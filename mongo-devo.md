### Auth theory
- Mongo has two levels of auth
    - Internal (node-to-node)
        - Controlled by the keyFile.
        - As long as every mongod and mongos uses the same key file, they can all talk to each other.
        - This has nothing to do with usernames/passwords.
    - External (humans & applications)
        - Controlled by the users you create in admin.
        - You can create them individually on any shard primary, config primary, or via mongos.
        - These do not interfere with the internal cluster authentication.
        - But this file has to be common across all the nodes
- So i can create the users individually as well and it wont hamper the connection between configs, router and shards? Yes
- ✅ Safe Pattern
    - Bring up your cluster with keyFile + authorization: enabled everywhere.
    - Cluster forms successfully (because all instances share the key file).
    - Then you connect via the localhost exception (first login without credentials on each primary if no users exist yet), or temporarily run without authorization: enabled just once.
    - Create admin/root user(s).
    - Restart instances with full auth.
    - Now:
    - Config servers ↔ Shards ↔ Mongos trust each other (keyFile).
    - You/applications log in with users (username/password)
- Setup 
    - machine1: shard1 (with two mongo servers)
    - machine2: shard2 (with two mongo servers)
    - machine3: running configserver (with one replica of config server only) and mongos (router)


### Generating keyfile and copying to all the nodes
```bash
shard1=10.21.14.17
shard2=10.21.3.192
router=10.21.31.167
pemkeyfile=mongo-shard.pem

openssl rand -base64 756 > mongokey
chmod 400 mongokey
ls -ltr | grep mongokey


# --- Shard 1 ---
scp -i mongo-shard.pem mongokey ubuntu@10.21.14.17:/tmp/mongokey
ssh -i mongo-shard.pem ubuntu@10.21.14.17 "sudo mkdir -p /etc/mongodb/keyfile && sudo mv /tmp/mongokey /etc/mongodb/keyfile/mongokey && sudo chown mongodb:mongodb /etc/mongodb/keyfile/mongokey && sudo chmod 400 /etc/mongodb/keyfile/mongokey"
ssh -i mongo-shard.pem ubuntu@10.21.14.17 "ls -l /etc/mongodb/keyfile/mongokey"

# --- Shard 2 ---
scp -i mongo-shard.pem mongokey ubuntu@10.21.3.192:/tmp/mongokey
ssh -i mongo-shard.pem ubuntu@10.21.3.192 "sudo mkdir -p /etc/mongodb/keyfile && sudo mv /tmp/mongokey /etc/mongodb/keyfile/mongokey && sudo chown mongodb:mongodb /etc/mongodb/keyfile/mongokey && sudo chmod 400 /etc/mongodb/keyfile/mongokey"
ssh -i mongo-shard.pem ubuntu@10.21.3.192 "ls -l /etc/mongodb/keyfile/mongokey"

# --- Router ---
scp -i mongo-shard.pem mongokey ubuntu@10.21.31.167:/tmp/mongokey
ssh -i mongo-shard.pem ubuntu@10.21.31.167 "sudo mkdir -p /etc/mongodb/keyfile && sudo mv /tmp/mongokey /etc/mongodb/keyfile/mongokey && sudo chown mongodb:mongodb /etc/mongodb/keyfile/mongokey && sudo chmod 400 /etc/mongodb/keyfile/mongokey"
ssh -i mongo-shard.pem ubuntu@10.21.31.167 "ls -l /etc/mongodb/keyfile/mongokey"
```


### shard-1
```bash
ssh -i mongo-shard.pem ubuntu@10.21.14.17

## 0. Cleanup
sudo apt install net-tools -y
netstat -tulnp
ps aux | grep mongo
sudo systemctl stop mongod
sudo pkill -f "mongod --shardsvr"
sudo pkill -f mongos

# remove shard data
rm -rf ~/shard-demo/shardrep1/*
rm -rf ~/shard-demo/shardrep2/*
rm -rf ~/shard-demo/shardrep3/*
rm -rf ~/shard-demo/config-*
sudo rm -rf /var/lib/mongodb/* 
ps aux | grep mongo
netstat -tulnp


# 1. Create Directories for Each Instance
sudo mkdir -p /data/shard1/node1 /data/shard1/node2
sudo mkdir -p /var/log/mongodb/shard1-node1 /var/log/mongodb/shard1-node2
sudo chown -R mongodb:mongodb /data/shard1 /var/log/mongodb/shard1-node*


# 2. Create conf files
sudo vi /etc/mongod-shard1-node1.conf
"""
systemLog:
  destination: file
  path: /var/log/mongodb/shard1-node1/mongod.log
  logAppend: true

storage:
  dbPath: /data/shard1/node1
net:
  port: 27017
  bindIp: 0.0.0.0

replication:
  replSetName: shard1

sharding:
  clusterRole: shardsvr

security:
  authorization: enabled
  keyFile: /etc/mongodb/keyfile/mongokey
"""


sudo vi /etc/mongod-shard1-node2.conf
"""
systemLog:
  destination: file
  path: /var/log/mongodb/shard1-node2/mongod.log
  logAppend: true

storage:
  dbPath: /data/shard1/node2
net:
  port: 27018
  bindIp: 0.0.0.0

replication:
  replSetName: shard1

sharding:
  clusterRole: shardsvr

security:
  authorization: enabled
  keyFile: /etc/mongodb/keyfile/mongokey
"""


# 3. Create Systemd Units
sudo vi /etc/systemd/system/mongod-shard1-node1.service
"""
[Unit]
Description=MongoDB Shard1 Node1
After=network.target

[Service]
User=mongodb
Group=mongodb
ExecStart=/usr/bin/mongod --config /etc/mongod-shard1-node1.conf
PIDFile=/var/run/mongodb-shard1-node1.pid
Restart=always
LimitNOFILE=64000

[Install]
WantedBy=multi-user.target
"""

sudo vi /etc/systemd/system/mongod-shard1-node2.service
"""
[Unit]
Description=MongoDB Shard1 Node2
After=network.target

[Service]
User=mongodb
Group=mongodb
ExecStart=/usr/bin/mongod --config /etc/mongod-shard1-node2.conf
PIDFile=/var/run/mongodb-shard1-node2.pid
Restart=always
LimitNOFILE=64000

[Install]
WantedBy=multi-user.target
"""


sudo systemctl daemon-reload
sudo systemctl enable mongod-shard1-node1 mongod-shard1-node2
sudo systemctl start mongod-shard1-node1 mongod-shard1-node2
netstat -tulnp

# 4. Initiate the Replica Set
mongosh --port 27017
>> rs.initiate({
  _id: "shard1",
  members: [
    { _id: 0, host: "10.21.14.17:27017" },
    { _id: 1, host: "10.21.14.17:27018" }
  ]
})
```


### shard-2
```bash
ssh -i mongo-shard.pem ubuntu@10.21.3.192

## 0. Cleanup
sudo apt install net-tools -y
netstat -tulnp
ps aux | grep mongo
sudo systemctl stop mongod
sudo pkill -f "mongod --shardsvr"
sudo pkill -f mongos

# remove shard data
rm -rf ~/shard-demo/shard2rep1/*
rm -rf ~/shard-demo/shard2rep2/*
rm -rf ~/shard-demo/shard2rep3/*
sudo rm -rf /var/lib/mongodb/* 
ps aux | grep mongo
netstat -tulnp


# 1. Create Directories for Each Instance
sudo mkdir -p /data/shard2/node1 /data/shard2/node2
sudo mkdir -p /var/log/mongodb/shard2-node1 /var/log/mongodb/shard2-node2
sudo chown -R mongodb:mongodb /data/shard2 /var/log/mongodb/shard2-node*


# 2. Create conf files
sudo vi /etc/mongod-shard2-node1.conf
"""
systemLog:
  destination: file
  path: /var/log/mongodb/shard2-node1/mongod.log
  logAppend: true

storage:
  dbPath: /data/shard2/node1

net:
  port: 27017
  bindIp: 0.0.0.0

replication:
  replSetName: shard2

sharding:
  clusterRole: shardsvr

security:
  authorization: enabled
  keyFile: /etc/mongodb/keyfile/mongokey
"""


sudo vi /etc/mongod-shard2-node2.conf
"""
systemLog:
  destination: file
  path: /var/log/mongodb/shard2-node2/mongod.log
  logAppend: true

storage:
  dbPath: /data/shard2/node2

net:
  port: 27018
  bindIp: 0.0.0.0

replication:
  replSetName: shard2

sharding:
  clusterRole: shardsvr

security:
  authorization: enabled
  keyFile: /etc/mongodb/keyfile/mongokey
"""


# 3. Create Systemd Units
sudo vi /etc/systemd/system/mongod-shard2-node1.service
"""
[Unit]
Description=MongoDB Shard2 Node1
After=network.target

[Service]
User=mongodb
Group=mongodb
ExecStart=/usr/bin/mongod --config /etc/mongod-shard2-node1.conf
PIDFile=/var/run/mongodb-shard2-node1.pid
Restart=always
LimitNOFILE=64000

[Install]
WantedBy=multi-user.target
"""

sudo vi /etc/systemd/system/mongod-shard2-node2.service
"""
[Unit]
Description=MongoDB Shard2 Node2
After=network.target

[Service]
User=mongodb
Group=mongodb
ExecStart=/usr/bin/mongod --config /etc/mongod-shard2-node2.conf
PIDFile=/var/run/mongodb-shard2-node2.pid
Restart=always
LimitNOFILE=64000

[Install]
WantedBy=multi-user.target
"""


sudo systemctl daemon-reload
sudo systemctl enable mongod-shard2-node1 mongod-shard2-node2
sudo systemctl start mongod-shard2-node1 mongod-shard2-node2
netstat -tulnp

# 4. Initiate the Replica Set
mongosh --port 27017
>> rs.initiate({
  _id: "shard2",
  members: [
    { _id: 0, host: "10.21.3.192:27017" },
    { _id: 1, host: "10.21.3.192:27018" }
  ]
})
```


### router
```bash
ssh -i mongo-shard.pem ubuntu@10.21.31.167

## 0. Cleanup
sudo apt install net-tools -y
netstat -tulnp
ps aux | grep mongo
sudo systemctl stop mongod
sudo pkill -9 mongod
sudo pkill -9 mongos
ps aux | grep mongo
netstat -tulnp

# remove socket files and data
sudo rm -f /tmp/mongodb-*.sock
sudo rm -rf ~/shard-demo/configsrv
sudo rm -rf ~/shard-demo/configsrv1
sudo rm -rf ~/shard-demo/configsrv2
sudo ls -l ~/shard-demo/



# 1. Create Directories for Each Instance
# Config server data
sudo mkdir -p /data/configdb
sudo chown -R mongodb:mongodb /data/configdb

# Config server log
sudo mkdir -p /var/log/mongodb/config
sudo chown -R mongodb:mongodb /var/log/mongodb/config

# Mongos router log
sudo mkdir -p /var/log/mongodb/mongos
sudo chown -R mongodb:mongodb /var/log/mongodb/mongos


# 2. Create conf files for config-server
sudo vi /etc/mongod-configsvr.conf
"""
systemLog:
  destination: file
  path: /var/log/mongodb/config/mongod.log
  logAppend: true

storage:
  dbPath: /data/configdb

net:
  port: 27019
  bindIp: 0.0.0.0

processManagement:
  timeZoneInfo: /usr/share/zoneinfo

replication:
  replSetName: config_repl

sharding:
  clusterRole: configsvr

security:
  authorization: enabled
  keyFile: /etc/mongodb/keyfile/mongokey
"""

# 3. Create Systemd Units for config-server
sudo vi /etc/systemd/system/mongod-configsvr.service
"""
[Unit]
Description=MongoDB Config Server
After=network.target

[Service]
User=mongodb
Group=mongodb
ExecStart=/usr/bin/mongod --config /etc/mongod-configsvr.conf
PIDFile=/var/run/mongodb-configsvr.pid
Restart=always
LimitNOFILE=64000

[Install]
WantedBy=multi-user.target
"""

sudo systemctl daemon-reload
sudo systemctl enable mongod-configsvr
sudo systemctl start mongod-configsvr
sudo systemctl status mongod-configsvr

# 4. Initiate the Replica Set
mongosh --port 27019

rs.initiate({
  _id: "config_repl",
  configsvr: true,
  members: [
    { _id: 0, host: "10.21.31.167:27019" }
  ]
})

# Create admin user for the cluster
use admin
db.createUser({
  user: "routerAdmin",
  pwd: "RouterPass123!",
  roles: [ { role: "root", db: "admin" } ]
})


# 5. Configure mongos (router)
sudo vi /etc/mongos.conf
"""
systemLog:
  destination: file
  path: /var/log/mongodb/mongos/mongos.log
  logAppend: true

net:
  port: 27020
  bindIp: 0.0.0.0

sharding:
  configDB: config_repl/10.21.31.167:27019

security:
  keyFile: /etc/mongodb/keyfile/mongokey
"""

# 6. Systemd Unit for mongos (router)
sudo vi /etc/systemd/system/mongos.service 
"""
[Unit]
Description=MongoDB Router
After=network.target

[Service]
User=mongodb
Group=mongodb
ExecStart=/usr/bin/mongos --config /etc/mongos.conf
PIDFile=/var/run/mongos.pid
Restart=always
LimitNOFILE=64000

[Install]
WantedBy=multi-user.target
"""

sudo systemctl daemon-reload
sudo systemctl enable mongos
sudo systemctl start mongos
sudo systemctl status mongos


# 7. Add Shards to the Cluster
mongosh --port 27020
use admin
db.auth("routerAdmin", "RouterPass123!")

sh.addShard("shard1/10.21.14.17:27017,10.21.14.17:27018")
sh.addShard("shard2/10.21.3.192:27017,10.21.3.192:27018")
sh.status()

# 8. Validate sharding
sh.enableSharding("sharding_db_test")
sh.status()
use sharding_db_test
db.createCollection("test_collection")
sh.shardCollection("sharding_db_test.test_collection", { "userId": 1 })
for (let i = 1; i <= 20; i++) {
    db.test_collection.insertOne({ userId: i, name: "User" + i })
}
db.test_collection.getShardDistribution()
db.test_collection.find({ userId: 5 })

>> "mongodb://routerAdmin:RouterPass123!@10.21.31.167:27020/admin"


# 9. Shard-Level Admin Users (Optional)
# Shard1 primary
# remember: this will only with localhost exception
ssh -i mongo-shard.pem ubuntu@10.21.14.17
mongosh --port 27017
use admin
db.createUser({ user: "shard1Admin", pwd: "Shard1Passbnhwidy921!", roles: [ { role: "root", db: "admin" } ] })

# Shard2 primary
ssh -i mongo-shard.pem ubuntu@10.21.3.192
mongosh --port 27017
use admin
db.createUser({ user: "shard2Admin", pwd: "Shard2Pass!123nni1d", roles: [ { role: "root", db: "admin" } ] })
```