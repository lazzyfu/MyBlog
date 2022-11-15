- [参数介绍](#参数介绍)
- [用途](#用途)
	- [1.报告有问题的实例](#1报告有问题的实例)
		- [调用流程](#调用流程)
		- [HttpAPI引用](#httpapi引用)
		- [结论](#结论)
	- [2.检查复制问题](#2检查复制问题)
		- [调用流程](#调用流程-1)
		- [然后赋值给变量`CountLaggingReplicas`](#然后赋值给变量countlaggingreplicas)
		- [`CountLaggingReplicas`行为](#countlaggingreplicas行为)
		- [看下`UnreachableMasterWithLaggingReplicas`的官方定义](#看下unreachablemasterwithlaggingreplicas的官方定义)
		- [`UnreachableMasterWithLaggingReplicas`会触发什么紧急操作](#unreachablemasterwithlaggingreplicas会触发什么紧急操作)
		- [`emergentlyRestartReplicationOnTopologyInstanceReplicas`做了什么](#emergentlyrestartreplicationontopologyinstancereplicas做了什么)
		- [结论](#结论-1)
	- [3.观测SQL线程是否应用完中继日志](#3观测sql线程是否应用完中继日志)
		- [分析代码](#分析代码)
		- [结论](#结论-2)
	- [4.作为变量`ReasonableLockedSemiSyncMasterSeconds`的兜底](#4作为变量reasonablelockedsemisyncmasterseconds的兜底)

## 参数介绍
参数: `ReasonableReplicationLagSeconds`
释义: 合理的复制延迟秒数

该参数到底是干嘛使用的, 文档解释有些含糊, 也没太多的解释, 我们不妨看下代码, 深入分析下。


## 用途
### 1.报告有问题的实例
#### 调用流程
函数:`ReadProblemInstances`
下面SQL用于获取延时大于`ReasonableReplicationLagSeconds`的实例。
```sql
...
or (abs(cast(seconds_behind_master as signed) - cast(sql_delay as signed)) > ?)
...
```

#### HttpAPI引用
引用方法`Problems`
```go
// Problems provides list of instances with known problems
func (this *HttpAPI) Problems(params martini.Params, r render.Render, req *http.Request) {
	clusterName := params["clusterName"]
	instances, err := inst.ReadProblemInstances(clusterName)

	if err != nil {
		Respond(r, &APIResponse{Code: ERROR, Message: fmt.Sprintf("%+v", err)})
		return
	}

	r.JSON(http.StatusOK, instances)
}
```

#### 结论
被接口`Problems`调用, 该接口用于获取有问题的复制, 一般用于前端页面显示复制延时, 比如:哪些实例有延时。

### 2.检查复制问题
#### 调用流程
函数: `GetReplicationAnalysis`
```go
// 整合参数
args := sqlutils.Args(config.Config.ReasonableLockedSemiSyncMasterSeconds, ValidSecondsFromSeenToLastAttemptedCheck(), config.Config.ReasonableReplicationLagSeconds, clusterName)

// 查询的SQL语句
...
IFNULL(
    SUM(replica_instance.slave_lag_seconds > ?),
    0
) AS count_lagging_replicas,
...
```

#### 然后赋值给变量`CountLaggingReplicas`
```go
a.CountLaggingReplicas = m.GetUint("count_lagging_replicas")
```

#### `CountLaggingReplicas`行为
> 下面我们看下该变量`CountLaggingReplicas`做什么操作（精简了）
```go
if !a.IsReplicationGroupMember /* Traditional Async/Semi-sync replication issue detection */ {
	...
	else if a.IsMaster && !a.LastCheckValid && a.CountLaggingReplicas == a.CountReplicas && a.CountDelayedReplicas < a.CountReplicas && a.CountValidReplicatingReplicas > 0 {
		a.Analysis = UnreachableMasterWithLaggingReplicas
		a.Description = "Master cannot be reached by orchestrator and all of its replicas are lagging"
		//
	}
	...
	// 我们没有中间主, 不考虑
	else if !a.IsMaster && !a.LastCheckValid && a.CountLaggingReplicas == a.CountReplicas && a.CountDelayedReplicas < a.CountReplicas && a.CountValidReplicatingReplicas > 0 {
		a.Analysis = UnreachableIntermediateMasterWithLaggingReplicas
		a.Description = "Intermediate master cannot be reached by orchestrator and all of its replicas are lagging"
		//
	}
}
```
返回的`analysisCode`为`UnreachableMasterWithLaggingReplicas`

#### 看下`UnreachableMasterWithLaggingReplicas`的官方定义
> [https://github.com/openark/orchestrator/blob/master/docs/failure-detection.md#unreachablemasterwithlaggingreplicas](https://github.com/openark/orchestrator/blob/master/docs/failure-detection.md#unreachablemasterwithlaggingreplicas)

**UnreachableMasterWithLaggingReplicas**
* 主库无法访问
* 所有直属副本（除了延时从库）都是滞后的

当主服务器过载时, 可能会发生这种情况。客户端会收到"Too many connections"错误, 而很久以前就连接的副本却会声称主库没有问题(因为io_thread是长连接)。同样, 如果主服务器由于某些元数据操作而被锁定, 客户端将在连接上被阻止, 而副本可能会声称一切正常。 但是, 由于应用程序无法连接到主库, 因此不会写入任何实际数据, 并且当使用诸如`pt-heartbeat`之类的心跳机制时, 我们可以观察到副本的延迟越来越大。
ochestrator通过在所有master的直属副本上重新启动复制来响应这种情况。这将关闭这些副本上的旧客户端连接并尝试启动新的连接, 如果此时无法建立连接（IO Thread）, 将会导致所有副本复制失败, 从而导致orchestrator认为Master节点状态为`DeadMaster`。

#### `UnreachableMasterWithLaggingReplicas`会触发什么紧急操作
分析`runEmergentOperations`代码
```go
switch analysisEntry.Analysis {
...
case inst.UnreachableMasterWithLaggingReplicas:
	go emergentlyRestartReplicationOnTopologyInstanceReplicas(&analysisEntry.AnalyzedInstanceKey, analysisEntry.Analysis)
...
}
```
从代码可以看到执行了紧急操作`emergentlyRestartReplicationOnTopologyInstanceReplicas`

#### `emergentlyRestartReplicationOnTopologyInstanceReplicas`做了什么
orchestrator会在所有副本上强制执行`stop slave` + `start slave`, 试图重新评估其复制状态。当副本执行stop和start时, 副本和主节点之间会重新进行身份认证, 判断主服务器是否有问题。

#### 结论
当主库不可达时, 同时所有的副本延时大于`ReasonableReplicationLagSeconds`时, 会触发orchestrator对当前拓扑所有的副本进行主动探测, 表现为强制执行`stop slave` + `start slave`, 以此来评估副本是否可以访问主节点。如果所有副本无法访问主节点, 则判断当前主库为`DeadMaster`。

### 3.观测SQL线程是否应用完中继日志
#### 分析代码
请看下面代码注释即可, 逻辑有些绕圈圈。
```go
func WaitForSQLThreadUpToDate(instanceKey *InstanceKey, overallTimeout time.Duration, staleCoordinatesTimeout time.Duration) (instance *Instance, err error) {
	// Otherwise we don't bother.
	// lastExecBinlogCoordinates最新执行的binlog坐标
	var lastExecBinlogCoordinates BinlogCoordinates

	// 总超时
	if overallTimeout == 0 {
		overallTimeout = 24 * time.Hour
	}
	// 过时的坐标超时
	if staleCoordinatesTimeout == 0 {
		staleCoordinatesTimeout = time.Duration(config.Config.ReasonableReplicationLagSeconds) * time.Second
	}
	// 通用定时器,24小时后发送当前时间到chan
	generalTimer := time.NewTimer(overallTimeout)
	// 过时的定时器,10秒后发送当前时间到chan
	staleTimer := time.NewTimer(staleCoordinatesTimeout)

	for {
		instance, err := RetryInstanceFunction(func() (*Instance, error) {
			return ReadTopologyInstance(instanceKey)
		})
		if err != nil {
			return instance, log.Errore(err)
		}
		// 当消费完所有的中继日志后, 返回true
		// 判断方法: 读取到的binlog坐标【Read_Master_Log_Pos】等于执行到的binlog坐标【Exec_Master_Log_Pos】
		if instance.SQLThreadUpToDate() {
			// Woohoo
			return instance, nil
		}
		// 延时从库
		if instance.SQLDelay != 0 {
			return instance, log.Errorf("WaitForSQLThreadUpToDate: instance %+v has SQL Delay %+v. Operation is irrelevant", *instanceKey, instance.SQLDelay)
		}
		// 假设Read_Master_Log_Pos = 500

		// loop 1
		// ExecBinlogCoordinates = 100
		// lastExecBinlogCoordinates = null

		// loop 2
		// ExecBinlogCoordinates = 200
		// lastExecBinlogCoordinates = 100

		// 第一种情况:loop 3
		// ExecBinlogCoordinates = 500
		// lastExecBinlogCoordinates = 200
		// 此时SQLThreadUpToDate判断Read_Master_Log_Pos == ExecBinlogCoordinates,  返回了

		// 第二种情况:loop 3
		// ExecBinlogCoordinates = 200
		// lastExecBinlogCoordinates = 200
		// 此时instance.ExecBinlogCoordinates.Equals(&lastExecBinlogCoordinates), 继续休眠500ms, 不会重置定时器staleCoordinatesTimeout, 
		// 继续循环, 如果接下来的循环, ExecBinlogCoordinates仍然等于lastExecBinlogCoordinates, 则会触发case <-staleTimer.C, 导致超时退出。
		// 如从库有锁, 比如全局锁定, 大事务等情况, 执行时间超过了ReasonableReplicationLagSeconds, 就会退出
		if !instance.ExecBinlogCoordinates.Equals(&lastExecBinlogCoordinates) {
			// 只要进入, 表示中继日志在继续应用, 重置staleCoordinatesTimeout, 下面的select就不会执行case <-staleTimer.C
			fmt.Println("------------reset staleTimer---------------")
			// means we managed to apply binlog events. We made progress...
			// so we reset the "staleness" timer
			if !staleTimer.Stop() {
				<-staleTimer.C
			}
			staleTimer.Reset(staleCoordinatesTimeout)
		}
		lastExecBinlogCoordinates = instance.ExecBinlogCoordinates

		select {
		case <-generalTimer.C:
			// 超时退出, 等待时间超过24小时
			return instance, log.Errorf("WaitForSQLThreadUpToDate timeout on %+v after duration %+v", *instanceKey, overallTimeout)
		case <-staleTimer.C:
			// 超过退出, 等待时间超过ReasonableReplicationLagSeconds
			return instance, log.Errorf("WaitForSQLThreadUpToDate stale coordinates timeout on %+v after duration %+v", *instanceKey, staleCoordinatesTimeout)
		default:
			log.Debugf("WaitForSQLThreadUpToDate waiting on %+v", *instanceKey)
			// 休眠500ms, 继续循环
			time.Sleep(retryInterval)
		}
	}
}
```

#### 结论
当参数`DelayMasterPromotionIfSQLThreadNotUpToDate`为`true`时, 表示有延时需要等待应用完`relay-log`才进行切换。
此时参数`ReasonableReplicationLagSeconds`也控制等待应用完中继日志的超时。因此切换期间, 不应该有DDL操作, 大事务等, 否则从库会阻塞, 导致`ExecBinlogCoordinates`等于`lastExecBinlogCoordinates`, 最终触发`ReasonableReplicationLagSeconds`超时, 切换失败。

该参数建议调大些, 比如:120s

### 4.作为变量`ReasonableLockedSemiSyncMasterSeconds`的兜底
如果变量`ReasonableLockedSemiSyncMasterSeconds`没有指定, 就使用`ReasonableReplicationLagSeconds`