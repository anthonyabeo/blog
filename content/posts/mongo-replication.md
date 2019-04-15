---
title: "Setting Up MongoDB Replica Set On Ubuntu 12.04"
date: 2018-11-20T15:36:33Z
---
### Introduction
This is a tutorial detailing how to configure a MongoDB replica set. If you are reading this, we can assume you know what a replica set, why is is necessary and how it works. Otherwise you can checkout the MongoDB documentation for replica sets and clusters. 

In this tutorial, i will be using 3 Vagrant Ubuntu 12.04 boxes and MongoDB version 2.4.10. You should be comfortable spinning up vagrant boxes as this tutorial will not cover that. So let's get started.

### Spin up the three Vagrant boxes

#### Configure Vagrant VMs
I am assuming you have downloaded the `ubuntu/precise64` 12.04 box locally. So run the following commands to create vagrant boxes.

```shell
# Create a directory for each VM.
$ mkdir -p mongo01
$ mkdir -p mongo02
$ mkdir -p mongo03

# Create the Vagrant configuration files.
$ vagrant init ubuntu/precise64 --output mongo01/Vagrantfile
$ vagrant init ubuntu/precise64 --output mongo02/Vagrantfile
$ vagrant init ubuntu/precise64 --output mongo03/Vagrantfile
``` 

Next thing is to edit the Vagrantfile for each VM to have a specific hostname and IP address. Open each Vagrantfile and look for the lines that start with `config.vm.network`, uncomment it and assign it any IP address you want and add a line for the hostname as indicated below.

```python
# The IPs and the names are up to you.

# mongo01
config.vm.hostname = "mongo01"
config.vm.network private_network, ip: "192.168.33.1"

# mongo02
config.vm.hostname = "mongo02"
config.vm.network private_network, ip: "192.168.33.2"

# mongo03
config.vm.hostname = "mongo03"
config.vm.network private_network, ip: "192.168.33.3"
```
#### Find VMs by hostname or DNS
Now start the VMs with `vagrant up` and `ssh` into each with `vagrant ssh`. Now to avoid referring to each VM by the IP address, let's make sure we can refer to it by hostname or DNS by adding the IP addresses to the `/etc/hosts` files. Open this file in each VM and
add the following lines.

```python
192.169.33.1    mongo01
192.169.33.2    mongo02
192.169.33.3    mongo03
```
Save the file. `ping` the VMs to ensure that you can reach one from the other. You can run `ping mongo02` from the `mongo01` VM. When you are certain the machines can find each other, we can now set up the replica set.

### MongoDB Replica Set

#### Installing MongoDB on each VM
Run the following commands to install MongoDB 2.4.10. Do same for the other VMs.

```shell
# import the MongoDB public GPG Key
$ sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 7F0CEB10

# Create a /etc/apt/sources.list.d/mongodb.list file
$ echo 'deb http://downloads-distro.mongodb.org/repo/ubuntu-upstart dist 10gen' | sudo tee /etc/apt/sources.list.d/mongodb.list

# reload your repository
$ sudo apt-get update

# Install MongoDB 2.4.10
$ sudo apt-get install mongodb-10gen=2.4.10
```

#### Edit configuration file.
Now we need to make some changes to the configuration file. Open `/etc/mongodb.conf` and ensure that the following statements are uncommented. Add the ones that are not available like the line beginning with `keyFile`.

```python
# Supports Authentication
auth = true

dbpath=/var/lib/mongodb
logpath=/var/log/mongodb/mongod.log
logappend=true

# KeyFile for authenticating VMs in the replice set
keyFile=/var/lib/mongodb/keyFile

# Name of the repliceset. Set it to whatever you want. Must be the same for
# all the VMs in the repliceset.
replSet=demo
```

The replica set will use the `keyFile` to authenticate the nodes in the set. Therefore any machine that will be part of this replica set must have a keyfile. We will now create the key file for one VM. As usual, do same for the other VMs in the set.

```shell
$ echo -n "MyReplicaSetKey" | md5sum|grep -o "[0-9a-z]\+" > keyFile
```

Now we need to change the ownership and permission modes of the file; as well as moving it to the path specified in the configuration file.

```shell
$ sudo cp keyFile /var/lib/mongodb
$ sudo chown mongodb:nogroup /var/lib/mongodb/keyFile
$ sudo chmod 400 /var/lib/mongodb/keyFile
```

Next step, we need to create users in the DB for authentication and authorization. We need to first comment out the line beginning with `auth`. Because if authentication is enabled, we cannot even login to the DB since no user has been created yet.

Restart the mongodb service to ensure that the new configuration takes hold.
```shell
$ sudo service mongodb restart
```

#### Create Super Admin
Here we will create an admin accounts; The admin will have thr following roles: `userAdminAnyDatabase`, `clusterAdmin`, `readWriteAnyDatabase`, `dbAdminAnyDatabase`. This will allow the admin to create other users, manage the replica sets etc.

Choose any of the VMs as the primary node and connect to the MongoDB service running on it. The host and port are optional, unless your primary node exist on another machine. 
```shell
$ mongo [--host localhost --port 27017]
```

This should show the MongoDB shell prompt. Run the following commands to create the user.
```shell
# change to the admin database
> use admin

# create a superuser with username and password as "admin" and can 
# administer any database.
> db.addUser({
    user: "admin", pwd: "admin",
    roles: [ 
        "readWriteAnyDatabase"
        "userAdminAnyDatabase",
        "clusterAdmin",
        "dbAdminAnyDatabase" 
    ]
})
```

Exit the shell. Go to the configuration file and uncomment the line beginning with `auth` (remember we commented it out before creating the users) and restart the MongoDB service. Make sure the other nodes have the same configuration as well. They should have `auth` enabled, have the same `replSet` name as the others and the `keyFile` path should be same. Restart the other MongoDB services as well.

#### Add nodes to the replica set
Enter the mongodb shell again and authenticate with the username and password as "admin". 

```python
$ mongo

> use admin
> db.auth("admin", "admin")
1 # if successful and 0 otherwise
```
Then run the following commands to initiate the replica set and add nodes.

```python
# Initiate replica set
> use admin
> rs.initiate()

# Verify the configuration
> rs.conf()

# Add mongo02 as a member or the replica set
> rs.add("mongo02:27017")

# Add mongo03 as a member or the replica set
> rs.add("mongo03:27017")
```

Check the status of the replica set by running `rs.status()` which should return something similar to
```python
> rs.status() 
{
	"set" : "demo",
	"date" : ISODate("2014-10-28T23:12:29Z"),
	"myState" : 1,
	"members" : [
		{
			"_id" : 0,
			"name" : "mongo01:27017",
			"health" : 1,
			"state" : 1,
			"stateStr" : "PRIMARY",
			"optime" : {
				"t" : 1414522392000,
				"i" : 1
			},
			"optimeDate" : ISODate("2014-10-28T18:53:12Z"),
			"self" : true
		},
		{
			"_id" : 1,
			"name" : "mongo02:27017",
			"health" : 1,
			"state" : 2,
			"stateStr" : "SECONDARY",
			"uptime" : 15754,
			"optime" : {
				"t" : 1414522392000,
				"i" : 1
			},
			"optimeDate" : ISODate("2014-10-28T18:53:12Z"),
			"lastHeartbeat" : ISODate("2014-10-28T23:12:29Z"),
			"pingMs" : 0
		},
		{
			"_id" : 2,
			"name" : "mongo03:27017",
			"health" : 1,
			"state" : 2,
			"stateStr" : "SECONDARY",
			"uptime" : 7953,
			"optime" : {
				"t" : 1414522392000,
				"i" : 1
			},
			"optimeDate" : ISODate("2014-10-28T18:53:12Z"),
			"lastHeartbeat" : ISODate("2014-10-28T23:12:27Z"),
			"pingMs" : 0
		}
	],
	"ok" : 1
}
```

#### Add regular users to specific databases
At this point, our replica set has been created. We can create a database and add a user to that database like so.
```python
# create a database called books
> use books

# Insert a record into the "novels" collection
> db.novels.insert({"title": "The Arrival"})

# add user with read and write access to the "books" database
> db.addUser({
    user: "bob", pwd: "bob", 
    roles: ["readWrite"]
})
```

Now the next time we try to access the `books` database, we have to authenticate. When we reconnect to the shell, the prompt will be the same as before. We can authenticate as the super admin and add users as well as manage the cluster set. When we authenticate, the shell prompt changes.

```shell
> use admin
> db.auth("fred", "fred")
1
demo:PRIMARY> 
```
`demo` is the name used for the replica set and the `PRIMARY` indicate that this a primary node and the others will be `SECONDARY` nodes. We can now insert, update and delete in the `PRIMARY` node and it will be replicated in the `SECONDARY` nodes.