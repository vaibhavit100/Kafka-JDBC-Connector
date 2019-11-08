Installation Steps:

 

1) Install Docker

2)  Launch Kafka cluster based  on  landoop image in docker-compose file since landoop image has already loaded lot of connectors in the image. Run this command after downloading docker-compose file from here:

## docker-compose.yml

version: '2'<br>
services:<br>
  // this is our kafka cluster.<br>
  kafka-cluster:
    image: landoop/fast-data-dev:cp3.3.0<br>
    environment:
      ADV_HOST: 127.0.0.1        // Change to 192.168.99.100 if using Docker Toolbox<br>
      RUNTESTS: 0                // Disable Running tests so the cluster starts faster<br>
    ports:
      - 2181:2181                 // Zookeeper<br>
      - 3030:3030                 // Landoop UI<br>
      - 8081-8083:8081-8083       // REST Proxy, Schema Registry, Kafka Connect ports<br>
      - 9581-9585:9581-9585       // JMX Ports<br>
      - 9092:9092                 // Kafka Broker<br>



Start kafka Cluster  by the command:

docker-compose up kafka-cluster

3) Table Structure





4) Once kafka cluster is up, create a Kafka connect cluster:

 a) Download Kafka at https://kafka.apache.org/downloads and unzip Kafka to a directory of your choice

 b) Go to Kafka/config/distributed-config properties and change bootstrap.servers to the correct ones(based on landoop image, set it as 127.0.0.1:9092)

c) Uncomment rest.port to 8083 or  change to 8084 (or any available port if process is running on 8083 or 8084)

d) Set plugin.path=/directory/of/your/choice

e)  Download mysql connector jar from here and place in kafka/lib folder

f)  Download kafka jdbc connector (jars)  from here and set its path in Kafka/config/distributed-config under plugin.path 

g) Start the connector by running commands from kafka folder:

bin/connect-distributed.sh config/connect-distributed.properties



5) Create Kafka Connector Task by calling REST API: localhost:8083/connectors

Note: Port number should be same as present as rest.port  in Kafka/config/distributed-config properties 

Check all configurable kafka connect properties from here



Sample payload:

{
"name": "jdbc_source_mysql_account",
"config": {
"connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
"connection.url": "jdbc:mysql://127.0.0.1:3306/kafka_connect_test?user=root&password=password",
"table.whitelist": "accounts",
"mode": "incrementing",
 
"incrementing.column.name": "id",
"validate.non.null": "false",
"topic.prefix": "mysql-account"
  }
}
 

It will create the topic automatically based on topic.prefix value above  and append table name in the topic (topic.prefix+table name), you can view topic name and data at http://127.0.0.1:3030/



 

6) Create Kafka console consumer by running below command from kafka folder:

bin/kafka-console-consumer.sh --topic topicname  --bootstrap-server 127.0.0.1:9092

7) (Optional) Use kafka Connect UI to view Kafka Connector task by running below command:

docker run --rm -it -p 8000:8000  -e "CONNECT_URL=http://docker.for.mac.localhost:8085"  landoop/kafka-connect-ui

Access link : http://0.0.0.0:8000/



 

## Findings:

## Testing With Auto-Increment Id 

1) Created table with  Auto-Increment Id as primaryKey and 15 column and and 1 million rows

2) Started Kafka Connector and created single task with increment mode on Id



3) On starting kafka console consumer, initially it took almost 6 minutes to read entire 1 miliions rows with default settings like batch.max.rows 100 and poll.interval.ms 5000

4) On inserting new column with increasting Id, seeing events in kafka consumer

5) But suppose Auto-Increment  id order is 1,4,5,9 then on inserting new rows with id as 2, there is no event on kafka

6) Updating existing data doesn't result in any event in kafka

7) Deleting existing rows doesn't result in any event in kafka

8) Following scenario also doesn't result in any kafka event:

 a) Auto-Increment id order is 1,4,5,9

 b) Deletion of row with Id 9 and 

 c)  Inserting with Id 6, 7,8,9

But it works while inserting data with id 10, means kafka connect set the offset internally with the latest Auto-Increment id(9 in this case) and it remains same even after deletion of that Id, so to get the next row, Auto-IncrementId while insertion should be greater than the deleted one which was latest earlier.



## Testing With Timestamp Column:

1) Created table with  timestamp as primaryKey and 15 column and and 1 million rows

2) Started Kafka Connector and created single task with timestamp mode

3) On starting kafka console consumer, initially it took almost 6 minutes to read entire 1 miliions rows with default settings like batch.max.rows 100 and poll.interval.ms 5000

4) Observed the same results as above ie. no events on updates, delete and if inserting previous timestamp values



## Testing With Custom Query:

1) Use above existing table with  Auto-Increment Id as primaryKey and 15 column and and 1 million rows
2) Started Kafka Connector and created single task with SELECT query on table instead of using entire table in increment mode



3) Observed same issues as above

## Limitations:

Source Connector: The connector works by executing a query, over JDBC, against the source database. It does this to pull in all rows (bulk) or those that changed since previously (incremental).

1) The relational db table from which Kafka connect getting the data should either contain the auto increment column or timestamp field or  if plan to use custom query even then mode(timestamp or incrementing) is mandatory

2) There are no update, alter, delete events in kafka jdbc connector

