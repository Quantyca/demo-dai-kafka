= Realtime stock with Kafka and KSQL - Hands on Guide
Giulio Scotti <giulio.scotti@quantyca.it>
v0.10, May 27, 2019

1. Bring up the stack
+
[source,bash]
----
git clone https://github.com/Quantyca/kafka-realtime-stock.git
cd kafka-realtime-stock
docker-compose up -d
----
+
This brings up the stack and loads the necessary configuration. 

2. Launch the KSQL CLI: 
