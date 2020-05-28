# IBM Event Streams on Cloud Pak for Integration Story/Lab (AWS S3 Specific)

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

## Setting up the Kafka Strimzi Operator - 

1. As part of the pre-requisites this assumes that you have a 4.x OpenShift Container Platform cluster we will use the Strimzi Operator to deploy our Kafka cluster. 

2. In your OpenShift Web Console, in the "ADMINISTRATOR" view. This is in the top left most portion of the menu. Go to "Operators" > "OperatorHub".

![OperatorHub](https://github.com/jackyng88/cloudpak-eventstreams-story/blob/master/supporting-pictures/Operator%20Hub.png)

3. Type "Strimzi" into the Search Bar.

![Strimzi](https://github.com/jackyng88/cloudpak-eventstreams-story/blob/master/supporting-pictures/Strimzi.png)

4. Click on the Strimzi Operator and then click "Install".

![Operator Install](https://github.com/jackyng88/cloudpak-eventstreams-story/blob/master/supporting-pictures/Operator%20Install.png)

5. Make sure that the option to have "All namespaces on the cluster (default)" is checked.

![Operator Subscription](https://github.com/jackyng88/cloudpak-eventstreams-story/blob/master/supporting-pictures/Operator%20Subscription.png)

6. Tail the status of your Strimzi operator install either through the web console or doing while logged in through OpenShift through your terminal. 

```oc get pods -n openshift-operators```

![Operator Installing](https://github.com/jackyng88/cloudpak-eventstreams-story/blob/master/supporting-pictures/Strimzi%20Operator%20Installing.png)



7. When the Strimzi Operator finally says Succeeded in the "Installed Operators" section in the Web console or 1/1 Running in the Pod status we may proceed.



![Operator Success](https://github.com/jackyng88/cloudpak-eventstreams-story/blob/master/supporting-pictures/Strimzi%20Operator%20Success.png)

![Operator Console](https://github.com/jackyng88/cloudpak-eventstreams-story/blob/master/supporting-pictures/Strimzi%20Operator%20Console.png)

## Setting up the Kafka Connect Cluster - 

Note - As stated in the pre-requisites section we will be mirroring the steps followed here at [Kafka Connect to S3 Sink & Source](https://ibm-cloud-architecture.github.io/refarch-eda/scenarios/connect-s3/) for more granular information and reading.

1. Now that we have our Strimzi Kafka Operator installed we need the secrets and appropriate credentials set up.

2. You will need to be logged into your OpenShift Cluster through the terminal. You can do this by going to the OpenShift Web UI and going to the top right and hitting the User (likely kube:admin in this case) and then "Copy Login command" and then "Display Token". Copy and paste the "Log in with this token" command into your terminal.

3. I would advise you to create a new Project/Namespace to separate secrets and logic but that's up to you.

```oc new-project es-s3-test```

4. We will create a new file to store our AWS credentials. Create a new file named ```aws-credentials.properties```

```vi aws-credentials.properties```

5. Place the below into the properties file and use your user/IAM credentials in place of <>.

```
aws_access_key_id=<AKIA123456EXAMPLE>
aws_secret_access_key=<strWrz/bb8%c3po/r2d2EXAMPLEKEY>
```

6. Create the secret from the aws-credentials.properties file. You can use the ```oc get secrets``` command in the proper project/namespace to see if it was created. This secret will be injected into the KafkaConnect cluster later.

```oc create secret generic aws-credentials --from-file=aws-credentials.properties```

7. Like the previous step we need to create another secret for our Event Streams API Key that we gathered from the "Event Streams Security: API Key, Credentials and Certificates" Section earlier. This secret will be injected into the KafkaConnect cluster at run time as well. Replace <eventstreams_api_key> with your API key. This should also be in the ```es-api-key.json``` file earlier if you chose the "Download as JSON" option.

```oc create secret generic eventstreams-apikey --from-literal=password=<eventstreams_api_key>```

8. We will now create/generate the proper certificate for use with the Kafka Connect cluster. By default our Event Streams certificate is a .jks file but we need to convert this to a .crt file. Run the following commands. These commands convert the .jks file to a new es-cert.crt file and then creates a Kubernetes/OpenShift secret for use with the KafkaConnect cluster.

```
keytool -importkeystore -srckeystore es-cert.jks -destkeystore es-cert.p12 -deststoretype PKCS12
openssl pkcs12 -in es-cert.p12 -nokeys -out es-cert.crt
oc create secret generic eventstreams-truststore-cert --from-file=es-cert.crt
```

9. (OPTIONAL) This is an Optional step. Apache Camel by default can log potentially sensitive access key information to the log files. To remedy that we will use a log4j ConfigMap to filter out that potentially sensitive information. Create a log4j.properties file and paste the following into it.

```vi log4j.properties```

```
# Do not change this generated file. Logging can be configured in the corresponding kubernetes/openshift resource.
log4j.appender.CONSOLE=org.apache.log4j.ConsoleAppender
log4j.appender.CONSOLE.layout=org.apache.log4j.PatternLayout
log4j.appender.CONSOLE.layout.ConversionPattern=%d{ISO8601} %p %m (%c) [%t]%n
connect.root.logger.level=INFO
log4j.rootLogger=${connect.root.logger.level}, CONSOLE
log4j.logger.org.apache.zookeeper=ERROR
log4j.logger.org.I0Itec.zkclient=ERROR
log4j.logger.org.reflections=ERROR

# Due to back-leveled version of log4j that is included in Kafka Connect,
# we can use multiple StringMatchFilters to remove all the permutations
# of the AWS accessKey and secretKey values that may get dumped to stdout
# and thus into any connected logging system.
log4j.appender.CONSOLE.filter.a=org.apache.log4j.varia.StringMatchFilter
log4j.appender.CONSOLE.filter.a.StringToMatch=accesskey
log4j.appender.CONSOLE.filter.a.AcceptOnMatch=false
log4j.appender.CONSOLE.filter.b=org.apache.log4j.varia.StringMatchFilter
log4j.appender.CONSOLE.filter.b.StringToMatch=accessKey
log4j.appender.CONSOLE.filter.b.AcceptOnMatch=false
log4j.appender.CONSOLE.filter.c=org.apache.log4j.varia.StringMatchFilter
log4j.appender.CONSOLE.filter.c.StringToMatch=AccessKey
log4j.appender.CONSOLE.filter.c.AcceptOnMatch=false
log4j.appender.CONSOLE.filter.d=org.apache.log4j.varia.StringMatchFilter
log4j.appender.CONSOLE.filter.d.StringToMatch=ACCESSKEY
log4j.appender.CONSOLE.filter.d.AcceptOnMatch=false

log4j.appender.CONSOLE.filter.e=org.apache.log4j.varia.StringMatchFilter
log4j.appender.CONSOLE.filter.e.StringToMatch=secretkey
log4j.appender.CONSOLE.filter.e.AcceptOnMatch=false
log4j.appender.CONSOLE.filter.f=org.apache.log4j.varia.StringMatchFilter
log4j.appender.CONSOLE.filter.f.StringToMatch=secretKey
log4j.appender.CONSOLE.filter.f.AcceptOnMatch=false
log4j.appender.CONSOLE.filter.g=org.apache.log4j.varia.StringMatchFilter
log4j.appender.CONSOLE.filter.g.StringToMatch=SecretKey
log4j.appender.CONSOLE.filter.g.AcceptOnMatch=false
log4j.appender.CONSOLE.filter.h=org.apache.log4j.varia.StringMatchFilter
log4j.appender.CONSOLE.filter.h.StringToMatch=SECRETKEY
log4j.appender.CONSOLE.filter.h.AcceptOnMatch=false
```

10. (OPTIONAL) We can now create the ConfigMap from the newly created properties file.

```oc create configmap custom-connect-log4j --from-file=log4j.properties```

11. We will now deploy the base KafkaConnect Cluster using KafkaConnectS2I (Source to Image) custom resource. Create a new ```kafka-connect.yaml``` file and paste the following. 

```vi kafka-connect.yaml```

```
apiVersion: kafka.strimzi.io/v1alpha1
kind: KafkaConnectS2I
metadata:
  name: connect-cluster-101
  annotations:
    strimzi.io/use-connector-resources: "true"
spec:
  #logging:
  #  type: external
  #  name: custom-connect-log4j
  replicas: 1
  bootstrapServers: <your-bootstrap-server-address:443>
  tls:
    trustedCertificates:
      - certificate: es-cert.crt
        secretName: eventstreams-truststore-cert
  authentication:
    passwordSecret:
      secretName: eventstreams-apikey
      password: password
    username: token
    type: plain
  externalConfiguration:
    volumes:
      - name: aws-credentials
        secret:
          secretName: aws-credentials
  config:
    group.id: connect-cluster-101
    config.providers: file
    config.providers.file.class: org.apache.kafka.common.config.provider.FileConfigProvider
    key.converter: org.apache.kafka.connect.json.JsonConverter
    value.converter: org.apache.kafka.connect.json.JsonConverter
    key.converter.schemas.enable: false
    value.converter.schemas.enable: false
    offset.storage.topic: connect-cluster-101-offsets
    config.storage.topic: connect-cluster-101-configs
    status.storage.topic: connect-cluster-101-status
 ```
 
 NOTE - The following options are things we need to take note of/configure. 
 
 - (OPTIONAL)```spec.logging.name```: The name of the previously configured log4j ConfigMap. You can uncomment those previous three lines if you opted to create the ConfigMap.
 - ```spec.bootstrapServers```: Replace with your Event Streams instance bootstrap server address.
 - ```spec.tls.trustedCertificates[0].secretName```: The name of the OpenShift secret created in Step 8.
 - ```spec.authentication.passwordSecret.secretName```: The name of the OpenShift secret created from the Event Streams API Key created in Step 7.
 - ```spec.externalConfiguration.volumes[0].secret.secretName```: The name of the OpenShift secret created from the AWS credentials created in Step 6.
 - ```spec.config['group.id']```: This should be a unique ID for connecting to the same set of Kafka brokers. If we do not specify a name, multiple KafkaConnect instances will end up using the default id and end up in a race condition as they all try to vie for access.
 - ```spec.config['*.storage.topic']```: As noted in the .yaml file we have ```offset.storage.topic```, ```config.storage.topic```, and ```status.storage.topic```. The name of these topics will need to be created in your Event Streams instance to store the metadata. See the "Creating Event Streams Topics" section for a refresher on creating Event Streams topics. 
 
 
 12. Make sure the ```kafka-connect.yaml``` files values are correctly configured and save it. From the terminal run the following - 
 ```oc apply -f kafka-connect.yaml``` 
 
 13. You can check the status of the pods by running ```oc get pods```. When they're all Running we can proceed.
 
 
 ## Building the Apache Camel Kafka Connect binaries - 
 
 1. We will need to clone from a github repository using HTTPS or SSH depending on preference. Note that the below github repository is a fork of the official [Apache Camel repository](https://github.com/apache/camel-kafka-connector).
 
 ```git clone https://github.com/osowski/camel-kafka-connector.git```
 
 ```git clone git@github.com:osowski/camel-kafka-connector.git```
 
 2. Once successfully cloned, change the directory to the root of the cloned repository. Change to the camel-kafka-connector-0.1.0-branch branch
 
 ```git checkout camel-kafka-connector-0.1.0-branch```
 
 3. We will now build the project with Maven. Note that this can take around 30 minutes! 
 
 ```mvn clean package```
 
 4. Once the build is complete copy the generated S3 artifacts to the core package build artifacts.
 

```cp connectors/camel-aws-s3-kafka-connector/target/camel-aws-s3-kafka-connector-0.1.0.jar core/target/camel-kafka-connector-0.1.0-package/share/java/camel-kafka-connector/```


5. We will now start an OpenShift build from the previously generated S3 artifacts. Note the ```connect-cluster-101-connect``` after start-build. The ```-connect``` was appended automatically when our Kafka Connect cluster pod was created.

```
oc start-build connect-cluster-101-connect --from-dir=./core/target/camel-kafka-connector-0.1.0-package/share/java --follow
```

6. Check the status of your new build. This build should've created a new pod with a name along the lines of ```connect-cluster-101-connect-2-[random-suffix]```. The original pod should have had a name similar to ```connect-cluster-101-connect-1-[random-suffix]``` with a 1 instead of a 2 for instance. Wait for the pod to go into a Running state.

```oc get pods -w```


## Deploying the Kafka to S3 Sink Connector - 

- The Sink connector's function is for taking data from an internal system and moving it to an external system. In this case, our Kafka Topic (internal) to an AWS S3 Bucket (external).

1. Now that we have a Kafka Connect Cluster up, we will need to create an S3 Sink Connector with a KafkaConnect custom resource definition managed by the Strimzi Operator that we deployed.

2. Create a ```kafka-sink-connector.yaml``` file.

```vi kafka-sink-connector.yaml```

3. Paste the following into the newly created .yaml file.

```apiVersion: kafka.strimzi.io/v1alpha1
kind: KafkaConnector
metadata:
  name: s3-sink-connector
  labels:
    strimzi.io/cluster: connect-cluster-101
spec:
  class: org.apache.camel.kafkaconnector.CamelSinkConnector
  tasksMax: 1
  config:
    key.converter: org.apache.kafka.connect.storage.StringConverter
    value.converter: org.apache.kafka.connect.storage.StringConverter
    topics: INBOUND
    camel.sink.url: aws-s3://<my-s3-bucket>?keyName=${date:now:yyyyMMdd-HHmmssSSS}-${exchangeId}
    camel.sink.maxPollDuration: 10000
    camel.component.aws-s3.configuration.autocloseBody: false
    camel.component.aws-s3.accessKey: ${file:/opt/kafka/external-configuration/aws-credentials/aws-credentials.properties:aws_access_key_id}
    camel.component.aws-s3.secretKey: ${file:/opt/kafka/external-configuration/aws-credentials/aws-credentials.properties:aws_secret_access_key}
    camel.component.aws-s3.region: <US_EAST_1>
```

Similar to the kafka-connect.yaml file there are a few things here to keep note of - 

- ```metadata.labels.strimzi.io/cluster```: This needs to match the name of the Kafka Connect cluster.
- ```spec.config.topics```: The name of the topic we created earlier for Inbound messages. The INBOUND topic was created in the "Creating Event Streams Topics" section for example.
- ```spec.config.camel.sink.url"```: You will need to replace <my-s3-bucket> with the name of your AWS S3 bucket. If the name of the created bucket is ```my-s3-bucket``` then ```camel.sink.url``` will look like ```camel.sink.url: aws-s3://my-s3-bucket?keyName=${date:now:yyyyMMdd-HHmmssSSS}-${exchangeId}```. 
- ```spec.config.camel.component.aws-s3.region```: You will need to indicate the region that your S3 bucket is in. Replace <US_EAST_1> with your bucket's region.

4. Save the file and create the custom resource definition for the connector by applying it.

```oc apply -f kafka-sink-connector.yaml```

5. You can see if the connector was created by running the following - 

```oc get kafkaconnector```

6. This can take a few minutes but you can check the logs of the KafkaConnector by using the following command - 

```oc get kafkaconnector s3-sink-connector -o yaml```

7. In the logs once the section of the ```Status.connectorStatus.connector.state:``` object returns `RUNNING` it means that the connector was successfully created and we can proceed!

![S3 Sink Connector Running](https://github.com/jackyng88/cloudpak-eventstreams-story/blob/master/supporting-pictures/S3%20Sink%20Running.png)


## Deploying the Kafka to S3 Source Connector - 

- The function of the Source connector is similar to the Sink connector but does it in the opposite direction. It takes data from an external system and moves it into an internal system. In this case AWS S3 Bucket (external) to our Kafka topic (internal).

1. Now that we have a Kafka Connect Cluster up and the S3 Sink connector, it's time to deploy the S3 Source connector. You can forego this section initially if you want to test if your messages are arriving into your INBOUND topic if you so choose.

2. Create a ```kafka-source-connector.yaml``` file.

```vi kafka-source-connector.yaml```

3. Paste the following into the newly created .yaml file.

```apiVersion: kafka.strimzi.io/v1alpha1
kind: KafkaConnector
metadata:
  name: s3-source-connector
  labels:
    strimzi.io/cluster: connect-cluster-101
spec:
  class: org.apache.camel.kafkaconnector.CamelSourceConnector
  tasksMax: 1
  config:
    key.converter: org.apache.kafka.connect.storage.StringConverter
    value.converter: org.apache.camel.kafkaconnector.awss3.converters.S3ObjectConverter
    # topics: my-target-topic
    camel.source.kafka.topic: OUTBOUND
    camel.source.url: aws-s3://<my-s3-bucket>?autocloseBody=false
    camel.source.maxPollDuration: 10000
    camel.component.aws-s3.accessKey: ${file:/opt/kafka/external-configuration/aws-credentials/aws-credentials-secret.properties:aws_access_key_id}
    camel.component.aws-s3.secretKey: ${file:/opt/kafka/external-configuration/aws-credentials/aws-credentials-secret.properties:aws_secret_access_key}
    camel.component.aws-s3.region: <US_EAST_1>
```

Similar to the kafka-connect.yaml (and kafka-sink-connector.yaml) files there are a few things here to keep note of - 

- ```metadata.labels.strimzi.io/cluster```: This needs to match the name of the Kafka Connect cluster.
- ```spec.config.topics```: The name of the topic we created earlier for Outbind messages (in this scenario). The OUTBOUND topic was created in the "Creating Event Streams Topics" section for example.
- ```spec.config.camel.source.url"```: You will need to replace <my-s3-bucket> with the name of your AWS S3 bucket. If the name of the created bucket is ```my-s3-bucket``` then ```camel.source.url``` will look like ```camel.source.url: aws-s3://my-s3-bucket?autocloseBody=false```. 
- ```spec.config.camel.component.aws-s3.region```: You will need to indicate the region that your S3 bucket is in. Replace <US_EAST_1> with your bucket's region.

4. Save the file and create the custom resource definition for the connector by applying it.

```oc apply -f kafka-source-connector.yaml```

5. You can see if the connector was created by running the following - 

```oc get kafkaconnector```

6. This can take a few minutes but you can check the logs of the KafkaConnector by using the following command - 

```oc get kafkaconnector s3-source-connector -o yaml```

7. Like the Sink connector within the logs once the section of the ```Status.connectorStatus.connector.state:``` object returns `RUNNING` it means that the connector was successfully created and we can proceed!


![S3 Source Connector Running](https://github.com/jackyng88/cloudpak-eventstreams-story/blob/master/supporting-pictures/S3%20Source%20Running.png)



## Summary of the End-to-End Scenario - 

Below are the following steps that will happen.

- With the Quarkus Kafka application we will send messages every 5 seconds to our INBOUND topic.

- Once the messages make it ot the INBOUND topic, the Kafka Connect S3 Sink Connector will pull the messages from the INBOUND topic and place them into the AWS S3 Bucket.

- Once the messages/objects are in the AWS S3 bucket, the Kafka Connect S3 Source connector will pull (and delete) the objects from the S3 bucket and put them into the OUTBOUND topic.


## Test the Entire Flow - 

1. Now that we have all the previous steps setup we can now test the entire flow.

2. Start our Quarkus Kafka Producer application to send messages to the Event Streams INBOUND topic.

`./mvnw quarkus:dev`

![Quarkus Run Success](https://github.com/jackyng88/cloudpak-eventstreams-story/blob/master/supporting-pictures/Quarkus%20Run%20Success.png)

3. Go to your Event Streams instance on Cloud Pak for Integration. Traverse to the Topics menu and select your OUTBOUND topic and then go to "Messages". Choose the "Live" option to see an up-to-date stream of your incoming messages.

4. Your messages should be propagating the OUTBOUND topic automatically even though we sent messages to the INBOUND topic.

![End to End Success](https://github.com/jackyng88/cloudpak-eventstreams-story/blob/master/supporting-pictures/End%20to%20End%20Success.png)
