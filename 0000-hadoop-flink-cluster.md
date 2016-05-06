# Setting up a simple Hadoop/Flink cluster

Here are reported the steps to create a simple Hadoop/Flink cluster. This
guide was inspired from

* https://districtdatalabs.silvrback.com/creating-a-hadoop-pseudo-distributed-environment
* http://chaalpritam.blogspot.it/2015/05/hadoop-270-single-node-cluster-setup-on.html
* http://chaalpritam.blogspot.it/2015/05/hadoop-270-multi-node-cluster-setup-on.html

## System requirements

Three or more (virtual) machines with

* minimal installation of Debian 8
* at least 1GB RAM
* at least 10GB disk

## Preparing the environment

### Network

I'm using FQDNs to identify cluster nodes. You can configure a DNS or simply
put the names in `/etc/hosts` of each cluster node. The result should be
something like that

    $ cat /etc/hosts
    ...
    10.0.100.254    master.cluster    master
    10.0.100.1      slave1.cluster    slave1
    10.0.100.2      slave2.cluster    slave2
    ...
    10.0.100.N      slaveN.cluster    slaveN
    ...

If the hostname was not set during the installation of the base system, you
can change it by editing `/etc/hostname` and rebooting the machine.
More details on the
[DebianWiKi/ChangeHostname](https://wiki.debian.org/HowTo/ChangeHostname).

### SSH hardening

Hadoop/Flink cluster requires SSH access without password properly working.
While the specific configuration will be treated in the next steps, in
[this post](http://www.cyberciti.biz/tips/linux-unix-bsd-openssh-server-best-practices.html)
you may found some best practices about the hardening process.

### Firewall configuration

Hadoop requires some `tcp` ports to be opened on the firewall. Below the
`iptables` configuration. It will create a dedicated chain named (with a lot
of fantasy) `HADOOP` and assumes Hadoop working on the `10.0.100.0/24`
network. Further reading on `iptables` on the
[DebianWiki/iptables](https://wiki.debian.org/iptables).

    iptables -N HADOOP
    iptables -A INPUT -j HADOOP
    iptables -A HADOOP -p tcp -s 10.0.100.0/24 --dport 8025 -j ACCEPT
    iptables -A HADOOP -p tcp -s 10.0.100.0/24 --dport 8030 -j ACCEPT
    iptables -A HADOOP -p tcp -s 10.0.100.0/24 --dport 8033 -j ACCEPT
    iptables -A HADOOP -p tcp -s 10.0.100.0/24 --dport 8050 -j ACCEPT
    iptables -A HADOOP -p tcp -s 10.0.100.0/24 --dport 8088 -j ACCEPT
    iptables -A HADOOP -p tcp -s 10.0.100.0/24 --dport 9000 -j ACCEPT
    iptables -A HADOOP -p tcp -s 10.0.100.0/24 --dport 50070 -j ACCEPT
    iptables -A HADOOP -p tcp -s 10.0.100.0/24 --dport 50090 -j ACCEPT

Same approach for Flink

    iptables -N FLINK
    iptables -A INPUT -j FLINK
    # todo

## Common steps (master and slaves)

This section presents the installation steps that have to be performed on all
the cluster nodes.

### Generic

* Install useful tools (optional)

        $ apt-get update
        $ apt-get install rsync multitail tree vim

* Install Java and SSH

        $ apt-get update
        $ apt-get install openjdk-7-jdk
        $ apt-get install ssh

* Create `hadoop` and `flink` users. Since they will be considered something
like system users, I created them with disabled password.

        $ adduser --disabled-password hadoop
        $ adduser --disabled-password flink

### Hadoop specific

* Generate a keypair without passphrase for the `hadoop` user

        $ su - hadoop
        hadoop:~$ ssh-keygen -b 4092 -P '' -t rsa -f ~/.ssh/id_rsa
        Generating public/private rsa key pair.
        Created directory '/home/hadoop/.ssh'.
        Your identification has been saved in /home/hadoop/.ssh/id_rsa.
        Your public key has been saved in /home/hadoop/.ssh/id_rsa.pub
        ...

* Import all the other public keys of the cluster nodes

        hadoop:~$ cd .ssh
        hadoop:~$ cat id_rsa.pub >> authorized_keys
        hadoop:~$ cat /path/to/master.pub >> authorized_keys
        hadoop:~$ cat /path/to/slave1.pub >> authorized_keys
        hadoop:~$ cat /path/to/slave2.pub >> authorized_keys
        ...
        hadoop:~$ cat /path/to/slaveN.pub >> authorized_keys
        hadoop:~$ chmod 600 authorized_keys
        hadoop:~$ exit

* Allow `hadoop` user in SSH configuration

        $ cat /etc/ssh/sshd_config
        ...
        AllowUsers hadoop
        ...

* Download and untar Hadoop binaries (here I'm using v2.7.2)

        $ cd /opt
        $ wget http://it.apache.contactlab.it/hadoop/common/hadoop-2.7.2/hadoop-2.7.2.tar.gz
        $ tar zxvf ~/hadoop-2.7.2.tar.gz
        $ ln -s hadoop-2.7.2 hadoop
        $ chown -R hadoop:hadoop hadoop-2.7.2
        $ rm hadoop-2.7.2.tar.gz

* In the `hadoop` homedir, create the `.javarc` file as follows

        hadoop:~$ cat .javarc
        export JAVA_HOME="/usr/lib/jvm/default-java"

* In the `hadoop` homedir, create `.hadooprc` file as follows

        hadoop:~$ cat .hadooprc
        export HADOOP_HOME="/opt/hadoop"
        export PATH=$PATH:$HADOOP_HOME/bin
        export PATH=$PATH:$HADOOP_HOME/sbin
        export HADOOP_MAPRED_HOME=$HADOOP_HOME
        export HADOOP_COMMON_HOME=$HADOOP_HOME
        export HADOOP_HDFS_HOME=$HADOOP_HOME
        export YARN_HOME=$HADOOP_HOME
        export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
        export HADOOP_OPTS="-Djava.library.path=$HADOOP_HOME/lib"

* In the `hadoop` homedir, append to `.bashrc` the following code (it loads in
  the right order the just created rc files)

        hadoop_includes=( ".javarc" ".hadooprc" )
        for f in ${hadoop_includes[@]}; do
            if [ -f $f ]; then
                source $f
            fi
        done

* File: `/opt/hadoop/etc/hadoop/hadoop-env.sh`

        $ cat /opt/hadoop/etc/hadoop/hadoop-env.sh
        ...
        # The only required environment variable is JAVA_HOME.  All others are
        # optional.  When running a distributed configuration it is best to
        # set JAVA_HOME in this file, so that it is correctly defined on
        # remote nodes.

        # The java implementation to use.
        #export JAVA_HOME=${JAVA_HOME}
        export JAVA_HOME="/usr/lib/jvm/default-java"
        ...

* Create the Hadoop storage paths

        $ mkdir -p /opt/hadoop/var/hadoop_data/hdfs/datanode
        $ mkdir -p /opt/hadoop/var/hadoop_data/hdfs/namenode
        $ chown -R hadoop:hadoop /opt/hadoop

### Flink

> Todo

### On the master node

All the mentioned files are in `$HADOOP_HOME/etc/hadoop`. Make the changes as
`hadoop` user.

* File: `core-site.xml`

        $ cat core-site.xml
        ...
        <configuration>
            <property>
                <name>fs.defaultFS</name>
                <value>hdfs://master:9000</value>
            </property>
        </configuration>
        ...

* File: `yarn-site.xml`

        $ cat yarn-site.xml
        ...
        <configuration>
            <property>
                <name>yarn.nodemanager.aux-services</name>
                <value>mapreduce_shuffle</value>
            </property>
            <property>
                <name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>
                <value>org.apache.hadoop.mapred.ShuffleHandler</value>
            </property>
            <property>
                <name>yarn.resourcemanager.resource-tracker.address</name>
                <value>master:8025</value>
            </property>
            <property>
                <name>yarn.resourcemanager.scheduler.address</name>
                <value>master:8030</value>
            </property>
            <property>
                <name>yarn.resourcemanager.address</name>
                <value>master:8050</value>
            </property>
        </configuration>

* File: `mapred-site.xml`

        $ cp mapred-site.xml.template mapred-site.xml
        ... do changes ...
        $ cat mapred-site.xml
        ...
        <property>
            <name>mapreduce.framework.name</name>
            <value>yarn</value>
        </property>
        ...

* File: `hdfs-site.xml`

        $ cat hdfs-site.xml
        ...
        <configuration>
            <property>
                <!-- set it according to the number of slaves -->
                <name>dfs.replication</name>
                <value>2</value>
            </property>
            <property>
                <name>dfs.namenode.name.dir</name>
                <value>file:/opt/hadoop/var/hadoop_data/hdfs/namenode</value>
            </property>
        </configuration>
        ...

* File: `masters`

        $ touch masters
        $ cat /dev/null > masters
        $ echo "master" >> masters

* File: `slaves`

        $ touch slaves
        $ cat /dev/null > slaves
        $ echo "slave1" >> slaves
        $ echo "slave2" >> slaves
        ...
        $ echo "slaveN" >> slaves

* Format the `namenode`.

        $ hadoop namenode -format

* Start the master node of the cluster

        $ start-dfs.sh
        $ start-yarn.sh

### On the slave nodes

> Todo
