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

Useful info:

    https://dev.mysql.com/doc/refman/5.7/en/group-replication-launching.html
