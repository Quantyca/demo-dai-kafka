# Realtime stock with Kafka and Confluent tools - Hands on Guide

A demo to compute stock in realtime with Kafka.

## Changelog
* v0.11 2019-05-28 giulio.scotti@quantyca.it
    * TO BE DEFINED
* v0.10 2019-05-27 pietro.latorre@quantyca.it
    * setup infrastructure

## Requirements

* TO BE DEFINED

## Prepare the environment

1. Bring up the stack
    ```
    git clone https://github.com/Quantyca/kafka-realtime-stock.git
    cd kafka-realtime-stock
    docker-compose up -d --build
    ```
    This brings up the stack and loads the necessary configuration:
    * a Kafka node
    * a Zookeeper node
    * a Kafka Connect server
    * a Kafka REST Server
    * a MySQL Database

2. Open a terminal and create the Kafka topics:
	
	
	```
    curl -X POST -H "Content-Type: application/vnd.kafka.json.v2+json" --data '{"records":[{"key":"STORE2","value":{"STORE_COD":"STORE2", "PRODUCT_COD":"PROD2", "SOLD_QTY":4}}]}' "http://ext_broker:8082/topics/ORDERS_LINES_TOPIC"
    ```
	
	
3. Deploy the JDBC Source Connector:	
	
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
	
	
4. Launch the KSQL CLI: 

	```
    docker-compose exec ksql-cli ksql http://ksql-server:8088
    ```
	
5. From withing the KSQL cli, create the KSQL Stream on movements topic; this topic contains Avro formatted messages: 

	```
    CREATE STREAM SOURCE_MOVEMENTS_STREAM WITH (kafka_topic='SOURCE_MOVEMENTS_TABLE',value_format='AVRO');
    ```
	
6. From withing the KSQL cli, create the KSQL Stream on sales topic; this topic contains JSON formatted messages: 
	
	```
    CREATE STREAM ORDERS_LINES_STREAM ( \
    STORE_COD STRING, \
    PRODUCT_COD STRING, \
    SOLD_QTY INT) \
    WITH (kafka_topic='ORDERS_LINES_TOPIC', value_format='JSON');
    ```

7. From withing the KSQL cli, convert the JSON formatted sales messages in Avro records:


	```
    CREATE STREAM SOURCE_ORDERS_STREAM \
    WITH (value_format='AVRO') \
    AS SELECT * FROM ORDERS_LINES_STREAM;
	
    ```
	
8. From withing the KSQL cli, create the delta stock stream:

	
	```
	CREATE STREAM DELTA_STOCK_STREAM \
    AS SELECT \
    STORE_COD AS STORE_COD, \
    PRODUCT_COD AS PRODUCT_COD, \
    MOV_QTA AS DELTA_QTY \
    FROM SOURCE_MOVEMENTS_STREAM;
	
    ```
	
9. From withing the KSQL cli, add the orders contribute to delta stock:

	
	```
    INSERT INTO DELTA_STOCK_STREAM \
    SELECT \
    STORE_COD AS STORE_COD, \
    PRODUCT_COD AS PRODUCT_COD, \
    (-1) * SOLD_QTY AS DELTA_QTY \
    FROM SOURCE_ORDERS_STREAM;
	
    ```
	
10. From withing the KSQL cli, add the orders contribute to delta stock:

	
	```
    CREATE TABLE STOCK_TABLE \ 
    WITH (value_format='AVRO') \
    AS SELECT \
    STORE_COD, \
    PRODUCT_COD, \ 
    SUM(DELTA_QTY) AS CURRENT_STOCK_VAL \
    FROM DELTA_STOCK_STREAM \
    GROUP BY STORE_COD, PRODUCT_COD;
	
    ```
	
	
	
	
	

	
	

	
