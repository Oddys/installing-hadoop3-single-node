# Installing Hadoop 3 in pseudo-distributed mode

## (Version 3.2.1 in Docker container from Windows 10 host)

## Links

[Hadoop: Setting up a Single Node Cluster](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/SingleCluster.html)

[Install Hadoop 3.2: Setting up a Single Node Hadoop Cluster](https://medium.com/@thedsa.in/install-hadoop-3-2-setting-up-a-single-node-hadoop-cluster-22a5754bd9fc)

## Start Docker container

Download openjdk:8 image:
```sh
docker pull openjdk:8
```

(Optional) Create Docker network:
```sh
docker network create --driver bridge myhadoop
```

Run container (exposing ports for the resource manager, NameNode and
	MapReduce history server):
```
docker run --name myhadoop --network myhadoop -p 8088:8088 -p 9870:9870 `
-p 19888:19888 -itd openjdk:8 bash
```

## Prepare configuration files for your cluster

Create following files in hadoop_config directory on your host:

###### core-site.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
	<property>
		<name>fs.defaultFS</name>
		<value>hdfs://localhost:9000</value>
	</property>
	<property>
		<name>hadoop.tmp.dir</name>
		<value>/usr/local/hadoop_store/tmp</value>     
	</property>
</configuration>
```


###### hdfs-site.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
	<property>
		<name>dfs.name.dir</name>
		<value>/usr/local/hdfs_store/namenode</value>
	</property>
	<property>
		<name>dfs.data.dir</name>
		<value>/usr/local/hdfs_store/datanode</value>
	</property>
</configuration>
```

###### yarn-site.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <property>
        <name>yarn.nodemanager.env-whitelist</name>
        <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_MAPRED_HOME</value>
    </property>
</configuration>
```

###### mapred-site.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
    <property>
        <name>mapreduce.application.classpath</name>
        <value>$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/*:$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/lib/*</value>
    </property>
</configuration>
```

Copy your configuration files to the __root__ user directory in the container:
```sh
docker cp hadoop_config myhadoop:/root
```

## Install Hadoop

Go to your container:
```sh
docker exec -it myhadoop bash
```

Go to /tmp direcotry:
```sh
cd /tmp
```

Download a release from one of the [Apache Download Mirrors](http://www.apache.org/dyn/closer.cgi/hadoop/common/) (replace with your link):
```sh
wget https://apache.volia.net/hadoop/common/hadoop-3.2.1/hadoop-3.2.1.tar.gz
```

Untar Hadoop files to a chosen directory:
```sh
tar xzf hadoop-3.2.1.tar.gz -C /usr/local
```

## Configure Hadoop

Go to the Hadoop installation directory:
```sh
cd /usr/local/hadoop-3.2.1
```

Copy the configuration files from __root__ user directory to the Hadoop configuration directory:
```sh
mv /root/*.xml etc/hadoop
```

Create HADOOP_HOME environment variable and add Hadoop binaries to PATH  
(add these lines to ~/.bashrc file), so that you will be able to use the variable
and call Hadoop scripts from anywhere:
```sh
export HADOOP_HOME=/usr/local/hadoop-3.2.1
export PATH=$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$PATH
```

Put these lines to etc/hadoop/hadoop-env.sh:
```sh
export JAVA_HOME=/usr/local/openjdk-8
export HADOOP_HOME=/usr/local/hadoop-3.2.1
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
export HADOOP_LOG_DIR=$HADOOP_HOME/logs
export HDFS_NAMENODE_USER=root
export HDFS_DATANODE_USER=root
export HDFS_SECONDARYNAMENODE_USER=root
export YARN_RESOURCEMANAGER_USER=root
export YARN_NODEMANAGER_USER=root
```

Create directories for HDFS (to make it easy to look at, for example,
the contents of NameNode's local directory):
```sh
mkdir -p $HADOOP_HOME/hdfs_store/tmp
chmod -R 755 $HADOOP_HOME/hdfs_store/tmp
mkdir -p $HADOOP_HOME/hdfs_store/namenode
chmod -R 755 $HADOOP_HOME/hdfs_store/namenode
mkdir -p $HADOOP_HOME/hdfs_store/datanode
chmod -R 755 $HADOOP_HOME/hdfs_store/datanode
```

## Prepare additional software
Install SSH:
```sh
apt update
apt install ssh
```

Run SSH service:
```sh
service ssh start
```

Create SSH keys:
```sh
ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 0600 ~/.ssh/authorized_keys
```

## Start the cluster

Format NameNode:
```sh
hfds namenode -format
```

Start NameNode daemon and DataNode daemon (can be stopped with stop-dfs.sh):
```sh
start-dfs.sh
```

Start YARN (can be stopped with stop-yarn.sh):
```sh
start-yarn.sh
```

Create your users's directory in HDFS:
```sh
hdfs dfs -mkdir /user
hdfs dfs -mkdir /user/root
```

(Optionally) Start MapReduce history server (can be stopped with ```mapred --daemon stop historyserver```):
```sh
mapred --daemon start historyserver
```

## Check installation in your browser

NameNode - http://localhost:9870/

ResourceManager - http://localhost:8088/

(if started) MapReduce History Server - http://localhost:19888/
