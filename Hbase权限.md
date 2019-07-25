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

public boolean authorize(User user, String namespace, Permission.Action action) {
  // Global authorizations supercede namespace level
  if (authorize(user, action)) {
    return true;
  }
  // Check namespace permissions
  PermissionCache<TablePermission> tablePerms = nsCache.get(namespace);
  if (tablePerms != null) {
    List<TablePermission> userPerms = tablePerms.getUser(user.getShortName());
    if (authorize(userPerms, namespace, action)) {
      return true;
    }
    String[] groupNames = user.getGroupNames();
    if (groupNames != null) {
      for (String group : groupNames) {
        List<TablePermission> groupPerms = tablePerms.getGroup(group);
        if (authorize(groupPerms, namespace, action)) {
          return true;
        }
      }
    }
  }
  return false;
}
public String getShortUserName() {
    Iterator i$ = this.subject.getPrincipals(User.class).iterator();
    if (i$.hasNext()) {
        User p = (User)i$.next();
        return p.getShortName();
    } else {
        return null;
    }
}
public String[] getGroupNames() {
  if (cache != null) {
    try {
      return this.cache.get(getShortName());
    } catch (ExecutionException e) {
      return new String[0];
    }
  }
  return ugi.getGroupNames();
}
```

- HMaster处理逻辑

  1. 校验权限

  2. 操作

  以**addColumn**为例，

  - preAddColumn 校验权限
  - postAddColumn 操作

- 用户获取过程

hbase-common/src/main/java/org/apache/hadoop/hbase/security/User.getCurrent()

```java
  public static User getCurrent() throws IOException {
    User user = new SecureHadoopUser();
    if (user.getUGI() == null) {
      return null;
    }
    return user;
  }
```

hadoop-common/org/apache/hadoop/security/UserGroupInformation.loginUserFromSubject()

```java
public static synchronized void loginUserFromSubject(Subject subject) throws IOException {
    ensureInitialized();

    try {
        if (subject == null) {
            subject = new Subject();
        }

        LoginContext login = newLoginContext(authenticationMethod.getLoginAppName(), subject, new UserGroupInformation.HadoopConfiguration());
        login.login();
        UserGroupInformation realUser = new UserGroupInformation(subject);
        realUser.setLogin(login);
        realUser.setAuthenticationMethod(authenticationMethod);
        realUser = new UserGroupInformation(login.getSubject());
        String proxyUser = System.getenv("HADOOP_PROXY_USER");
        if (proxyUser == null) {
            proxyUser = System.getProperty("HADOOP_PROXY_USER");
        }

        loginUser = proxyUser == null ? realUser : createProxyUser(proxyUser, realUser);
        String fileLocation = System.getenv("HADOOP_TOKEN_FILE_LOCATION");
        if (fileLocation != null) {
            Credentials cred = Credentials.readTokenStorageFile(new File(fileLocation), conf);
            loginUser.addCredentials(cred);
        }

        loginUser.spawnAutoRenewalThreadForUserCreds();
    } catch (LoginException var6) {
        LOG.debug("failure to login", var6);
        throw new IOException("failure to login", var6);
    }

    if (LOG.isDebugEnabled()) {
        LOG.debug("UGI loginUser:" + loginUser);
    }

}

public static UserGroupInformation createProxyUser(String user, UserGroupInformation realUser) {
    if (user != null && !user.isEmpty()) {
        if (realUser == null) {
            throw new IllegalArgumentException("Null real user");
        } else {
            Subject subject = new Subject();
            Set<Principal> principals = subject.getPrincipals();
            principals.add(new User(user));
            principals.add(new UserGroupInformation.RealUser(realUser));
            UserGroupInformation result = new UserGroupInformation(subject);
            result.setAuthenticationMethod(UserGroupInformation.AuthenticationMethod.PROXY);
            return result;
        }
    } else {
        throw new IllegalArgumentException("Null user");
    }
}
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

private void writeRequest(Call call, final int priority, Span span) throws IOException {
    RequestHeader.Builder builder = RequestHeader.newBuilder();
    builder.setCallId(call.id);
    if (span != null) {
      builder.setTraceInfo(
          RPCTInfo.newBuilder().setParentId(span.getSpanId()).setTraceId(span.getTraceId()));
    }
    builder.setMethodName(call.md.getName());
    builder.setRequestParam(call.param != null);
    ByteBuffer cellBlock = ipcUtil.buildCellBlock(this.codec, this.compressor, call.cells);
    if (cellBlock != null) {
      CellBlockMeta.Builder cellBlockBuilder = CellBlockMeta.newBuilder();
      cellBlockBuilder.setLength(cellBlock.limit());
      builder.setCellBlockMeta(cellBlockBuilder.build());
    }
    // Only pass priority if there is one set.
    if (priority != PayloadCarryingRpcController.PRIORITY_UNSET) {
      builder.setPriority(priority);
    }
    builder.setTimeout(call.timeout);
    RequestHeader header = builder.build();

    setupIOstreams();

    // Now we're going to write the call. We take the lock, then check that the connection
    //  is still valid, and, if so we do the write to the socket. If the write fails, we don't
    //  know where we stand, we have to close the connection.
    checkIsOpen();
    IOException writeException = null;
    synchronized (this.outLock) {
      if (Thread.interrupted()) throw new InterruptedIOException();

      calls.put(call.id, call); // We put first as we don't want the connection to become idle.
      checkIsOpen(); // Now we're checking that it didn't became idle in between.

      try {
        call.callStats.setRequestSizeBytes(IPCUtil.write(this.out, header, call.param,
            cellBlock));
      } catch (Throwable t) {
        if (LOG.isTraceEnabled()) {
          LOG.trace("Error while writing call, call_id:" + call.id, t);
        }
        // We set the value inside the synchronized block, this way the next in line
        //  won't even try to write. Otherwise we might miss a call in the calls map?
        shouldCloseConnection.set(true);
        writeException = IPCUtil.toIOE(t);
        interrupt();
      }
    }

    // call close outside of the synchronized (outLock) to prevent deadlock - HBASE-14474
    if (writeException != null) {
      markClosed(writeException);
      close();
    }

    // We added a call, and may be started the connection close. In both cases, we
    //  need to notify the reader.
    doNotify();

    // Now that we notified, we can rethrow the exception if any. Otherwise we're good.
    if (writeException != null) throw writeException;
  }

protected synchronized void setupIOstreams() throws IOException {
  if (socket != null) {
    // The connection is already available. Perfect.
    return;
  }

  if (shouldCloseConnection.get()){
    throw new ConnectionClosingException("This connection is closing");
  }

  if (failedServers.isFailedServer(remoteId.getAddress())) {
    if (LOG.isDebugEnabled()) {
      LOG.debug("Not trying to connect to " + server +
          " this server is in the failed servers list");
    }
    IOException e = new FailedServerException(
        "This server is in the failed servers list: " + server);
    markClosed(e);
    close();
    throw e;
  }

  try {
    if (LOG.isDebugEnabled()) {
      LOG.debug("Connecting to " + server);
    }
    short numRetries = 0;
    final short MAX_RETRIES = 5;
    Random rand = null;
    while (true) {
      setupConnection();
      InputStream inStream = NetUtils.getInputStream(socket);
      // This creates a socket with a write timeout. This timeout cannot be changed.
      OutputStream outStream = NetUtils.getOutputStream(socket, writeTO);
      // Write out the preamble -- MAGIC, version, and auth to use.
      writeConnectionHeaderPreamble(outStream);
      if (useSasl) {
        final InputStream in2 = inStream;
        final OutputStream out2 = outStream;
        UserGroupInformation ticket = remoteId.getTicket().getUGI();
        if (authMethod == AuthMethod.KERBEROS) {
          if (ticket != null && ticket.getRealUser() != null) {
            ticket = ticket.getRealUser();
          }
        }
        boolean continueSasl;
        if (ticket == null) throw new FatalConnectionException("ticket/user is null");
        try {
          continueSasl = ticket.doAs(new PrivilegedExceptionAction<Boolean>() {
            @Override
            public Boolean run() throws IOException {
              return setupSaslConnection(in2, out2);
            }
          });
        } catch (Exception ex) {
          ExceptionUtil.rethrowIfInterrupt(ex);
          if (rand == null) {
            rand = new Random();
          }
          handleSaslConnectionFailure(numRetries++, MAX_RETRIES, ex, rand, ticket);
          continue;
        }
        if (continueSasl) {
          // Sasl connect is successful. Let's set up Sasl i/o streams.
          inStream = saslRpcClient.getInputStream(inStream);
          outStream = saslRpcClient.getOutputStream(outStream);
        } else {
          // fall back to simple auth because server told us so.
          authMethod = AuthMethod.SIMPLE;
          useSasl = false;
        }
      }
      this.in = new DataInputStream(new BufferedInputStream(inStream));
      synchronized (this.outLock) {
        this.out = new DataOutputStream(new BufferedOutputStream(outStream));
      }
      // Now write out the connection header
      writeConnectionHeader();

      // start the receiver thread after the socket connection has been set up
      start();
      return;
    }
  } catch (Throwable t) {
    IOException e = ExceptionUtil.asInterrupt(t);
    if (e == null) {
      failedServers.addToFailedServers(remoteId.address);
      if (t instanceof LinkageError) {
        // probably the hbase hadoop version does not match the running hadoop version
        e = new DoNotRetryIOException(t);
      } else if (t instanceof IOException) {
        e = (IOException) t;
      } else {
        e = new IOException("Could not set up IO Streams to " + server, t);
      }
    }
    markClosed(e);
    close();
    throw e;
  }
}
```

