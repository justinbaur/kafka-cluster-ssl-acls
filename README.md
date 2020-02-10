# kafka-cluster-ssl-acls
Setup a kafka cluster in docker to test ssl acls pattern

# Installation
## Requirements
- Docker
- Java for keytool
- Openssl
- Kafka CLI

## Cluster Setup
- Export KAFKA_SSL_SECRETS_DIR to the location of this repositories secrets folder
- Run secrets/create-certs.sh
- Run docker-compose up

# Provision
- Setup a test user topic against zookeeper as it is not setup with SSL ACLs for this cluster
```bash
kafka-topics --zookeeper localhost:22181 --create --topic tf_test_acl --replication-factor 3 --partitions 12
```
- Setup the test admin topic
```bash
kafka-topics --zookeeper localhost:22181 --create --topic tf_test_acl_admin --replication-factor 3 --partitions 12
```
- Setup ACLs on user topic 
```bash
kafka-acls --authorizer-properties zookeeper.connect=localhost:22181 --add --allow-principal User:CN=TF,OU=kafka,O=kafka,L=kafka,ST=kafka,C=XX --operation All --topic tf_test_acl --cluster --group mygroup
```

# Test
- Start a user consumer to start pulling from the user topic
```bash
kafka-console-consumer --bootstrap-server localhost:19092,localhost:29092,localhost:39092 --topic tf_test_acl --group mygroup --consumer.config secrets/tf-ssl.properties --from-beginning
```

- Start a user producing to the user topic
```bash
kafka-console-producer --broker-list localhost:19092,localhost:29092,localhost:39092 --topic tf_test_acl --producer.config secrets/tf-ssl.properties
```

- Test that the user cannot consume from the admin topic
```bash
kafka-console-consumer --bootstrap-server localhost:19092,localhost:29092,localhost:39092 --topic tf_test_acl_admin --group mygroup --consumer.config secrets/tf-ssl.properties --from-beginning
```

The logs should be something like the following
```
[2020-02-09 23:38:07,723] WARN [Consumer clientId=consumer-mygroup-1, groupId=mygroup] Error while fetching metadata with correlation id 2 : {tf_test_acl_admin=TOPIC_AUTHORIZATION_FAILED} (org.apache.kafka.clients.NetworkClient)
[2020-02-09 23:38:07,725] ERROR [Consumer clientId=consumer-mygroup-1, groupId=mygroup] Topic authorization failed for topics [tf_test_acl_admin] (org.apache.kafka.clients.Metadata)
[2020-02-09 23:38:07,726] ERROR Error processing message, terminating consumer process:  (kafka.tools.ConsoleConsumer$)
org.apache.kafka.common.errors.TopicAuthorizationException: Not authorized to access topics: [tf_test_acl_admin]
Processed a total of 0 messages
```


# Reference
## Client cert setup
The following are the portions of the create-certs.sh that enable the client cert authentication when using the kafka CLI

```bash
keytool -noprompt -keystore kafka.client.keystore.jks -alias TF -validity 365 -genkey -dname CN=TF,OU=kafka,O=kafka,L=kafka,ST=kafka,C=XX -keypass confluent -storepass confluent
keytool -noprompt -keystore kafka.admin.keystore.jks -alias TF -validity 365 -genkey -dname CN=TF_ADMIN,OU=kafka,O=kafka,L=kafka,ST=kafka,C=XX -keypass confluent -storepass confluent

keytool -noprompt -keystore kafka.client.truststore.jks -alias CARoot -import -file snakeoil-ca-1.crt -storepass confluent
keytool -noprompt -keystore kafka.admin.truststore.jks -alias CARoot -import -file snakeoil-ca-1.crt -storepass confluent

keytool -noprompt -keystore kafka.client.keystore.jks -alias TF -certreq -file cert-file-client-tf -storepass confluent
keytool -noprompt -keystore kafka.admin.keystore.jks -alias TF -certreq -file cert-file-client-tf-admin -storepass confluent

openssl x509 -req -CA snakeoil-ca-1.crt -CAkey snakeoil-ca-1.key -in cert-file-client-tf -out cert-signed-client-tf -days 365 -CAcreateserial -passin pass:confluent
openssl x509 -req -CA snakeoil-ca-1.crt -CAkey snakeoil-ca-1.key -in cert-file-client-tf-admin -out cert-signed-client-tf-admin -days 365 -CAcreateserial -passin pass:confluent

keytool -keystore kafka.broker1.truststore.jks -alias TF -import -file cert-signed-client-tf -storepass confluent
keytool -keystore kafka.broker1.truststore.jks -alias TF_ADMIN -import -file cert-signed-client-tf-admin -storepass confluent
keytool -keystore kafka.broker2.truststore.jks -alias TF -import -file cert-signed-client-tf -storepass confluent
keytool -keystore kafka.broker2.truststore.jks -alias TF_ADMIN -import -file cert-signed-client-tf-admin -storepass confluent
keytool -keystore kafka.broker3.truststore.jks -alias TF -import -file cert-signed-client-tf -storepass confluent
keytool -keystore kafka.broker3.truststore.jks -alias TF_ADMIN -import -file cert-signed-client-tf-admin -storepass confluent
```

The following properties in the docker-compose.yml that enable the acls are
```
KAFKA_SSL_CLIENT_AUTH: required
KAFKA_SECURITY_INTER_BROKER_PROTOCOL: SSL
KAFKA_AUTHORIZER_CLASS_NAME: kafka.security.auth.SimpleAclAuthorizer
KAFKA_ALLOW_EVERYONE_IF_NO_ACL_FOUND: "false"
KAFKA_SUPER_USERS: User:CN=TF_ADMIN,OU=kafka,O=kafka,L=kafka,ST=kafka,C=XX;User:CN=broker3.test.confluent.io,OU=TEST,O=CONFLUENT,L=PaloAlto,ST=Ca,C=US;User:CN=broker2.test.confluent.io,OU=TEST,O=CONFLUENT,L=PaloAlto,ST=Ca,C=US;User:CN=broker1.test.confluent.io,OU=TEST,O=CONFLUENT,L=PaloAlto,ST=Ca,C=US
```

User Client SSL Properties
```
ssl.truststore.location=secrets/kafka.client.truststore.jks
ssl.truststore.password=confluent

ssl.keystore.location=secrets/kafka.client.keystore.jks
ssl.keystore.password=confluent

ssl.key.password=confluent
ssl.endpoint.identification.algorithm= 

security.protocol=SSL
```

## Kafka CLI
```bash
export KAFKA_HEAP_OPTS="-Xmx1G -Xms1G"
```