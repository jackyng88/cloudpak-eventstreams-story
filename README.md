# cloudpak-eventstreams-story (AWS S3 Specific)

## Overall Use Case and Goal - 
1. Now that you have an Event Streams instance installed on Cloud Pak for Integration on top of OpenShift Container Platform the goal of this story is to show a possible use case that we can use with this technology.
2. With IBM Event Streams we have access to the powerful capabilities of Kafka in addition to all the monitoring and logging capabilities that IBM provides on top of that with Event Streams.
3. We will create a simple Quarkus (a super sonic and sub-atomic Kubernetes native framework for Java) application that utilizes MicroProfile Reactive Messaging in order for us to send a stream of data to our Event Streams/Kafka topic.
4. After that we will use the capabilities of the Strimzi Operator on OpenShift to deploy a KafkaConnect S2I cluster and also AWS S3 Sink and Source connectors built using the Apache Camel KafkaConnect binaries. From our Quarkus application we will send messages to an Inbound (Sink) topic which will trigger the Camel S3 Sink Connector to pull from that topic and put that data into an AWS S3 Bucket. We will then have the Camel S3 Sink Connector pull from the AWS S3 bucket and place the contents into an Outbound (Source) Topic. We will follow the steps outlined here [Kafka Connect to S3 Sink & Source](https://ibm-cloud-architecture.github.io/refarch-eda/scenarios/connect-s3/)

![Architecture Diagram](https://github.com/jackyng88/cloudpak-eventstreams-story/raw/master/supporting-pictures/Quarkus%20to%20Event%20Streams%20to%20S3%20Arch%20Diagram%20image.png)


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

3. Keep note of your Bootstrap Server Address. Save this somewhere as we will need this later to configure our Quarkus Application's connection to the Event Streams instance.

![Bootstrap Address](https://github.com/jackyng88/cloudpak-eventstreams-story/raw/master/supporting-pictures/Bootstrap%20Server.png)


4. Generate your API Key. Click the Generate API Key button.

![Generate API Key 1](https://github.com/jackyng88/cloudpak-eventstreams-story/raw/master/supporting-pictures/Generate%20API%20Key%201.png)


5. Select a name for your application. It doesn't really matter too much what you name it. Also choose the Produce, Consume, Create Topics and Schema Option.

![Generate API Key 2](https://github.com/jackyng88/cloudpak-eventstreams-story/raw/master/supporting-pictures/Generate%20API%20Key%202.png)

6. Select All Topics and then click Next.

![Generate API Key 3](https://github.com/jackyng88/cloudpak-eventstreams-story/raw/master/supporting-pictures/Generate%20API%20Key%203.png)

7. Leave it All Consumer Groups on "ON" and click Next.

![Generate API Key 4](https://github.com/jackyng88/cloudpak-eventstreams-story/raw/master/supporting-pictures/Generate%20API%20Key%204.png)

8. You can copy down your API Key by hitting the Copy API Key button, or you can select Download as JSON so you can have a .json file with your API Key for better organization. Afterwards hit Close.

![Generate API Key 5](https://github.com/jackyng88/cloudpak-eventstreams-story/raw/master/supporting-pictures/Generate%20API%20Key%205.png)

9. Download the Java truststore .jks certificate.

![JKS Truststore](https://github.com/jackyng88/cloudpak-eventstreams-story/raw/master/supporting-pictures/JKS%20Cert.png)


10. Write/keep track of the truststore password.

![Truststore password](https://github.com/jackyng88/cloudpak-eventstreams-story/raw/master/supporting-pictures/JKS%20Truststore%20password.png)

11. Make sure you have these files in the same folder.


<ins>Summary</ins> - We now have the bootstrap server address, API Key, .jks truststore certificate, and the truststore password associated with that certificate to allow us the ability to connect to our Event Streams instance.



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

    @Outgoing("<TOPIC-NAME>")      
    public Flowable<KafkaRecord<Integer, String>> generate() {
        return Flowable.interval(5, TimeUnit.SECONDS)    
                .onBackpressureDrop()
                .map(tick -> {      
                    return KafkaRecord.of(random.nextInt(100), String.valueOf(random.nextInt(100)));
                });
    }                  
}

```

Take note on the line that says @Outgoing("<TOPIC-NAME>"). For the purposes of this story we will use the INBOUND topic name that we created in the Event Streams Topic step earlier. Replace whatever is inside the quotation marks.

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
mp.messaging.outgoing.INBOUND.ssl.truststore.password=<password>
```

7. Replace <es-bootstrap-address> with the address of your Event Streams bootstrap server address that we took note of earlier. 
   
```mp.messaging.connector.smallrye-kafka.bootstrap.servers=<es-bootstrap-address>```

8. Replace <APIKey> with your API Key obtained earlier.
   
```mp.messaging.outgoing.INBOUND.sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
            username="token" \
            password="<APIKey>";
```
            
9. Provide the file path to your Event Streams .jks certificate file. Replace </filepath-to-es-truststorefile/>

```mp.messaging.outgoing.INBOUND.ssl.truststore.location=</filepath-to-es-truststorefile/>es-cert.jks```

10. Provide the truststore password. By default it should just be password.

```mp.messaging.outgoing.INBOUND.ssl.truststore.password=<password>```


11. Great! We now have our simple Quarkus Kafka Producer with our Event Streams credentials. We can now test the connection.

12. Run the producer code by running the following command 

```./mvnw quarkus:dev```

13. Since the code sends a message every 5 seconds, you can leave it on for a bit or you can change it to send it more frequently. Check out the Event Streams instance in the browser UI topic for messages. You can click the message under "Indexed Timestamp" to see the contents and details of the message.

![ES Topic Messages](https://github.com/jackyng88/cloudpak-eventstreams-story/raw/master/supporting-pictures/Event%20Streams%20topic%20messages.png)


## Creating AWS S3 Bucket and setting up the AWS S3 Bucket Policy - 

For more information on S3 Bucket policies you can read up [here](https://docs.aws.amazon.com/AmazonS3/latest/dev/example-bucket-policies.html)

1. Assuming your AWS username/IAM has the proper AmazonS3FullAccessPolicy we can proceed with setting up the AWS S3 bucket for use with the connectors. 

2. Traverse to the AWS S3 section.

3. Create an S3 Bucket. 

![Create S3 Bucket](https://github.com/jackyng88/cloudpak-eventstreams-story/raw/master/supporting-pictures/Create%20S3%20Bucket.png)

4. In the Configure Options section you have access to extra features for your S3 Bucket. For the purposes of this flow you won't need any of these but you can feel free to use them if you would like.

![Configure S3 Options](https://github.com/jackyng88/cloudpak-eventstreams-story/raw/master/supporting-pictures/AWS%20S3%20Configure%20Options.png)

5. For the purposes of this simple use case we can uncheck "Block all public access" and then check "I acknowledge that the current settings may result in this bucket and the objects within becoming public". You can change this back to block access to the bucket later if you so choose.

![S3 Permissions](https://github.com/jackyng88/cloudpak-eventstreams-story/raw/master/supporting-pictures/AWS%20S3%20Permissions.png)

6. Review your bucket options and then Create bucket.

7. Once the S3 bucket has been created; click on your newly created bucket and then from the menu select "Permission" and then "Bucket Policy".

![S3 Bucket Policy](https://github.com/jackyng88/cloudpak-eventstreams-story/raw/master/supporting-pictures/AWS%20S3%20Bucket%20Policy.png)

8. Next, click on "Policy generator". This will open a new window/tab.

![S3 Policy Generator](https://github.com/jackyng88/cloudpak-eventstreams-story/raw/master/supporting-pictures/AWS%20S3%20Policy%20Generator.png)

9. In this Policy generator, from the "Select type of Policy drop-down" select "S3 Bucket Policy".

10. Under principal you will need to fill it with the IAM user ID in various fashions. For more information on that you can go to [AWS Principal Documentation](https://docs.aws.amazon.com/AmazonS3/latest/dev/s3-bucket-user-policy-specifying-principal-intro.html). For instance replace the below - 

```arn:aws:iam::<account-id>:user/<iam-user-id>```

```arn:aws:iam::123456789012:user/user@test.com```


11. From the "Actions" drop-down select "List Bucket"

12. For "Amazon Resource Name (ARN)" fill in the below and replace <bucket-name> with your S3 bucket name - 

```arn:aws:s3:::<bucket-name>/```

13. Hit Create Statement.

14. Now we need to repeat the previous Steps from 9 to 13. We need a second statement to allow us to access the Objects themselves in the bucket. The previous "List Bucket" steps works on a Bucket-level and the next policy statement is for the Object-level.

15. In this new statement the "Policy type", "Principal", are the same. Under the "Actions" drop-down menu select the following options - "DeleteObject", "GetObject", and "PutObject". 

16. For "Amazon Resource Name (ARN)" it's almost the same as what we entered previously i.e. arn:aws:s3:::bucket-name/ however we need to append a * at the end. The reason for this is to allow us to have access to all the Objects within that bucket. Without this it will not behave properly.
   
```arn:aws:s3:::<bucket-name>/*```

17. Create the Statement and should look similar to below - 

![S3 Policy Generator Statements](https://github.com/jackyng88/cloudpak-eventstreams-story/blob/master/supporting-pictures/AWS%20S3%20Policy%20Generator%20Statements.png)


18. Finally hit "Generate Policy" and copy this JSON into your S3 Bucket's Bucket policy. It should look similar to something like this - 

![S3 Bucket Policy Implemented](https://github.com/jackyng88/cloudpak-eventstreams-story/blob/master/supporting-pictures/AWS%20S3%20Bucket%20Policy%20implemented.png)

19. Lastly hit the "Save" button.

## Setting up the KafkaConnect Strimzi Operator - 

1. As part of the pre-requisites this assumes that you have a 4.x OpenShift Container Platform cluster we will use the Strimzi Operator to deploy our Kafka cluster. 

2. In your OpenShift Web Console, in the "ADMINISTRATOR" view. This is in the top left most portion of the menu. Go to "Operators" > "OperatorHub".

![OperatorHub](https://github.com/jackyng88/cloudpak-eventstreams-story/blob/master/supporting-pictures/Operator%20Hub.png)

3. Type "Strimzi" into the Search Bar.

![Strimzi](https://github.com/jackyng88/cloudpak-eventstreams-story/blob/master/supporting-pictures/Strimzi.png)

4. 
