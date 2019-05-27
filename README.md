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

2. Launch the KSQL CLI: 

    * TO DO