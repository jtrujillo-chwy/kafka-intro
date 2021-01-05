# Security and ACLs

Security on Kafka consists of authentication using certificates, plus ACLs. The ACLs are configured using Kafka's CLI tools.

## Sources:

- <https://docs.confluent.io/platform/current/security/security_tutorial.html>
- <https://docs.confluent.io/platform/current/kafka/authorization.html>
- <https://github.com/confluentinc/confluent-platform-security-tools/blob/master/kafka-generate-ssl.sh>

## Creating certificates

### Create the CA

First create a CA:

```
openssl req -new -x509 -keyout ca-key -out ca-cert -days 3650
```

### Create the truststore

Then create a truststore. This defines the certificates a client or server trust.

```
keytool -keystore kafka.client.truststore.jks -alias CARoot -importcert -file ca-cert
```

### Create the server keystore

Keystores store the signed certificates that prove identity. You need to create one for each client or server. Note that there should be no spaces in the CN name!

```
keytool -keystore kafka.server.keystore.jks \
  -alias localhost \
  -keyalg RSA \
  -validity 3650 \
  -genkey \
  -storepass changeme \
  -keypass changeme \
  -dname "CN=root"
```

Sign it with the CA:

```
keytool -keystore kafka.server.keystore.jks -alias localhost -certreq -file cert-file
openssl x509 -req -CA ca-cert -CAkey ca-key -in cert-file -out cert-signed -days 365 -CAcreateserial -passin pass:changeme
keytool -keystore kafka.server.keystore.jks -alias CARoot -importcert -file ca-cert
keytool -keystore kafka.server.keystore.jks -alias localhost -importcert -file cert-signed
rm cert-file ca-cert.srl cert-signed
```

### Create the client keystore

```
keytool -keystore kafka.client.keystore.jks \
  -alias localhost \
  -keyalg RSA \
  -validity 3650 \
  -genkey \
  -storepass changeme \
  -keypass changeme \
  -dname "CN=com.chewy.cse.client,OU=DEMM,O=Chewy,C=US"
```

Sign it with the CA:

```
keytool -keystore kafka.client.keystore.jks -alias localhost -certreq -file cert-file
openssl x509 -req -CA ca-cert -CAkey ca-key -in cert-file -out cert-signed -days 365 -CAcreateserial -passin pass:changeme
keytool -keystore kafka.client.keystore.jks -alias CARoot -importcert -file ca-cert
keytool -keystore kafka.client.keystore.jks -alias localhost -importcert -file cert-signed
```

You can verify the keystores with the below command:

```
keytool -list -v -keystore server.keystore.jks
```

### Publishing

You can publish to the topics now, and the server will authenticate you.

```
kafka-console-producer.sh \
  --broker-list kafka-east-1:9093 \
  --topic origin.test-topic \
  --producer.config /libs/client-ssl.properties

kafka-console-consumer.sh \
  --bootstrap-server kafka-east-1:9093 \
  --topic origin.test-topic \
  --consumer.config /libs/client-ssl.properties \
  --from-beginning
```

### ACLs

Add the following to `server.properties` to enable authorization:

```
authorizer.class.name=kafka.security.authorizer.AclAuthorizer
```

Create a topic:

```
kafka-topics.sh \
  --create \
  --command-config libs/root.properties \
  --topic origin.test-topic --partitions 5 --bootstrap-server kafka-east-1:9093
```

Add producer permissions:

```
kafka-acls.sh \
  --bootstrap-server kafka-east-1:9093 \
  --command-config libs/root.properties \
  --add --allow-principal User:CN=com.chewy.cse.client,OU=DEMM,O=Chewy,C=US \
  --producer \
  --topic origin.test-topic
```

Add consumer permissions. It is broken in 2 commands to enable group wildcarding:

```
kafka-acls.sh \
  --bootstrap-server kafka-east-1:9093 \
  --command-config libs/root.properties \
  --add --allow-principal User:CN=com.chewy.cse.client,OU=DEMM,O=Chewy,C=US \
  --operation Read \
  --operation Describe \
  --topic origin.test-topic

kafka-acls.sh \
  --bootstrap-server kafka-east-1:9093 \
  --command-config libs/root.properties \
  --add --allow-principal User:CN=com.chewy.cse.client,OU=DEMM,O=Chewy,C=US \
  --operation Read \
  --group '*'
```

Verify ACLs:

```
kafka-acls.sh \
  --bootstrap-server kafka-east-1:9093 \
  --list \
  --command-config /libs/root.properties
```

Producing and consuming should now work again.
