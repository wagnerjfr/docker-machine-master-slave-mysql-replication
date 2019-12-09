# docker-machine-master-slave-mysql-replication
This project: 
- creates two virtual box hosts using docker-machine
- deploys one MySQL container in which of the hosts
- sets up master ➡ slave replication

For running the steps below, it's better to open 2 terminals and run the commands separately.
- `Terminal 1`: **MASTER**
- `Terminal 2`: **SLAVE**

***References***: [Replication with Docker MySQL Images](https://github.com/wagnerjfr/mysql-master-slaves-replication-docker)

## 1. Creating the hosts master and slave
In `Terminal 1`, run the command below:
```
$ docker-machine create --driver virtualbox master
```
At the same time, you can run the command below in `Terminal 2`:
```
$ docker-machine create --driver virtualbox slave
```
It will take some seconds till the virtual boxes are up and running.

Both will output somethig similar to:
```console
Running pre-create checks...
Creating machine...
(master) Copying /home/wfranchi/.docker/machine/cache/boot2docker.iso to /home/wfranchi/.docker/machine/machines/master/boot2docker.iso...
(master) Creating VirtualBox VM...
(master) Creating SSH key...
(master) Starting the VM...
(master) Check network to re-create if needed...
(master) Waiting for an IP...
Waiting for machine to be running, this may take a few minutes...
Detecting operating system of created instance...
Waiting for SSH to be available...
Detecting the provisioner...
Provisioning with boot2docker...
Copying certs to the local machine directory...
Copying certs to the remote machine...
Setting Docker configuration on the remote daemon...

This machine has been allocated an IP address, but Docker Machine could not
reach it successfully.

SSH for the machine should still work, but connecting to exposed ports, such as
the Docker daemon port (usually <ip>:2376), may not work properly.

You may need to add the route manually, or use another related workaround.

This could be due to a VPN, proxy, or host file configuration issue.

You also might want to clear any VirtualBox host only interfaces you are not using.
Checking connection to Docker...
Docker is up and running!
To see how to connect your Docker Client to the Docker Engine running on this virtual machine, run: docker-machine env master
```
## 2. Deploying the MySQL Docker container in the hosts
In `Terminal 1` run:
```
$ docker-machine ssh master docker run -d --rm --name=master_mysql --hostname=master \
  -v $PWD/d0:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=mypass \
  -p 3306:3306 \
  mysql/mysql-server:8.0 \
  --server-id=1 \
  --log-bin='mysql-bin-1.log'
```
In `Terminal 2` run:
```
$ docker-machine ssh slave docker run -d --rm --name=slave_mysql --hostname=slave \
  -v $PWD/d1:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=mypass \
  mysql/mysql-server:8.0 \
  --server-id=2
```
In both terminals, we need to wait for the MySQL container to be with status `healthy`.

It's possible to check it by running:
```
$ docker-machine ssh master docker ps -a

$ docker-machine ssh slave docker ps -a
```

## 3. Setting up MySQL Replication
In `Terminal 1`, ssh into the host:
```
$ docker-machine ssh master
```
Finally run:
```
$ docker exec -it master_mysql mysql -uroot -pmypass \
  -e "CREATE USER 'repl'@'%' IDENTIFIED WITH mysql_native_password BY 'slavepass';" \
  -e "GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';" \
  -e "SHOW MASTER STATUS;"
```
Output:
```console
mysql: [Warning] Using a password on the command line interface can be insecure.
+--------------------+----------+--------------+------------------+-------------------+
| File               | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+--------------------+----------+--------------+------------------+-------------------+
| mysql-bin-1.000003 |      595 |              |                  |                   |
+--------------------+----------+--------------+------------------+-------------------+
```

In `Terminal 2`,  let's first grab the master host IP:
```
MASTER_IP=$(docker-machine ip master)
echo $MASTER_IP
```
My outuput is:
```console
192.168.99.100
```
Then, ssh into the host:
```
$ docker-machine ssh slave
```
Change the value of `MASTER_HOST='192.168.99.100'` if your master IP is different, before running the command below:
```
$ docker exec -t slave_mysql mysql -uroot -pmypass \
    -e "CHANGE MASTER TO MASTER_HOST='192.168.99.100', MASTER_USER='repl', \
      MASTER_PASSWORD='slavepass', MASTER_LOG_FILE='mysql-bin-1.000003';"
```
Then run:
```
$ docker exec slave_mysql mysql -uroot -pmypass -e "START SLAVE;"
```
Finally, to check whether it's ok, run:
```
$ docker exec slave_mysql mysql -uroot -pmypass -e "SHOW SLAVE STATUS\G"
```
Output:
```console
mysql: [Warning] Using a password on the command line interface can be insecure.
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.99.100
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin-1.000003
          Read_Master_Log_Pos: 595
               Relay_Log_File: slave-relay-bin.000002
                Relay_Log_Pos: 812
        Relay_Master_Log_File: mysql-bin-1.000003
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
            ....
```
Note that both `Slave_IO_Running: Yes` and `Slave_SQL_Running: Yes` are running.

## 4. Adding some data:

Now it's time to test whether data is replicated to slaves. We are going to create a new database named "TEST" in master.

In `Terminal 1`, run:
```
$ docker exec -it master_mysql mysql -uroot -pmypass -e "CREATE DATABASE TEST; SHOW DATABASES;"
```
Output:
```console
mysql: [Warning] Using a password on the command line interface can be insecure.
+--------------------+
| Database           |
+--------------------+
| information_schema |
| TEST               |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
```
In `Terminal 2`, run the code below to check whether the database was replicated:
```
$ docker exec -t slave_mysql mysql -uroot -pmypass -e "SHOW DATABASES";
```
Output:
```console
mysql: [Warning] Using a password on the command line interface can be insecure.
+--------------------+
| Database           |
+--------------------+
| information_schema |
| TEST               |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
```
## 5. Clean up
In `Terminal 1` (or `Terminal 2`), type `exit` and then press `ENTER`.

Run the command below to delete the hosts:
```
$ docker-machine rm -f master slave
```
Output:
```console
About to remove master, slave
WARNING: This action will delete both local reference and remote instance.
Successfully removed master
Successfully removed slave
```
