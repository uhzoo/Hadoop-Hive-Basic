# Installing Hadoop and Hive
Step-by-step guide for installing a basic Hadoop and Hive.

| Service  | Version |
| ------------- | ------------- |
|OS|Rocky Linux 9.4 Minimal, 4 vCPUs, 8 GB RAM|
|Hadoop|3.4.1|
|Hive|4.0.1|
|Hive MetaStore|Postgresql 16|

## Install Hadoop
Before installing Hadoop, disable the firewall for convenience.
```bash
systemctl disable firewalld --now;
```
Hadoop requires Java (JDK 8), so you need to install it first.
```bash
dnf install java-1.8.0-*;
```
To download and extract the installation file, install wget and tar.
```bash
dnf install wget tar;
```
Configure `/etc/hosts` to allow Hadoop access using the fully qualified domain name (FQDN).
```bash
hostnamectl set-hostname test.hadoop.com;
echo $(hostname -I | awk '{print $1}') $(hostname -f) >> /etc/hosts;
```
or
```bash
echo ${YOUR_IP} $(hostname -f) >> /etc/hosts;
```
Add a user named hadoop and set its password (in this example, the password is also hadoop).
```bash
useradd hadoop;
passwd hadoop;
```
Download Hadoop, extract it to `/opt`, and change the ownership to the hadoop user.
```bash
wget https://dlcdn.apache.org/hadoop/common/hadoop-3.4.1/hadoop-3.4.1.tar.gz;
tar -zxf hadoop-3.4.1.tar.gz -C /opt;
chown -R hadoop. /opt/hadoop-3.4.1;
```
We will store all Hadoop-related data under /data, so let's create the directory and give ownership to the hadoop user.
```bash
mkdir -p /data;
chown -R hadoop. /data;
```
Now login to hadoop.
```bash
su - hadoop;
```
In order to use useful commands like `start-all.sh`, passwordless SSH access is required.
Generate an SSH key and copy it to the authorized hosts.
When prompted during `ssh-keygen`, simply press Enter to accept the defaults.
```bash
ssh-keygen -t rsa -m PEM;
ssh-copy-id -i /home/hadoop/.ssh/id_rsa.pub $(hostname -f);
```
Now configure the `HADOOP_HOME` and `JAVA_HOME` environment variables in `/opt/hadoop-3.4.1/etc/hadoop/hadoop-env.sh`.
You can determine your `JAVA_HOME` path with the following command: `readlink -f /usr/bin/java`
```bash
vi /opt/hadoop-3.4.1/etc/hadoop/hadoop-env.sh;
```
```text
# export JAVA_HOME=
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.462.b08-3.el9.x86_64/jre
# export HADOOP_HOME=
export HADOOP_HOME=/opt/hadoop-3.4.1
```
Next, configure the following Hadoop XML files to set up the core components.
Use vi (or your preferred editor) to modify each file as shown below.
1. core-site.xml
```bash
vi /opt/hadoop-3.4.1/etc/hadoop/core-site.xml;
```
```xml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://test.hadoop.com:9000</value>
        <description>The default filesystem URI. All HDFS paths will be resolved relative to this.</description>
    </property>
</configuration>
```
2. hdfs-site.xml
```bash
vi /opt/hadoop-3.4.1/etc/hadoop/hdfs-site.xml;
```
```xml
<configuration>
    <property>
        <name>dfs.namenode.http-address</name>
        <value>test.hadoop.com:9870</value>
    </property>
    <property>
        <name>dfs.secondary.http.address</name>
        <value>test.hadoop.com:9868</value>
    </property>
    <property>
        <name>dfs.datanode.http.address</name>
        <value>test.hadoop.com:9864</value>
    </property>
    <property>
        <name>dfs.namenode.name.dir</name>
        <value>file:/data/namenode</value>
    </property>
    <property>
        <name>dfs.datanode.data.dir</name>
        <value>file:/data/datanode</value>
    </property>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
        <description>The default number of data block replications. Set to 1 for single-node or test environments.</description>
    </property>
</configuration>
```
3. yarn-site.xml
```bash
vi /opt/hadoop-3.4.1/etc/hadoop/yarn-site.xml;
```
```xml
<configuration>
    <property>
        <name>yarn.resourcemanager.webapp.address</name>
        <value>test.hadoop.com:8088</value>
    </property>
    <property>
        <name>yarn.nodemanager.webapp.address</name>
        <value>test.hadoop.com:8042</value>
    </property>
    <property>
        <name>yarn.nodemanager.local-dirs</name>
        <value>/data/yarn/local</value>
    </property>
    <property>
        <name>yarn.nodemanager.log-dirs</name>
        <value>/data/yarn/logs</value>
    </property>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
        <description>Enables shuffle service for MapReduce jobs in the NodeManager.</description>
    </property>
    <property>
        <name>yarn.nodemanager.env-whitelist</name>
        <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_HOME,PATH,LANG,TZ,HADOOP_MAPRED_HOME</value>
        <description>Specifies which environment variables are passed from NodeManager to containers.</description>
    </property>
</configuration>
```
4. mapred-site.xml
```bash
vi /opt/hadoop-3.4.1/etc/hadoop/mapred-site.xml;
```
```xml
<configuration>
    <property>
        <name>mapreduce.jobhistory.address</name>
        <value>test.hadoop.com:10020</value>
    </property>
    <property>
        <name>mapreduce.jobhistory.webapp.address</name>
        <value>test.hadoop.com:19888</value>
    </property>
    <property>
        <name>mapreduce.jobhistory.done-dir</name>
        <value>/mr-history/done</value>
    </property>
    <property>
        <name>mapreduce.jobhistory.intermediate-done-dir</name>
        <value>/mr-history/tmp</value>
    </property>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
        <description>Specifies YARN as the execution framework for MapReduce jobs.</description>
    </property>
    <property>
        <name>mapreduce.application.classpath</name>
        <value>$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/*:$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/lib/*</value>
        <description>Defines the classpath for MapReduce applications to find required libraries.</description>
    </property>
</configuration>
```
Now that the basic configuration is complete, format the NameNode to initialize the HDFS filesystem.
```bash
/opt/hadoop-3.4.1/bin/hdfs namenode -format;
```
Now let’s start all Hadoop services.
```bash
/opt/hadoop-3.4.1/sbin/start-all.sh;
```
The MapReduce JobHistory Server is not started or stoped automatically, so we need to start it manually.
```bash
/opt/hadoop-3.4.1/bin/mapred --daemon start historyserver;
```
If you see the following output from `jps -m`, it indicates that Hadoop services are running correctly.
```bash
jps -m;
```
```text
9729 ResourceManager
9320 DataNode
10328 Jps -m
9132 NameNode
9853 NodeManager
10285 JobHistoryServer
9518 SecondaryNameNode
```
You can access the Web UIs of each Hadoop service using the following URLs.
| Service  | URL |
| ------------- | ------------- |
|NameNode|http://test.hadoop.com:9870|
|DataNode|http://test.hadoop.com:9864|
|SecondaryNameNode|http://test.hadoop.com:9868|
|ResourceManager|http://test.hadoop.com:8088|
|NodeManager|http://test.hadoop.com:8042|
|JobHistoryServer|http://test.hadoop.com:19888|

Now stop all Hadoop services before proceeding with the Hive installation.
```bash
/opt/hadoop-3.4.1/sbin/stop-all.sh;
/opt/hadoop-3.4.1/bin/mapred --daemon stop historyserver;
```
---
## Install Hive
In this step, we’ll install Hive and configure PostgreSQL 16 as its metastore database.
Use the following commands to install PostgreSQL 16 on your system via dnf.
```bash
dnf install https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm;
dnf install https://rpmfind.net/linux/centos-stream/9-stream/CRB/x86_64/os/Packages/perl-IO-Tty-1.16-4.el9.x86_64.rpm;
dnf install https://rpmfind.net/linux/centos-stream/9-stream/CRB/aarch64/os/Packages/perl-IPC-Run-20200505.0-6.el9.noarch.rpm;
dnf install postgresql16 postgresql16-devel postgresql16-server;
```
PostgreSQL 16 requires an initial database setup before it can be started.
```bash
postgresql-16-setup initdb;
```
To allow both local and remote connections to PostgreSQL, set listen_addresses to '*' in /var/lib/pgsql/16/data/postgresql.conf, and modify /var/lib/pgsql/16/data/pg_hba.conf to allow remote client access.
```bash
vi /var/lib/pgsql/16/data/postgresql.conf;
```
```text
# listen_addresses = 'localhost'
listen_addresses = '*'
```
This change allows PostgreSQL to accept connections from all IP addresses.
```bash
vi /var/lib/pgsql/16/data/pg_hba.conf;
```
```text
# host    all             all             127.0.0.1/32            scram-sha-256
host    all             all             ${YOUR_IP}/24            scram-sha-256
```
This rule allows clients in the ${YOUR_IP}/24 subnet to connect to any database as any user using scram-sha-256 authentication.

Let’s start PostgreSQL 16 and connect to it.
```bash
systemctl start postgresql-16;
su - postgres -c "psql";
```
Now let’s create the Hive user and database.
```sql
CREATE USER hive WITH PASSWORD 'hive';
CREATE DATABASE hive OWNER hive;
\q;
```
The Hive metastore is now ready to be installed.
Let’s create the hive user and add it to the hadoop group
```bash
useradd -G hadoop hive;
```
Download Hive and the PostgreSQL JDBC driver
```bash
wget https://dlcdn.apache.org/hive/hive-4.0.1/apache-hive-4.0.1-bin.tar.gz;
wget https://repo1.maven.org/maven2/org/postgresql/postgresql/42.5.4/postgresql-42.5.4.jar;
```
Extract Hive to the `/opt` directory, and move the JDBC driver into Hive’s lib directory so it can connect to the PostgreSQL metastore.
```bash
tar -zxf apache-hive-4.0.1-bin.tar.gz -C /opt;
mv postgresql-42.5.4.jar /opt/apache-hive-4.0.1-bin/lib;
chown -R hive. /opt/apache-hive-4.0.1-bin;
```
Now log in as the hive user, create the `hive-env.sh` file from the provided template, and configure the `HADOOP_HOME` environment variable.
Also, set conditional logging configuration for the metastore and HiveServer2 based on the service type.
```bash
su - hive;
cp /opt/apache-hive-4.0.1-bin/conf/hive-env.sh.template /opt/apache-hive-4.0.1-bin/conf/hive-env.sh;
vi /opt/apache-hive-4.0.1-bin/conf/hive-env.sh;
```
```text
# HADOOP_HOME=${bin}/../../hadoop
export HADOOP_HOME=/opt/hadoop-3.4.1
if [[ "$SERVICE" == "metastore" ]]; then
   export HIVE_LOG4J_FILE=/opt/apache-hive-4.0.1-bin/conf/hive-metastore-log4j2.properties
   export HADOOP_OPTS="$HADOOP_OPTS -Dlog4j2.configurationFile=$HIVE_LOG4J_FILE"
elif [[ "$SERVICE" == "hiveserver2" ]]; then
   export HIVE_LOG4J_FILE=/opt/apache-hive-4.0.1-bin/conf/hive-server2-log4j2.properties
   export HADOOP_OPTS="$HADOOP_OPTS -Dlog4j2.configurationFile=$HIVE_LOG4J_FILE"
fi
```
Create `/opt/apache-hive-4.0.1-bin/conf/hive-site.xml` and set the configuration variables according to your environment.
```bash
vi /opt/apache-hive-4.0.1-bin/conf/hive-site.xml;
```
```xml
<configuration>
   <property>
      <name>javax.jdo.option.ConnectionURL</name>
      <value>jdbc:postgresql://test.hadoop.com:5432/hive?createDatabaseIfNotExist=true</value>
      <description>JDBC connect string for a JDBC metastore</description>
   </property>
   <property>
      <name>javax.jdo.option.ConnectionDriverName</name>
      <value>org.postgresql.Driver</value>
      <description>Driver class name for a JDBC metastore</description>
   </property>
   <property>
      <name>javax.jdo.option.ConnectionUserName</name>
      <value>hive</value>
      <description>Username to use against metastore database</description>
   </property>
   <property>
      <name>javax.jdo.option.ConnectionPassword</name>
      <value>hive</value>
      <description>Password to use against metastore database</description>
   </property>
   <property>
      <name>datanucleus.autoCreateSchema</name>
      <value>true</value>
   </property>
   <property>
      <name>datanucleus.fixedDatastore</name>
      <value>true</value>
   </property>
   <property>
      <name>datanucleus.autoCreateTables</name>
      <value>True</value>
   </property>
   <property>
      <name>hive.metastore.warehouse.dir</name>
      <value>/user/hive/warehouse</value>
   </property>
   <property>
      <name>hive.metastore.uris</name>
      <value>thrift://test.hadoop.com:9083</value>
   </property>
   <property>
      <name>hive.server2.enable.doAs</name>
      <value>true</value>
   </property>
</configuration>
```
For convenience, create `/opt/apache-hive-4.0.1-bin/conf/beeline-hs2-connection.xml` to preconfigure Beeline login credentials.
```bash
vi /opt/apache-hive-4.0.1-bin/conf/beeline-hs2-connection.xml;
```
```xml
<configuration>
   <property>
      <name>beeline.hs2.connection.user</name>
      <value>hive</value>
   </property>
   <property>
      <name>beeline.hs2.connection.password</name>
      <value>hive</value>
   </property>
</configuration>
```
Create the following three configuration files from `/opt/apache-hive-4.0.1-bin/conf/hive-log4j2.properties.template` for logging purposes. For each file, update the log directory and file name appropriately.
```bash
cp /opt/apache-hive-4.0.1-bin/conf/hive-log4j2.properties.template /opt/apache-hive-4.0.1-bin/conf/hive-log4j2.properties;
vi /opt/apache-hive-4.0.1-bin/conf/hive-log4j2.properties;
```
```text
# property.hive.log.dir = ${sys:java.io.tmpdir}/${sys:user.name}
property.hive.log.dir = /opt/apache-hive-4.0.1-bin/logs
```
```bash
cp /opt/apache-hive-4.0.1-bin/conf/hive-log4j2.properties.template /opt/apache-hive-4.0.1-bin/conf/hive-server2-log4j2.properties;
vi /opt/apache-hive-4.0.1-bin/conf/hive-server2-log4j2.properties;
```
```text
# property.hive.log.dir = ${sys:java.io.tmpdir}/${sys:user.name}
# property.hive.log.file = hive.log
property.hive.log.dir = /opt/apache-hive-4.0.1-bin/logs
property.hive.log.file = hiveserver2.log
```
```bash
cp /opt/apache-hive-4.0.1-bin/conf/hive-log4j2.properties.template /opt/apache-hive-4.0.1-bin/conf/hive-metastore-log4j2.properties;
vi /opt/apache-hive-4.0.1-bin/conf/hive-metastore-log4j2.properties;
```
```text
# property.hive.log.dir = ${sys:java.io.tmpdir}/${sys:user.name}
# property.hive.log.file = hive.log
property.hive.log.dir = /opt/apache-hive-4.0.1-bin/logs
property.hive.log.file = metastore.log
```
---
Append the following to the end of `/opt/hadoop-3.4.1/etc/hadoop/core-site.xml` as user hadoop. 
```bash
su - hadoop;
vi /opt/hadoop-3.4.1/etc/hadoop/core-site.xml;
```
```xml
    <property>
        <name>hadoop.proxyuser.hive.hosts</name>
        <value>*</value>
    </property>
    <property>
        <name>hadoop.proxyuser.hive.groups</name>
        <value>*</value>
    </property>
```
And start hadoop. And Create the default warehouse directory for Hive.
```bash
/opt/hadoop-3.4.1/sbin/start-all.sh;
/opt/hadoop-3.4.1/bin/hdfs dfs -mkdir -p /user/hive/warehouse;
/opt/hadoop-3.4.1/bin/hdfs dfs -chmod 770 /user/hive/warehouse;
/opt/hadoop-3.4.1/bin/hdfs dfs -chown -R hive:hadoop /user/hive;
```
---
Initialize the Hive metastore schema in the PostgreSQL database using the following command.
```bash
/opt/apache-hive-4.0.1-bin/bin/schematool -dbType postgres -initSchema;
```
Start the Hive Metastore in the background using nohup so it continues running after you log out.
```bash
nohup /opt/apache-hive-4.0.1-bin/bin/hive --service metastore > /dev/null 2>&1 &
```
Start HiveServer2 in the background using nohup so it keeps running after logout.
```bash
nohup /opt/apache-hive-4.0.1-bin/bin/hiveserver2 > /dev/null 2>&1 &
```
If you see the following output from `jps -m`, it means that the Hive Metastore and HiveServer2 are running properly.
```bash
jps -m;
```
```text
29139 Jps -m
27764 RunJar /opt/apache-hive-4.0.1-bin/lib/hive-service-4.0.1.jar org.apache.hive.service.server.HiveServer2
27671 RunJar /opt/apache-hive-4.0.1-bin/lib/hive-metastore-4.0.1.jar org.apache.hadoop.hive.metastore.HiveMetaStore
```
