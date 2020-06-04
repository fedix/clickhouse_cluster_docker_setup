# clickhouse_cluster_docker_setup

Guide for setting up a ClickHouse cluster using multiple docker containers

1. Install docker

2. Create docker containers:

   Create a folder for your data, config.xml and users.xml files. We will later mount these files to docker containers. This is handy becuase all the changes in files outside the containers will be automatically applied to the files inside containers too.

   We connect the first instance to localhost:8123 for http requests to clickhouse(e.g., using tabix):

   ```bash
   sudo docker run -d --name clickhouse_1 \
       --ulimit nofile=262144:262144 \
       -p 127.0.0.1:8123:8123 -p 127.0.0.1:9000:9000 \
       -v /Users/f/dev/clickhouse/etc/users.xml:/etc/clickhouse-server/users.xml \
       -v /Users/f/dev/clickhouse/etc/config.xml:/etc/clickhouse-server/config.xml \
       -v /Users/f/dev/clickhouse/transactions.json:/transactions.json \
       yandex/clickhouse-server
   ```

   Almost the same for the second container:

   ```bash
   sudo docker run -d --name clickhouse_2 \
       --ulimit nofile=262144:262144 \
       -v /Users/f/dev/clickhouse/etc/users.xml:/etc/clickhouse-server/users.xml \
       -v /Users/f/dev/clickhouse/etc/config.xml:/etc/clickhouse-server/config.xml \
       -v /Users/f/dev/clickhouse/transactions.json:/transactions.json \
       yandex/clickhouse-server
   ```

3. You can check if containers were created by running 

   ```bash
   docker ps
   ```

4. You can download config.xml and users.xml from my repository (for lazy ones) or get them from container by running

   ```bash
   sudo docker exec -it clickhouse_1 cat /etc/clickhouse-server/config.xml > config.xml
   sudo docker exec -it clickhouse_1 cat /etc/clickhouse-server/users.xml > users.xml
   ```

5. In users.xml provide password for default user and add more users if needed (users section):

   ```xml
           <qwerty>
                   <password>qwerty</password>
                   <networks incl="networks" replace="replace">
                           <ip>::/0</ip>
                   </networks>
                   <profile>default</profile>
                   <quota>default</quota>
           </qwerty>
   ```

   

6. In config.xml allow connecting from everywhere by uncommenting this line:

   ```xml
   <listen_host>0.0.0.0</listen_host>
   ```

7. Configure a cluster in config.xml (remote_servers section):

   ```xml
   <remote_servers incl="clickhouse_remote_servers" >
   		<awesome_cluster>
           <shard>
               <replica>
                   <host>172.17.0.2</host>
                   <port>9000</port>
                   <user>default</user>
                   <password></password>
               </replica>
           </shard>
           
           <shard>
               <replica>
                   <host>172.17.0.3</host>
                   <port>9000</port>
                   <user>default</user>
                   <password></password>
               </replica>
           </shard>
       </awesome_cluster>
   </remote_servers>
   ```

   Host adress can be accessed by running:

   ```xml
   docker inspect *container_name*
   ```

   Check the "IPAddress" field in the outputted .json file for host address.

8. Run

   ```bash
   clickhouse-client -u qwerty --password=qwerty
   ```

   and then in clickhouse-client run

   ```sql
   select * from system.clusters
   ```

   Proceed if you see a cluster consisting of your docker containers and one of the shards is_local.

9. Now it's time to create a database. In clickhouse-client run

   ```sql
   create database practice
   ```

   Then create a table

   ```sql
   create table practice.transactions
   (
   amount Nullable(Float64), 
   datetime DateTime,
   important Nullable(UInt64),
   user_id_in Int64,
   user_id_out Nullable(Int64),
   __index_level_0__ Int64
   )
   ENGINE = ReplacingMergeTree(datetime)
   PARTITION BY toMonth(datetime)
   ORDER BY (__index_level_0__)
   SETTINGS
       index_granularity=4096;
   ```

   Do it on both containers. Then create a distributed table on both containers as well:

   ```sql
   create table practice.distr_transactions AS practice.transactions
   ENGINE = Distributed(awesome_cluster, practice, transactions, xxHash64(__index_level_0__))
   ```

   9. Finally insert the data into. On one of ClickHouse instances run

      ```bash
      cat transactions.json | clickhouse-client -u qwerty --password=qwerty --query="INSERT INTO practice.distr_transactions FORMAT JSONEachRow"
      ```

      Be careful, I converted the transactions.parquet to transactions.json because inserting in parquet didn't work for me for some reason. But you can try it this way:

      ```bash
      cat transactions.parquet | clickhouse-client -u qwerty --password=qwerty --query="INSERT INTO practice.distr_transactions FORMAT Parquet"
      ```

   10. Check if data was actually distributed by running queries inside both containers:

       ```sql
       select count() from practice.transactions
       select count() from practice.distr_transactions
       ```

       and pray for good...
