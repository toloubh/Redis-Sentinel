# Redis-Sentinel -High availability with Redis Sentinel
Installing Redis Master-Slave with Sentinel for Auto Failover

--------------------
Redis Sentinel provides high availability for Redis when not using Redis Cluster.
Redis Sentinel is a distributed system:

Sentinel itself is designed to run in a configuration where there are multiple Sentinel processes cooperating together. The advantage of having multiple Sentinel processes cooperating are the following:

Failure detection is performed when multiple Sentinels agree about the fact a given master is no longer available. This lowers the probability of false positives.
Sentinel works even if not all the Sentinel processes are working, making the system robust against failures. There is no fun in having a failover system which is itself a single point of failure, after all.
The sum of Sentinels, Redis instances (masters and replicas) and clients connecting to Sentinel and Redis, are also a larger distributed system with specific properties.

--------------------

Obtaining Sentinel 
The current version of Sentinel is called Sentinel 2. It is a rewrite of the initial Sentinel implementation using stronger and simpler-to-predict algorithms (that are explained in this documentation).

A stable release of Redis Sentinel is shipped since Redis 2.8.

New developments are performed in the unstable branch, and new features sometimes are back ported into the latest stable branch as soon as they are considered to be stable.

Redis Sentinel version 1, shipped with Redis 2.6, is deprecated and should not be used.
* Sentinels by default run listening for connections to TCP port 26379, so for Sentinels to work, port 26379 of your servers must be open to receive connections from the IP addresses of the other Sentinel instances. Otherwise Sentinels can't talk and can't agree about what to do, so failover will never be performed.
--------------------

***** Step-by-Step: Add Redis Repository on Ubuntu *****
1. Install prerequisites:
```
sudo apt update
sudo apt install -y curl gnupg lsb-release
```
2. Add the Redis GPG key:
```
curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg
```
3. Add the Redis repository:
```
echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list
```
4. Update apt and install Redis:
```
sudo apt update
sudo apt install redis-server
sudo apt install redis-sentinel
```
5. Verify Redis Installation
redis-server --version
sudo systemctl status redis-server.service
We can repeat the installation on each node.
--------------------
***** Setting up Redis Master and Slave *****
After the installation is finished, the next part is we need to configure the Redis master and slave node. Redis configuration resides on the /etc/redis/redis.conf path and there are some parameters we need to adjust to ensure the service can be accessed from another host. The parameters are:
```
bind 127.0.0.1 192.168.13.X
masterauth MasterPassword # masterauth is used for authentication between the master and slave.
requirepass MasterPassword
protected-mode no
```
*** In the Redis slave nodes, we need to configure the Redis master node, master user and auth as below:
```
bind: 127.0.0.1 192.168.13.x
masterauth MasterPassword
requirepass MasterPassword
replicaof 192.168.13.x 6379
```
After that, we need to restart the Redis service and test and check the replication process.

*** To configure and integrate the Sentinel with Redis Server, we need to add the parameters in "sentinel.conf".
****** Starting with Redis 6, user authentication and permission is managed with the Access Control List (ACL).
In order for Sentinels to connect to Redis server instances when they are configured with requirepass, the Sentinel configuration must include the sentinel auth-pass directive, in the format:
```
#sentinel auth-pass <master-name> <password>
```
```
sentinel monitor redis-master 192.168.13.x 6379 2
sentinel auth-pass redis-master MasterPassword
sentinel down-after-milliseconds redis-master 1500
sentinel failover-timeout redis-master 3000
protected-mode no

```
After the configuration was added, we needed to restart the Redis service and Sentinel service.
```
root@redis-master:~# systemctl restart redis-server
root@redis-master:~# systemctl restart redis-sentinel
```
Finally, after the configuration has been finished, we can verify that the Sentinels work by grabbing the heartbeat in the Sentinel logs:
```
root@redis-master:~# tail -f /var/log/redis/redis-sentinel.log
14454:X 02 Jul 15:16:49.895 * +sentinel-address-switch master redis-master 192.168.13.50 6379 ip 192.168.13.52 port 26379 for cc9f9d652f2f870d1725d74fdd99d3edaf97294b

14454:X 02 Jul 15:16:49.895 * +sentinel-address-update sentinel cc9f9d652f2f870d1725d74fdd99d3edaf97294b 192.168.13.52 26379 @ redis-master 192.168.13.50 6379 1 additional matching instances

14454:X 02 Jul 15:16:51.742 * +sentinel-address-switch master redis-master 192.168.13.50 6379 ip 192.168.13.50 port 26379 for cc9f9d652f2f870d1725d74fdd99d3edaf97294b

14454:X 02 Jul 15:16:51.742 * +sentinel-address-update sentinel cc9f9d652f2f870d1725d74fdd99d3edaf97294b 192.168.13.50 26379 @ redis-master 192.168.13.50 6379 1 additional matching instances

14454:X 02 Jul 15:16:52.014 * +sentinel-address-switch master redis-master 192.168.13.50 6379 ip 192.168.13.52 port 26379 for cc9f9d652f2f870d1725d74fdd99d3edaf97294b

14454:X 02 Jul 15:16:52.015 * +sentinel-address-update sentinel cc9f9d652f2f870d1725d74fdd99d3edaf97294b 192.168.13.52 26379 @ redis-master 192.168.13.50 6379 1 additional matching instances
```
And check the replication information:
```
root@redis-master:/var/log/redis# redis-cli -a Masterpassword

192.168.13.50:6379> info replication

# Replication

role:master

connected_slaves:2

slave0_ip=192.168.13.52,port=6379,state=online,offset=2781219,lag=1

slave1_ip=192.168.13.53,port=6379,state=online,offset=2781503,lag=1

master_replid:075e651ef398ad202003105053091b4e6b43d323

master_replid2:0000000000000000000000000000000000000000

master_repl_offset:2781503

second_repl_offset:-1

repl_backlog_active:1

repl_backlog_size:1048576

repl_backlog_first_byte_offset:1732928

repl_backlog_histlen:1048576

192.168.13.50:6379>
```
