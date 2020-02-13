# kafka-cluster-ssl-acls
Setup a kafka cluster in docker to test ssl acls pattern

# Installation
## Requirements
- Docker
- Java for keytool
- Openssl
- Kafka CLI

### Optional
- Terraform

## Cluster Setup
- Export KAFKA_SSL_SECRETS_DIR to the location of this repositories secrets folder
- Run secrets/create-certs.sh
- Run docker-compose up

# Provision
- Setup a test user topic against zookeeper as it is not setup with SSL ACLs for this cluster
```bash
kafka-topics --zookeeper localhost:22181 --create --topic test_acl --replication-factor 3 --partitions 12
```
- Setup the test admin topic
```bash
kafka-topics --zookeeper localhost:22181 --create --topic test_acl_admin --replication-factor 3 --partitions 12
```
- Setup ACLs on user topic 
```bash
kafka-acls --authorizer-properties zookeeper.connect=localhost:22181 --add --allow-principal User:CN=user.test,OU=TEST,O=TEST,L=TEST,ST=TEST,C=US --operation All --topic test_acl --cluster --group mygroup
```

# Testing
## Kafka CLI
- Start a user consumer to start pulling from the user topic
```bash
kafka-console-consumer --bootstrap-server localhost:19092,localhost:29092,localhost:39092 --topic test_acl --group mygroup --consumer.config secrets/user-ssl.properties --from-beginning
```

- Start a user producing to the user topic
```bash
kafka-console-producer --broker-list localhost:19092,localhost:29092,localhost:39092 --topic test_acl --producer.config secrets/user-ssl.properties
```

- Test that the user cannot consume from the admin topic
```bash
kafka-console-consumer --bootstrap-server localhost:19092,localhost:29092,localhost:39092 --topic test_acl_admin --group mygroup --consumer.config secrets/user-ssl.properties --from-beginning
```

The logs should be something like the following
```
[2020-02-09 23:38:07,723] WARN [Consumer clientId=consumer-mygroup-1, groupId=mygroup] Error while fetching metadata with correlation id 2 : {tf_test_acl_admin=TOPIC_AUTHORIZATION_FAILED} (org.apache.kafka.clients.NetworkClient)
[2020-02-09 23:38:07,725] ERROR [Consumer clientId=consumer-mygroup-1, groupId=mygroup] Topic authorization failed for topics [tf_test_acl_admin] (org.apache.kafka.clients.Metadata)
[2020-02-09 23:38:07,726] ERROR Error processing message, terminating consumer process:  (kafka.tools.ConsoleConsumer$)
org.apache.kafka.common.errors.TopicAuthorizationException: Not authorized to access topics: [tf_test_acl_admin]
Processed a total of 0 messages
```

## Terraform
### Setup
- [Get Terraform](https://learn.hashicorp.com/terraform/getting-started/install)
- [Grab the kafka provider](https://github.com/Mongey/terraform-provider-kafka/releases)

### Usage
- Run terraform against templates in ./terraform

# Reference
## Client cert setup
The following are the portions of the create-certs.sh that enable the client cert authentication when using the kafka CLI

- Generate a key pair for the client
- Sign it with the CA
- Import signed cert into broker trust

```bash
	keytool -genkey -noprompt \
				 -alias $i \
				 -dname "CN=$i.test,OU=TEST,O=TEST,L=TEST,ST=TEST,C=US" \
				 -keystore kafka.$i.keystore.jks \
				 -keyalg RSA \
				 -storepass kafkatest \
				 -keypass kafkatest

	keytool -noprompt -keystore kafka.$i.keystore.jks -alias CARoot -import -file snakeoil-ca-1.crt -storepass kafkatest

	keytool -noprompt -keystore kafka.$i.keystore.jks -alias $i -certreq -file cert-file-client-$i -storepass kafkatest

	openssl x509 -req -CA snakeoil-ca-1.crt -CAkey snakeoil-ca-1.key -in cert-file-client-$i -out cert-signed-client-$i -days 365 -CAcreateserial -passin pass:kafkatest

	# Create truststore and import the CA cert.
	keytool -noprompt -keystore kafka.$i.truststore.jks -alias CARoot -import -file snakeoil-ca-1.crt -storepass kafkatest -keypass kafkatest

	# Add signed client cert to brokers trust
	keytool -noprompt -keystore kafka.broker1.truststore.jks -alias $i -import -file cert-signed-client-$i -storepass kafkatest
	keytool -noprompt -keystore kafka.broker2.truststore.jks -alias $i -import -file cert-signed-client-$i -storepass kafkatest
	keytool -noprompt -keystore kafka.broker3.truststore.jks -alias $i -import -file cert-signed-client-$i -storepass kafkatest
```

The following properties in the docker-compose.yml that enable the acls are
```
      KAFKA_SSL_CLIENT_AUTH: required
      KAFKA_SECURITY_INTER_BROKER_PROTOCOL: SSL
      KAFKA_AUTHORIZER_CLASS_NAME: kafka.security.auth.SimpleAclAuthorizer
      KAFKA_ALLOW_EVERYONE_IF_NO_ACL_FOUND: "false"
      KAFKA_SUPER_USERS: User:CN=admin.test,OU=TEST,O=TEST,L=TEST,ST=TEST,C=US;User:CN=broker3.test,OU=TEST,O=TEST,L=TEST,ST=TEST,C=US;User:CN=broker2.test,OU=TEST,O=TEST,L=TEST,ST=TEST,C=US;User:CN=broker1.test,OU=TEST,O=TEST,L=TEST,ST=TEST,C=US
```

User SSL Properties
```
ssl.truststore.location=secrets/kafka.user.truststore.jks
ssl.truststore.password=kafkatest

ssl.keystore.location=secrets/kafka.user.keystore.jks
ssl.keystore.password=kafkatest

ssl.key.password=kafkatest
ssl.endpoint.identification.algorithm= 

security.protocol=SSL
```

## Kafka CLI
```bash
export KAFKA_HEAP_OPTS="-Xmx1G -Xms1G"
```