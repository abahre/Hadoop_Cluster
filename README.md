# Hadoop_Cluster
This repository contains all the steps required to  create a hadoop cluster along with enabling kerberos on cluster

->Fix Swappiness

cat /proc/sys/vm/swappiness
sudo sysctl -w vm.swappiness=1
cat /proc/sys/vm/swappiness
sudo vi /etc/sysctl.conf


#add this line (vm.swappiness=1)


#Disable firewall and selinux

sudo /etc/init.d/iptables stop
sudo chkconfig iptables off
sudo chkconfig ip6tables off
sudo nano /etc/selinux/config
--> change "enforcing" to "permissive" or "disabled"

#Disable transparent hugepage support

sudo vi /etc/grub.conf
--> add "transparent_hugepage=never" to the kernel 

sudo reboot

#check that it worked
cat /sys/kernel/mm/transparent_hugepage/enabled (always madvise [never])


#Set ulimits for common users

sudo nano /etc/security/limits.d/90-nproc.conf

#change the value 1024 on category (*) to 4096
#logout and login

#check to see that the limit has been set

ulimit -u (should respond with 4096)
ulimit -n (should respond with 1024)


#Install ntpd and nscd

sudo yum install -y ntp
sudo chkconfig ntpd on
sudo service ntpd start
sudo service ntpd status

sudo yum install -y nscd
sudo service nscd start
sudo chkconfig nscd on

# on all nodes, download mysql rpm package

wget https://dev.mysql.com/get/Downloads/MySQL-server-5.5.55-1.el6.x86_64.rpm
sudo rpm -Uvh MySQL-server-5.5.55-1.el6.x86_64.rpm
sudo yum -y update

# download the mysql jdbc connector

wget https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-5.1.42.tar.gz
tar zxvf mysql-connector-java-5.1.42.tar.gz
sudo mkdir -p /usr/share/java
sudo cp mysql-connector-java-5.1.42/mysql-connector-java-5.1.42-bin.jar /usr/share/java/mysql-connector-java.jar

#on the server and possible replica nodes
	~sudo yum install -y mysql-server (1 replica node)

#start the mysql server

sudo service mysqld start
sudo /usr/bin/mysql_secure_installation
current password: (just press "Enter" for fresh installations)
set Root Pasword: Y
enter new pass: *******
reenter new pass: ******
remove anonymous users?: Y
disallow root login remotely?: N
remove test database?: Y
reload privilege tables?: Y

sudo service mysqld restart


#On the master MySQL node, grant replication privileges for your replica node:
sudo vi /etc/my.cnf
 
[mysqld]
log-bin=mysql-bin
server-id=1
transaction-isolation = READ-COMMITTED
innodb_flush_method = O_DIRECT
binlog_format = mixed

sudo service mysqld restart

#Log in to mysql

mysql -u root -p 

#Note the FQDN of your replica host.

mysql> GRANT REPLICATION SLAVE ON *.* TO 'user'@'FQDN' IDENTIFIED BY 'password';
mysql> SET GLOBAL binlog_format = 'ROW'; 
mysql> FLUSH TABLES WITH READ LOCK;


#In a second terminal session, log into the MySQL master and show its status:
mysql> SHOW MASTER STATUS;

#Capture the file name and byte offset. The replica uses this info to sync to the master.

#Logout and dismiss the second session; 

#on the first session

#remove the lock on the first with 

mysql> UNLOCK TABLES;

#Now log on to the replica. 

sudo vi /etc/my.cnf
		[mysqld]
		server-id=2

sudo service mysqld restart

#Use the following statements to connect with the master:

mysql> CHANGE MASTER TO MASTER_HOST='master host', MASTER_USER='replica user', 	MASTER_PASSWORD='replica password', MASTER_LOG_FILE='master file name', 	MASTER_LOG_POS=master file offset;

#Now start slave

mysql> START SLAVE;
mysql> SHOW SLAVE STATUS \G

#If successful, the Slave_IO_State field will read Waiting for master to send event

#On the Master MySQL server, login and create databases for the services

mysql -u root -p

# Maria DB Config for CentOS 7 

Only on Server

nano /etc/yum.repos.d/MariaDB.repo

[mariadb]
name = MariaDB
baseurl = http://yum.mariadb.org/10.1/centos7-amd64
gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=1

yum install MariaDB-server MariaDB-client -y

systemctl start mariadb
systemctl enable mariadb
systemctl status mariadb
mysql_secure_installation

mysql -V

mysqld --print-defaults

mysql -u root -p

**********************


#create databases for the SCM (scm), Activity Monitor (amon), Reports Manager (rman), Hive Metastore Server (hive), Hue Server (hue), Sentry Server (sentry), Cloudera Navigator Audit Server (nav), Cloudera Navigator Metadata Server (navms) and Oozie (oozie)	

create database amon DEFAULT CHARACTER SET utf8;
grant all on amon.* TO 'amon'@'%' IDENTIFIED BY 'amon_password';

amon, rman, hive, hue, sentry, nav, navms



create database rman DEFAULT CHARACTER SET utf8;
grant all on rman.* TO 'rman'@'%' IDENTIFIED BY 'rman_password';
create database amon DEFAULT CHARACTER SET utf8;
grant all on amon.* TO 'amon'@'%' IDENTIFIED BY 'amon_password';
create database hive DEFAULT CHARACTER SET utf8;
grant all on hive.* TO 'hive'@'%' IDENTIFIED BY 'hive_password';
create database hue DEFAULT CHARACTER SET utf8;
grant all on hue.* TO 'hue'@'%' IDENTIFIED BY 'hue_password';
create database sentry DEFAULT CHARACTER SET utf8;
grant all on sentry.* TO 'sentry'@'%' IDENTIFIED BY 'sentry_password';
create database nav DEFAULT CHARACTER SET utf8;
grant all on nav.* TO 'nav'@'%' IDENTIFIED BY 'nav_password';
create database navms DEFAULT CHARACTER SET utf8;
grant all on navms.* TO 'navms'@'%' IDENTIFIED BY 'navms_password';
create database scm DEFAULT CHARACTER SET utf8;
grant all on scm.* TO 'scm'@'%' IDENTIFIED BY 'scm_password';
create database oozie DEFAULT CHARACTER SET utf8;
grant all on oozie.* TO 'oozie'@'%' IDENTIFIED BY 'oozie_password';

#on the server, Install the Cloudera Manager
#first download the version suiting your OS

wget https://archive.cloudera.com/cm5/redhat/6/x86_64/cm/cloudera-manager.repo
sudo cp cloudera-manager.repo /etc/yum.repos.d/

#install JDK

sudo yum install -y oracle-j2sdk1.7

#Install cloudera-manager server packages

sudo yum install -y cloudera-manager-daemons cloudera-manager-server

#prepare database for cloudera manager

sudo /usr/share/cmf/schema/scm_prepare_database.sh mysql scm scm scm_password

#optional (add the -h option to specify host if the database server is situated on a server different from the cloudera server) e.g. -h cm-oracle.example.com

#Start cloudera manager

sudo service cloudera-scm-server start


#login on a browser using the public ip and the default port 7180
default credentials are username:admin, password, admin
choose cdh parcels, use pem file to login and don't select single use mode
copy the Master's internal address and paste in /etc/cloudera-scm-agent/config.ini
before pushing parcels

#Push the parcels to the agents (u can tail the master node to see what's going on in the field) - var'log/cloudera-scm/cloudera-scm-log

#all the services should be having good health (green sign)
#check hosts heartbeats

Log into a host in the cluster.

Run the Hadoop PiEstimator example using one of the following commands:

#Parcel

sudo -u hdfs hadoop jar /opt/cloudera/parcels/CDH/lib/hadoop-mapreduce/hadoop-mapreduce-examples.jar pi 10 100

#Package
sudo -u hdfs hadoop jar /usr/lib/hadoop-mapreduce/hadoop-mapreduce-examples.jar pi 10 100


# Enable Kerberos

http://blog.puneethabm.in/configure-hadoop-security-with-cloudera-manager-using-kerberos/

# Enable High Availability
The following manual steps must be performed after completing this wizard:

        Configure the HDFS Web Interface Role of Hue service(s) Hue to be an HTTPFS role instead of a NameNode.


https://www.cloudera.com/documentation/enterprise/latest/topics/cdh_hag_hdfs_ha_config.html

https://www.cloudera.com/documentation/enterprise/latest/topics/cdh_hag_hdfs_ha_enabling.html


For each of the Hive service(s) Hive, stop the Hive service, back up the Hive Metastore Database to a persistent store, run the service command "Update Hive Metastore NameNodes", then restart the Hive services.

https://www.cloudera.com/documentation/enterprise/latest/topics/cdh_hag_hdfs_ha_cdh_components_config.html

Back up the Hive metastore database.
(For MariaDB)
mysqldump -u hive -phive_password hive > /tmp/hive-backup.sql

Back up the Hive Metastore Database before running this command. If using Impala, after running this command you must either restart Impala or execute an 'invalidate metadata' query.

(For MySQL)

mysqldump -hhostname -uusername -ppassword database > /tmp/database-backup.sql
