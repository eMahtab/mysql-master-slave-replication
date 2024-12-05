# MySQL Master Slave Replication

**A demo showing MySQL data replication from Master to Slave**, in this example we setup a single master and a single slave or replica.

# Replication :
Replication enables data from one database server (known as a source or master) to be copied to one or more database servers (known as replicas or slaves).
Replication is usually asynchronous by default; replicas do not need to be connected permanently to receive updates from a source.

## Note : There can be many Replication topologies possible, in this one we setup a single MySQL Master and a single MySQL Slave(Replica). Data is replicated from the master to the slave/replica but not the other way. In Master-Slave topology data is replicated in only one way that is from master to slave/replica. 

## Step 1 : Create the Docker compose file and execute docker compose up
Below docker-compose.yml declares two services, named as mysql_master and mysql_slave_1 (in docker compose file actual containers are named as mysql-master and mysql-slave-1). We are using **mysql:8.0** as the docker image, and declare root user password as `toor` and create a test database.
**Make sure docker engine is running on your host machine before running the `docker compose up` command.**

```yml
---
version: "2"
services:
  mysql_master:
    image: mysql:8.0
    container_name: mysql-master
    volumes:
      - mysql-master-volume:/tmp
    command:
      [
        "mysqld",
        "--datadir=/tmp/master/data",
        "--log-bin=bin.log",
        "--server-id=1"
      ]
    environment:
      &mysql-default-environment
      MYSQL_ROOT_PASSWORD: toor
      MYSQL_DATABASE: test
      MYSQL_USER: test_user
      MYSQL_PASSWORD: insecure
    ports:
      - "3308:3306"

  mysql_slave_1:
    image: mysql:8.0
    container_name: mysql-slave-1
    volumes:
      - mysql-replica1-volume:/tmp
    command:
      [
        "mysqld",
        "--datadir=/tmp/slave/data1",
        "--log-bin=bin.log",
        "--server-id=2"
      ]
    environment: *mysql-default-environment
    ports:
      - "3309:3306"

volumes:
  mysql-master-volume:
  mysql-replica1-volume:
```

!["Running MySQL Master and Slave as Docker Containers"](docker-compose-up.png?raw=true)

!["MySQL Master and Slave containers"](docker-containers.png?raw=true)

## Step 2 : Create Replication user on Master with REPLICATION SLAVE privilege
Next we need to create a replication user on Master and grant that user `REPLICATION SLAVE` privilege.
To do this, we execute bash against master **`docker exec -it mysql-master bash`** and connect to mysql **`mysql -uroot -ptoor`** running on master, then execute below mysql commands.
```sql
CREATE USER 'replicator'@'%' IDENTIFIED BY 'rotacilper';
GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'%';
FLUSH PRIVILEGES;
```
Here we create a replication user called `replicator` with password `rotacilper` and grant this user **`REPLICATION SLAVE`** privilege, and finally flush privileges.
!["Create Replication user on Master"](create-replication-user.png?raw=true)

## Step 3 : Execute SHOW MASTER STATUS on MySQL Master 
Get the master status, execute the command **`SHOW MASTER STATUS;`** on Mysql Master to find the Binlog file and position.

!["Get Master status"](show-master-status.png?raw=true)

## Step 4 : Connect to MySQL slave and execute CHANGE MASTER TO
Next we need to execute **`CHANGE MASTER TO`** command on MySQL slave. Connect to MySQL slave and execute below command, update MASTER_LOG_FILE and MASTER_LOG_POS values which you get from executing SHOW MASTER STATUS command on MySQL master.
```sql
CHANGE MASTER TO
  MASTER_HOST='mysql_master',
  MASTER_PORT=3306,
  MASTER_USER='replicator',
  MASTER_PASSWORD='rotacilper',
  MASTER_LOG_FILE='bin.000004',
  MASTER_LOG_POS=871,
  GET_MASTER_PUBLIC_KEY=1;
```
!["Execute Change Master to command on MySQL Slave"](change-master-to.png?raw=true)

## Step 5 : Execute START SLAVE on MySQL slave
Execute the command **`START SLAVE;`** on MySQL slave to start the replication, after executing **`START SLAVE;`** you can optionally run **`SHOW REPLICA STATUS;`** to get the status of replica.
One of the most important parameter is **`Seconds_Behind_Source`** which tells how much behind, replica is from master, ideally **`Seconds_Behind_Source`** should always be 0, which means replica is up to date with master.

!["Start slave for replication"](start-slave.png?raw=true)

## Step 6 : See Replication in action
By performing the step 1 to 5, we have set the replication on Slave from Master, now whatever changes are done on master mysql will be replicated to slave automatically.

!["Update Master database"](update-database.png?raw=true)

We create `users` table under test database on the Master and insert records into the users table.

!["Replication on Slave"](replication-on-slave.png?raw=true)

## Step 6a : See Replication in action : update on master database are replicated to slave
We delete 4 users having id as either 3, 7, 9 or 10 on the mysql master.

!["Deleting some of the user records"](delete-records-on-master.png?raw=true)

The changes in mysql master are replicated to mysql replica.

!["Replication on slave"](slave-after-delete-on-master.png?raw=true)

## !Important :  Data doesn't replicate from replica to master

**Creating a table on replica and inserting a record**

!["In master-slave replication data is not replicated from slave to master"](write-on-replica.png?raw=true)

**Data is not replicated to master in Master-Slave replication**

!["Unidirectional Replication"](unidirectional-replication.png?raw=true)

# References :
1. https://victoronsoftware.com/posts/mysql-master-slave-replication/
2. https://www.youtube.com/watch?v=nMbb1199HQU
3. https://dev.mysql.com/doc/refman/8.4/en/replication.html
4. https://dev.mysql.com/doc/refman/8.4/en/faqs-replication.html
