- [思考](#思考)
- [MHA](#mha)
  - [结论](#结论)
  - [原因](#原因)
- [Orchestrator](#orchestrator)
  - [结论](#结论-1)
  - [原因](#原因-1)
  - [不切](#不切)
  - [切](#切)

## 思考
MHA和Orchestrator在遇到`Too Many Connections`错误时,是切还是不切？
相信很多人多这个问题一知半解,没有深入追究。

## MHA
### 结论
1. `Too many connection`不会自动Failover
2. `Too many connection`也不能手动Failover

### 原因
因为MHA内部定义了一些Error Code
```perl
our @ALIVE_ERROR_CODES = (
  1040,    # ER_CON_COUNT_ERROR                  -- too many connection
  1042,    # ER_BAD_HOST_ERROR                   -- Can't get hostname for your address
  1043,    # ER_HANDSHAKE_ERROR                  -- Bad handshake
  1044,    # ER_DBACCESS_DENIED_ERROR            -- Access denied for user '%s'@'%s' to database '%s'
  1045,    # ER_ACCESS_DENIED_ERROR              -- Access denied for user '%s'@'%s' (using password: %s)
  1129,    # ER_HOST_IS_BLOCKED                  -- Host '%s' is blocked because of many connection errors; unblock with 'mysqladmin flush-hosts'
  1130,    # ER_HOST_NOT_PRIVILEGED              -- Host '%s' is not allowed to connect to this MySQL server
  1203,    # ER_TOO_MANY_USER_CONNECTIONS        -- User %s already has more than 'max_user_connections' active connections
  1226,    # ER_USER_LIMIT_REACHED               -- User '%s' has exceeded the '%s' resource (current value: %ld)
  1251,    # ER_NOT_SUPPORTED_AUTH_MODE          -- Client does not support authentication protocol requested by server; consider upgrading MySQL client
  1275,    # ER_SERVER_IS_IN_SECURE_AUTH_MODE    -- Server is running in --secure-auth mode, but '%s'@'%s' has a password in the old format; please change the password to the new format
);
```

## Orchestrator
### 结论
可切可不切,这里很多人搞不明白。是不是很诡异？

### 原因
Orchestrator默认使用`长连接`来探测目标数据库集群,当然也可以修改为`短连接`进行探测
控制参数：MySQLConnectionLifetimeSeconds
- MySQLConnectionLifetimeSeconds = 0 表示orc探测目标数据库使用的是长连接
- MySQLConnectionLifetimeSeconds > 0 表示orc探测目标数据库使用的是短连接

### 不切
> 建议选择不切,如果线上代码异常,连接数打满,切了也是做无用功,同时会触发`anti-flapping`

默认参数：`MySQLConnectionLifetimeSeconds = 0`
即ORC默认使用长连接,因此ORC可以避免`Too Many Connections`触发Failover（除非你手动把主库的orc连接全部kill）

### 切
调整参数值,大于0即可,如：MySQLConnectionLifetimeSeconds = 10
这种情况下,ORC就可能会进行Failover