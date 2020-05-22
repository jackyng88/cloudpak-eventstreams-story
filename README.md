# cloudpak-eventstreams-story (AWS S3 Specific)

## Overall Use Case and Goal - 
1. Now that you have an Event Streams instance installed on Cloud Pak for Integration on top of OpenShift Container Platform the goal of this story is to show a possible use case that we can use with this technology.
2. With IBM Event Streams we have access to the powerful capabilities of Kafka in addition to all the monitoring and logging capabilities that IBM provides on top of that with Event Streams.
3. We will create a simple Quarkus (a super sonic and sub-atomic Kubernetes native framework for Java) application that utilizes MicroProfile Reactive Messaging in order for us to send a stream of data to our Event Streams/Kafka topic.
4. After that we will use the capabilities of the Strimzi Operator on OpenShift to deploy a KafkaConnect S2I cluster and also AWS S3 Sink and Source connectors built using the Apache Camel KafkaConnect binaries. From our Quarkus application we will send messages to an Inbound (Sink) topic which will trigger the Camel S3 Sink Connector to pull from that topic and put that data into an AWS S3 Bucket. We will then have the Camel S3 Sink Connector pull from the AWS S3 bucket and place the contents into an Outbound (Source) Topic. We will follow the steps outlined here [Kafka Connect to S3 Sink & Source](https://ibm-cloud-architecture.github.io/refarch-eda/scenarios/connect-s3/)


## Pre-requisites - 
1. OpenShift Container Platform Cluster - This story will assume you have a 4.x Cluster.
2. Cloud Pak for Integration - This will assume you have probably at least a 2019.4.1 or 2020.x.x release of the Cloud Pak for Integration installed on OpenShift. This story will also assume you have followed the installation instructions for Event Streams outlined here from the [Cloud Pak Playbook](https://cloudpak8s.io/integration/cp4i-deploy-eventstreams/) and have a working Event Streams instance.
3. Java Development Kit (JDK) v1.8+
4. Apache Maven v3.6.2+
5. An IDE of your choice 
6. An AWS Account.
   - An IAM user with AmazonS3FullAccess Policy
   - An S3 Bucket policy so we can perform actions on the S3 bucket.


## Creating Event Streams Topics - 
1. Navigate to the Cloud Pak for Integration Platform Navigator. 

2. Click View Instances and click the Event Streams instance that you have created.

3. Click the Topics option on the left. Create the INBOUND topic.

![Create Topic](https://github.com/jackyng88/cloudpak-eventstreams-story/raw/master/supporting-pictures/create%20topic.png)

![Topic Name](https://github.com/jackyng88/cloudpak-eventstreams-story/raw/master/supporting-pictures/inbound%20topic%20name.png)

4. Leave Partitions at 1.

![Partition](https://github.com/jackyng88/cloudpak-eventstreams-story/raw/master/supporting-pictures/partitions.png)

5. Depending on how long you want messages to persist you can change this.

![Message Retention](https://github.com/jackyng88/cloudpak-eventstreams-story/raw/master/supporting-pictures/message%20retention.png)

6. You can leave Replication Factor at the default 3.

![Replication](https://github.com/jackyng88/cloudpak-eventstreams-story/raw/master/supporting-pictures/replicas.png)

7. Click Create.

8. Create an OUTBOUND topic following the prior steps as well.


## Event Streams Security: API Key, Credentials and Certificates - 

1. To connect to our Event Streams Instance we will need to follow a few steps to properly connect to it.

2. While viewing our Event Streams Instance, navigate to the Topics menu from the left. Click Connect to this Cluster - 

![Connect to this Cluster](https://github.com/jackyng88/cloudpak-eventstreams-story/raw/master/supporting-pictures/Connect%20to%20this%20Cluster.png)

3. 


## Creating the Quarkus with MicroProfile Reactive Messaging Application - 
1. Create the Quarkus project. You can replace <> and the contents inside of <> with whatever you would like.

```
mvn io.quarkus:quarkus-maven-plugin:1.4.2.Final:create \
    -DprojectGroupId=<org.acme> \
    -DprojectArtifactId=<quarkus-kafka> \
    -Dextensions="kafka"
```

2. Open the project in your IDE of choice. 

3. Create the following folder structure and then create the Producer.java file.

```
src/main/java/org/acme/kafka/producer/Producer.java
```

![Quarkus Project Folder Structure](https://github.com/jackyng88/cloudpak-eventstreams-story/raw/master/supporting-pictures/Quarkus%20folder%20structure.png)


4. Within your Producer.java file add the following code - 

```
package org.acme.kafka.producer;

import io.reactivex.Flowable;
import io.smallrye.reactive.messaging.kafka.KafkaRecord;

import org.eclipse.microprofile.reactive.messaging.Outgoing;

import javax.enterprise.context.ApplicationScoped;
import java.util.Random;
import java.util.concurrent.TimeUnit;

/**
 * This class produces a message every 5 seconds.
 * The Kafka configuration is specified in the application.properties file.
*/
@ApplicationScoped
public class Producer {

    private Random random = new Random();

    @Outgoing("TOPIC-NAME")      
    public Flowable<KafkaRecord<Integer, String>> generate() {
        return Flowable.interval(5, TimeUnit.SECONDS)    
                .onBackpressureDrop()
                .map(tick -> {      
                    return KafkaRecord.of(random.nextInt(100), String.valueOf(random.nextInt(100)));
                });
    }                  
}

```

Take note on the line tha says @Outgoing("TOPIC-NAME"). Choose a topic name. For the purposes of this story we will use the INBOUND topic name. 

```@Outgoing("INBOUND") ```

Technically in @Outgoing("") is for specifying the name of the Channel, but it will default to a topic if a topic name is not provided in the application.properties file. We will address that a little bit later.


* What does this Producer.java code do? 
   - The @Outgoing annotation indicates that we're sending to a Channel (or Topic) and we're not expecting any data.
   - The generate() function returns an [RX Java 2 Flowable Object](https://www.baeldung.com/rxjava-2-flowable) emmitted every 5 seconds. 
   - The Flowable object returns a KafkaRecord of type <Integer, String>.
   
   
5. We will now need to update our applications.properties file that was automatically generated when the Quarkus project was created located here - 

``` src/main/resources/application.properties```

![application properties structure](https://github.com/jackyng88/cloudpak-eventstreams-story/raw/master/supporting-pictures/application%20properties%20structure.png)

6. Copy and paste the following into your application.properties file - 

```
# Event Streams instance connection details. The channel here (INBOUND) will by default be set as the topic.
mp.messaging.connector.smallrye-kafka.bootstrap.servers=<es-bootstrap-address>
mp.messaging.outgoing.INBOUND.connector=smallrye-kafka

# Event Streams security credentials if necessary (in cases where SSL is enabled). Serializers used for outgoing
# and deserializers are used for incoming messages.
mp.messaging.outgoing.INBOUND.key.serializer=org.apache.kafka.common.serialization.IntegerSerializer
mp.messaging.outgoing.INBOUND.value.serializer=org.apache.kafka.common.serialization.StringSerializer
mp.messaging.outgoing.INBOUND.sasl.mechanism=PLAIN
mp.messaging.outgoing.INBOUND.security.protocol=SASL_SSL
mp.messaging.outgoing.INBOUND.ssl.protocol=TLSv1.2
mp.messaging.outgoing.INBOUND.sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
            username="token" \
            password="<APIKey>";
mp.messaging.outgoing.INBOUND.ssl.truststore.location=</filepath-to-es-truststorefile/>es-cert.jks
mp.messaging.outgoing.INBOUND.ssl.truststore.password=password
```
