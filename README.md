
# kafka-mqtt-source-connector
A connector plugin to use with Kafka's Connect API. It can be configured to map any topic from an mqtt broker to any topic on the Kafka broker. The connector can be configured to use SSL in communication with the mqtt broker.

## Setting up a single Zookeeper and Kafka instance
To be able to test the connector, we first need to set up the Kafka infrastructure. For simplicity, we start by configuring single nodes (one Zookeeper and one Kafka).

###Prerequisites###
- Java version >8 installed to run Kafka and this source-connector. Check if Java is installed, and which version by running `java -version` in your terminal. We use `openjdk version "11.0.6" 2020-01-14`.
- Linux. We are running this setup on Ubuntu 16.4.
- A mqtt-broker. We use EMQX
- Maven. Check if maven is installed properly with running `mvn -v` in your terminal. We use Maven 3.6.0

###Mqtt Broker - EMQX###
- Download EMQX from [https://www.emqx.io/downloads](https://www.emqx.io/downloads)
- Extract the download to your desired destination, here termed _"path-to-emqx"_.
- Run the following command in your terminal to start the EMQX broker:
```
"path-to-emqx"/emqx/bin/emqx start
```
- Check that EMQX is running with the following terminal command:
 ```
"path-to-emqx"/emqx/bin/emqx_ctl status
```
 
###Download Kafka binaries###
Download a binary Kafka release from https://kafka.apache.org/downloads. We work with the compressed download:
>kafka_2.13-2.4.1.tgz
Extract the download to your desired destination, here termed _"path-to-kafka"_.

###Zookeeper###
About Zookeeper:
>"Zookeeper is a top-level software developed by Apache that acts as a centralized service and is used to maintain naming and configuration data and to provide flexible and robust synchronization within distributed systems. Zookeeper keeps track of status of the Kafka cluster nodes and it also keeps track of Kafka topics, partitions etc.
Zookeeper it self is allowing multiple clients to perform simultaneous reads and writes and acts as a shared configuration service within the system. The Zookeeper atomic broadcast (ZAB) protocol i s the brains of the whole system, making it possible for Zookeeper to act as an atomic broadcast system and issue orderly updates." [Cloudkarafka](https://www.cloudkarafka.com/blog/2018-07-04-cloudkarafka_what_is_zookeeper.html)

**Start Zookeeper**
```
"path-to-kafka"/kafka_2.13-2.4.1/bin/zookeeper-server-start.sh "path-to-kafka"/kafka_2.13-2.4.1/config/zookeeper.properties
```
P.S. The default properties of zookeeper.properties works well for this tutorial's purpose. It will start Zookeeper on the default port `2181`.

###Kafka Broker###
As mentioned, we will only kick up a single instance Kafka Broker. The Kafka Broker will use `"path-to-kafka"/kafka_2.13-2.4.1/config/server.properties`, and it could be worth checking that 
```
zookeeper.connect=localhost:2181
``` 
or set according to your custom configuration in `zookeeper.properties`.

**Start Kafka Broker**
```
"path-to-kafka"/kafka_2.13-2.4.1/bin/kafka-server-start.sh "path-to-kafka"/kafka_2.13-2.4.1/config/server.properties
```

**Create Kafka Topic**
```
"path-to-kafka"/kafka_2.13-2.4.1/bin/kafka-topics --create --bootstrap-server 127.0.0.1:9092 --replication-factor 1 --partitions 1 --topic test
```

###Kafka Connect###
The Kafka Connect API is what we utilise as a framework around our connectors, to handle scaling, polling from Kafka, work distribution etc. Kafka Connect can run as _connect-standalone_ or as _connect-distributed_. The _connect-standalone_ is engineered for demo and test purposes, as it cannot provide fallback in a production environment. 

**Start Kafka Connect**
Follow the respective steps below to start Kafka Connect in preferred mode. 

_Connect in general_
Build this java maven project, but navigating to root `kafka-mqtt-source-connector` in a terminal and typing:
```
mvn install
``` 
`Copy the kafka-mqtt-source-connector-"version".jar` from your maven target directory to the directory `/usr/share/java/kafka`.

**Insecure - using tcp**
__*Connect Standalone*__ 
1. Uncomment `plugin.path` in `"path-to-kafka"/kafka_2.13-2.4.1/config/connect-standalone.properties`, so that it is set to 
```
plugin.path=/usr/share/java,/usr/local/share/kafka/plugins,/usr/local/share/java/
``` 
2. Copy  the accompanying source connector properties file in this repository, **[source-connect-mqtt.properties](https://github.com/SINTEF-9012/kafka-mqtt-source-connector/src/main/resources/source-connect-mqtt.properties)**, to `"path-to-kafka"/kafka_2.13-2.4.1/config/` (or create a new properties file with the same name in the given directory).
3. Ensure the following configuration in `source-connect-mqtt.properties`:
```
name=mqtt-source-connector 
tasks.max=1
connector.class=com.sintef.asam.MqttSourceConnector
mqtt.connector.broker.uri=tcp://0.0.0.0:1883
mqtt.connector.broker.topic=test/#
mqtt.conncetor.kafka.topic=test
```
where `mqtt.connector.broker.topic` sets the topic one wants to subscribe to in the mqtt broker, while `mqtt.connector.kafka.topic` sets the topic for publishing to the Kafka broker. The `mqtt.connector.broker.uri` needs to be set according to your own mqtt broker, but the default for mosquitto and emqx will be the abovementioned. 

4. Start _Connect Standalone_ with our connector by typing (this may take a minute or two):
```
"path-to-kafka"/kafka_2.13-2.4.1/bin/connect-standalone.sh "path-to-kafka"/kafka_2.13-2.4.1/config/connect-standalone.properties "path-to-kafka"/kafka_2.13-2.4.1/config/source-connect-mqtt.properties
```

__*Connect Distributed*__ 
Kafka Connect Distributed does not need properties files to configure connectors. It uses the Kafka Connect REST-interface. 
5. Uncomment `plugin.path` in `"path-to-kafka"/kafka_2.13-2.4.1/config/connect-distributed.properties`, so that it is set to 
```
plugin.path=/usr/share/java,/usr/local/share/kafka/plugins,/usr/local/share/java/
``` 
and that `rest.port` so that it is set to
```
rest.port=19005
``` 
which will help one to avoid some "bind" exceptions. This will be the port for the Connect REST-interface.
6. Start _Connect Distributed_ with by typing (this may take a minute or two):
```
"path-to-kafka"/kafka_2.13-2.4.1/bin/connect-distributed.sh "path-to-kafka"/kafka_2.13-2.4.1/config/connect-distributed.properties
```
7. Start our connector by posting the following command to the Connect REST-interface:
```
curl -s -X POST -H 'Content-Type: application/json' http://127.0.0.1:19005/connectors -d '{"name":"mqtt-source-connector","config":{"connector.class":"com.sintef.asam.MqttSourceConnector","tasks.max":"1","mqtt.connector.broker.uri":"tcp://localhost:1883", "mqtt.connector.broker.topic":"test/#","mqtt.connector.kafka.topic":"test"}}'
```
8. Inspect the terminal where you started Conncet Distributed, and after the connector seem to have successfully started, check the existence by typing:
```
curl 'Content-Type: application/json' http://127.0.0.1:19005/connectors
```
where the response is an array with connectors by name.
9. Test the connector by making a Kafka Consumer subscribing to the topic `test`:
```
Documents/confluent-5.4.0/bin/kafka-console-consumer --bootstrap-server 127.0.0.1:9092 --topic test
``` 
10. Then publish something to the test topic on your EMQX broker, using e.g. mosquitto:
```
mosquitto_pub -h 127.0.0.1 -p 1883 -t test -m "Hello, world!"
```
and see the message appear base64 encoded to your Kafka Consumer.

**Secure - using SSL**  

__*Connect Distributed*__ 
12. Delete the previous made connector using TCP, if one is running, using the following call to the Connect REST-interface:
 ```
curl -X DELETE 'Content-Type: application/json' http://127.0.0.1:19005/connectors/mqtt-source-connector
```
if your connector was named `mqtt-source-connector`. Check running connectors by name using:
```
curl 'Content-Type: application/json' http://127.0.0.1:19005/connectors
``` 
13. Copy your CA certificate, client certificate and client key to desired directory. In our test case we have the full paths `/home/ca.crt`, `/home/client.crt` and `/home/client.key`. 
14. Start our connector by posting the following command to the Connect REST-interface:
```
curl -s -X POST -H 'Content-Type: application/json' http://127.0.0.1:19005/connectors -d '{"name":"mqtt-source-connector","config":{"connector.class":"com.sintef.asam.MqttSourceConnector","tasks.max":"1","mqtt.connector.broker.uri":"ssl://localhost:8883", "mqtt.connector.broker.topic":"test/#", "mqtt.connector.kafka.topic":"test","mqtt.connector.ssl":true, "mqtt.connector.ssl.ca":"/home/ca.crt/","mqtt.connector.ssl.crt":"/home/client.crt","mqtt.connector.ssl.key":"/home/client.key"}}'
```
15. Test the connector by making a Kafka Consumer subscribing to the topic `test`:
```
Documents/confluent-5.4.0/bin/kafka-console-consumer --bootstrap-server 127.0.0.1:9092 --topic test
``` 
16. Then publish something to the test topic on your EMQX broker, using e.g. mosquitto:
```
mosquitto_pub --url mqtts://127.0.0.1:8883/test --cafile /home/ca.crt --cert /home/client.crt --key /home/client.key --insecure --tls-version tlsv1.2 -m "Hello, world!"
```
and see the message appear base64 encoded to your Kafka Consumer.
