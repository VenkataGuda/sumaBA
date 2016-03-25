# sumaBA
Notes
YARN:-
---------

core-site.xml:-
--------------
       <property>
               <name>hadoop.http.staticuser.user</name>
               <value>hdfs</value>
       </property>
which will set the default user name to hdfs.

hdfs-site.xml:-
--------------
<property>
   <name>fs.checkpoint.dir</name>
   <value>file:/var/data/hadoop/hdfs/snn</value>
 </property>
 <property>
   <name>fs.checkpoint.edits.dir</name>
   <value>file:/var/data/hadoop/hdfs/snn</value>
 </property>
 
 yarn-site.xml:-
 ----------------
 <property>
   <name>yarn.nodemanager.aux-services</name>
   <value>mapreduce_shuffle</value>
 </property>
 <property>
   <name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>
   <value>org.apache.hadoop.mapred.ShuffleHandler</value>
 </property>
 
 The yarn.nodemanager.aux-services property tells NodeManagers that there will be an auxiliary service called mapreduce.shuffle that they need to implement. After we tell the NodeManagers to implement that service, we give it a class name as the means to implement that service. This particular configuration tells MapReduce how to do its shuffle. Because NodeManagers won’t shuffle data for a non-MapReduce job by default, we need to configure such a service for MapReduce.
 
 hadoop-env.sh:-
 --------------
 HADOOP_HEAPSIZE="500"
HADOOP_NAMENODE_INIT_HEAPSIZE="500"

mapred-env.sh:-
----------------
HADOOP_JOB_HISTORYSERVER_HEAPSIZE=250

yarn-env.sh:-
------------
JAVA_HEAP_MAX=-Xmx500m

YARN_HEAPSIZE=500

Scheduler:-
-----------

The scheduler class is set in yarn-default.xml. Information about the currently running scheduler can be found by opening the ResourceManager web UI and selecting the Scheduler option under the Cluster menu on the left (e.g., http://your_cluster:8088/cluster/scheduler).


Localization:
--------------
The process of copying or downloading remote resources onto the local file system. Instead of always accessing a resource remotely, that resource is copied to the local machine, so it can then be accessed locally from that point of time.

YARN INSTALLATION:-
-------------------
We describe two methods of installation here: a script-based install and a GUI-based install using Apache Ambari. In both cases, a certain minimum amount of user is input required for successful installation.

1.SYSTEM PREPARATION
Once your system requirements are confirmed and you have downloaded the latest version of Hadoop 2, you will need some information that will make the scripted installation easier. The workhorse of this method is the open-source tool Parallel Distributed Shell (http://sourceforge.net/projects/pdsh), commonly referred to as simply pdsh, which describes itself as “an efficient, multithreaded remote shell client which executes commands on multiple remote hosts in parallel.” In simple terms, pdsh will execute commands remotely on hosts specified either on the command line or in a file. As Hadoop is a distributed system that potentially spans thousands of hosts, pdsh can be a very valuable tool for managing systems. Also included in the pdsh distribution is the pdcp command, which performs distributed copying of files. We’ll use both the pdsh and pdcp commands to install Hadoop 2.

Note

The following procedure describes an RPM (Red Hat)-based installation. The scripts described here are available for download from the book repository (see Appendix A), along with instructions for Ubuntu installation.

Step 1: Install EPEL and pdsh
The pdsh tool is easily installed using an existing RPM package. It is also possible to install pdsh by downloading prebuilt binaries or by compiling the tool from its source files. In most cases, this tool will be available through your system’s existing software installation or update mechanism. For Red Hat–based systems, this is the yum RPM repository; for SUSE systems, it is the zypper RPM repository.

For the purposes of this installation, we will use Red Hat RPM-based system. A version of the pdsh package is distributed in the Extra Packages for Enterprise Linux (EPEL) repository. EPEL has extra packages not in the standard RPM repositories for distributions based on Red Hat Linux. The following steps, performed as root, are needed to install the EPEL repository the pdsh RPM.

Click here to view code image

# rpm -Uvh http://download.fedoraproject.org/pub/epel/6/i386/epel-release-6-8.noarch.rpm
...
# yum install pdsh

Step 2: Generate and Distribute ssh Keys
For pdsh to work effectively, we need to configure “password-less” ssh (secure shell). When pdsh executes remote commands, it will attempt to do so without the need for a password, similar to a user executing an ssh command. The first step is to generate the public and private keys for any user executing pdsh commands (at a minimum, for the root user). On Linux, the OpenSSH package is generally used for this task. This package includes the ssh-keygen command shown later in this subsection. For the easiest installation, as root we execute the ssh-keygen command and accept all the defaults, making sure not to enter a passphrase. If we did specify a passphrase, we would need to enter this passphrase every time we used pdsh or any other tool that accessed the private key.

After generating the keys, we need to copy the public key to the hosts to which we want to log in via ssh without a password. While this might seem like a painstaking task, OpenSSH has the tools to make things easier. The OpenSSH clients package, which is usually installed by default, provides ssh-copy-id, a command that copies a public key to another host and adds the host to the ssh known_hosts file. During an installation using pdsh, we’ll want to use the host’s fully qualified domain name (FQDN), as this is also the hostname we’ll use in Hadoop configuration files.

Click here to view code image

# ssh-keygen -t rsa
...
# ssh-copy-id -i /root/.ssh/id_rsa.pub my.fqdn.tld
...

Once pdsh is installed and password-less ssh is working, the following type of command should be possible. See the pdsh man page for more information.

Click here to view code image

# pdsh –w my.fqdn.tld hostname

One feature of pdsh that will be useful to us is the ability to use host lists. For example, if we create a file called all_hosts and include the FQDNs of all the nodes in the cluster, then pshd can use this list as follows:

Click here to view code image

# pdsh -w ^all_hosts uptime

This feature will be used extensively in the installation script.



2.SCRIPT-BASED INSTALLATION OF HADOOP 2:-
-----------------------------------------
B. YARN Installation Scripts
The following is a listing of the installation scripts discussed in Chapter 5, “Installing Apache Hadoop YARN.” They can be used to help follow the installation discussion. All of the scripts are available from the download page listed in Appendix A.

INSTALL-HADOOP2.SH
Click here to view code image

#!/bin/bash
#
# Install Hadoop 2 using pdsh/pdcp where possible.
#
# Command can be interactive or file-based. This script sets up
# a Hadoop 2 cluster with basic configuration. Modify data, log, and pid
# directories as desired. Further configure your cluster with ./conf-hadoop2.sh
# after running this installation script.
#

# Basic environment variables. Edit as necessary
HADOOP_VERSION=2.2.0
HADOOP_HOME="/opt/hadoop-${HADOOP_VERSION}"
NN_DATA_DIR=/var/data/hadoop/hdfs/nn
SNN_DATA_DIR=/var/data/hadoop/hdfs/snn
DN_DATA_DIR=/var/data/hadoop/hdfs/dn
YARN_LOG_DIR=/var/log/hadoop/yarn
HADOOP_LOG_DIR=/var/log/hadoop/hdfs
HADOOP_MAPRED_LOG_DIR=/var/log/hadoop/mapred
YARN_PID_DIR=/var/run/hadoop/yarn
HADOOP_PID_DIR=/var/run/hadoop/hdfs
HADOOP_MAPRED_PID_DIR=/var/run/hadoop/mapred
HTTP_STATIC_USER=hdfs
YARN_PROXY_PORT=8081

source hadoop-xml-conf.sh
CMD_OPTIONS=$(getopt -n "$0"  -o hif --long "help,interactive,file"  -- "$@")

# Take care of bad options in the command
if [ $? -ne 0 ];
then
  exit 1
fi
eval set -- "$CMD_OPTIONS"

all_hosts="all_hosts"
nn_host="nn_host"
snn_host="snn_host"
dn_hosts="dn_hosts"
rm_host="rm_host"
nm_hosts="nm_hosts"
mr_history_host="mr_history_host"
yarn_proxy_host="yarn_proxy_host"

install()
{
        echo "Copying Hadoop $HADOOP_VERSION to all hosts..."
        pdcp -w ^all_hosts hadoop-"$HADOOP_VERSION".tar.gz /opt

        echo "Copying JDK 1.6.0_31 to all hosts..."
        pdcp -w ^all_hosts jdk-6u31-linux-x64-rpm.bin /opt

        echo "Installing JDK 1.6.0_31 on all hosts..."
        pdsh -w ^all_hosts chmod a+x /opt/jdk-6u31-linux-x64-rpm.bin
        pdsh -w ^all_hosts /opt/jdk-6u31-linux-x64-rpm.bin -noregister 1>&- 2>&-

        echo "Setting JAVA_HOME and HADOOP_HOME environment variables on all hosts..."
        pdsh -w ^all_hosts 'echo export JAVA_HOME=/usr/java/jdk1.6.0_31 > /etc/profile.d/java.sh'
        pdsh -w ^all_hosts "source /etc/profile.d/java.sh"
        pdsh -w ^all_hosts "echo export HADOOP_HOME=$HADOOP_HOME > /etc/profile.d/hadoop.sh"
        pdsh -w ^all_hosts 'echo export HADOOP_PREFIX=$HADOOP_HOME >> /etc/profile.d/hadoop.sh'
        pdsh -w ^all_hosts "source /etc/profile.d/hadoop.sh"

        echo "Extracting Hadoop $HADOOP_VERSION distribution on all hosts..."
        pdsh -w ^all_hosts tar -zxf /opt/hadoop-"$HADOOP_VERSION".tar.gz -C /opt

        echo "Creating system accounts and groups on all hosts..."
        pdsh -w ^all_hosts groupadd hadoop
        pdsh -w ^all_hosts useradd -g hadoop yarn
        pdsh -w ^all_hosts useradd -g hadoop hdfs
        pdsh -w ^all_hosts useradd -g hadoop mapred

        echo "Creating HDFS data directories on NameNode host, Secondary NameNode host, and DataNode hosts..."
        pdsh -w ^nn_host "mkdir -p $NN_DATA_DIR && chown hdfs:hadoop $NN_DATA_DIR"
        pdsh -w ^snn_host "mkdir -p $SNN_DATA_DIR && chown hdfs:hadoop $SNN_DATA_DIR"
        pdsh -w ^dn_hosts "mkdir -p $DN_DATA_DIR && chown hdfs:hadoop $DN_DATA_DIR"

        echo "Creating log directories on all hosts..."
        pdsh -w ^all_hosts "mkdir -p $YARN_LOG_DIR && chown yarn:hadoop $YARN_LOG_DIR"
        pdsh -w ^all_hosts "mkdir -p $HADOOP_LOG_DIR && chown hdfs:hadoop $HADOOP_LOG_DIR"
        pdsh -w ^all_hosts "mkdir -p $HADOOP_MAPRED_LOG_DIR && chown mapred:hadoop $HADOOP_MAPRED_LOG_DIR"

        echo "Creating pid directories on all hosts..."
        pdsh -w ^all_hosts "mkdir -p $YARN_PID_DIR && chown yarn:hadoop $YARN_PID_DIR"
        pdsh -w ^all_hosts "mkdir -p $HADOOP_PID_DIR && chown hdfs:hadoop $HADOOP_PID_DIR"
        pdsh -w ^all_hosts "mkdir -p $HADOOP_MAPRED_PID_DIR && chown mapred:hadoop $HADOOP_MAPRED_PID_DIR"

        echo "Editing Hadoop environment scripts for log directories on all hosts..."
        pdsh -w ^all_hosts echo "export HADOOP_LOG_DIR=$HADOOP_LOG_DIR >> $HADOOP_HOME/etc/hadoop/hadoop-env.sh"
        pdsh -w ^all_hosts echo "export YARN_LOG_DIR=$YARN_LOG_DIR >> $HADOOP_HOME/etc/hadoop/yarn-env.sh"
        pdsh -w ^all_hosts echo "export HADOOP_MAPRED_LOG_DIR=$HADOOP_MAPRED_LOG_DIR >> $HADOOP_HOME/etc/hadoop/mapred-env.sh"

        echo "Editing Hadoop environment scripts for pid directories on all hosts..."
        pdsh -w ^all_hosts echo "export HADOOP_PID_DIR=$HADOOP_PID_DIR >> $HADOOP_HOME/etc/hadoop/hadoop-env.sh"
        pdsh -w ^all_hosts echo "export YARN_PID_DIR=$YARN_PID_DIR >> $HADOOP_HOME/etc/hadoop/yarn-env.sh"
        pdsh -w ^all_hosts echo "export HADOOP_MAPRED_PID_DIR=$HADOOP_MAPRED_PID_DIR >> $HADOOP_HOME/etc/hadoop/mapred-env.sh"

        echo "Creating base Hadoop XML config files..."
        create_config --file core-site.xml
        put_config --file core-site.xml --property fs.default.name --value "hdfs://$nn:9000"
        put_config --file core-site.xml --property hadoop.http.staticuser.user --value "$HTTP_STATIC_USER"

        create_config --file hdfs-site.xml
        put_config --file hdfs-site.xml --property dfs.namenode.name.dir --value "$NN_DATA_DIR"
        put_config --file hdfs-site.xml --property fs.checkpoint.dir --value "$SNN_DATA_DIR"
        put_config --file hdfs-site.xml --property fs.checkpoint.edits.dir --value "$SNN_DATA_DIR"
        put_config --file hdfs-site.xml --property dfs.datanode.data.dir --value "$DN_DATA_DIR"
        put_config --file hdfs-site.xml --property dfs.namenode.http-address --value "$nn:50070"
        put_config --file hdfs-site.xml --property dfs.namenode.secondary.http-address --value "$snn:50090"

        create_config --file mapred-site.xml
        put_config --file mapred-site.xml --property mapreduce.framework.name --value yarn
        put_config --file mapred-site.xml --property mapreduce.jobhistory.address --value "$mr_hist:10020"
        put_config --file mapred-site.xml --property mapreduce.jobhistory.webapp.address --value "$mr_hist:19888"
        put_config --file mapred-site.xml --property yarn.app.mapreduce.am.staging-dir --value /mapred

        create_config --file yarn-site.xml
        put_config --file yarn-site.xml --property yarn.nodemanager.aux-services --value mapreduce.shuffle
        put_config --file yarn-site.xml --property yarn.nodemanager.aux-services.mapreduce.shuffle.class --value org.apache.hadoop.mapred.ShuffleHandler
        put_config --file yarn-site.xml --property yarn.web-proxy.address --value "$yarn_proxy:$YARN_PROXY_PORT"
        put_config --file yarn-site.xml -–property yarn.resourcemanager.scheduler.address --value "$rmgr:8030"
        put_config --file yarn-site.xml --property yarn.resourcemanager.resource-tracker.address --value "$rmgr:8031"
        put_config --file yarn-site.xml --property yarn.resourcemanager.address --value "$rmgr:8032"
        put_config --file yarn-site.xml --property yarn.resourcemanager.admin.address --value "$rmgr:8033"
        put_config --file yarn-site.xml --property yarn.resourcemanager.webapp.address --value "$rmgr:8088"

        echo "Copying base Hadoop XML config files to all hosts..."
        pdcp -w ^all_hosts core-site.xml hdfs-site.xml mapred-site.xml yarn-site.xml $HADOOP_HOME/etc/hadoop/

        echo "Creating configuration, command, and script links on all hosts..."
        pdsh -w ^all_hosts "ln -s $HADOOP_HOME/etc/hadoop /etc/hadoop"
        pdsh -w ^all_hosts "ln -s $HADOOP_HOME/bin/* /usr/bin"
        pdsh -w ^all_hosts "ln -s $HADOOP_HOME/libexec/* /usr/libexec"

        echo "Formatting the NameNode..."
        pdsh -w ^nn_host "su - hdfs -c '$HADOOP_HOME/bin/hdfs namenode -format'"

        echo "Copying startup scripts to all hosts..."
        pdcp -w ^nn_host hadoop-namenode /etc/init.d/
        pdcp -w ^snn_host hadoop-secondarynamenode /etc/init.d/
        pdcp -w ^dn_hosts hadoop-datanode /etc/init.d/
        pdcp -w ^rm_host hadoop-resourcemanager /etc/init.d/
        pdcp -w ^nm_hosts hadoop-nodemanager /etc/init.d/
        pdcp -w ^mr_history_host hadoop-historyserver /etc/init.d/
        pdcp -w ^yarn_proxy_host hadoop-proxyserver /etc/init.d/

        echo "Starting Hadoop $HADOOP_VERSION services on all hosts..."
        pdsh -w ^nn_host "chmod 755 /etc/init.d/hadoop-namenode && chkconfig hadoop-namenode on && service hadoop-namenode start"
        pdsh -w ^snn_host "chmod 755 /etc/init.d/hadoop-secondarynamenode && chkconfig hadoop-secondarynamenode on && service hadoop-secondarynamenode start"
        pdsh -w ^dn_hosts "chmod 755 /etc/init.d/hadoop-datanode && chkconfig hadoop-datanode on && service hadoop-datanode start"
        pdsh -w ^rm_host "chmod 755 /etc/init.d/hadoop-resourcemanager && chkconfig hadoop-resourcemanager on && service hadoop-resourcemanager start"
        pdsh -w ^nm_hosts "chmod 755 /etc/init.d/hadoop-nodemanager && chkconfig hadoop-nodemanager on && service hadoop-nodemanager start"

        pdsh -w ^yarn_proxy_host "chmod 755 /etc/init.d/hadoop-proxyserver && chkconfig hadoop-proxyserver on && service hadoop-proxyserver start"

        echo "Creating MapReduce Job History directories..."
        su - hdfs -c "hadoop fs -mkdir -p /mapred/history/done_intermediate"
        su - hdfs -c "hadoop fs -chown -R mapred:hadoop /mapred"
        su - hdfs -c "hadoop fs -chmod -R g+rwx /mapred"

        pdsh -w ^mr_history_host "chmod 755 /etc/init.d/hadoop-historyserver && chkconfig hadoop-historyserver on && service hadoop-historyserver start"

        echo "Running YARN smoke test..."
        pdsh -w ^all_hosts "usermod -a -G hadoop $(whoami)"
        su - hdfs -c "hadoop fs -mkdir -p /user/$(whoami)"
        su - hdfs -c "hadoop fs -chown $(whoami):$(whoami) /user/$(whoami)"
        source /etc/profile.d/java.sh
        source /etc/profile.d/hadoop.sh
        source /etc/hadoop/hadoop-env.sh
        source /etc/hadoop/yarn-env.sh
        hadoop jar $HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-examples-$HADOOP_VERSION.jar pi -Dmapreduce.clientfactory.class.name=org.apache.hadoop.mapred.YarnClientFactory -libjars $HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-client-jobclient-$HADOOP_VERSION.jar 16 10000
}

interactive()
{
        echo -n "Enter NameNode hostname: "
        read nn
        echo -n "Enter Secondary NameNode hostname: "
        read snn
        echo -n "Enter ResourceManager hostname: "
        read rmgr
        echo -n "Enter Job History Server hostname: "
        read mr_hist
        echo -n "Enter YARN Proxy hostname: "
        read yarn_proxy
        echo -n "Enter DataNode hostnames (comma-separated or hostlist syntax): "
        read dns
        echo -n "Enter NodeManager hostnames (comma-separated or hostlist syntax): "
        read nms

        echo "$nn" > "$nn_host"
        echo "$snn" > "$snn_host"
        echo "$rmgr" > "$rm_host"
        echo "$mr_hist" > "$mr_history_host"
        echo "$yarn_proxy" > "$yarn_proxy_host"
        dn_hosts_var=$(sed 's/\,/\n/g' <<< $dns)
        nm_hosts_var=$(sed 's/\,/\n/g' <<< $nms)
        echo "$dn_hosts_var" > "$dn_hosts"
        echo "$nm_hosts_var" > "$nm_hosts"
        echo "$(echo "$nn $snn $rmgr $mr_hist $yarn_proxy $dn_hosts_var $nm_hosts_var" | tr ' ' '\n' | sort -u)" > "$all_hosts"
}

file()
{
        nn=$(cat nn_host)
        snn=$(cat snn_host)
        rmgr=$(cat rm_host)
        mr_hist=$(cat mr_history_host)
        yarn_proxy=$(cat yarn_proxy_host)
        dns=$(cat dn_hosts)
        nms=$(cat nm_hosts)

        echo "$(echo "$nn $snn $rmgr $mr_hist $dns $nms" | tr ' ' '\n' | sort -u)" > "$all_hosts"
}

help()
{
cat << EOF
install-hadoop2.sh

This script installs Hadoop 2 with basic data, log, and pid directories.

USAGE:  install-hadoop2.sh [options]

OPTIONS:
   -i, --interactive      Prompt for fully qualified domain names (FQDN) of the NameNode,
                          Secondary NameNode, DataNodes, ResourceManager, NodeManagers,
                          MapReduce Job History Server, and YARN Proxy server. Values
                          entered are stored in files in the same directory as this command.

   -f, --file             Use files with fully qualified domain names (FQDN), newline
                          separated. Place files in the same directory as this script.
                          Services and file name are as follows:
                          NameNode = nn_host
                          Secondary NameNode = snn_host
                          DataNodes = dn_hosts
                          ResourceManager = rm_host
                          NodeManagers = nm_hosts
                          MapReduce Job History Server = mr_history_host
                          YARN Proxy Server = yarn_proxy_host

   -h, --help             Show this message.

EXAMPLES:
   Prompt for host names:
     install-hadoop2.sh -i
     install-hadoop2.sh --interactive

   Use values from files in the same directory:
     install-hadoop2.sh -f
     install-hadoop2.sh --file

EOF
}

while true;
do
  case "$1" in

    -h|--help)
      help
      exit 0
      ;;
    -i|--interactive)
      interactive
      install
      shift
      ;;
    -f|--file)
      file
      install
      shift
      ;;
    --)
      shift
      break
      ;;
  esac
done

UNINSTALL-HADOOP2.SH
Click here to view code image

#!/bin/bash

HADOOP_VERSION=2.0.5-alpha
HADOOP_HOME="/opt/hadoop-${HADOOP_VERSION}"
NN_DATA_DIR=/var/data/hadoop/hdfs/nn
SNN_DATA_DIR=/var/data/hadoop/hdfs/snn
DN_DATA_DIR=/var/data/hadoop/hdfs/dn
YARN_LOG_DIR=/var/log/hadoop/yarn
HADOOP_LOG_DIR=/var/log/hadoop/hdfs
HADOOP_MAPRED_LOG_DIR=/var/log/hadoop/mapred

echo "Stopping Hadoop 2 services..."
pdsh -w ^dn_hosts "service hadoop-datanode stop"
pdsh -w ^snn_host "service hadoop-secondarynamenode stop"
pdsh -w ^nn_host "service hadoop-namenode stop"
pdsh -w ^mr_history_host "service hadoop-historyserver stop"
pdsh -w ^yarn_proxy_host "service hadoop-proxyserver stop"
pdsh -w ^nm_hosts "service hadoop-nodemanager stop"
pdsh -w ^rm_host "service hadoop-resourcemanager stop"

echo "Removing Hadoop 2 services from run levels..."
pdsh -w ^dn_hosts "chkconfig --del hadoop-datanode"
pdsh -w ^snn_host "chkconfig --del hadoop-secondarynamenode"
pdsh -w ^nn_host "chkconfig --del hadoop-namenode"
pdsh -w ^mr_history_host "chkconfig --del hadoop-historyserver"
pdsh -w ^yarn_proxy_host "chkconfig --del hadoop-proxyserver"
pdsh -w ^nm_hosts "chkconfig --del hadoop-nodemanager"
pdsh -w ^rm_host "chkconfig --del hadoop-resourcemanager"

echo "Removing Hadoop 2 startup scripts..."
pdsh -w ^all_hosts "rm -f /etc/init.d/hadoop-*"

echo "Removing Hadoop 2 distribution tarball..."
pdsh -w ^all_hosts "rm -f /opt/hadoop-2*.tar.gz"

echo "Removing JDK 1.6.0_31 distribution..."
pdsh -w ^all_hosts "rm -f /opt/jdk*"

echo "Removing JDK 1.6.0_31 artifacts..."
pdsh -w ^all_hosts "rm -f sun-java*"
pdsh -w ^all_hosts "rm -f jdk*"

echo "Removing Hadoop 2 home directory..."
pdsh -w ^all_hosts "rm -Rf $HADOOP_HOME"

echo "Removing Hadoop 2 bash environment setting..."
pdsh -w ^all_hosts "rm -f /etc/profile.d/hadoop.sh"

echo "Removing Java bash environment setting..."
pdsh -w ^all_hosts "rm -f /etc/profile.d/java.sh"

echo "Removing /etc/hadoop link..."
pdsh -w ^all_hosts "unlink /etc/hadoop"

echo "Removing Hadoop 2 command links..."
pdsh -w ^all_hosts "unlink /usr/bin/container-executor"
pdsh -w ^all_hosts "unlink /usr/bin/hadoop"
pdsh -w ^all_hosts "unlink /usr/bin/hdfs"
pdsh -w ^all_hosts "unlink /usr/bin/mapred"
pdsh -w ^all_hosts "unlink /usr/bin/rcc"
pdsh -w ^all_hosts "unlink /usr/bin/test-container-executor"
pdsh -w ^all_hosts "unlink /usr/bin/yarn"

echo "Removing Hadoop 2 script links..."
pdsh -w ^all_hosts "unlink /usr/libexec/hadoop-config.sh"
pdsh -w ^all_hosts "unlink /usr/libexec/hdfs-config.sh"
pdsh -w ^all_hosts "unlink /usr/libexec/httpfs-config.sh"
pdsh -w ^all_hosts "unlink /usr/libexec/mapred-config.sh"
pdsh -w ^all_hosts "unlink /usr/libexec/yarn-config.sh"

echo "Uninstalling JDK 1.6.0_31 RPM..."
pdsh -w ^all_hosts "rpm -ev jdk-1.6.0_31-fcs.x86_64"

echo "Removing NameNode data directory..."
pdsh -w ^nn_host "rm -Rf $NN_DATA_DIR"

echo "Removing Secondary NameNode data directory..."
pdsh -w ^snn_host "rm -Rf $SNN_DATA_DIR"

echo "Removing DataNode data directories..."
pdsh -w ^dn_hosts "rm -Rf $DN_DATA_DIR"

echo "Removing YARN log directories..."
pdsh -w ^all_hosts "rm -Rf $YARN_LOG_DIR"

echo "Removing HDFS log directories..."
pdsh -w ^all_hosts "rm -Rf $HADOOP_LOG_DIR"

echo "Removing MapReduce log directories..."
pdsh -w ^all_hosts "rm -Rf $HADOOP_MAPRED_LOG_DIR"

echo "Removing HDFS account..."
pdsh -w ^all_hosts "userdel -r hdfs"

echo "Removing MapReduce system account..."
pdsh -w ^all_hosts "userdel -r mapred"

echo "Removing YARN system account..."
pdsh -w ^all_hosts "userdel -r yarn"

echo "Removing Hadoop system group..."
pdsh -w ^all_hosts "groupdel hadoop"

HADOOP-XML-CONF.SH
Click here to view code image

#!/bin/bash
#
# Utility functions for processing Hadoop 2 XML configuration files.
#
# Depends on Python built-in XML processing and libxml2 for formatting.
#

installed=false
if [ -f /etc/profile.d/hadoop.sh ]; then
    source /etc/profile.d/hadoop.sh
    source $HADOOP_HOME/etc/hadoop/hadoop-env.sh
    installed=true
fi


create_config()
{
        local filename=

        case $1 in
            '')    echo $"$0: Usage: create_config --file"
                   return 1;;
            --file)
                   filename=$2
                   ;;
        esac

        python - <<END
from xml.etree import ElementTree
from xml.etree.ElementTree import Element

conf = Element('configuration')

conf_file = open("$filename",'w')
conf_file.write(ElementTree.tostring(conf))
conf_file.close()
END
        write_file $filename
}

put_config()
{
        local filename= property= value=

        while [ "$1" != "" ]; do
        case $1 in
            '')    echo $"$0: Usage: put_config --file --property --value"
                   return 1;;
            --file)
                   filename=$2
                   shift 2
                   ;;
            --property)
                   property=$2
                   shift 2
                   ;;
            --value)
                   value=$2
                   shift 2
                   ;;
        esac
        done

        python - <<END
from xml.etree import ElementTree
from xml.etree.ElementTree import Element
from xml.etree.ElementTree import SubElement

def putconfig(root, name, value):
        for existing_prop in root.getchildren():
                if existing_prop.find('name').text == name:
                        root.remove(existing_prop)
                        break
        property = SubElement(root, 'property')
        name_elem = SubElement(property, 'name')
        name_elem.text = name
        value_elem = SubElement(property, 'value')
        value_elem.text = value

path = ''
if "$installed" == 'true':
        path = "$HADOOP_CONF_DIR" + '/'

conf = ElementTree.parse(path + "$filename").getroot()
putconfig(root = conf, name = "$property", value = "$value")

conf_file = open("$filename",'w')
conf_file.write(ElementTree.tostring(conf))
conf_file.close()
END
        write_file $filename
}

del_config()
{
        local filename= property=

        while [ "$1" != "" ]; do
        case $1 in
            '')    echo $"$0: Usage: del_config --file --property"
                   return 1;;
            --file)
                   filename=$2
                   shift 2
                   ;;
            --property)
                   property=$2
                   shift 2
                   ;;
        esac
        done

        python - <<END
from xml.etree import ElementTree
from xml.etree.ElementTree import Element
from xml.etree.ElementTree import SubElement

def delconfig(root, name):
        for existing_prop in root.getchildren():
                if existing_prop.find('name').text == name:
                        root.remove(existing_prop)
                        break

path = ''
if "$installed" == 'true':
        path = "$HADOOP_CONF_DIR" + '/'

conf = ElementTree.parse(path + "$filename").getroot()
delconfig(root = conf, name = "$property")

conf_file = open("$filename",'w')
conf_file.write(ElementTree.tostring(conf))
conf_file.close()
END
        write_file $filename
}

write_file()
{
        local file=$1
        xmllint --format "$file" > "$file".pp && mv "$file".pp "$file"
}


JDK Options
There are two options for installing a Java JDK. The first is to install the OpenJDK that is part of the distribution. You can test whether the OpenJDK is installed by issuing the following command. If the OpenJDK is installed, the packages will be listed.

Click here to view code image

# rpm -qa|grep jdk
java-1.6.0-openjdk-devel-1.6.0.0-1.62.1.11.11.90.el6_4.x86_64
java-1.6.0-openjdk-1.6.0.0-1.62.1.11.11.90.el6_4.x86_64

Both the base and devel versions should be installed. The other recommended JVM is jdk-6u31-linux-x64-rpm.bin from Oracle. This version can be downloaded from http://www.oracle.com/technetwork/java/javasebusiness/downloads/java-archive-downloads-javase6-419409.html.

If you choose to use the Oracle version, make sure you have removed the OpenJDK from all your systems using the following command:

Click here to view code image

# rpm –e java-1.6.0-openjdk-devel java-1.6.0-openjdk-devel

You may need to add the “--nodeps” option to remove the packages. Keep in mind that if there are dependencies on the OpenJDK, you may need to change some settings to use the Oracle JDK.

Step 1: Download and Extract the Scripts
The install scripts and supporting files are available from the book repository (see Appendix A). You can simply use wget to pull down the hadoop2-install-scripts.tgz file.

As root, extract the file move to the hadoop2-install-scripts working directory:

Click here to view code image

# tar xvzf hadoop2-install-scripts.tgz
# cd hadoop2-install-scripts

If you have not done so already, download your Hadoop version and place it in this directory (do not extract it). Also, if you are using the Oracle SDK, place it in this directory as well.

Step 2: Set the Script Variables
The main script is called install-hadoop2.sh. The following is a list of the user-defined variables that appear at the beginning of this script. You will want to make sure your version matches the HADOOP_VERSION variable. You can also change the install path by changing HADOOP_HOME (in this case, the path is /opt). Next, there are various directories that are used by the various services. The following example assumes that /var has sufficient capacity to hold all the Hadoop 2 data and log files. These paths can also be changed to suit your hardware. It is a good idea to keep the default values for HTTP_STATIC_USER and YARN_PROXY_PORT. Finally, JAVA_HOME needs to be defined. If you are using the OpenJDK, make sure this definition corresponds to the OpenJDK path. If you are using the Oracle JDK, then download jdk-6u31-linux-x64-rpm.bin to this directly and define JAVA_HOME as empty: JAVA_HOME="".

Click here to view code image

# Basic environment variables. Edit as necessary
HADOOP_VERSION=2.2.0
HADOOP_HOME="/opt/hadoop-${HADOOP_VERSION}"
NN_DATA_DIR=/var/data/hadoop/hdfs/nn
SNN_DATA_DIR=/var/data/hadoop/hdfs/snn
DN_DATA_DIR=/var/data/hadoop/hdfs/dn
YARN_LOG_DIR=/var/log/hadoop/yarn
HADOOP_LOG_DIR=/var/log/hadoop/hdfs
HADOOP_MAPRED_LOG_DIR=/var/log/hadoop/mapred
YARN_PID_DIR=/var/run/hadoop/yarn
HADOOP_PID_DIR=/var/run/hadoop/hdfs
HADOOP_MAPRED_PID_DIR=/var/run/hadoop/mapred
HTTP_STATIC_USER=hdfs
YARN_PROXY_PORT=8081
# If using local OpenJDK, it must be installed on all nodes.
# If using jdk-6u31-linux-x64-rpm.bin, then
# set JAVA_HOME="" and place jdk-6u31-linux-x64-rpm.bin in this directory
JAVA_HOME=/usr/lib/jvm/java-1.6.0-openjdk-1.6.0.0.x86_64/

Step 3: Provide Node Names
Once you have set the options, it is time to perform the installation. Keep in mind that the script relies heavily on pdsh and pdcp. If these two commands are not working across the cluster, then the installation procedure will not work. You can get help for the script by running

Click here to view code image

# ./install-hadoop2.sh -h

The script offers two options: file based or interactive. In file-based mode, the script needs the following list of files with the appropriate node names for your cluster:

Image nn_host: HDFS NameNode hostname

Image rm_host: YARN ResourceManager hostname

Image snn_host: HDFS SecondaryNameNode hostname

Image mr_history_host: MapReduce Job History server hostname

Image yarn_proxy_host: YARN Web Proxy hostname

Image nm_hosts: YARN NodeManager hostnames

Image dn_hosts: HDFS DataNode hostnames, separated by a space

If you use the interactive method, the files will be created automatically. If you choose the file-based method, you can edit the files yourself. With exception of nm_hosts and dn_hosts, all of the files require one hostname. Depending on your installation, some of these hosts may be the same physical machine. The nm_hosts and dn_hosts files take a space-delimited list of hostnames, which will identify HDFS data nodes and/or YARN worker nodes.

Step 4: Run the Script
After you have checked over everything, you can run the script as follows, using tee to keep a record of the install:

Click here to view code image

# ./install-hadoop2.sh –f |tee install-hadoop2-results

Some steps may take longer than others. If everything worked correctly, a MapReduce job (the classic “calculate pi” example) will be run. If it is successful, the installation process is complete. You may wish to install other tools like Pig, Hive, or HBase as well. For your reference, the script does the following:

1. Copies the Hadoop tar file to all hosts.

2. Optionally copies and installs Oracle JDK 1.6.0_31 to all hosts.

3. Sets the JAVA_HOME and HADOOP_HOME environment variables on all hosts.

4. Extracts the Hadoop distribution on all hosts.

5. Creates system accounts and groups on all hosts (Group: hadoop, Users: yarn, hdfs, and mapred).

6. Creates HDFS data directories on the NameNode host, SecondaryNameNode host, and DataNode hosts.

7. Creates log directories on all hosts.

8. Creates pid directories on all hosts.

9. Edits Hadoop environment scripts for log directories on all hosts.

10. Edits Hadoop environment scripts for pid directories on all hosts.

11. Creates the base Hadoop XML config files (core-site.XML, hdfs-site.XML, mapred-site.XML, yarn-site.XML).

12. Copies the base Hadoop XML config files to all hosts.

13. Creates configuration, command, and script links on all hosts.

14. Formats the NameNode.

15. Copies start-up scripts to all hosts (hadoop-datanode, hadoop-historyserver, hadoop-namenode, hadoop-nodemanager, hadoop-proxyserver, hadoop-resourcemanager, and hadoop-secondarynamenode).

16. Starts Hadoop services on all hosts.

17. Creates MapReduce Job History directories.

18. Runs the YARN pi MapReduce job.

Step 5: Verify the Installation
There are a few points in the script-based installation process where problems may occur. One important step is formatting the NameNode (Step 14 in the preceding list). The results of this command will show up in the script output. Make sure there were no errors with this command.

If you note other errors, such as when starting the Hadoop daemons, check the log files located under $HADOOP_HOME/logs. The final part of the script runs the following example “pi” MapReduce job command.

Click here to view code image

hadoop jar \
$HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-examples-$HADOOP_VERSION.jar \
pi -Dmapreduce.clientfactory.class.name=org.apache.hadoop.mapred.YarnClientFactory \
-libjars $HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-client-jobclient-$HADOOP_VERSION.jar \
16 10000

If this test is successful, you should see the following at the end of the output. (Your run-time time will vary.)

Click here to view code image

Job Finished in 25.854 seconds
Estimated value of Pi is 3.14127500000000000000

You can also examine the HDFS file system using the following command:

Click here to view code image

# hdfs dfs -ls /
Found 6 items
drwxr-xr-x   - hdfs   hdfs            0 2013-02-06 21:17 /apps
drwxr-xr-x   - hdfs   hadoop          0 2013-02-06 22:26 /benchmarks
drwx------   - mapred hdfs            0 2013-04-25 16:20 /mapred
drwxr-xr-x   - hdfs   hdfs            0 2013-05-16 11:53 /system
drwxrwxr--   - hdfs   hadoop          0 2013-05-14 14:54 /tmp
drwxrwxr-x   - hdfs   hadoop          0 2013-04-26 02:00 /user

Similar to Hadoop version 1, many aspects of Hadoop version 2 are available via the built-in web UI. The web UI can be accessed by directly accessing the following URL in your favorite browser or by issuing the following command as the user hdfs (using your local hostname):

Click here to view code image

$ firefox http://hostname:8088/cluster

If the HDFS file system looks fine and there are no dead nodes, then your Hadoop cluster should be fully functional. You can investigate other aspects of your Hadoop 2 cluster by exploring some of the links on both the HDFS and YARN UI pages.

SCRIPT-BASED UNINSTALL
To uninstall the Hadoop2 installation, use the uninstall-hadoop2.sh script. Make sure any changes you made to the install-hadoop2.sh script, such as the Hadoop version number (HADOOP_VERSION), are copied to the uninstall script.

CONFIGURATION FILE PROCESSING
The install script provides some commands that you will find useful for processing Hadoop XML files. If you examine the install-hadoop2.sh script, you should notice that several commands are used to create the Hadoop XML configuration files:

Click here to view code image

create_config() --file filename
put_config --file filename --property-property pname --value pvalue
del_config()--file filename --property-property pname

where filename is the name of the XML file, pname is the property to be defined, and pvalue is the actual value. The functions are defined in hadoop-xml-conf.sh, which is part of the hadoop2-install-scripts.tgz archive. See the book repository (Appendix A) for download instructions. Basically, these scripts facilitate the creation of Hadoop XML configuration files. If you want to customize the installation or make changes to the XML files, these scripts may be useful.

CONFIGURATION FILE SETTINGS
The following is a brief description of the settings specified in the Hadoop configuration files by the installation script. These files are located in $HADOOP_HOME/etc/hadoop.

core-site.xml
In this file, we define two essential properties for the entire system.

Click here to view code image

hdfs://$nn:9000 --> fs.default.name
$HTTP_STATIC_USER --> hadoop.http.staticuser.user

First, we define the name of the default file system we wish to use. Because we are using HDFS, we will set this value to hdfs://$nn:9000 ($nn is the NameNode we specified in the script and 9000 is the standard HDFS port). Next we add the hadoop.http.staticuser.user (hdfs) that we defined in the install script. This login is used as the default user for the built-in web user interfaces.

hdfs-site.xml
The hdfs-site.xml file holds information about the Hadoop HDFS file system. Most of these values were set at the beginning of the script. They are copied as follows:

Click here to view code image

$NN_DATA_DIR --> dfs.namenode.name.dir
$SNN_DATA_DIR --> fs.checkpoint.dir
$SNN_DATA_DIR --> fs.checkpoint.edits.dir
$DN_DATA_DIR --> dfs.datanode.data.dir

The remaining two values are set to the standard default port numbers ($nn is the NameNode and $snn is the SecondaryNameNode we input to the script):

Click here to view code image

$nn:50070 --> dfs.namenode.http-address
$snn:50090 --> dfs.namenode.secondary.http-address

mapred-site.xml
Users who are familiar with Hadoop version 1 may notice that this is a known configuration file. Given that MapReduce is just another YARN framework, it needs its own configuration file. The script specifies the following settings:

Click here to view code image

yarn --> mapreduce.framework.name
$mr_hist:10020 --> mapreduce.jobhistory.address
$mr_hist:19888 --> mapreduce.jobhistory.webapp.address
/mapred --> yarn.app.mapreduce.am.staging-dir

The first property is mapreduce.framework.name. For this property, there are three valid values: local (default), classic, or yarn. Specifying “local” for this value means that the MapReduce Application is run locally in a process on the client machine, without using any cluster resources. This local process will execute the map and reduce tasks for a given job; because it is local, it doesn’t need to shuffle data from map task output on one server to reduce task input on another server. Typically, this means that there will be one map task and one reduce task.

Specifying “classic” for the mapreduce.framework.name property is appropriate when there is a Hadoop 1.x JobTracker running in your cluster where Hadoop can submit the job. This property exists to accommodate unforeseen situations where there are backward-compatibility problems with users’ MapReduce jobs and the need for a classic job submission process to a JobTracker is unavoidable.

As we are interested in using yarn, we will set mapreduce.framework.name to “yarn” and use the new MapReduce framework provided by YARN.

The next two properties, mapreduce.jobhistory.address and mapreduce.jobhistory.webapp.address, may seem similar but have some subtle differences. The mapreduce.jobhistory.address property is the host and port where the MapReduce application will send job history via its own internal protocol. The mapreduce.jobhistory.webapp.address is where an administrator or a user can view the details of MapReduce jobs that have completed.

Finally, we specify a property for a MapReduce staging directory. When a Map-Reduce job is submitted to YARN, the MapReduce ApplicationMaster will create temporary data in HDFS for the job and will need a staging area for this data. The property yarn.app.mapreduce.am.staging-dir is where we designate such a directory in HDFS (i.e., /mapred).

This staging area will also be used by the job history server. The installation script will make sure that the proper permissions and subdirectories are created (i.e., /mapred/history/done_intermediate).

yarn-site.xml
The final configuration file is yarn-site.xml. The script sets the following values:

Click here to view code image

mapreduce.shuffle --> yarn.nodemanager.aux-services
org.apache.hadoop.mapred.ShuffleHandler --> yarn.nodemanager.aux-services.mapreduce.shuffle.class
$yarn_proxy:$YARN_PROXY_PORT --> yarn.web-proxy.address
$rmgr:8030 --> yarn.resourcemanager.scheduler.address
$rmgr:8031 --> yarn.resourcemanager.resource-tracker.address
$rmgr:8032 --> yarn.resourcemanager.address
$rmgr:8033 --> yarn.resourcemanager.admin.address
$rmgr:8088 --> yarn.resourcemanager.webapp.address

The yarn.nodemanager.aux-services property tells the NodeManager that a MapReduce container will have to do a shuffle from the map tasks to the reduce tasks. Previously, the shuffle step was part of the monolithic MapReduce TaskTracker. With YARN, the shuffle is an auxiliary service and must be set in the configuration file. In addition, the yarn.nodemanager.aux-services.mapreduce.shuffle.class property tells YARN which class to use to do the actual shuffle. The class we use for the shuffle handler is org.apache.hadoop.mapred.ShuffleHandler. Although it’s possible to write your own shuffle handler by extending this class, it is recommended that the default class be used.

The next property is the yarn.web-proxy.address. This property is part of the installation process because we decided to run the YARN Proxy Server as a separate process. If we didn’t configure it this way, the Proxy Server would run as part of the ResourceManager process. The Proxy Server aims to lessen the possibility of security issues. An ApplicationMaster will send to the ResourceManager a link for the application’s web UI but, in reality, this link can point anywhere. The YARN Proxy Server lessens the risk associated with this link, but it doesn’t eliminate it.

The remaining settings are the default ResourceManager port addresses.

START-UP SCRIPTS
The Hadoop distribution includes a lot of convenient scripts to start and stop services such as the ResourceManager and NodeManagers. In a production cluster, however, we want the ability to integrate our scripts with the system’s services management. On most Linux systems today, that means integrating with the init system.

Instead of relying on the built-in scripts that ship with Hadoop, we provide a set of init scripts that can be placed in /etc/init.d and used to start, stop, and monitor the Hadoop services. The files are as follows, with each service being identified by name:

hadoop-namenode
hadoop-datanode
hadoop-secondarynamenode
hadoop-resourcemanager
hadoop-nodemanager
hadoop-proxyserver
hadoop-historyserver

Of course, not all services run on all nodes; thus the installation script places the correct files on the requisite nodes. Before starting the service, the script also registers each service with chkconfig. Once these services are installed, they can be easily managed with commands like the following:

Click here to view code image

# service hadoop-namenode start
# hadoop-resourcemanager status
# hadoop-proxyserver restart
# hadoop-historyserver stop












