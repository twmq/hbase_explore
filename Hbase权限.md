表结构相关操作入口

```java
hbase-client/src/main/java/org/apache/hadoop/hbase/client/ConnectionManager.java
hbase-server/src/main/java/org/apache/hadoop/hbase/master/MasterRpcServices.java
hbase-server/src/main/java/org/apache/hadoop/hbase/master/HMaster.java
```

- 校验权限过程

  - 获取请求用户（为空时用本地用户校验）

  - 校验用户权限

  - 校验对应用户组对应权限

    对应类如下（AccessController.java包含表结构和表数据操作校验）

```java
hbase-server/src/main/java/org/apache/hadoop/hbase/security/access/AccessController.java
hbase-server/src/main/java/org/apache/hadoop/hbase/security/access/AccessChecker.java
hbase-server/src/main/java/org/apache/hadoop/hbase/security/access/TableAuthManager.java
```

- HMaster处理逻辑

  1. 校验权限

  2. 操作

  以**addColumn**为例，

  - preAddColumn 校验权限
  - postAddColumn 操作

- 用户获取过程

```
hbase-common/src/main/java/org/apache/hadoop/hbase/security/User.SecureHadoopUser()
hadoop-common/2.5.1/hadoop-common-2.5.1.jar!/org/apache/hadoop/security/UserGroupInformation.class getCurrentUser()
获取 System.getenv("HADOOP_USER_NAME")
其次 System.getProperty("HADOOP_USER_NAME")
其次 系统用户
```

- 用户使用过程

```java
在 ConnectionFactory.createConnection() 进行获取并设置为Connection的成员变量
user 的传递过程 connection -> Table,HBaseAdmin ->RpcClient.call方法 -> RpcClient.getConnection

protected Connection getConnection(User ticket, Call call, InetSocketAddress addr)
  throws IOException {
  if (!running.get()) throw new StoppedRpcClientException();
  Connection connection;
  ConnectionId remoteId =
  new ConnectionId(ticket, call.md.getService().getName(), addr);
  synchronized (connections) {
  connection = connections.get(remoteId);
  if (connection == null) {
  connection = createConnection(remoteId, this.codec, this.compressor);
  connections.put(remoteId, connection);
  }
  }

  return connection;
}
```

