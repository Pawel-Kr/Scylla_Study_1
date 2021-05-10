# Scylla_Study_1
### Tasks to complete:
1. Create a 3 nodes cluster, with one datacenter, each of the nodes in a different rack , dc:north, racks north1, north2, north3.
2. Create a keyspace with 3 tables, one of the tables using STCS, another LCS, another TWCS.
3. Insert 10,000 records in each of the tables, using loop and cqlsh.
4. In the TWCS table, when creating the table and inserting data use time-window 30 minutes and the data to expire with 1 hour TTL.
5. Add a DC with 3 more nodes , each of the nodes in a different rack, dc: south, racks south1, south2, south3.
6. Install Scylla Manager.
7. Run repair using Scylla manager.
8. Decommission the old DC, keeping only the new created DC.
9. Add a node, decommission a node.
10. Then kill one of the nodes, destroy one of the containers (kill the seed node).
11. Replace procedure to replace this node we've killed.
### Prerequisition
1. Linux
2. Docker
3. Docker-Compose
### Misc
All required files are uploaded to this repo.
### Ex. 1
```
[pawel@dell-pawel ScyllaTraining]$ docker-compose -f docker-compose-dc1.yml up -d

[pawel@dell-pawel ScyllaTraining]$ sudo docker exec -it DC1_node1 nodetool status
Using /etc/scylla/scylla.yaml as the config file
Datacenter: north
=================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address     Load       Tokens       Owns    Host ID                               Rack
UN  172.18.0.2  252.48 KB  256          ?       d001367e-2b3e-4bb2-8384-dabb0c04e201  north3
UN  172.18.0.3  251.71 KB  256          ?       f3df05f1-4391-4ef1-9b85-cb425977c102  north2
UN  172.18.0.4  250.46 KB  256          ?       a2cd371c-f2c6-4ec3-8746-b60e109e7153  north1
```
### Ex. 2 & Ex. 4
```
cqlsh> CREATE KEYSPACE mykeyspace WITH replication = {'class': 'NetworkTopologyStrategy', 'north' : 3};

cqlsh> use mykeyspace ;

cqlsh:mykeyspace> CONSISTENCY QUORUM       
Consistency level set to QUORUM.

cqlsh:mykeyspace> CREATE TABLE STCS_table (playername text, playerteam text, age int, PRIMARY KEY (playername)) WITH compaction = {'class': 'SizeTieredCompactionStrategy'};
cqlsh:mykeyspace> CREATE TABLE LCS_table (playername text, playerteam text, age int, PRIMARY KEY (playername)) WITH compaction = {'class': 'LeveledCompactionStrategy'};
cqlsh:mykeyspace> CREATE TABLE TWCS_table (playername text, playerteam text, age int, PRIMARY KEY (playername)) WITH compaction = {'class': 'TimeWindowCompactionStrategy', 'compaction_window_unit': 'MINUTES', 'compaction_window_size': 30} AND default_time_to_live = 3600;
```
### Ex. 3
I've chosen inserting 10k rows from a file created prior to doing an exercise. The file was created by using MS Excel and random auto-fill of cells and then converted to the CSV file.
```
[pawel@dell-pawel ScyllaTraining]$ sudo docker cp /home/pawel/ScyllaTraining/tabs.csv DC1_node1:/etc/

[pawel@dell-pawel ScyllaTraining]$ sudo docker exec -it DC1_node1 cqlsh
cqlsh> use mykeyspace ;

cqlsh:mykeyspace> COPY lcs_table (playername , playerteam , age) FROM '/etc/tabs.csv' WITH DELIMITER = ';' AND HEADER = true ;
cqlsh:mykeyspace> COPY stcs_table (playername , playerteam , age) FROM '/etc/tabs.csv' WITH DELIMITER = ';' AND HEADER = true ;
cqlsh:mykeyspace> COPY twcs_table (playername , playerteam , age) FROM '/etc/tabs.csv' WITH DELIMITER = ';' AND HEADER = true ;
```
### Ex. 5
[Procedure to follow](https://docs.scylladb.com/operating-scylla/procedures/cluster-management/add_dc_to_existing_dc/)
```
cqlsh> ALTER KEYSPACE mykeyspace WITH replication = {'class': 'NetworkTopologyStrategy', 'north': 3, 'south': 0};

[pawel@dell-pawel ScyllaTraining]$ sudo docker-compose -f docker-compose-dc2.yml up -d

[pawel@dell-pawel ScyllaTraining]$ sudo docker exec -it DC1_node1 nodetool status
Using /etc/scylla/scylla.yaml as the config file
Datacenter: north
=================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address     Load       Tokens       Owns    Host ID                               Rack
UN  172.18.0.2  252.48 KB  256          ?       d001367e-2b3e-4bb2-8384-dabb0c04e201  north3
UN  172.18.0.3  251.71 KB  256          ?       f3df05f1-4391-4ef1-9b85-cb425977c102  north2
UN  172.18.0.4  250.46 KB  256          ?       a2cd371c-f2c6-4ec3-8746-b60e109e7153  north1
Datacenter: south
=================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address     Load       Tokens       Owns    Host ID                               Rack
UJ  172.18.0.6  ?          256          ?       8570fa72-09f2-41c2-aa0d-86696cec2242  south1
UJ  172.18.0.7  ?          256          ?       a9964600-2ff1-4419-b69a-2f578d1870a6  south2
UJ  172.18.0.5  ?          256          ?       7a6c23c8-a304-4c7e-8497-6a61e37f4bb1  south3

cqlsh> ALTER KEYSPACE mykeyspace WITH replication = {'class': 'NetworkTopologyStrategy', 'north': 3, 'south': 3};
cqlsh> ALTER KEYSPACE system_auth WITH replication = {'class': 'NetworkTopologyStrategy', 'north': 3, 'south': 3};
cqlsh> ALTER KEYSPACE system_distributed WITH replication = {'class': 'NetworkTopologyStrategy', 'north': 3, 'south': 3};
cqlsh> ALTER KEYSPACE system_traces WITH replication = {'class': 'NetworkTopologyStrategy', 'north': 3, 'south': 3};

[pawel@dell-pawel ScyllaTraining]$ sudo docker exec -it DC2_node1 nodetool rebuild --DC1
[pawel@dell-pawel ScyllaTraining]$ sudo docker exec -it DC2_node2 nodetool rebuild --DC1
[pawel@dell-pawel ScyllaTraining]$ sudo docker exec -it DC2_node3 nodetool rebuild --DC1

[pawel@dell-pawel ScyllaTraining]$ sudo docker exec -it DC2_node1 nodetool repair -pr
[pawel@dell-pawel ScyllaTraining]$ sudo docker exec -it DC2_node2 nodetool repair -pr
[pawel@dell-pawel ScyllaTraining]$ sudo docker exec -it DC2_node3 nodetool repair -pr
```
### Ex. 6
[Procedure to follow](https://hub.docker.com/r/scylladb/scylla-manager)
```
[pawel@dell-pawel ScyllaTraining]$ sudo docker-compose -f docker-compose.yml up -d

[pawel@dell-pawel ScyllaTraining]$ sudo docker pull scylladb/scylla-manager-agent:2.3.0 (outside of nodes)
```
Now it's required to install Scylla Manager Agent on every node. For example:
```
[pawel@dell-pawel ScyllaTraining]$ sudo docker exec -it DC1_node1 bash
[root@5e7e1e66af2f /]# yum -y install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
[root@5e7e1e66af2f /]# curl -o /etc/yum.repos.d/scylla-manager.repo -L http://downloads.scylladb.com/rpm/centos/scylladb-manager-2.3.repo
[root@5e7e1e66af2f /]# yum -y install scylla-manager-agent
```
Next, it's required to generate authentication token. Token is the same for every node IN THE SAME cluster. Tokens must be different on other clusters.
```
[root@5e7e1e66af2f /]# scyllamgr_auth_token_gen
```
After token is created, it's required to paste it into `/etc/scylla-manager-agent/scylla-manager-agent.yaml` file on every node.
```
[root@5e7e1e66af2f /]# vi /etc/scylla-manager-agent/scylla-manager-agent.yaml
```
Finally, the Agent can be started on every node. For exaple:
```
[root@5e7e1e66af2f /]# scylla-manager-agent -c /etc/scylla-manager-agent/scylla-manager-agent.yaml &
```
Final step is to add a cluster to the Manager:
```
[pawel@dell-pawel ScyllaTraining]$ sudo docker exec -it scyllatraining_scylla-manager_1 sctool cluster add -c 'Test Cluster' --host=<host ip> --auth-token=<token>
```
### Ex. 7
```
[pawel@dell-pawel ScyllaTraining]$ sudo docker exec -it scyllatraining_scylla-manager_1sctool repair -c 'Test Cluster'
```
### Ex. 8
[Procedure to follow](https://docs.scylladb.com/operating-scylla/procedures/cluster-management/decommissioning_data_center/)
```
[pawel@dell-pawel ScyllaTraining]$ sudo docker exec -it DC1_node1 nodetool repair -pr
[pawel@dell-pawel ScyllaTraining]$ sudo docker exec -it DC1_node2 nodetool repair -pr
[pawel@dell-pawel ScyllaTraining]$ sudo docker exec -it DC1_node3 nodetool repair -pr

cqlsh> ALTER KEYSPACE mykeyspace WITH replication = {'class': 'NetworkTopologyStrategy', 'south': 3};

[pawel@dell-pawel ScyllaTraining]$ sudo docker exec -it DC1_node1 nodetool decommission
[pawel@dell-pawel ScyllaTraining]$ sudo docker exec -it DC1_node2 nodetool decommission
[pawel@dell-pawel ScyllaTraining]$ sudo docker exec -it DC1_node3 nodetool decommission

[pawel@dell-pawel ScyllaTraining]$ sudo docker exec -it DC2_node1 nodetool status
Using /etc/scylla/scylla.yaml as the config file
Datacenter: south
=================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address     Load       Tokens       Owns    Host ID                               Rack
UJ  172.18.0.6  ?          256          ?       8570fa72-09f2-41c2-aa0d-86696cec2242  south1
UJ  172.18.0.7  ?          256          ?       a9964600-2ff1-4419-b69a-2f578d1870a6  south2
UJ  172.18.0.5  ?          256          ?       7a6c23c8-a304-4c7e-8497-6a61e37f4bb1  south3
```
### Ex. 9
```
[pawel@dell-pawel ScyllaTraining]$ sudo docker-compose -f docker-compose-dc2-n4.yml up -d

[pawel@dell-pawel ScyllaTraining]$ sudo docker exec -it DC2_node1 nodetool status
Using /etc/scylla/scylla.yaml as the config file
Datacenter: south
=================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address     Load       Tokens       Owns    Host ID                               Rack
UN  172.18.0.6  316.19 KB  256          ?       8570fa72-09f2-41c2-aa0d-86696cec2242  south1
UN  172.18.0.7  389.58 KB  256          ?       a9964600-2ff1-4419-b69a-2f578d1870a6  south2
UN  172.18.0.8  462.24 KB  256          ?       a2cd371c-f2c6-4ec3-8746-b60e109e7153  south4
UN  172.18.0.5  289.26 KB  256          ?       7a6c23c8-a304-4c7e-8497-6a61e37f4bb1  south3

[pawel@dell-pawel ScyllaTraining]$ sudo docker exec -it DC2_node4 nodetool decommission

[pawel@dell-pawel ScyllaTraining]$ sudo docker exec -it DC2_node1 nodetool status
Using /etc/scylla/scylla.yaml as the config file
Datacenter: south
=================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address     Load       Tokens       Owns    Host ID                               Rack
UN  172.18.0.6  316.19 KB  256          ?       8570fa72-09f2-41c2-aa0d-86696cec2242  south1
UN  172.18.0.7  389.58 KB  256          ?       a9964600-2ff1-4419-b69a-2f578d1870a6  south2
UN  172.18.0.5  289.26 KB  256          ?       7a6c23c8-a304-4c7e-8497-6a61e37f4bb1  south3
```
### Ex. 10
```
[pawel@dell-pawel ScyllaTraining]$ sudo docker stop DC2_node1
[pawel@dell-pawel ScyllaTraining]$ sudo docker rm DC2_node1
```
### Ex. 11
[Procedure #1 to follow](https://docs.scylladb.com/operating-scylla/procedures/cluster-management/replace_seed_node/)  
[Procedure #2 to follow](https://docs.scylladb.com/operating-scylla/procedures/cluster-management/replace_dead_node/)
# Authors
Paweł Krawczyk (pawel.krawczyk@scylladb.com)
# Version History
1.0 - Initial release
