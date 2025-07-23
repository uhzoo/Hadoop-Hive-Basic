# Installing Hadoop and Hive
Step-by-step guide for installing a basic Hadoop and Hive.

| Service  | Version |
| ------------- | ------------- |
|OS|Rocky Linux 9.4 Minimal, 2 vCPUs, 4 GB RAM|
|Hadoop|3.4.1|
|Hive|4.0.1|
|Hive MetaStore|PostgreSQL 16|

The following guide uses `test.hadoop.com` in example URLs.  
To use them as-is, update your hosts file to map `test.hadoop.com` to your server’s IP address.
- Windows: `C:\Windows\System32\drivers\etc\hosts`
- Mac/Linux: `/etc/hosts`
  
Alternatively, you can access services directly using ${YOUR_IP}:${PORT}.

## Install Hadoop
Before installing Hadoop, disable the firewall for convenience.
```bash
systemctl disable firewalld --now;
```
Hadoop requires Java (JDK 8), so you need to install it first.
```bash
dnf install java-1.8.0-*;
```
To download and extract the installation file, install `wget` and `tar`.
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
We will store all Hadoop-related data under `/data`, so let's create the directory and give ownership to the hadoop user.
```bash
mkdir -p /data;
chown -R hadoop. /data;
```
Now login to hadoop.
```bash
su - hadoop;
```
In order to use useful commands like `start-all.sh`, passwordless SSH access is required. Generate an SSH key and copy it to the authorized hosts. When prompted during `ssh-keygen`, simply press Enter to accept the defaults.
```bash
ssh-keygen -t rsa -m PEM;
ssh-copy-id -i /home/hadoop/.ssh/id_rsa.pub $(hostname -f);
```
Now configure the `HADOOP_HOME` and `JAVA_HOME` environment variables in `/opt/hadoop-3.4.1/etc/hadoop/hadoop-env.sh`. You can determine your `JAVA_HOME` path with the following command: `readlink -f /usr/bin/java`
```bash
vi /opt/hadoop-3.4.1/etc/hadoop/hadoop-env.sh;
```
```text
# export JAVA_HOME=
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.462.b08-3.el9.x86_64/jre
# export HADOOP_HOME=
export HADOOP_HOME=/opt/hadoop-3.4.1
# export HADOOP_HEAPSIZE_MAX=
export HADOOP_HEAPSIZE_MAX=512
# export HADOOP_HEAPSIZE_MIN=
export HADOOP_HEAPSIZE_MIN=512
```
Next, configure the following Hadoop XML files to set up the core components.
1. core-site.xml
```bash
vi /opt/hadoop-3.4.1/etc/hadoop/core-site.xml;
```
```xml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://test.hadoop.com:9000</value>
        <description>The name of the default file system. A URI whose scheme and authority determine the FileSystem implementation. The uri's scheme determines the config property (fs.SCHEME.impl) naming the FileSystem implementation class. The uri's authority is used to determine the host, port, etc. for a filesystem.</description>
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
        <description>The address and the base port where the dfs namenode web ui will listen on.</description>
    </property>
    <property>
        <name>dfs.secondary.http.address</name>
        <value>test.hadoop.com:9868</value>
        <description>The address and the base port where the dfs secondary namenode web ui will listen on.</description>
    </property>
    <property>
        <name>dfs.datanode.http.address</name>
        <value>test.hadoop.com:9864</value>
        <description>The datanode http server address and port.</description>
    </property>
    <property>
        <name>dfs.namenode.name.dir</name>
        <value>file:/data/namenode</value>
        <description>Determines where on the local filesystem the DFS name node should store the name table(fsimage). If this is a comma-delimited list of directories then the name table is replicated in all of the directories, for redundancy.</description>
    </property>
    <property>
        <name>dfs.datanode.data.dir</name>
        <value>file:/data/datanode</value>
        <description>Determines where on the local filesystem an DFS data node should store its blocks. If this is a comma-delimited list of directories, then data will be stored in all named directories, typically on different devices. Directories that do not exist are ignored.</description>
    </property>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
        <description>Default block replication. The actual number of replications can be specified when the file is created. The default is used if replication is not specified in create time.</description>
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
        <description>The http address of the RM web application.</description>
    </property>
    <property>
        <name>yarn.nodemanager.webapp.address</name>
        <value>test.hadoop.com:8042</value>
        <description>NM Webapp address.</description>
    </property>
    <property>
        <name>yarn.nodemanager.local-dirs</name>
        <value>/data/yarn/local</value>
        <description>List of directories to store localized files in. An application's localized file directory will be found in: ${yarn.nodemanager.local-dirs}/usercache/${user}/appcache/application_${appid}. Individual containers' work directories, called container_${contid}, will be subdirectories of this.</description>
    </property>
    <property>
        <name>yarn.nodemanager.log-dirs</name>
        <value>/data/yarn/logs</value>
        <description>Where to store container logs. An application's localized log directory will be found in ${yarn.nodemanager.log-dirs}/application_${appid}. Individual containers' log directories will be below this, in directories named container_{$contid}. Each container directory will contain the files stderr, stdin, and syslog generated by that container.</description>
    </property>
    <property>
        <name>yarn.nodemanager.resource.memory-mb</name>
        <value>2048</value>
        <description>Amount of physical memory, in MB, that can be allocated for containers.</description>
    </property>
    <property>
        <name>yarn.nodemanager.resource.cpu-vcores</name>
        <value>2</value>
        <description>Number of vcores that can be allocated for containers. This is used by the RM scheduler when allocating resources for containers. This is not used to limit the number of physical cores used by YARN containers.</description>
    </property>
    <property>
        <name>yarn.scheduler.maximum-allocation-mb</name>
        <value>2048</value>
        <description>The maximum allocation for every container request at the RM, in MBs. Memory requests higher than this will throw a InvalidResourceRequestException.</description>
    </property>
    <property>
        <name>yarn.scheduler.maximum-allocation-vcores</name>
        <value>2</value>
        <description>The maximum allocation for every container request at the RM, in terms of virtual CPU cores. Requests higher than this will throw a InvalidResourceRequestException.</description>
    </property>
    <property>
        <name>yarn.scheduler.minimum-allocation-mb</name>
        <value>512</value>
        <description>The minimum allocation for every container request at the RM, in MBs. Memory requests lower than this will throw a InvalidResourceRequestException.</description>
    </property>
    <property>
        <name>yarn.scheduler.minimum-allocation-vcores</name>
        <value>1</value>
        <description>The minimum allocation for every container request at the RM, in terms of virtual CPU cores. Requests lower than this will throw a InvalidResourceRequestException.</description>
    </property>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
        <description>A comma separated list of services where service name should only contain a-zA-Z0-9_ and can not start with numbers</description>
    </property>
    <property>
        <name>yarn.nodemanager.env-whitelist</name>
        <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_HOME,PATH,LANG,TZ,HADOOP_MAPRED_HOME</value>
        <description>Environment variables that containers may override rather than use NodeManager's default.</description>
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
        <description>MapReduce JobHistory Server IPC host:port</description>
    </property>
    <property>
        <name>mapreduce.jobhistory.webapp.address</name>
        <value>test.hadoop.com:19888</value>
        <description>MapReduce JobHistory Server Web UI host:port</description>
    </property>
    <property>
        <name>mapreduce.jobhistory.done-dir</name>
        <value>/mr-history/done</value>
        <description></description>
    </property>
    <property>
        <name>mapreduce.jobhistory.intermediate-done-dir</name>
        <value>/mr-history/tmp</value>
        <description></description>
    </property>
    <property>
        <name>mapreduce.map.memory.mb</name>
        <value>512</value>
        <description>The amount of memory to request from the scheduler for each map task.</description>
    </property>
    <property>
        <name>mapreduce.map.java.opts</name>
        <value>-Xmx400m</value>
        <description></description>
    </property>
    <property>
        <name>mapreduce.reduce.memory.mb</name>
        <value>512</value>
        <description>The amount of memory to request from the scheduler for each reduce task.</description>
    </property>
    <property>
        <name>mapreduce.reduce.java.opts</name>
        <value>-Xmx400m</value>
        <description></description>
    </property>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
        <description>The runtime framework for executing MapReduce jobs. Can be one of local, classic or yarn.</description>
    </property>
    <property>
        <name>mapreduce.application.classpath</name>
        <value>$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/*:$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/lib/*</value>
        <description>CLASSPATH for MR applications. A comma-separated list of CLASSPATH entries. If mapreduce.application.framework is set then this must specify the appropriate classpath for that archive, and the name of the archive must be present in the classpath. If mapreduce.app-submission.cross-platform is false, platform-specific environment vairable expansion syntax would be used to construct the default CLASSPATH entries. For Linux: $HADOOP_MAPRED_HOME/share/hadoop/mapreduce/*, $HADOOP_MAPRED_HOME/share/hadoop/mapreduce/lib/*. For Windows: %HADOOP_MAPRED_HOME%/share/hadoop/mapreduce/*, %HADOOP_MAPRED_HOME%/share/hadoop/mapreduce/lib/*. If mapreduce.app-submission.cross-platform is true, platform-agnostic default CLASSPATH for MR applications would be used: {{HADOOP_MAPRED_HOME}}/share/hadoop/mapreduce/*, {{HADOOP_MAPRED_HOME}}/share/hadoop/mapreduce/lib/* Parameter expansion marker will be replaced by NodeManager on container launch based on the underlying OS accordingly.</description>
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

After creating some default HDFS directories required for MapReduce and Hive, stop all Hadoop services before proceeding with the Hive installation.
```bash
/opt/hadoop-3.4.1/bin/hdfs dfs -mkdir -p /mr-history;
/opt/hadoop-3.4.1/bin/hdfs dfs -chmod -R 1777 /mr-history;
/opt/hadoop-3.4.1/bin/hdfs dfs -mkdir -p /tmp;
/opt/hadoop-3.4.1/bin/hdfs dfs -chmod -R 1777 /tmp;
/opt/hadoop-3.4.1/bin/mapred --daemon stop historyserver;
/opt/hadoop-3.4.1/sbin/stop-all.sh;
```
---
## Install Hive
In this step, we’ll install Hive and configure PostgreSQL 16 as its metastore database. Use the following commands to install PostgreSQL 16 on your system via dnf.
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
To allow both local and remote connections to PostgreSQL, set listen_addresses to '*' in `/var/lib/pgsql/16/data/postgresql.conf`, and modify `/var/lib/pgsql/16/data/pg_hba.conf` to allow remote client access.
```bash
vi /var/lib/pgsql/16/data/postgresql.conf;
```
```text
# listen_addresses = '*'          # what IP address(es) to listen on;
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
This rule allows clients in the `${YOUR_IP}/24` subnet to connect to any database as any user using scram-sha-256 authentication.

Let’s start PostgreSQL 16 and connect to it.
```bash
systemctl start postgresql-16;
su - postgres -c "psql";
```
Now let’s create the Hive user and database.
```sql
CREATE USER hive WITH PASSWORD 'hive';
CREATE DATABASE hive OWNER hive;
\q
```
The Hive metastore is now ready to be installed. Let’s create the hive user and add it to the hadoop group
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
Now log in as the hive user, create the `hive-env.sh` file from the provided template, and configure the `HADOOP_HOME` environment variable. Also, set conditional logging configuration for the metastore and HiveServer2 based on the service type.
```bash
su - hive;
cp /opt/apache-hive-4.0.1-bin/conf/hive-env.sh.template /opt/apache-hive-4.0.1-bin/conf/hive-env.sh;
```
```bash
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
        <description>
          JDBC connect string for a JDBC metastore.
          To use SSL to encrypt/authenticate the connection, provide database-specific SSL flag in the connection URL.
          For example, jdbc:postgresql://myhost/db?ssl=true for postgres database.
        </description>
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
        <description>password to use against metastore database</description>
    </property>
    <property>
        <name>hive.metastore.warehouse.dir</name>
        <value>/user/hive/warehouse</value>
        <description>location of default database for the warehouse</description>
    </property>
    <property>
        <name>hive.server2.enable.doAs</name>
        <value>true</value>
        <description>
          Setting this property to true will have HiveServer2 execute
          Hive operations as the user making the calls to it.
        </description>
    </property>
    <property>
        <name>hive.server2.webui.host</name>
        <value>test.hadoop.com</value>
        <description>The host address the HiveServer2 WebUI will listen on</description>
    </property>
    <property>
        <name>hive.server2.webui.port</name>
        <value>10002</value>
        <description>The port the HiveServer2 WebUI will listen on. This can beset to 0 or a negative integer to disable the web UI</description>
    </property>
    <property>
        <name>hive.server2.thrift.bind.host</name>
        <value>test.hadoop.com</value>
        <description>Bind host on which to run the HiveServer2 Thrift service.</description>
    </property>
    <property>
        <name>hive.server2.thrift.port</name>
        <value>10000</value>
        <description>Port number of HiveServer2 Thrift interface when hive.server2.transport.mode is 'binary'.</description>
    </property>
    <property>
        <name>hive.exec.scratchdir</name>
        <value>/tmp/hive</value>
        <description>HDFS root scratch dir for Hive jobs which gets created with write all (733) permission. For each connecting user, an HDFS scratch dir: ${hive.exec.scratchdir}/&lt;username&gt; is created, with ${hive.scratch.dir.permission}.</description>
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
        <description></description>
    </property>
    <property>
        <name>beeline.hs2.connection.password</name>
        <value>hive</value>
        <description></description>
    </property>
</configuration>
```
Create the following three configuration files from `/opt/apache-hive-4.0.1-bin/conf/hive-log4j2.properties.template` for logging purposes. For each file, update the log directory and file name appropriately.
1. hive-log4j2.properties
```bash
cp /opt/apache-hive-4.0.1-bin/conf/hive-log4j2.properties.template /opt/apache-hive-4.0.1-bin/conf/hive-log4j2.properties;
```
```bash
vi /opt/apache-hive-4.0.1-bin/conf/hive-log4j2.properties;
```
```text
# property.hive.log.dir = ${sys:java.io.tmpdir}/${sys:user.name}
property.hive.log.dir = /opt/apache-hive-4.0.1-bin/logs
```
2. hive-server2-log4j2.properties
```bash
cp /opt/apache-hive-4.0.1-bin/conf/hive-log4j2.properties.template /opt/apache-hive-4.0.1-bin/conf/hive-server2-log4j2.properties;
```
```bash
vi /opt/apache-hive-4.0.1-bin/conf/hive-server2-log4j2.properties;
```
```text
# property.hive.log.dir = ${sys:java.io.tmpdir}/${sys:user.name}
# property.hive.log.file = hive.log
property.hive.log.dir = /opt/apache-hive-4.0.1-bin/logs
property.hive.log.file = hiveserver2.log
```
3. hive-metastore-log4j2.properties
```bash
cp /opt/apache-hive-4.0.1-bin/conf/hive-log4j2.properties.template /opt/apache-hive-4.0.1-bin/conf/hive-metastore-log4j2.properties;
```
```bash
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
```
```
vi /opt/hadoop-3.4.1/etc/hadoop/core-site.xml;
```
```xml
    <property>
        <name>hadoop.proxyuser.hive.hosts</name>
        <value>*</value>
        <description></description>
    </property>
    <property>
        <name>hadoop.proxyuser.hive.groups</name>
        <value>*</value>
        <description></description>
    </property>
```
Now start hadoop, and Create the default directory for Hive.
```bash
/opt/hadoop-3.4.1/sbin/start-all.sh;
/opt/hadoop-3.4.1/bin/mapred --daemon start historyserver;
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
You can access the Web UI for Hive service using the following URLs.
| Service  | URL |
| ------------- | ------------- |
|HiveServer2|http://test.hadoop.com:10002|

---
## Simple Hadoop Test Jobs.
```bash
# Generate 1GB of synthetic data (100 bytes × 10 million rows) for TeraSort benchmark
/opt/hadoop-3.4.1/bin/hadoop jar /opt/hadoop-3.4.1/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.4.1.jar teragen 10000000 /tmp/teragen.dat

# Sort the generated data using TeraSort
/opt/hadoop-3.4.1/bin/hadoop jar /opt/hadoop-3.4.1/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.4.1.jar terasort /tmp/teragen.dat /tmp/terasort.dat

# Validate the sorted data to ensure correct ordering
/opt/hadoop-3.4.1/bin/hadoop jar /opt/hadoop-3.4.1/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.4.1.jar teravalidate /tmp/terasort.dat /tmp/teravalidate.dat

# Generate 1GB of random text data (used to test WordCount)
/opt/hadoop-3.4.1/bin/hadoop jar /opt/hadoop-3.4.1/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.4.1.jar randomtextwriter -D mapreduce.randomtextwriter.totalbytes=1000000000 /tmp/randomtextwriter.dat

# Run WordCount job on the generated random text data
/opt/hadoop-3.4.1/bin/hadoop jar /opt/hadoop-3.4.1/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.4.1.jar wordcount /tmp/randomtextwriter.dat /tmp/wordcount.dat
```
## Simple Hive Test.
```bash
wget https://raw.githubusercontent.com/datasciencedojo/datasets/master/titanic.csv -O titanic.csv;
tail -n +2 titanic.csv > titanic_noheader.csv;
/opt/hadoop-3.4.1/bin/hdfs dfs -mkdir /tmp/hive_input;
/opt/hadoop-3.4.1/bin/hdfs dfs -put -f titanic_noheader.csv /tmp/hive_input/titanic_noheader.csv;
```
```sql
CREATE EXTERNAL TABLE titanic (
  passengerid INT,
  survived INT,
  pclass INT,
  name STRING,
  sex STRING,
  age DOUBLE,
  sibsp INT,
  parch INT,
  ticket STRING,
  fare DOUBLE,
  cabin STRING,
  embarked STRING
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
WITH SERDEPROPERTIES (
  "separatorChar" = ",",
  "quoteChar"     = "\""
)
STORED AS TEXTFILE
LOCATION '/tmp/hive_input';
```
```sql
-- Survival rate by gender
SELECT sex, AVG(survived) AS survival_rate FROM titanic GROUP BY sex;

-- Survival rate by passenger class (1st, 2nd, 3rd)
SELECT pclass, AVG(survived) AS survival_rate FROM titanic GROUP BY pclass;

-- Survival rate by gender and class
SELECT sex, pclass, AVG(survived) AS survival_rate FROM titanic GROUP BY sex, pclass;

-- Survival rate by family status (alone vs. with family)
SELECT
  CASE WHEN sibsp + parch = 0 THEN 'Alone' ELSE 'With Family' END AS family_status,
  AVG(survived) AS survival_rate
FROM titanic
GROUP BY CASE WHEN sibsp + parch = 0 THEN 'Alone' ELSE 'With Family' END;

-- Average age by survival status
SELECT survived, AVG(age) AS avg_age FROM titanic GROUP BY survived;

-- Fare distribution by survival status
SELECT survived, AVG(fare) AS avg_fare FROM titanic GROUP BY survived;

-- Average age and fare overall
SELECT AVG(age) AS avg_age, AVG(fare) AS avg_fare FROM titanic;

-- Average fare by passenger class
SELECT pclass, AVG(fare) AS avg_fare FROM titanic GROUP BY pclass;

-- Survival rate by embarkation port
SELECT embarked, AVG(survived) AS survival_rate FROM titanic GROUP BY embarked;

-- Average fare by embarkation port
SELECT embarked, AVG(fare) AS avg_fare FROM titanic GROUP BY embarked;
```
