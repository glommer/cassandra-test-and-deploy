# Ansible Scylla and Cassandra Cluster Stress 

A framework for running and automating Scylla/Cassandra stress tests on Amazon EC2.

This repository contains Ansible playbooks and scripts for running Scylla/Cassandra **multi region** performance tests on EC2 with multiple DB servers and multiple cassandra-stress loaders, as well as adding, stopping and starting nodes.


### Introduction
The following will compare Scylla cluster to Cassandra cluster performance on EC2, using 6 loader servers
```
./ec2-setup-scylla.sh -e "cluster_nodes=2"
./ec2-setup-cassandra.sh -e "cluster_nodes=2"
./ec2-setup-loadgen.sh -e "load_nodes=6"
./ec2-stress.sh 1 -e "load_name=cassandra.v1" -e "server=Cassandra" -e "stress_options='-errors ignore'"
./ec2-stress.sh 1 -e "load_name=scylla.v1" -e "server=Scylla" -e "stress_options='-errors ignore'"
```
Note you will need more load nodes to stress Scylla.

### Prerequisites

#### install
Make sure you installed the following:
* [ansible](http://docs.ansible.com/ansible/intro_installation.html)
* [python boto](https://github.com/boto/boto#installation)

#### config

Configure EC2 credentials:

```sh
export AWS_ACCESS_KEY_ID=<...>
export AWS_SECRET_ACCESS_KEY=<...>
```

Make sure your EC2 SSH keys for each EC2 region is included in the keychain:

```sh
ssh-add <key-file>
```
**You should do it for each of EC2 region!**

To avoid prompting for SSH key confirmation
```
export ANSIBLE_HOST_KEY_CHECKING=False
```
Make sure you have boto version 2.34.0 or above. To check boto version:
```
pip show boto
```
To update boto
```
sudo pip install --upgrade boto
```

To make sure your server use a unique name on EC2, set ANSIBLE_EC2_PREFIX
```
export ANSIBLE_EC2_PREFIX=tzach
```

### Setup
The default setup is using one region for all nodes, and the reflector to attache them to a cluster.
To test a multi region setup, use ```-e "ec2_multiregion=true"``` as an extra parameter for the setup commands below.
The default EC2 regions are define in ```inventories/ec2/group_vars/all.yaml```, with the AMI, security group, and key-pair per region. **You must update the key-pair** to your own.

#### Create security group
The following will create a EC2 security group called "cassandra-security-group", which is later used for all EC2 servers.
```
ansible-playbook -i inventories/ec2/ configure-security-group.yaml -e "security_group=cassandra-security-group" -e "region=your-ec2-region" -e "-e "vpc_id=your-vpc"
```
You only need to run this once. Once security group exists, there is no need to repeat this step. You can use a different security group by adding ```-e "security_group=your_group_name"``` option to all ec2-setup-* scripts below.

#### Launch Scylla cluster
Scylla servers launch from Scylla AMI, base on Fedora 22 (login fedora)

```
./ec2-setup-scylla.sh <options>
```

  **options**
  * ```-e "cluster_nodes=2"``` - number of nodes **per region** (default 2)
  * ```-e "instance_type=c3.8xlarge"``` - type of EC2 instance
  * ```-e ec2_multiregion=true```- a multi region EC2 deployment **[does not work yet!]**
  * ```-e "collectd_server=your-collectd-server-ip"``` - collectd server to collect metric. See [scylla-monitoring](https://github.com/scylladb/scylla-monitoring) for an example monitoring server

Server are created with EC2 name *DB*, and tag "server=Scylla"

#### Launch Cassandra cluster
Cassandra servers launch from AMI, base on Ubuntu 14 (login ubuntu)

```
./ec2-setup-cassandra.sh <options>
```

  **options**
  * ```-e "cluster_nodes=2"``` - number of nodes **per region**  (default 2)
  * ```-e "instance_type=m3.large"``` - type of EC2 instance
  * ```-e "num_tokens=6"``` - set number of vnode per server
  * ```-e ec2_multiregion=true``` - a multi region EC2 deployment
  ```-e "collectd_server=your-collectd-server-ip"``` - collectd server to collect metric. See [scylla-monitoring](https://github.com/scylladb/scylla-monitoring) for an example monitoring server

Server are created with EC2 name *DB*, and tag "server=Cassandra"

#### Launch Loaders
Loaders are launch from Scylla AMI, base on Fedora 22 (login fedora), including a version of cassandra-stress which only use CQL, not thrift.

```
./ec2-setup-loadgen.sh <options>
```

  **options**
  * ```-e "load_nodes=2"``` - number of loaders **per region** (default 2)
  * ```-e "instance_type=m3.large"``` - type of EC2 instance

#### Add nodes to existing cluster
```
./ec2-add-node-to-cluster.sh
```

  **options**
  * ```-e "stopped=true"``` - start server is stopped state
  * ```-e "instance_type=m3.large"``` - type of EC2 instance
  * ```-e "cluster_nodes=2"``` - number of nodes to add (default is 2)
  * ```-e ec2_multiregion=true```- a multi region EC2 deployment

#### Start stopped nodes (one by one)
```
./ec2-start-server.sh
```

### Run Load

```
./ec2-stress.sh <iterations> -e "load_name=late_night" -e "server=Scylla" <more options>
```

for multi region (update the regions and replication for your needs)
```
./ec2-stress.sh 1 -e "load_name=cassandra.multi.v1" -e "server=Cassandra" -e "stress_options='-schema \"replication(strategy=NetworkTopologyStrategy,us-east_test=1,us-west-2_test=1)\" keyspace=\"mykeyspace\"'" 
```

**Options**
All parameters are available at
[group_vars/all](https://github.com/cloudius-systems/ansible-cassandra-cluster-stress/blob/master/group_vars/all)

To override each of the parameter, use the ``-e "param=value"
notation. Examples below:

* ```-e "server=Cassandra"``` - test cassandra servers (default is scylla)
* ```-e "populate=2500000"``` - key population per loader (default is 1000000)
* ```-e "command=write"``` [read, write,mixed,..] setting stress command
* ```-e "stress_options='-schema \"replication(factor=2)\"'"```
* ```-e "stress_options='-errors ignore'"```
* ```-e "command_options='cl=LOCAL_ONE -mode native cql3'"```
* ```-e "command_options=duration=60m"```
* ```-e "threads=200"```
* ```-e "profile_dir=/tmp/cassandra-stress-profiles/" -e "profile_file=cust_a/users/users1.yaml" -e "command_options=ops\\(insert=1\\)"```

Make sure **load name is unique**  so the new results will not
override an old run in the same day.

Load name can not not include *-* (dash)

### Results

The results are automatically collected and copy to your local
```/tmp/cassandra-results/``` directory, including summary files per load. 

### Stop and start nodes
Stop a server in the cluster

```
./ec2-stop-server.sh external_ip(s)
```

Stop and clean node data
```
./ec2-stop-server.sh external_ip -e "clean_data=true"
```

This will stop Cassandra service on this server, while keeping it in
the cluster.

Use ```./ec2-start-server.sh``` to start stopped servers, or just
update the **stopped** tag back to false.
The next stress test will restart the Cassandra service.

### Terminate Servers
```
./ec2-terminate.sh
```

## Todo
Scylla cluster does not yet support:
* clean_data option, to clean data files before each stress
* ec2-add-node-to-cluster, ec2-start-server.sh and ec2-stop-server.sh


## License
The Scylla/Cassandra Cluster Stress is distributed under the Apache License.
See LICENSE.TXT for more.
