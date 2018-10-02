---
type: post
title: Build a High Available MySQL Cluster with Percona XtraDB and Ubuntu 16.04
---

## Introduction
This tutorial will show how to install the _Percona XtraDB Cluster_ on three _Ubuntu_ 16.04 LTS servers, using the packages from the official Percona repositories. After this tutorial you should be able to install, configure and secure a high available MySQL cluster.

## Prerequisites

To follow this tutorial you need 3 machines running Ubuntu 16.04. All machines need to be reachable over SSH and they need to be in the same network.

```
node #1
hostname: mysql01
IP: 10.100.0.200

node #2
hostname: mysql02
IP:10.100.0.201

node #3 
hostname: mysql03 
IP:10.100.0.202
```

### Configuring the repositories

To install Percona XtraDB you will have to add the appropriate repositories to your servers. Run the following commands on the three nodes.

```bash
wget https://repo.percona.com/apt/percona-release_0.1-4.$(lsb_release -sc)_all.deb
sudo dpkg -i percona-release_0.1-4.$(lsb_release -sc)_all.deb
sudo apt-get update
sudo apt-get install percona-xtradb-cluster-57
```

### Configure Apparmor & SElinux

In order to successfully start Percona XtraDB, it is advised to disable SElinux. When SElinux is enabled the communication between nodes does not work and the nodes won't be able to join the cluster successfully. By default SElinux is not enabled in Ubuntu, if you have installed it previously there are some options for disabling it. To test if SElinux is enabled on your system run the following command.

```bash
selinuxenabled && echo enabled || echo disabled
```

If SElinux is enabled you can completely remove it from your system by uninstalling it

```bash
sudo apt-get remove selinux* --purge -y
```

Or you can disable it by editing the config file for SElinux.

```bash
SELINUX=disabled
```

Percona doesn’t provide any AppArmor profile for PXC, but it seems that on this server (Ubuntu TLS), a previous version of MySQL was installed and then removed but the AppArmor profile was still present. So if you use apparmor (or if you don’t know) and you want to check is there is a profile for mysql, you can run the following command :

```bash
apparmor_status
```

If a mysql profile is available you can disable it like this.

```bash
sudo ln -s /etc/apparmor.d/usr.sbin.mysqld /etc/apparmor.d/disable/
sudo apparmor_parser -R /etc/apparmor.d/usr.sbin.mysqld
```

If SElinux is enabled you can completely remove it from your system by uninstalling it

> Remember to check all your nodes, it prevents a lot of issues later on.

## Configure your firewall

Percona XtraDB uses some ports asside from the standard 3306. You need to open these ports on your firewall for the cluster to function well. 

**MySQL:** 3306  
**Cluster Communication:** 4567  
**SST:** 4568 

The main cluster communication happens over port 4567. You can change this by specifying this option.

```
wsrep_provider_options="gmcast.listen_addr=tcp://0.0.0.0:4010;" 
```
## Configure the bootstrap node

Go to your first node and open /etc/mysql/my.cnf and copy the following configuration at the end of the file and change the IP's on line 2 and 6 to your configuration.

```bash
wsrep_provider=/usr/lib/libgalera_smm.so
wsrep_cluster_address=gcomm://10.100.0.200,10.100.0.201,10.100.0.203 # Change the IP's to your IP's (ip1,ip2,ip3) 
binlog_format=ROW
default_storage_engine=InnoDB
innodb_autoinc_lock_mode=2
address wsrep_node_address=10.100.0.200 # The IP of the node that you are configuring
wsrep_sst_method=xtrabackup-v2 
wsrep_cluster_name=my_mysql_cluster 
wsrep_sst_auth="username:password" # A username and password for SST that we will configure later
```

The first node is ready to be started. Execute the following command to bootsrap the cluster.
```bash
/etc/init.d/mysql bootstrap-pxc
```
Now we have to check if the first node has started successfully.

```bash
sudo mysql -u root -p 
mysql > SHOW STATUS LIKE 'wsrep%';
+----------------------------+--------------------------------------+
| Variable_name              | Value                                |
+----------------------------+--------------------------------------+
| wsrep_local_state_uuid     | b598af3e-ace3-11e2-0800-3e90eb9cd5d3 |
...
| wsrep_local_state          | 4                                    |
| wsrep_local_state_comment  | Synced                               |
...
| wsrep_cluster_size         | 1                                    |
| wsrep_cluster_status       | Primary                              |
| wsrep_connected            | ON                                   |
...
| wsrep_ready                | ON                                   |
+----------------------------+--------------------------------------+
60 rows in set (0.01 sec) 
```
As you can see the first node started successfully and it is ready to accept other nodes to join the cluster. Now we have to configure a user to make use of State Snapshot Transfer. Change the username and password to the values you chose for wsrep\_sst\_auth in /etc/mysql/my.conf.

```bash
mysql> CREATE USER 'username'@'localhost' IDENTIFIED BY 'password';
mysql> GRANT PROCESS, RELOAD, LOCK TABLES, REPLICATION CLIENT ON \*.\* TO 'username'@'password';
mysql> FLUSH PRIVILEGES; 
```

> State Snapshot Transfer (SST) is a full data copy from one node (donor) to the joining node (joiner). It’s used when a new node joins the cluster. In order to be synchronized with the cluster, the new node has to receive data from a node that is already part of the cluster.

## Configure the two other nodes

Edit /etc/mysql/my.cnf on the two other nodes. Remember to change the IP of the host on line 6.

```bash
wsrep_provider=/usr/lib/libgalera_smm.so
wsrep_cluster_address=gcomm://10.100.0.200,10.100.0.201,10.100.0.203 # Change the IP's to your IP's (ip1,ip2,ip3) 
binlog\_format=ROW
default_storage_engine=InnoDB
innodb_autoinc_lock_mode=2
address wsrep\_node_address=10.100.0.200 # The IP of the node that you are configuring
wsrep_sst_method=xtrabackup-v2 
wsrep_cluster_name=my_mysql_cluster 
wsrep_sst_auth="username:password" # The username and password for SST
```
Then start MySQL on the nodes.
```bash
sudo service mysql start
```
If everything went right, you can check the cluster status to check if all nodes are connected and the cluster is in a good state.

```bash
sudo mysql -u root -p 
mysql> SHOW STATUS LIKE 'wsrep%';
+----------------------------+--------------------------------------+
| Variable_name              | Value                                |
+----------------------------+--------------------------------------+
| wsrep_local_state_uuid     | g4fd322e-whe3-11e2-0255\3e90eb9cd5d3 |
...
| wsrep_local_state          | 4                                    |
| wsrep_local_state_comment  | Synced                               |
...
| wsrep_cluster_size         | 3                                    |
| wsrep_cluster_status       | Primary                              |
| wsrep_connected            | ON                                   |
...
| wsrep_ready                | ON                                   |
+----------------------------+--------------------------------------+
40 rows in set (0.01 sec) 
```

### Conclusion

Setting up a Percona XtraDB cluster is relatively easy. There are however some things that could go wrong. 90% of the time it has something to do with SElinux. A whole other beast is when your cluster dies, there are a lot of scenario's that could break the cluster. I will get more in depth in a later article.

> TIP: In case of any errors look for the file /var/lib/mysql/mysqlxx.err this file has saved me in a lot of situations when troubleshooting MySQL.
