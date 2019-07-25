- 从github下载hbase代码到本地，如提示版本相关报错，修改/hbase/conf/hbase-site.xml增加如下内容并重新编译

```
<configuration>
    <property>
        <name>hbase.defaults.for.version.skip</name>
        <value>true</value>
    </property>
</configuration>
```

- 编译项目 mvn clean package -DskipTests assembly:single

- 增加两个Run/Debug Application 配置如下

```
main class: org.apache.hadoop.hbase.master.HMaster
VM options: -Dlog4j.configuration=file:/……/hbase/conf/log4j.properties
program arguments: start
working directory: /……/hbase
use class path of model: hbase-server

main class: org.jruby.Main
VM options: -Dhbase.ruby.sources=/……/hbase/hbase-shell/src/main/ruby -Dlog4j.configuration=file:/……/hbase/conf/log4j.properties
program arguments: /……/hbase/bin/hirb.rb
working directory: /……/hbase
use class path of model: hbase-shell
```

- 可将/hbase/conf/hbase-site.xml拷贝至hbase-server/target/classes/hbase-site.xml 则以main 方法启动时可生效
- 启动HMaster 访问[http://localhost:16010](http://localhost:16010/)可查看页面
- 启动hbase-shell 可在命令行执行相关操作

 