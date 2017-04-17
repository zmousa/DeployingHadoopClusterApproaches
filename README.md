Deploying Hadoop Cluster Approaches
=========================

**Table of Contents**

+ Introduction
+ Approaches
  + BigTop cluster
  + Hadoop Cluster based on vagrant and shell scripts provisioning
  + Cloudera Cluster based on vagrant and Ansible provisioning
+ References

------------

# Introduction
When we work with setting-up some sort of cluster, we have to repeat the same commands to setup the host OS or to initialize deployment environment, what make some sysadmins prepare a set of scripts that hold most of the work as a semi-automated way to speed-up the process.
But when we work with master/slave clusters like Hadoop it become a pain to manage all configurations and redo the process to hundred (or even thousands) of nodes, which opened the door to companies to invest in this field and provide a management tool that lead the cluster installation and configuration process.
But since the hadoop core tools are open-source, people keep looking for an whole open-source solution that give them less pain and keep open for development, and achieving that using some sort of provisioning tool.

We have many criteria that involve in the deployment of the hadoop cluster, which make a difference in the tools and ways of this deployment:
1. **Commercial vs open source tools**
Hadoop itself is an open-source framework, but some companies release or sell products that include the official Hadoop release files, and their own and other useful tools.
[Cloudera](http://www.cloudera.com/) and [Hortonworks](http://www.hortonworks.com/) are the most common distributions who provide a platform contains majority of related big data projects with technical support and training sessions.
In addition some open source projects also provide a repository of all open source tools that might needed for a hadoop cluster and can provide an alternative of the commercial platform like BigTop project.
2. **Automated vs manual provisioning**
Previsioning is the process of tools installation and configuration in a host machine, and this could be a pain when working with cluster of too much nodes, so sysadmins either use some scripts that hold all commands and configuration and apply it on all machines.
Instead of achieving in-house developed scripts, deployment management tools and configuration management tools enable you to use recipes, templates, or whatever scripts to simplify automation across environment in order to provide a standard, consistent deployment, and most popular tools are [Ansible](https://www.ansible.com/  "Ansible"), [Chef](https://www.chef.io/  "Chef") and [Puppet](https://puppet.com/ "Puppet").
3. **Virtual vs real testing clusters**
There is always need to have a testing Hadoop cluster that enable teams and individuals to learn and test their big data parallel jobs and tools, and in this case either you go with real hosting cluster with different nodes for testing, or just use the a virtual machine based cluster.
virtual machine managers tools allow you to script the virtual machine configuration as well as the provisioning, depending on Virtual Box (or others), and the most popular tool in this domain is [Vagrant](https://www.vagrantup.com/ "Vagrant")

Under above criteria we will check different solutions for deploying Hadoop cluster:
1. Cloudera cluster, throughout cloudera manager.
2. Cloudera cluster, throughout Ansible provisioning.
3. Hadoop cluster, throughout Ansible provisioning and based on CDH repository.
4. Hadoop cluster, throughout shell scripting provisioning.
5. Hadoop cluster, throughout BigTop project.


------------

# Approaches

## BigTop cluster
BigTop is an open source project contains a deployment part of Apache Hadoop ecosystem, where it provides a distribution repository that contains a compatible versions for a majority of open source components including (hadoop, hbase, hive, flume, hue, kafka, mahout, oozie, solr, spark, sqoop, sqoop2, zookeeper …)
In addition to puppet provisioning modules that provide an automated way for deployment.
**Source**: https://github.com/apache/bigtop
**Steps**:
+ Setup virtual machine cluster nodes using Vagrant, by setting VM settings, installing the proper OS and configure the private network.
  + As BitgTop predefined vagrant file depends on a property file vagrantconfig.yaml to set the main attributes:
      +  Set the memory size for each node, number of CPU’s and the virtual box OS used:
 ```bash
memory_size  "4096"
number_cpus  "1"
box "puppetlabs/centos-7.2-64-nocm" 
```
    + Set the number of nodes in the clusters.
```bash
num_instances"3"
```
    + Set the repository where hadoop services will pulled from:
```bash
repo "http://bigtop-repos.s3.amazonaws.com/releases/1.1.0/centos/7/x86_64"
```
    + Set up the components that will installed in the cluster:
```bash
components [hadoop, yarn, hive, hue, spark]
```
    + Set the JDK version to be installed
```bash
jdk  "java-1.7.0-openjdk-devel.x86_64"
```
  +  Setup the hadoop master node from  Vagrantfile
    + Change the property
```bash
bigtop_master  "bigtop1.vagrant"
```
    + User also able to set other role for specific nodes, not just setting master and slaves nodes by setting custom properties in the hiera property file cluster.yaml:
```bash
hadoop_head_node  "bigtop1.vagrant" 
hadoop_standby_head_node "bigtop2.vagrant" 
hadoop_gateway_node  "bigtop3.vagrant" 
```
  + Before running the vagrant file, two pulgins must be downloaded, the host manager plugin that manages the hosts file on guest machines, and cacher plugin that share the common package cache among similar VM instances.
```bash
$ vagrant plugin install vagrant-hostmanager
$ vagrant plugin install vagrant-cachier
```
  + Run the vagrant file
```bash
$ Vagrant up
```
+ Puppet provisioning, responsible for automating the environment initialization, deploying tools and configuring communication.
  + Some default values realted to the tools and services, like hosts and ports could be changed by altering hiera property file `cluster.yamal`:
```bash
kafka:: server::port "9092"
kafka:: server::zookeeper_connection_string "%{hiera('bigtop::hadoop_head_node')}:2181"
```
  + Installing java will take a place inside the puppet files in bigtop-toolchain component.
  ```json
package { 'openjdk-7-jdk' :
     ensure => present,
  }
```
  + For each hadoop component there is a puppet module, and this includes tasks such as:
    + Service installation.
    + Pointing slaves to masters (i.e. regionservers, nodemanagers to their respective master) 
    + Starting the services.

+ Adding additional modules to BigTop project to facilitate the full big data project usage:
  + Adding MySQL and SQL JDBC connector puppet modules to be used in the projects that use RDBMS database and migration processes.
    + Changing the mysql server configurations from site.pp file as below:
 ```json
 class { ':: mysql::server':
       root_password    => 'strongpassword',
       override_options => { 
	      'mysqld' => { 'max_connections' => '1024' }    
       }
  }
```
    + With Ability to create default database, by adding such configurations:
 ```json
 mysql::db { 'mydb':
    user     => 'myuser',
    password => 'mypass',
    host     => 'localhost',
    grant    => ['SELECT', 'UPDATE'],
  }
```
  + Installing Ganglia module, which is a distributed monitoring system to view live statistics covering metrics such as CPU load averages or network utilization for cluster nodes, below configuration should be changed at site.pp in order to be provisioned to the monitoring daemon (updated manually at “/etc/ganglia/gmond.conf”).
    + Change gmetad (Ganglia meta daemon) configuration to setup the cluster name
  ```json
class { 'ganglia::gmetad':
         clusters      => $clusters,
     gridname      => 'my grid',
     rras          => $rras,
     all_trusted   => false,
     trusted_hosts => [],
   }
```
    + Change gmond (Ganglia monitoring daemon) configuration to setup the configuration of sending and receiving metrics.
  ```json
$udp_recv_channel = [ 
         { host => '192.168.1.10', port => 8649 }
  ]
    class{ 'ganglia::gmond':
          cluster_name             => 'cluster-name',
        udp_recv_channel          => $udp_recv_channel,
     udp_send_channel          => $udp_send_channel
   }
```
    + Setting up Ganglia web module
  ```json
 class{ 'ganglia::web':
    ganglia_ip => '192.168.0.10',
    ganglia_port => 8652,
  }
```
    + You may also need to change the permissions of apache web server as below modifications for file “/etc/httpd/conf.d/ganglia.conf”
```xml
<Location /ganglia>
    Allow from all 
    Require all granted
</Location>
```
+ In case some problem faced throughout provisioning, re-running the provisioing part using:
 ```bash
 $ Vagrant provision
```
+ For accessing the nodes through ssh, you need to use the private key generated from vagrant as below:
```bash
  $ ssh vagrant@10.10.10.13 -i './.vagrant/machines/bigtop3/virtualbox/private_key'
```
+ Links for accessing the cluster components:
  + Hadoop dashboard: http://bigtop1.vagrant:50070/
  + Hadoop Cluster Manager: http://bigtop1.vagrant:8088/cluster/apps
  + Hadoop job History: http://10.10.10.11:19888/jobhistory
  + Spark: http://bigtop1.vagrant:8081/
  + Hue: http://bigtop1.vagrant:8888
  + Ganglia: http://bigtop1.vagrant/ganglia
  + Oozie: http://bigtop1.vagrant:11000/oozie/


------------

## Hadoop Cluster based on vagrant and shell scripts provisioning
Most sysadmins usually use scripts that hold all commands and configuration for the deployment processes, so this solution provide an automated way to provision virtual machine cluster nodes using these scripts.
**Source**: https://github.com/dnafrance/vagrant-hadoop-spark-cluster
**Steps**:
+ Setup virtual machine cluster nodes using Vagrant, by setting VM settings, installing the proper OS and configure the private network.
  + Configuring Vagrantfile by setting the VM box url and memory information for each node:
```bash
vm.box_url "https://github.com/2creatives/vagrant-centos/releases/download/v6.5.1/centos65-x86_64-20131205.box"
vm.customize ["modifyvm", :id, "--memory", "1024"]
```
  + Setting number of nodes:
```bash
numNodes "4"
```
These nodes will take the roles based on it’s order as below:
    + **node1** : HDFS NameNode + Spark Master
    + **node2** : YARN ResourceManager + JobHistoryServer + ProxyServer
    + **node3** : HDFS DataNode + YARN NodeManager + Spark Slave
    + **node4** : HDFS DataNode + YARN NodeManager + Spark Slave
  And all nodes from 3 on will considered as slaves
  + Running vagrant
```bash
$ Vagrant up
```
+ Use Vagrant shell provisioning functionality to run these scripts in order (with taking in consideration the role of the node master/slave), and noticing that recourse folder contains templates and scripts for the hadoop components to be used while provisioning:
  + Common script, used to set the tools versions and environment variables.
  + Setup centos, used to stop firewall and call common script.
 ```bash
$ node.vm.provision "shell", path: "scripts/setup-centos.sh"
```
  + Setup centos ssh, used to install sshpass rpm, generate the key and modify the hosts file in each node.
```bash
 $ node.vm.provision "shell", path: "scripts/setup-centos-ssh.sh"
```
  + Setup Java, used to install java and set the environment variable.
```bash
 $ node.vm.provision "shell", path: "scripts/setup-java.sh"
```
  + Setup hadoop, used to install hadoop, copy configuration and set the environment variables.
```bash
 $ node.vm.provision "shell", path: "scripts/setup-hadoop.sh"
```
  + Setup hadoop slaves, used to edit hadoop configuration file and set the slaves hosts.
```bash
 $ node.vm.provision "shell", path: "scripts/setup-hadoop-slaves.sh"
```
  + Setup spark, used for spark installation, setup and setting environment variable.
```bash
 $ node.vm.provision "shell", path: "scripts/setup-spark.sh"
```
  + Setup spark slaves, used to modify spark workers.
```bash
 $ node.vm.provision "shell", path: "scripts/setup-spark-slaves.sh"
```
  + Setup hive, used for hive installation, setup and setting environment variable.
```bash
 $ node.vm.provision "shell", path: "scripts/setup-hive.sh"
```
  + Post provisioning, after provisioning the cluster, we need to run some commands to initialize Hadoop cluster.
    + Accessing the nodes throughout ssh:
```bash
$ ssh vagrant@10.211.55.101
```
    + Running below on NameNode
```bash
   $HADOOP_PREFIX/sbin/hadoop-daemon.sh --config $HADOOP_CONF_DIR --script hdfs start namenode
   $HADOOP_PREFIX/sbin/hadoop-daemons.sh --config $HADOOP_CONF_DIR --script hdfs start datanode
   $SPARK_HOME/sbin/start-all.sh
```
    + Running below on ResourceManager
   ```bash
$HADOOP_YARN_HOME/sbin/yarn-daemon.sh --config $HADOOP_CONF_DIR start resourcemanager
   $HADOOP_YARN_HOME/sbin/yarn-daemons.sh --config $HADOOP_CONF_DIR start nodemanager
   $HADOOP_YARN_HOME/sbin/yarn-daemon.sh start proxyserver --config $HADOOP_CONF_DIR
   $HADOOP_PREFIX/sbin/mr-jobhistory-daemon.sh start historyserver --config $HADOOP_CONF_DIR
```
+ Links
  + [NameNode](http://10.211.55.101:50070/dfshealth.html "NameNode")
  + [ResourceManager](http://10.211.55.102:8088/cluster "ResourceManager")
  + [JobHistory](http://10.211.55.102:19888/jobhistory "JobHistory")
  + [Spark](http://10.211.55.101:4040/ "Spark")
+ Testing
  + MapReduce
```bash
$yarn jar /usr/local/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.6.0.jar pi 2 100
```
  + Spark
```bash
$SPARK_HOME/bin/spark-submit --class org.apache.spark.examples.SparkPi --master yarn-cluster --num-executors 10 --executor-cores 2 lib/spark-examples*.jar 100
```

------------


## Cloudera Cluster based on vagrant and Ansible provisioning
Cloudera Manager is a great tool to orchestrate CDH-based Apache Hadoop cluster, which used  for cluster installation, deploying configurations, restarting daemons to monitoring each cluster component.
Cloudera proposed an official project to automate the manual part of installing and configuring cloudera manager, thereby being able to replay steps for upcoming cluster installations.
**Source**: https://github.com/cloudera/cloudera-playbook
The automation tall used here is Ansible, and below are the steps to start this cluster:
+ Updating the hosts file that will assign nodes into the cluster roles.
```bash
[gateway_servers] 
<host>  
[master_servers] 
<host> 
[worker_servers] 
<host>
….
```
+ The Ansible playbook holds the basic roles that will start running on the hosts-groups, like installing java, Mariadb, cloudera manager agent…
```bash
  name: Install Java 
    hosts: cdh_servers 
    roles:
           - java
….
```
+ Run the Ansible playbook
```bash
$ ansible-playbook -i ./hosts ./site.yml -u vagrant --sudo  -K
```

Unfortunately, this repository is not well supported and contains real problems throughout installation.


------------


# References
+ [BitTop project](https://github.com/apache/bigtop)
+ [Cloudera Ansible playbook](https://github.com/cloudera/cloudera-playbook)
+ [Shell provision based cluster](https://github.com/dnafrance/vagrant-hadoop-spark-cluster)

