- [为什么需要VIP](#为什么需要vip)
- [MHA实现方式](#mha实现方式)
- [一些痛点](#一些痛点)
- [思考](#思考)
- [方案1.使用consul](#方案1使用consul)
    - [ORC注册时机](#orc注册时机)
    - [编辑配置文件](#编辑配置文件)
    - [获取指定集群的Master](#获取指定集群的master)
- [方案2.智能DNS](#方案2智能dns)
  - [实现逻辑](#实现逻辑)
    - [Etcd存储格式](#etcd存储格式)
    - [写脚本（举例）](#写脚本举例)
    - [执行Hook](#执行hook)
    - [更新DNS解析](#更新dns解析)
    - [验证方法](#验证方法)
      - [查看当前的vip域名解析](#查看当前的vip域名解析)
      - [执行切换](#执行切换)
      - [查看切换日志](#查看切换日志)
      - [验证域名解析是否进行了变更](#验证域名解析是否进行了变更)
- [方案3.继续使用vip切换脚本](#方案3继续使用vip切换脚本)

## 为什么需要VIP
数据库为什么需要vip？主要是解决如下场景
- 数据同步场景,如Canal/TiDB DM等支持GTID,可自动恢复的工具
- 一些想直连主库的业务场景

## MHA实现方式
MHA通过调用VIP切换脚本,将VIP绑定到新主库上

## 一些痛点
1. 比如一些云服务,数据库如果采用跨可用区部署,vip可能不支持在跨可用区ECS上进行绑定。如果候选主库和宕机主库不在一个可用区,就会非常麻烦。
2. VIP脚本需要使用到SSH,需要免密访问MySQL实例,安全层面不合规,存在一定的风险（如果公司安全规范没要求,当我没说）
3. VIP切换可能存在飘不过去的情况

## 思考
既然我们采用了orchestrator来构建拓扑,那么有没有更好一些的解决方案呢？

## 方案1.使用consul
ORC支持`consul`注册,可以支持将集群的`master`节点的ip推送到Consul中,可以支持服务发现,这需要下游组件支持。

#### ORC注册时机
1. ORC会定期运行检查,当遇到一个新的集群,或者外部存储没有现有KV条目的master节点时,orc会注入到外部KV系统（只注入一次）
2. 发生了故障转移,orc会覆盖master的值
3. 手动填充
    - orchestrator-client -c submit-masters-to-kv-stores to submit all clusters' masters to KV, or
    - orchestrator-client -c submit-masters-to-kv-stores -alias mycluster to submit the master of mycluster to KV

#### 编辑配置文件
配置文件: `orchestrator.conf.json`
增加下面参数,ConsulAddress改为真实的地址,建议为consul集群
```json
"KVClusterMasterPrefix": "mysql/master",
"ConsulAddress": "10.10.1.220:8500",
"ConsulCrossDataCenterDistribution": true
```

#### 获取指定集群的Master
```bash
[root@orc-220 ~]# curl -Ss http://10.10.1.220:8500/v1/kv/mysql/master/mha_rd_tt
[
    {
        "LockIndex": 0,
        "Key": "mysql/master/mha_rd_tt",
        "Flags": 0,
        "Value": "MTAuMTAuMS4yMjE6MzMwNg==",
        "CreateIndex": 57,
        "ModifyIndex": 185
    }
]
[root@orc-220 ~]# curl -Ss http://10.10.1.220:8500/v1/kv/mysql/master/mha_rd_tt/hostname |jq -r '.[].Value' |base64 -d
10.10.1.221
```

## 方案2.智能DNS
> 这里的dns架构为Coredns + Etcd,不使用DNS缓存,实时更新etcd值实现DNS解析实时生效

抛转引玉,您也可以有更好的类似实现,方案2注重一些思考分享。

### 实现逻辑
由于采用的是coredns+etcd实现且没有dns缓存,那么我们只需要修改etcd对应的key值即可实现域名实时更新。

#### Etcd存储格式
假设域名为: vip_prod_tt.mysql.local

在etcd中存储为:
key: /dns/local/mysql/vip_prod_tt/10_10_1_220
value: {"host": "10.10.1.220"}


#### 写脚本（举例）
脚本名: orc_change_vip
传参:  
   - **-vip_domain** vip_prod_tt.mysql.local
   - **-new_master_ip** 新主库的IP地址
   - **-endpoints** etcd集群地址


#### 执行Hook
ORC在发生切换成功后,会执行hook,执行PostFailoverProcesses钩子,并传递变量successorHost和failureClusterDomain
我们在PostFailoverProcesses里面是可以定义脚本
```json
"DetectClusterDomainQuery": "select cluster_domain from orc_meta.cluster where hostname=@@hostname",
"PostFailoverProcesses": [
    "/usr/local/orchestrator/orc_change_vip -vip_domain {failureClusterDomain} -new_master_ip {successorHost} -endpoints 10.10.1.220:2379",
    "echo '(for all types) Recovered -- {failureClusterDomain} -- from {failureType} on {failureCluster}. Failed: {failedHost}:{failedPort}; Successor: {successorHost}:{successorPort}' >> /tmp/recovery.log"
],
```

#### 更新DNS解析
当orc执行脚本`orc_change_vip`后,该脚本会修改`etcd`对应域名的解析为新`Master`的`IP`地址。
此时客户端解析域名获取到的解析会变为新的`Master`,客户端只需要访问域名,不需要关注谁是Master。

#### 验证方法
> 下面仅限于使用coredns+etcd的场景

##### 查看当前的vip域名解析
```bash
[root@orc-220 ~]# ETCDCTL_API=3 etcdctl get --prefix "/dns/local/mysql//vip_prod_tt" --endpoints "http://10.10.1.220:2379"
/dns/local/mysql/vip_prod_tt/10.10.1.220
{"host":"10.10.1.220"}
```

##### 执行切换
提升10.10.1.221:3306为新主
```bash

[root@orc-220 ~]# orchestrator-client -c graceful-master-takeover-auto -alias mha_rd_tt -d 10.10.1.221:3306 -b "admin:1234.com"
10.10.1.221:3306
```

##### 查看切换日志
```bash
[root@orc-220 ~]# grep PostFailoverProcesses /tmp/orchestrator.log
2022-06-07 16:44:22 INFO topology_recovery: Running 2 PostFailoverProcesses hooks
2022-06-07 16:44:22 INFO topology_recovery: Running PostFailoverProcesses hook 1 of 2: /usr/local/orchestrator/orcChangeVip -vip_domain vip_prod_tt.mysql.local -new_master_ip 10.10.1.221 -endpoints 10.10.1.220:2379
2022-06-07 16:44:23 INFO topology_recovery: Completed PostFailoverProcesses hook 1 of 2 in 13.092554ms
2022-06-07 16:44:23 INFO topology_recovery: Running PostFailoverProcesses hook 2 of 2: echo '(for all types) Recovered -- vip_prod_tt..mysql.local -- from DeadMaster on 10.10.1.220:3306. Failed: 10.10.1.220:3306; Successor: 10.10.1.221:3306' >> /tmp/recovery.log
2022-06-07 16:44:23 INFO topology_recovery: Completed PostFailoverProcesses hook 2 of 2 in 1.356151ms
2022-06-07 16:44:23 INFO topology_recovery: done running PostFailoverProcesses hooks
```

##### 验证域名解析是否进行了变更
```bash
[root@orc-220 ~]# ETCDCTL_API=3 etcdctl get --prefix "/dns/local/mysql/vip_prod_tt" --endpoints "http://10.10.1.220:2379"
/dns/local/mysql/rd/proxysql/vip_prod_tt/10.10.1.221
{"host":"10.10.1.221"}
```

## 方案3.继续使用vip切换脚本
和MHA一样,编写orc版的vip切换脚本,然后在hook里面调用即可