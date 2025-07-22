# Installing Hadoop and Hive
Step-by-step guide for installing a basic Hadoop and Hive.

| Service  | Version |
| ------------- | ------------- |
|OS|Rocky Linux 9.4 Minimal, 4 vCPUs, 8 GB RAM|
|Hadoop|3.4.1|
|Hive|4.0.1
|Hive MetaStore|Postgresql 16|

## Install Hadoop
Before installing Hadoop, disable the firewall to allow cluster communication.
```
systemctl disable firewalld --now
```

Hadoop requires Java (JDK 8), so install it first.
```
dnf install java-1.8.0-*
```
In order to download and extract the installation file, we need to install wget and tar.
```
dnf install wget tar
```
Configure `/etc/hosts` to enable access to Hadoop using FQDN.
```
hostnamectl set-hostname test.hadoop.com
echo $(hostname -I | awk '{print $1}') $(hostname -f) >> /etc/hosts
or
echo ${YOUR_IP} $(hostname -f) >> /etc/hosts
```
Add user hadoop and set password.
```
useradd hadoop
passwd hadoop
```
Now login to hadoop.
```
su - hadoop
```
Useful commands like start-all.sh require passwordless SSH access, so we need to generate an SSH key and copy it to authorized hosts.
```
ssh-keygen -t rsa -m PEM
ssh-copy-id -i /home/hadoop/.ssh/id_rsa.pub $(hostname -f)
```
Now let’s download the Hadoop installation file and extract it.
```
wget https://dlcdn.apache.org/hadoop/common/hadoop-3.4.1/hadoop-3.4.1.tar.gz
tar -zxf hadoop-3.4.1.tar.gz
```
Now we need to configure `HADOOP_HOME` and `JAVA_HOME` in `/home/hadoop/hadoop-3.4.1/etc/hadoop/hadoop-env.sh`
You can find your `JAVA_HOME` path using the command: `readlink -f /usr/bin/java`
```
vi /home/hadoop/hadoop-3.4.1/etc/hadoop/hadoop-env.sh
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.462.b08-3.el9.x86_64/jre
export HADOOP_HOME=/home/hadoop/hadoop-3.4.1
```
Now we need to configure below files.

`/home/hadoop/hadoop-3.4.1/etc/hadoop/core-site.xml`
```
# core-site.xml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://test.hadoop.com:9000</value>
        <description>The default filesystem URI. All HDFS paths will be resolved relative to this.</description>
    </property>
</configuration>
```
`/home/hadoop/hadoop-3.4.1/etc/hadoop/hdfs-site.xml`
```
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
        <description>The default number of data block replications. Set to 1 for single-node or test environments.</description>
    </property>
</configuration>
```
`/home/hadoop/hadoop-3.4.1/etc/hadoop/yarn-site.xml`
```
<configuration>
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
`/home/hadoop/hadoop-3.4.1/etc/hadoop/mapred-site.xml`
```
<configuration>
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
Now that the basic configuration is complete, let’s format the NameNode.
```
/home/hadoop/hadoop-3.4.1/bin/hdfs namenode -format
```
Now lets start all services.
```
/home/hadoop/hadoop-3.4.1/sbin/start-all.sh
```
If you see the following output from `jps -m`, it means the setup was successful.
```
16098 ResourceManager
16210 NodeManager
15559 NameNode
15720 DataNode
16556 Jps -m
15902 SecondaryNameNode
```
Now shutdown all services.
```
/home/hadoop/hadoop-3.4.1/sbin/stop-all.sh
```
## Install Hive
Now we’re going to install Hive and set up PostgreSQL 16 as its metastore database.
Follow the commands below to install PostgreSQL 16 using dnf.
```
dnf -y install https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm
dnf -y install https://rpmfind.net/linux/centos-stream/9-stream/CRB/x86_64/os/Packages/perl-IO-Tty-1.16-4.el9.x86_64.rpm 
dnf -y install https://rpmfind.net/linux/centos-stream/9-stream/CRB/aarch64/os/Packages/perl-IPC-Run-20200505.0-6.el9.noarch.rpm
dnf -y install postgresql16 postgresql16-devel postgresql16-server
```
In order to start PostgreSQL 16, we need to initialize the database first.
```
postgresql-16-setup initdb
```
To allow both local and remote connections to PostgreSQL, set listen_addresses to '*' from `/var/lib/pgsql/16/data/postgresql.conf` and update `/var/lib/pgsql/16/data/pg_hba.conf` as shown below.
```
vi /var/lib/pgsql/16/data/postgresql.conf
listen_addresses = '*'
vi /var/lib/pgsql/16/data/pg_hba.conf
local   all             all                                     trust
host    all             all             ${YOUR_IP}/24            scram-sha-256
```
Let’s start PostgreSQL 16 and connect to it.
```
systemctl start postgresql-16
psql -U postgres
```
Now let’s create the Hive database and user, and grant all privileges.
```
CREATE DATABASE hive;
CREATE USER hive WITH PASSWORD 'hive';
GRANT ALL PRIVILEGES ON DATABASE hive TO hive;
```
The Hive metastore is now ready to be installed.
Let’s switch to the hadoop user and begin the Hive installation.
First, we’ll download the installation files and extract them.
```
su - hadoop
wget https://dlcdn.apache.org/hive/hive-4.0.1/apache-hive-4.0.1-bin.tar.gz
wget https://repo1.maven.org/maven2/org/postgresql/postgresql/42.5.4/postgresql-42.5.4.jar
tar -zxf apache-hive-4.0.1-bin.tar.gz
```
Create `/home/hadoop/apache-hive-4.0.1-bin/conf/hive-env.sh` from the template file `/home/hadoop/apache-hive-4.0.1-bin/conf/hive-env.sh.template`, and set the `HADOOP_HOME` variable.
```
cp /home/hadoop/apache-hive-4.0.1-bin/conf/hive-env.sh.template /home/hadoop/apache-hive-4.0.1-bin/conf/hive-env.sh
vi /home/hadoop/apache-hive-4.0.1-bin/conf/hive-env.sh
export HADOOP_HOME=/home/hadoop/hadoop-3.4.1
```
Create `/home/hadoop/apache-hive-4.0.1-bin/conf/hive-site.xml` and set the configuration variables according to your environment.
```
vi /home/hadoop/apache-hive-4.0.1-bin/conf/hive-site.xml
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
</configuration>
```
Create `/home/hadoop/apache-hive-4.0.1-bin/conf/beeline-hs2-connection.xml` for convenience. 
```
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
Move the PostgreSQL JDBC driver into Hive’s library path
```
mv /home/hadoop/postgresql-42.5.4.jar /home/hadoop/apache-hive-4.0.1-bin/lib
```
Since we are running Hive as the hadoop user, Append the following to the end of `/home/hadoop/hadoop-3.4.1/etc/hadoop/core-site.xml`
```
    <property>
        <name>hadoop.proxyuser.hadoop.hosts</name>
        <value>*</value>
    </property>
    <property>
        <name>hadoop.proxyuser.hadoop.groups</name>
        <value>*</value>
    </property>
```
Initialize the Hive metastore schema using the following command.
```
/home/hadoop/apache-hive-4.0.1-bin/bin/schematool -dbType postgres -initSchema
```
Start hadoop.
```
/home/hadoop/hadoop-3.4.1/sbin/start-all.sh
```
Start hiveserver2 in background.
```
mkdir /home/hadoop/apache-hive-4.0.1-bin/logs
nohup /home/hadoop/apache-hive-4.0.1-bin/bin/hiveserver2 > /home/hadoop/apache-hive-4.0.1-bin/logs/hiveserver2.log 2>&1 &
```
Create default location for Hive.
```
hdfs dfs -mkdir /user/hive
```
