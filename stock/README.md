# Realtime stock with Kafka and Confluent tools - Hands on Guide

A demo to compute stock in realtime with Kafka.

## Changelog
* v0.11 2019-05-28 giulio.scotti@quantyca.it
    * TO BE DEFINED
* v0.10 2019-05-27 pietro.latorre@quantyca.it
    * setup infrastructure

## Requirements

* Git: you can download from it from https://git-scm.com/downloads
* For Linux: 
	1. Docker: you can download it from https://docs.docker.com/install/linux/docker-ce/centos/
	2. Docker Compose: you can download it from https://docs.docker.com/compose/install/ 
* For Windows: 
	1. Docker Toolbox: you can download it from https://docs.docker.com/toolbox/toolbox_install_windows/
* For Linux:
	1. Locate the hosts file in your file system, insert the following mapping (the actual ip you insert must be the current dynamic ip assigned to your machine):
	```
	x.x.x.x ext_broker
    ```
* For Windows:
	1. Locate the hosts file in your file system, insert the following mapping (the actual ip you insert must be the actual ip assigned to your Docker Toolbox VM):
	```
	192.168.99.100 ext_broker
    ```
	

## Prepare the environment

0. Open two distinct terminal windows

1. From terminal 1, clone the repository and bring up the stack
    ```
    git clone https://github.com/Quantyca/demo-dai-kafka.git;
    cd demo-dai-kafka/stock;
    docker-compose up -d --build;
    ```
    This brings up the stack and loads the necessary configuration:
    * a Kafka broker
    * a Zookeeper server
    * a Kafka Connect worker
    * a Kafka REST proxy
	* a Schema Registry sever
	* a KSQL Server
    * a MySQL database
	
	NB: this first setup could take a few minutes: wait that all containers are started and properly configured before going on (more or less 3 minutes).

2. One contribute to the stock is given by the online orders data. So, first you have to create a Kafka topic where these data will be   written to. You are going to write orders data in JSON format. For performing this task, from terminal n.1, launch the call to the REST proxy (producing a neutral order):
	
	```
    curl -X POST -H "Content-Type: application/vnd.kafka.json.v2+json" --data '{"records":[{"key":"STORE2","value":{"STORE_COD":"STORE2", "PRODUCT_COD":"PROD2", "SOLD_QTY":0}}]}' "http://ext_broker:8082/topics/ORDERS_LINES_TOPIC"
    ```
You should receive a response like this:
    ```
    {"offsets":[{"partition":0,"offset":0,"error_code":null,"error":null}],"key_schema_id":null,"value_schema_id":null}	
    ```
	
	
3. The other contribute is given by the warehouse movement, extracted from a database table. The table has been automatically created when you started the infrastructure and a neutral movement has been inserted. For extracting data, from terminal 1, deploy the JDBC Source Connector by launching the rest call to the Connect Worker REST API (this will automatically create a topic and write data extracted from the database table in Avro format, registering automatically the schema into the Schema Registry):	
	
	```
    curl -X POST http://ext_broker:8083/connectors -H "Content-Type: application/json" -d '{
    "name": "SOURCE_MOVEMENTS_CONNECTOR",
    "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
    "connection.url": "jdbc:mysql://mysql:3306/?characterEncoding=latin1&useConfigs=maxPerformance",
    "connection.user": "root",
    "connection.password": "ok",
    "topic.prefix": "SOURCE_MOVEMENTS_TABLE",
    "mode":"timestamp",
    "query" : "SELECT * FROM KAFKA.SOURCE_MOVEMENTS_TABLE",
    "timestamp.column.name" : "INSERT_UPDATE_TIMESTAMP",
    "timestamp.delay.interval.ms": "-7200000",
    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "value.converter" : "io.confluent.connect.avro.AvroConverter",
    "value.converter.schema.registry.url" : "http://schema-registry:8081",
    "transforms":"createKey,castInteger,timestampConverter", 
    "transforms.createKey.type":"org.apache.kafka.connect.transforms.ValueToKey",
    "transforms.createKey.fields":"STORE_COD",
    "transforms.castInteger.type":"org.apache.kafka.connect.transforms.Cast$Value", 
    "transforms.castInteger.spec":"MOV_ID:int64,MOV_QTA:int32",
    "transforms.timestampConverter.type":"org.apache.kafka.connect.transforms.TimestampConverter$Value", 
    "transforms.timestampConverter.target.type":"unix",
    "transforms.timestampConverter.field":"INSERT_UPDATE_UTC_TIMESTAMP",
    "transforms.timestampConverter.format":"yyyy-MM-dd HH:mm:ss.sss"
    }
    }'
    ```
	
You should receive a response like this:
    ```
	{"name":"SOURCE_MOVEMENTS_CONNECTOR","config":{"connector.class":"io.confluent.connect.jdbc.JdbcSourceConnector","connection.url":"jdbc:mysql://mysql:3306/?characterEncoding=latin1&useConfigs=maxPerformance","connection.user":"root","connection.password":"ok","topic.prefix":"SOURCE_MOVEMENTS_TABLE","mode":"timestamp","query":"SELECT * FROM KAFKA.SOURCE_MOVEMENTS_TABLE","timestamp.column.name":"INSERT_UPDATE_TIMESTAMP","timestamp.delay.interval.ms":"-7200000","key.converter":"org.apache.kafka.connect.storage.StringConverter","value.converter":"io.confluent.connect.avro.AvroConverter","value.converter.schema.registry.url":"http://schema-registry:8081","transforms":"createKey,castInteger,timestampConverter","transforms.createKey.type":"org.apache.kafka.connect.transforms.ValueToKey","transforms.createKey.fields":"STORE_COD","transforms.castInteger.type":"org.apache.kafka.connect.transforms.Cast$Value","transforms.castInteger.spec":"MOV_ID:int64,MOV_QTA:int32","transforms.timestampConverter.type":"org.apache.kafka.connect.transforms.TimestampConverter$Value","transforms.timestampConverter.target.type":"unix","transforms.timestampConverter.field":"INSERT_UPDATE_UTC_TIMESTAMP","transforms.timestampConverter.format":"yyyy-MM-dd HH:mm:ss.sss","name":"SOURCE_MOVEMENTS_CONNECTOR"},"tasks":[{"connector":"SOURCE_MOVEMENTS_CONNECTOR","task":0}],"type":"source"}
	```
	
4. For doing the stream processing needed, you have to create KSQL abstractions over Kafka topics. So, from terminal n.1, create the KSQL Stream on movements topic, by launching the rest call to the KSQL Server; this topic contains Avro formatted messages: 

	```
    curl -X "POST" -H "Content-Type: application/vnd.ksql.v1+json; charset=utf-8" \
	-d $'{"ksql":"CREATE STREAM SOURCE_MOVEMENTS_STREAM WITH (kafka_topic='\''SOURCE_MOVEMENTS_TABLE'\'',value_format='\''AVRO'\'');", "streamProperties":{}
	}' "http://ext_broker:8088/ksql"
    ```
	
You should receive a response like this:
    ```
	[{"@type":"currentStatus","statementText":"CREATE STREAM SOURCE_MOVEMENTS_STREAM \n(ID INTEGER, STORE_COD STRING, PRODUCT_COD STRING, MOV_QTA INTEGER, INSERT_UPDATE_TIMESTAMP BIGINT) WITH (KAFKA_TOPIC='SOURCE_MOVEMENTS_TABLE', VALUE_FORMAT='AVRO', AVRO_SCHEMA_ID='1');","commandId":"stream/SOURCE_MOVEMENTS_STREAM/create","commandStatus":{"status":"SUCCESS","message":"Stream created"}}]
	```
	
5. You have to do the same operation over the orders topic but, in this case, the messages are formatted in JSON. So, from terminal n.1, create the KSQL Stream on sales topic; you have to specify the schema esplicitly: 
	
	```
    curl -X "POST" -H "Content-Type: application/vnd.ksql.v1+json; charset=utf-8" \
	-d $'{"ksql":"CREATE STREAM ORDERS_LINES_STREAM (STORE_COD STRING, PRODUCT_COD STRING, SOLD_QTY INT) WITH (kafka_topic='\''ORDERS_LINES_TOPIC'\'', value_format='\''JSON'\'');", "streamProperties":{}
	}' "http://ext_broker:8088/ksql"	
    ```
	
You should receive a response like this:
    ```
	[{"@type":"currentStatus","statementText":"CREATE STREAM ORDERS_LINES_STREAM (STORE_COD STRING, PRODUCT_COD STRING, SOLD_QTY INT) WITH (kafka_topic='ORDERS_LINES_TOPIC', value_format='JSON');","commandId":"stream/ORDERS_LINES_STREAM/create","commandStatus":{"status":"SUCCESS","message":"Stream created"}}]
	```

6. In order to manage all the data in Avro format, use KSQL to cconvert the JSON formatted orders messages in Avro records:

	```
    curl -X "POST" -H "Content-Type: application/vnd.ksql.v1+json; charset=utf-8" \
	-d $'{"ksql":"CREATE STREAM SOURCE_ORDERS_STREAM WITH (value_format='\''AVRO'\'') AS SELECT * FROM ORDERS_LINES_STREAM;", "streamProperties":{}
	}' "http://ext_broker:8088/ksql"
    ```
	
You should receive a response like this:
    ```
	[{"@type":"currentStatus","statementText":"CREATE STREAM SOURCE_ORDERS_STREAM WITH (value_format='AVRO') AS SELECT * FROM ORDERS_LINES_STREAM;","commandId":"stream/SOURCE_ORDERS_STREAM/create","commandStatus":{"status":"SUCCESS","message":"Stream created and running"}}]
	```
	
7. You have to do a UNION operation between the two contributes, for building an unique stream of stock variations. So, from terminal n.1, use KSQL to create the delta stock stream as a select from the warehouse movements stream:
	
	```
    curl -X "POST" -H "Content-Type: application/vnd.ksql.v1+json; charset=utf-8" \
	-d $'{"ksql":"CREATE STREAM DELTA_STOCK_STREAM AS SELECT STORE_COD AS STORE_COD, PRODUCT_COD AS PRODUCT_COD, MOV_QTA AS DELTA_QTY FROM SOURCE_MOVEMENTS_STREAM;", "streamProperties":{}
	}' "http://ext_broker:8088/ksql"
    ```
	
You should receive a response like this:
    ```
	[{"@type":"currentStatus","statementText":"CREATE STREAM DELTA_STOCK_STREAM AS SELECT STORE_COD AS STORE_COD, PRODUCT_COD AS PRODUCT_COD, MOV_QTA AS DELTA_QTY FROM SOURCE_MOVEMENTS_STREAM;","commandId":"stream/DELTA_STOCK_STREAM/create","commandStatus":{"status":"SUCCESS","message":"Stream created and running"}}]
	```
	
8. In order to get the UNION, from terminal n.1, you have to add the orders contribute to delta stock stream (again using KSQL):
	
	```
    curl -X "POST" -H "Content-Type: application/vnd.ksql.v1+json; charset=utf-8" \
	-d $'{"ksql":"INSERT INTO DELTA_STOCK_STREAM SELECT STORE_COD AS STORE_COD, PRODUCT_COD AS PRODUCT_COD, (-1) * SOLD_QTY AS DELTA_QTY FROM SOURCE_ORDERS_STREAM;", "streamProperties":{}
	}' "http://ext_broker:8088/ksql"
    ```
	
You should receive a response like this:
    ```
	[{"@type":"currentStatus","statementText":"INSERT INTO DELTA_STOCK_STREAM SELECT STORE_COD AS STORE_COD, PRODUCT_COD AS PRODUCT_COD, (-1) * SOLD_QTY AS DELTA_QTY FROM SOURCE_ORDERS_STREAM;","commandId":"stream/DELTA_STOCK_STREAM/create","commandStatus":{"status":"SUCCESS","message":"Insert Into query is running."}}]
	```
	
9. Now that you have the unique source of stock variations, you can build a stream processing application that performs an aggregation over the delta stock and computes the real time stock snapshot.  The stock is calculated as a continuous sum of variations, grouping by the pair <RODUCT, STORE>. With KSQL, you can easily build such an application. From terminal n.1, launch the REST call:
	
	```
    curl -X "POST" -H "Content-Type: application/vnd.ksql.v1+json; charset=utf-8" \
	-d $'{"ksql":"CREATE TABLE STOCK_TABLE WITH (value_format='\''AVRO'\'') AS SELECT STORE_COD, PRODUCT_COD, SUM(DELTA_QTY) AS CURRENT_STOCK_VAL FROM DELTA_STOCK_STREAM GROUP BY STORE_COD, PRODUCT_COD;", "streamProperties":{}
	}' "http://ext_broker:8088/ksql"
    ```
	
You should receive a response like this:
    ```
	[{"@type":"currentStatus","statementText":"CREATE TABLE STOCK_TABLE WITH (value_format='AVRO') AS SELECT STORE_COD, PRODUCT_COD, SUM(DELTA_QTY) AS CURRENT_STOCK_VAL FROM DELTA_STOCK_STREAM GROUP BY STORE_COD, PRODUCT_COD;","commandId":"table/STOCK_TABLE/create","commandStatus":{"status":"SUCCESS","message":"Table created and running"}}]
	```
	
10. Now, you can start a continuous query over the Ktable representing the stock, for consuming results in real time. From terminal n.1, launch the query on the stock table and keep it running:
	
	```
    curl -X "POST" -H "Content-Type: application/vnd.ksql.v1+json; charset=utf-8" \
	-d $'{"ksql":"SELECT * FROM STOCK_TABLE;", "streamProperties":{}
	}' "http://ext_broker:8088/query"
    ```
	
You should see a scrolling view in the terminal.
	
11. Now, from terminal n.2, produce a movement into the database table and, keeping the query running on terminal n.1, look at what appears as query results:
	
	```
    docker exec mysql mysql -u root -pok -e "INSERT INTO KAFKA.SOURCE_MOVEMENTS_TABLE (STORE_COD,PRODUCT_COD,MOV_QTA,INSERT_UPDATE_TIMESTAMP) SELECT 'STORE8','PROD3',10,CURRENT_TIMESTAMP;"
    ```
	
Looking at termianl 1, you should see a line like this:
    ```
	{"row":{"columns":[1560410975591,"STORE8|+|PROD3","STORE8","PROD3",10]},"errorMessage":null,"finalMessage":null}
	```
	
12. Now, keeping the query running on terminal n.1, you can simulate the arrival of new orders by submitting REST call from terminal n.2; just later, by looking at terminal n.1, the updated stock quantity will appear:	
	
	```
    curl -X POST -H "Content-Type: application/vnd.kafka.json.v2+json" --data '{"records":[{"key":"STORE8","value":{"STORE_COD":"STORE8", "PRODUCT_COD":"PROD3", "SOLD_QTY":4}}]}' "http://ext_broker:8082/topics/ORDERS_LINES_TOPIC"
    ```
	
Looking at termianl 1, you should see a line like this:
    ```
	{"row":{"columns":[1560410975591,"STORE8|+|PROD3","STORE8","PROD3",6]},"errorMessage":null,"finalMessage":null}
	```

13. Now, from terminal n.2, simulate the arrival of new items in the store and check the updated value in terminal n.1:
	
	```
    docker exec mysql mysql -u root -pok -e "INSERT INTO KAFKA.SOURCE_MOVEMENTS_TABLE (STORE_COD,PRODUCT_COD,MOV_QTA,INSERT_UPDATE_TIMESTAMP) SELECT 'STORE8','PROD3',5,CURRENT_TIMESTAMP;"
    ```
	
Looking at termianl 1, you should see a line like this:
    ```
	{"row":{"columns":[1560411075743,"STORE8|+|PROD3","STORE8","PROD3",11]},"errorMessage":null,"finalMessage":null}
	```
	
14. Again from terminal n.2, another order; look at terminal n.1 and you will see the stock has changed again:	
	
	```
    curl -X POST -H "Content-Type: application/vnd.kafka.json.v2+json" --data '{"records":[{"key":"STORE8","value":{"STORE_COD":"STORE8", "PRODUCT_COD":"PROD3", "SOLD_QTY":9}}]}' "http://ext_broker:8082/topics/ORDERS_LINES_TOPIC"
    ```
	
Looking at termianl 1, you should see a line like this:
    ```
	{"row":{"columns":[1560411106820,"STORE8|+|PROD3","STORE8","PROD3",2]},"errorMessage":null,"finalMessage":null}
	```

15. When you are satisfied, kill the continuous query and destroy the infrastructure.

	```
    press CTRL+C
	docker-compose down	
    ```	
	

	
	
 

	

	
