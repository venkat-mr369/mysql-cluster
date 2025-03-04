
## Introduction

The MySQL Cluster distributed database provides high availability and throughput for your MySQL database management system. 
A MySQL Cluster consists of one or more management nodes (ndb_mgmd) that store the cluster’s configuration and control the data nodes (ndbd), 
where cluster data is stored. After communicating with the management node, clients (MySQL clients, servers, or native APIs) 
connect directly to these data nodes.

With MySQL Cluster there is typically no replication of data, but instead data node synchronization. For this purpose a special data engine 
must be used — NDBCluster (NDB). It’s helpful to think of the cluster as a single logical MySQL environment with redundant components. 
Thus, a MySQL Cluster can participate in replication with other MySQL Clusters.

MySQL Cluster works best in a shared-nothing environment. Ideally, no two components should share the same hardware. For simplicity and demonstration purposes, 
we’ll limit ourselves to using only three servers. We will set up two servers as data nodes which sync data between themselves. The third server 
will be used for the Cluster Manager and also for the MySQL server/client. If you spin up additional servers, you can add more data nodes to the cluster, 
decouple the cluster manager from the MySQL server/client, and configure more servers as Cluster Managers and MySQL servers/clients.

MySQL Clusters constitute the following components:

***Data Nodes***
With the help of Data Nodes, you can store data in a MySQL cluster environment. Actually, it imitates data nodes, making them available whenever 
one or more storage nodes fail. After, the stored data is visible to every MySQL server connected to the cluster. Moreover, data nodes handle 
the entire database transactions. However, some MySQL data, like permissions and stored procedures, cannot be stored in the cluster and should be updated 
on every MySQL server attached to the cluster.

***Management Client***
Additionally, this client program manages your cluster. Basically, it delivers every administrative functionality, like starting and stopping nodes, 
getting status information, and creating backups.

***Applications***
Applications connect to the MySQL cluster exactly as they connect to the MySQL cluster through MySQL Server. So, if you use another MySQL storage engine, 
they will use group applications. However, your application may specifically need to handle cluster specific features.

***Management Server Nodes***
Besides, Administration Server nodes handle system configuration an used it to reconfigure the cluster. As a result, the server node management should 
only run during system start up and reconfiguration. Other components of the MySQL cluster can work without relying on the Central Management Server node.

***MySQL Server Nodes***
Certainly, these nodes run on a MySQL server that has access to cluster storage. Multiple MySQL servers can be connected to a cluster. Consequently, 
it provides redundancy and performance through parallel processing. When you perform updates on one MySQL server, they are immediately reflected on 
other servers connected to the cluster.

## Installation

Adding the MySQL Yum Repository
First, add the MySQL Yum repository to your system's repository list. Follow these steps:

Go to the download page for MySQL Yum repository at https://dev.mysql.com/downloads/repo/yum/.

Select and download the release package for your platform.

Install the downloaded release package with the following command, replacing platform-and-version-specific-package-name with the name of the downloaded package:

```bash
$> sudo rpm -Uvh platform-and-version-specific-package-name.rpm
```
For example, for version n of the package for EL6-based systems, the command is:


rpm -ivh mysql80-community-release-el8-5.noarch.rpm
yum repolist all | grep mysql

```bash
$> sudo dnf config-manager --disable mysql80-community
$> sudo dnf config-manager --enable mysql-cluster-8.0-community

$> yum repolist enabled | grep mysql
!mysql-cluster-8.0-community/x86_64 MySQL Cluster 8.0 Community               18
!mysql-connectors-community/x86_64  MySQL Connectors Community                31
!mysql-tools-community/x86_64       MySQL Tools Community                     33
```

The subrepository for NDB Cluster 8.0 (Community edition) has now been enabled. Also in the list are a number of other subrepositories of the MySQL Yum repository that have been enabled by default.

Installing MySQL NDB Cluster
For a minimal installation of MySQL NDB Cluster, follow these steps (for dnf-enabled systems, replace yum in the commands with dnf):

Install the components for SQL nodes:
```bash
$> sudo yum install mysql-cluster-community-server
```
After the installation is completed, start and initialize the SQL node by following the steps given in Starting the MySQL Server.

If you choose to initialize the data directory manually using the mysqld --initialize command (see Initializing the Data Directory for details), a root password is going to be generated and stored in the SQL node's error log; see MySQL Server Initialization for how to find the password, and for a few things you need to know about it.

Install the executables for data nodes:

```bash
$> sudo yum install mysql-cluster-community-data-node
```
Install the executables for management nodes:
```bash
$> sudo yum install mysql-cluster-community-management-server
```

## Prerequisites

all machines /etc/hosts file should be added below machines information.
```bash
198.168.17.101 server1.localdomain server1
198.168.17.102 server2.localdomain server2
198.168.17.103 server3.localdomain server3
198.168.17.104 server4.localdomain server4
```
To complete this tutorial, you will need a total of 4 servers: two servers for the redundant MySQL data nodes (ndbd), and one server for the Cluster Manager (ndb_mgmd) and MySQL server/client (mysqld and mysql).

A non-root user with sudo privileges configured.
Be sure to note down the private IP addresses of your three Machines. In this tutorial our cluster nodes have the following private IP addresses:

198.168.17.101 will be the Cluster Manager
198.168.17.102 will be the first data node 
198.168.17.103 will be the second data node 
198.168.17.104 will be the mysqld node

Configuring the data nodes and SQL nodes.  The my.cnf file needed for the data nodes is fairly simple. The configuration file should be located in the /etc directory and can be edited using any text editor. (Create the file if it does not exist.) For example:
```bash
$> vi /etc/my.cnf
#Note
#We show vi being used here to create the file, but any text editor should work just as well.

#For each data node and SQL node in our example setup, my.cnf should look like this:

[mysqld]
# Options for mysqld process:
ndbcluster                      # run NDB storage engine

[mysql_cluster]
# Options for NDB Cluster processes:
ndb-connectstring=oel86mysql3  # location of management server
```
After entering the preceding information, save this file and exit the text editor. Do this for the machines hosting data node “A”, data node “B”, and the SQL node.

Important
Once you have started a mysqld process with the ndbcluster and ndb-connectstring parameters in the [mysqld] and [mysql_cluster] sections of the my.cnf file as shown previously, you cannot execute any CREATE TABLE or ALTER TABLE statements without having actually started the cluster. Otherwise, these statements fail with an error. This is by design.

Configuring the management node.  The first step in configuring the management node is to create the directory in which the configuration file can be found and then to create the file itself. For example (running as root):
```bash
$> mkdir /var/lib/mysql-cluster
$> cd /var/lib/mysql-cluster
$> vi config.ini
```
For our representative setup, the config.ini file should read as follows:

```bash
[ndbd default]
# Options affecting ndbd processes on all data nodes:
NoOfReplicas=2    # Number of fragment replicas
DataMemory=98M    # How much memory to allocate for data storage

Important note. Use host name not ip in config.ini file

[ndb_mgmd]
# Management process options:
HostName=server1          # Hostname or IP address of management node
DataDir=/var/lib/mysql-cluster  # Directory for management node log files

[ndbd]
# Options for data node "A":
                                # (one [ndbd] section per data node)
HostName=server2                # Hostname or IP address
NodeId=2                        # Node ID for this data node
DataDir=/var/lib/mysql-cluster/data   # Directory for this data node's data files

[ndbd]
# Options for data node "B":
HostName=server3                # Hostname or IP address
NodeId=3                        # Node ID for this data node
DataDir=/var/lib/mysql-cluster/data   # Directory for this data node's data files

[mysqld]
# SQL node options:
HostName=server4                # Hostname or IP address
NodeId=4
                                # (additional mysqld connections can be
                                # specified for this node for various
                                # purposes such as running ndb_restore)
```
save and close the file.

Note:-
The world database can be downloaded from https://dev.mysql.com/doc/index-other.html.

After all the configuration files have been created and these minimal options have been specified, you are ready to proceed with starting the cluster and 
verifying that all processes ar								

 ## Initial startup of ndb cluster
Starting the cluster is not very difficult after it has been configured. Each cluster node process must be started separately, and on the host where it resides. The management node should be started first, followed by the data nodes, and then finally by any SQL nodes:

On the management host, issue the following command from the system shell to start the management node process:
```bash
$> ndb_mgmd --initial -f /var/lib/mysql-cluster/config.ini
```
The first time that it is started, ndb_mgmd must be told where to find its configuration file, using the -f or --config-file option. This option requires that --initial or --reload also be specified; see Section 23.5.4, “ndb_mgmd — The NDB Cluster Management Server Daemon”, for details.

As you can see, the MySQL Cluster Manager is installed and running. Now, kill the running server and create a systemd file for Cluster manager:

```bash
pkill -f ndb_mgmd
vi /etc/systemd/system/ndb_mgmd.service
```
Add the following configurations:
```bash
[Unit]
Description=MySQL NDB Cluster Management Server
After=network.target auditd.service

[Service]
Type=forking
ExecStart=/usr/sbin/ndb_mgmd -f /var/lib/mysql-cluster/config.ini
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
After, save and close the file then reload the systemd daemon to apply the changes:
## on mgmt server
```bash
systemctl daemon-reload
```
Following is to start and enable the Cluster Manager with the following command:
```bash
systemctl start ndb_mgmd
systemctl enable ndb_mgmd
```
As noted, you can also check the active status with the following command:
```bash
systemctl status ndb_mgmd
```

On each of the data node hosts, run this command to start the ndbd process:

$> ndbd

Your MySQL data node Droplet can now communicate with both the Cluster Manager and other data node over the private network.

Finally, we’d also like the data node daemon to start up automatically when the server boots. We’ll follow the same procedure used for the Cluster Manager, and create a systemd service.

Before we create the service, we’ll kill the running ndbd process:
## On Data Node
```bash
sudo pkill -f ndbd
```
Now, open and edit the following systemd Unit file using your favorite editor:
```bash
sudo vi /etc/systemd/system/ndbd.service
```
Paste in the following code:
```bash
[Unit]
Description=MySQL NDB Data Node Daemon
After=network.target auditd.service

[Service]
Type=forking
ExecStart=/usr/sbin/ndbd
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
Here, we’ve added a minimal set of options instructing systemd on how to start, stop and restart the ndbd process. To learn more about the options used in this unit configuration, consult the systemd manual.

:wq! Save and close the file 

Now, reload systemd’s manager configuration using daemon-reload:
```bash
sudo systemctl daemon-reload
```
We’ll now enable the service we just created so that the data node daemon starts on reboot:
```bash
sudo systemctl enable ndbd
```
Finally, you can start the service:
```bash
sudo systemctl start ndbd
```
You can verify that the NDB Cluster Management service is running:
```bash
sudo systemctl status ndbd
```

If you used RPM files to install MySQL on the cluster host where the SQL node is to reside, you can (and should) use the supplied startup script to start the MySQL server process on the SQL node.

If all has gone well, and the cluster has been set up correctly, the cluster should now be operational. You can test this by invoking the ndb_mgm management node client. The output should look like that shown here, although you might see some slight differences in the output depending upon the exact version of MySQL that you are using:
-- if error for data nodes use below on management node.
```bash
ndb_mgm -e "SHUTDOWN"
ndb_mgmd --reload --config-file /var/lib/mysql-cluster/config.ini
systemctl start ndb_mgmd
```
then restart mysqld on all mysql or api nodes.
```bash
systemctl restart mysqld
```
$> ndb_mgm
-- NDB Cluster -- Management Client --
it will install packages for first time
```bash
ndb_mgm> show
Cluster Configuration
---------------------
[ndbd(NDB)]     2 node(s)
id=2    @192.168.17.102  (mysql-8.0.41 ndb-8.0.41, Nodegroup: 0, *)
id=3    @192.168.17.103  (mysql-8.0.41 ndb-8.0.41, Nodegroup: 0)

[ndb_mgmd(MGM)] 1 node(s)
id=1    @192.168.17.101  (mysql-8.0.41 ndb-8.0.41)

[mysqld(API)]   1 node(s)
id=4    @192.168.17.104  (mysql-8.0.41 ndb-8.0.41)
```

The SQL node is referenced here as [mysqld(API)], which reflects the fact that the mysqld process is acting as an NDB Cluster API node.

Note
The IP address shown for a given NDB Cluster SQL or other API node in the output of SHOW is the address used by the SQL or API node to connect to the cluster data nodes, and not to any management node.

You should now be ready to work with databases, tables, and data in NDB Cluster

Note
The default port for Cluster management nodes is 1186; the default port for data nodes is 2202. However, the cluster can automatically allocate ports for 
data nodes from those that are already free.

Test MySQL Cluster
At this point, the MySQL multi-node cluster is up and running. Now, its time to test it. First, log in to the MySQL shell using the following command:

reset the root mysql password
vi /var/log/mysqld.log 
to search password "/password"
get it and login 
mysql -u root -p
Once you are logged in, check the cluster status using the following command: to reset the password
set password='Oracle_123';

SHOW ENGINE NDB STATUS \G
```bash
mysql> SHOW ENGINE NDB STATUS \G
*************************** 1. row ***************************
  Type: ndbclus
  Name: connection
Status: cluster_node_id=12, connected_host=oel86mysql3, connected_port=1186, number_of_data_nodes=2, number_of_ready_data_nodes=2, connect_count=0
*************************** 2. row ***************************
  Type: ndbclus
  Name: NdbTransaction
Status: created=2, free=2, sizeof=392
*************************** 3. row ***************************
  Type: ndbclus
  Name: NdbOperation
Status: created=4, free=4, sizeof=944
*************************** 4. row ***************************
  Type: ndbclus
  Name: NdbIndexScanOperation
Status: created=0, free=0, sizeof=1152
*************************** 5. row ***************************
  Type: ndbclus
  Name: NdbIndexOperation
Status: created=0, free=0, sizeof=952
*************************** 6. row ***************************
  Type: ndbclus
  Name: NdbRecAttr
Status: created=0, free=0, sizeof=88
*************************** 7. row ***************************
  Type: ndbclus
  Name: NdbApiSignal
Status: created=16, free=16, sizeof=144
*************************** 8. row ***************************
  Type: ndbclus
  Name: NdbLabel
Status: created=0, free=0, sizeof=200
*************************** 9. row ***************************
  Type: ndbclus
  Name: NdbBranch
Status: created=0, free=0, sizeof=32
*************************** 10. row ***************************
  Type: ndbclus
  Name: NdbSubroutine
Status: created=0, free=0, sizeof=72
*************************** 11. row ***************************
  Type: ndbclus
  Name: NdbCall
Status: created=0, free=0, sizeof=24
*************************** 12. row ***************************
  Type: ndbclus
  Name: NdbBlob
Status: created=0, free=0, sizeof=592
*************************** 13. row ***************************
  Type: ndbclus
  Name: NdbReceiver
Status: created=0, free=0, sizeof=128
*************************** 14. row ***************************
  Type: ndbclus
  Name: NdbLockHandle
Status: created=0, free=0, sizeof=48
*************************** 15. row ***************************
  Type: ndbclus
  Name: binlog
Status: latest_epoch=1438814044161, latest_trans_epoch=700079669256, latest_received_binlog_epoch=0, latest_handled_binlog_epoch=773094113280, latest_applied_binlog_epoch=0
15 rows in set (0.03 sec)
```
** Test cases.

Let’s create a table in one of the API node and see if all that table is accessible from the other API node.
```bash
create database demo;
use demo;
CREATE TABLE NDB_test_emp(ename varchar(40),empid int) ENGINE=NDBCLUSTER;
insert into NDB_test_emp(ename, empid) values('venkat',1);
insert into NDB_test_emp(ename, empid) values('kishore',2);
```

--- startup shutdown sequence
1.    First, we need to stop all the API node.
2.    Once all the API is down shutdown all the database and management node from the Management node using single command.
systemctl stop mysqld on all mysql nodes
ndb_mgm -e shutdown on management node
