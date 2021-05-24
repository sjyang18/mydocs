# TLS support in datawave


## Software Patches

* Accumulo 2.0.1 with TLS patch: https://github.com/sjyang18/accumulo/tree/2.0.1-w-patch 

* Datawave patch with TLS support: https://github.com/sjyang18/datawave/tree/accumulo_tls_support


* Datawave-muchos patch for Accumulo 2.0.1: https://github.com/sjyang18/datawave-muchos/tree/support_accumulo_2.x


## Software Patch test/validation configuratiobn
* When you build your accumulo cluster with fluo-muchos, make sure to build Acccumulo 2.0.1 with the patch above and include (i.e under conf/upload) & configure muchos.properties. 
* Log in bastion host, clone datawave-muchos with the patch above, and go thru its setup instructions.

* Update dw_project_version, dw_repo, and dw_checkout_version to ansible/group_vars/datawave. 

* Log in to datawave ingesting node and add `ADDITIONAL_INGEST_ENV`. For example, in my cluster, I added.


```
export ADDITIONAL_INGEST_ENV=/home/azureuser/extra-ingest-env.sh
```
* In the same ingesting node, create extra-ingest-env.sh with the following instructions.
```
#!/bin/bash

ADDITIONAL_INGEST_LIBS="${ADDITIONAL_INGEST_LIBS}"

add_jar_prefix_to_ingest_libs() {
  for JAR in "$1"*jar; do
    ADDITIONAL_INGEST_LIBS="${ADDITIONAL_INGEST_LIBS}:${JAR}"
  done
}

add_jar_prefix_to_ingest_libs "${ZOOKEEPER_HOME}/lib/netty-"

export ACCUMULO_CLIENT_PROPS=/opt/muchos/install/accumulo-2.0.1/conf/accumulo-client.properties

CLIENT_JVMFLAGS="-Dzookeeper.clientCnxnSocket=org.apache.zookeeper.ClientCnxnSocketNetty \
  -Dzookeeper.client.secure=true \
  -Dzookeeper.ssl.keyStore.location=/opt/muchos/install/ssl/host-keystore.jks \
  -Dzookeeper.ssl.keyStore.password=hadoop \
  -Dzookeeper.ssl.trustStore.location=/opt/muchos/install/ssl/truststore.jks \
  -Dzookeeper.ssl.trustStore.password=hadoop"

CHILD_INGEST_OPTS="${CHILD_INGEST_OPTS} ${CLIENT_JVMFLAGS}"
HADOOP_INGEST_OPTS="${HADOOP_INGEST_OPTS} ${CLIENT_JVMFLAGS}"
MAPRED_INGEST_OPTS="${MAPRED_INGEST_OPTS} ${CLIENT_JVMFLAGS}"

``` 