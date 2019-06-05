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
    docker-compose up -d
    ```
    This brings up the stack and loads the necessary configuration:
    * a Kafka node
    * a Zookeeper node
    * a Kafka Connect server
    * a Kafka REST Server
    * a MySQL Database

2. Open a terminal inside the Confluent bin directory:

	* on Windows
	```
    cd /c/confluent-5.1.0/bin/windows
    ```
	* on Linux
	```
    cd /opt/confluent-5.1.0/bin/
    ```

3. Create the Kafka topics:
	
	* on Windows/Linux
	```
    curl -X POST -H "Content-Type: application/vnd.kafka.json.v2+json" --data '{"records":[{"key":"STORE2","value":{"STORE_COD":"STORE2", "PRODUCT_COD":"PROD2", "SOLD_QTY":4}}]}' "http://ext_broker:29092/topics/ORDERS_LINES_TOPIC"
    ```
	
	
4. Deploy the JDBC Source Connector:	
	
		* on Windows/Linux
	```
    curl -X POST http://ext_broker:8083/connectors -H "Content-Type: application/json" -d '{
    "name": "SOURCE_MOVEMENTS_CONNECTOR",
    "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
    "connection.url": "jdbc:mysql://mysql:3306/?characterEncoding=latin1&useConfigs=maxPerformance",
    "connection.user": "root",
    "connection.password": "ok",
    "topic.prefix": "SOURCE_MOVEMENTS_TABLE-value",
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
    "transforms.timestampConverter.field":"MOV_TIMESTAMP,INSERT_UPDATE_UTC_TIMESTAMP",
    "transforms.timestampConverter.format":"yyyy-MM-dd HH:mm:ss.sss"
    }
    }'
    ```
	
	
3. Launch the KSQL CLI: 

	* on Windows/Linux
	```
    docker-compose exec ksql-cli ksql http://ksql-server:8088
    ```
	
4. From withing the KSQL cli, create the KSQL Stream on movements topic; this topic contains Avro formatted messages: 

	* on Windows/Linux
	```
    CREATE STREAM SOURCE_MOVEMENTS_STREAM WITH (kafka_topic='SOURCE_MOVEMENTS_TABLE',value_format='AVRO');
    ```
	
5. From withing the KSQL cli, create the KSQL Stream on sales topic; this topic contains JSON formatted messages: 
	
	* on Windows/Linux
	```
    CREATE STREAM SALES_LINES_STREAM ( \
    STORE_COD STRING, \
    PRODUCT_COD STRING, \
    SOLD_QTY INT) \
    WITH (kafka_topic='SALES_LINES_TOPIC', value_format='JSON');
    ```

6. From withing the KSQL cli, convert the JSON formatted sales messages in Avro records:

	* on Windows/Linux
	```
    CREATE STREAM SOURCE_SALES_STREAM \
    WITH (value_format='AVRO') \
    AS SELECT * FROM SALES_LINES_STREAM;
    ```