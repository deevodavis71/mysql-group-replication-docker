It is a good idea to create a docker network in advance:

    docker network create cluster1

Start the first node:

    docker run -d --net=cluster1 --name=node1 -P perconalab/mysql-group-replication --group_replication_bootstrap_group=ON

Start the following nodes:

    docker run -d --net=cluster1 --name=node2 -P perconalab/mysql-group-replication --group_replication_group_seeds='node1:6606' 

Add new user:

    docker exec -it node1 mysql

    mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'MyNewPass';
    mysql> CREATE USER 'user'@'%' IDENTIFIED BY 'password';
    mysql> GRANT ALL PRIVILEGES ON *.* TO 'user'@'%' WITH GRANT OPTION;
    mysql> FLUSH PRIVILEGES;

List nodes and see mapped ports (use localhost/<mapped port> for SQL Client):

    docker ps -a
    
    CONTAINER ID        IMAGE                                COMMAND                  CREATED             STATUS              PORTS                     NAMES
    f6b083d5b51c        perconalab/mysql-group-replication   "/entrypoint.sh --gr…"   27 minutes ago      Up 29 minutes       0.0.0.0:32805->3306/tcp   node2
    4894e0726130        perconalab/mysql-group-replication   "/entrypoint.sh --gr…"   30 minutes ago      Up 33 minutes       0.0.0.0:32804->3306/tcp   node1

Useful info:

    https://dev.mysql.com/doc/refman/5.7/en/group-replication-launching.html

Interrogating Replication status:

    mysql> SELECT * FROM performance_schema.replication_group_members;
    +---------------------------+--------------------------------------+--------------+-------------+--------------+
    | CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST  | MEMBER_PORT | MEMBER_STATE |
    +---------------------------+--------------------------------------+--------------+-------------+--------------+
    | group_replication_applier | 1d3ce290-244f-11e8-9c16-0242ac130003 | f6b083d5b51c |        3306 | ONLINE       |
    | group_replication_applier | a991972b-244e-11e8-8b58-0242ac130002 | 4894e0726130 |        3306 | ONLINE       |
    +---------------------------+--------------------------------------+--------------+-------------+--------------+
    2 rows in set (0.01 sec)
    
    mysql> SHOW BINLOG EVENTS;
    +-------------------------+------+----------------+------------+-------------+---------------------------------------------------------------------------------------------------------------+
    | Log_name                | Pos  | Event_type     | Server_id  | End_log_pos | Info                                                                                                          |
    +-------------------------+------+----------------+------------+-------------+---------------------------------------------------------------------------------------------------------------+
    | 4894e0726130-bin.000001 |    4 | Format_desc    | 2886926338 |         123 | Server ver: 5.7.14-labs-gr080-log, Binlog ver: 4                                                              |

Testing the database:

    https://www.digitalocean.com/community/tutorials/how-to-measure-mysql-query-performance-with-mysqlslap
    
    Create a DB called testdb, and a table called table1
    
    brew install mysqlslap
     
    mysqlslap --user=user --iterations=50 --concurrency=10 -q "select * from table1" --port=32805 --host=192.168.1.110 --password=password --create-schema=testdb
