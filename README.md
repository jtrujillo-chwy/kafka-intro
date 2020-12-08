# Getting started with Kafka

## Start the server

You can use docker compose to start the server:
```bash
docker-compose -f kafka-compose.yml up -d
```

## Publish and Consume

You can test publishing and consuming using the console tools:

**Publish**

```bash
docker exec -ti kafka-test kafka-console-producer.sh --topic test-topic --bootstrap-server localhost:9092
```

**Consume**

```bash
docker exec -ti kafka-test kafka-console-consumer.sh --topic test-topic --group "consumer-1" --from-beginning --bootstrap-server localhost:9092
```

Note that if you specify a consumer group, it will show all messages the first time you invoke it, but upon second invocation it will only show new messages. You can see your consumer group info as well:

```bash
docker exec -ti kafka-test kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group "consumer-1"
```
