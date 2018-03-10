It is a good idea to create a docker network in advance:

    docker network create cluster1

Start the first node:

    # docker run -d --net=cluster1 --name=node1 -P perconalab/mysql-group-replication --group_replication_bootstrap_group=ON
    docker run -d --net=cluster1 --name=node1 --privileged -P -v $(pwd)/my.cnf:/etc/my.cnf perconalab/mysql-group-replication --group_replication_bootstrap_group=ON

Start the following nodes:

    # docker run -d --net=cluster1 --name=node2 -P perconalab/mysql-group-replication --group_replication_group_seeds='node1:6606' 
    docker run -d --net=cluster1 --name=node2 --privileged -P -v $(pwd)/my.cnf:/etc/my.cnf perconalab/mysql-group-replication --group_replication_group_seeds='node1:6606' 

Add new user:

    docker exec -it node1 mysql

    ALTER USER 'root'@'localhost' IDENTIFIED BY 'MyNewPass';
    CREATE USER 'user'@'%' IDENTIFIED BY 'password';
    GRANT ALL PRIVILEGES ON *.* TO 'user'@'%' WITH GRANT OPTION;
    FLUSH PRIVILEGES;

List nodes and see mapped ports (use localhost/<mapped port> for SQL Client):

    docker ps -a
    
    CONTAINER ID        IMAGE                                COMMAND                  CREATED             STATUS              PORTS                     NAMES
    f6b083d5b51c        perconalab/mysql-group-replication   "/entrypoint.sh --gr…"   27 minutes ago      Up 29 minutes       0.0.0.0:32805->3306/tcp   node2
    4894e0726130        perconalab/mysql-group-replication   "/entrypoint.sh --gr…"   30 minutes ago      Up 33 minutes       0.0.0.0:32804->3306/tcp   node1

Useful info:

    https://dev.mysql.com/doc/refman/5.7/en/group-replication-launching.html

Discovering which mode is the primary (in Single-Primary mode)

    http://lefred.be/content/mysql-group-replication-who-is-the-primary-master/
    
    mysql> SELECT member_host as "primary master"
              FROM performance_schema.global_status         
              JOIN performance_schema.replication_group_members         
              WHERE variable_name = 'group_replication_primary_member'         
                AND member_id=variable_value;
    
    +----------------+
    | primary master |
    +----------------+
    | mysql1         |
    +----------------+

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

Simulating Network Latency:

    docker exec -it node1 bash
    
    yum install iproute
    
    ping www.google.co.uk
    64 bytes from lhr26s04-in-f227.1e100.net (216.58.198.227): icmp_seq=3 ttl=37 time=21 ms
    
    tc qdisc add dev eth0 root netem delay 1000ms
    
    ping www.google.co.uk
    64 bytes from lhr26s04-in-f227.1e100.net (216.58.198.227): icmp_seq=3 ttl=37 time=1021 ms
    
    tc qdisc del dev eth0 root netem

Observations

In Multi-Primary mode the cluster works in a consistent fashion but the system as a whole is dictated by the speed of the slowest node.
So if cause a network latency delay on Node2, and then try to do an update on both Node1 and Node2 of the same record by ID then the transaction on the slowest node will be rolled back.
If you update different records by ID, then the system still runs at the speed of the slowest node but both transactions succeed (i.e. no conflict, all is good (well... slow!))
