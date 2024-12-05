# MySQL Master Slave Replication

**A demo showing MySQL data replication from Master to Slave**


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
