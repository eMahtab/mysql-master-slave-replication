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

## Step 2 : Create Replication user on Master
Next we need to create a replication user on Master and grant that user `REPLICATION SLAVE` privilege.
To do this, we execute bash against Master and connect to mysql, then execute below mysql commands.
```sql
CREATE USER 'replicator'@'%' IDENTIFIED BY 'rotacilper';
GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'%';
FLUSH PRIVILEGES;
```
Here we create a replication user called `replicator` with password `rotacilper` and grant this user REPLICATION SLAVE privilege, and finally flush privileges.
!["Create Replication user on Master"](create-replication-user.png?raw=true)
