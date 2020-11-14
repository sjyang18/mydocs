# Standalone Accumulo integrated with Azure Active Directory (AAD) and Domain Service Inegration, without Azure HDInsight Dependency

## Assumption
* You have already setup AAD and AAD Domain service, and tools
* You have a HDFS available in your Azure VNET and available in your subnet. 
* You have setup a Azure VM (testvm1 below) in the same VNET and subnet.


## Hadoop Client and Zookeeper Client Dependency
Even though you are not running your own HDFS and Zookeeper service for your Accumulo, you still need to set up client libraries for Accumulo.
Download from apache.org and unzip.

```
wget https://downloads.apache.org/hadoop/common/hadoop-3.3.0/hadoop-3.3.0.tar.gz

wget https://archive.apache.org/dist/zookeeper/zookeeper-3.6.1/apache-zookeeper-3.6.1-bin.tar.gz

tar xvf hadoop-3.3.0.tar.gz
tar xvf apache-zookeeper-3.6.1-bin.tar.gz
```
Set your HADOOP_HOME variable to your unzip folder. Copy the core-site.xml and hdfs-site.xml from your HDFS service into `${HADOOP_HOME}/etc/hadoop` and update $`{HADOOP_HOME}/etc/hadoop/hadoop-env.sh`. My includes the followings:

```
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
export HADOOP_OPTIONAL_TOOLS="hadoop-azure"
export HADOOP_CLASSPATH="${HADOOP_CLASSPATH}:${HADOOP_HOME}/share/hadoop/tools/lib/*"
# get path to libssl by 'whereis libssl'
export HADOOP_OPTS="-Dorg.wildfly.openssl.path=/usr/lib/x86_64-linux-gnu/libssl3.so ${HADOOP_OPTS}"
```

In my experiement, I export "hadoop-azure" in $HADOOP_OPTIONAL_TOOLS variable because I reused the HDFS created thru HDInsight in the previous experiment and its HADOOP client configuration includes wasb (https://hadoop.apache.org/docs/current/hadoop-azure/index.html).

I also updated `fs.defaultFS` in ${HADOOP_HOME}/etc/hadoop/core-site.xml so that my experment will use the external HDFS in Accumulo configuration. Update the value (e.g. `hdfs://mycluster` in this case) based on `dfs.internal.nameservices` value of your ${HADOOP_HOME}/etc/hadoop/hdfs-site.xml. 

```
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://mycluster</value>
        <final>true</final>
    </property>
```

## Accumulo Installation and accumulo-env.sh
Download Accumulo and upzip.
```
wget "https://apache.osuosl.org/accumulo/2.0.0/accumulo-2.0.0-bin.tar.gz"
tar xvf accumulo-2.0.0-bin.tar.gz
```

Update ${ACCUMULO_HOME}/conf/accumulo-env.sh with the correct HADOOP_HOME, ZOOKEEPER_HOME. Set CLASSPATH to include hadoop client,its tools lib, and zookeeper lib directories. 
```
CLASSPATH="${CLASSPATH}:${lib}/*:${HADOOP_CONF_DIR}:${ZOOKEEPER_HOME}/*:${HADOOP_HOME}/share/hadoop/client/*"
CLASSPATH="${CLASSPATH}:${HADOOP_HOME}/share/hadoop/tools/lib/*:${HADOOP_HOME}/share/hadoop/hdfs/*:${HADOOP_HOME}/share/hadoop/hdfs/lib/*"
CLASSPATH="${CLASSPATH}:${ZOOKEEPER_HOME}/lib/*"
export CLASSPATH
```

Include `Dorg.wildfly.openssl.path` in JAVA_OPTS.

```
JAVA_OPTS=("${ACCUMULO_JAVA_OPTS[@]}"
  '-XX:+UseConcMarkSweepGC'
  '-XX:CMSInitiatingOccupancyFraction=75'
  '-XX:+CMSClassUnloadingEnabled'
  '-XX:OnOutOfMemoryError=kill -9 %p'
  '-XX:-OmitStackTraceInFastThrow'
  '-Djava.net.preferIPv4Stack=true'
  '-Dorg.wildfly.openssl.path=/usr/lib/x86_64-linux-gnu/libssl3.so'
  "-Daccumulo.native.lib.path=${lib}/native")
```





