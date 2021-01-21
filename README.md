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

## MirrorMaker

MirrorMaker is a process that comes with Kafka and can mirror topics to a different region. The approach will be to have 4 regions, with 2 topics per domain:

- origin.topic-name
- topic-name

Then MirrorMaker will mirror from the origin topics to the destination topics. Producers will only produce to origin topics.

Replication path:

```
origin.topic-name -> MirrorMaker -> [east-1, east-2, west-1, west-2]
```

### Running MirrorMaker on the server

First, add the topic replace message handler to the class path. This is necessary to rename topics:

```
export CLASSPATH=$CLASSPATH:/libs/topic-name-replace.jar
```

- I have written code to use patterns, but it is similar to this: <https://github.com/opencore/mirrormaker_topic_rename>
- Repo is here: <https://github.com/jtrujillo-chwy/mirrormaker-topic-rename>

Then run MirrorMaker:

```
kafka-mirror-maker.sh \
  --consumer.config libs/mirrormaker-consumer.properties \
  --producer.config libs/mirrormaker-producer.properties \
  --num.streams 3 \
  --whitelist "origin.*" \
  --message.handler com.chewy.cse.kafkahandlers.RenameTopicHandler --message.handler.args "origin.->" &
```

### Producing and Consuming

You can write messages to east-1:

```bash
kafka-console-producer.sh --topic origin.test-topic --bootstrap-server kafka-east-1:9092
```

And then consume from any other broker and you should see the same messages:

```bash
kafka-console-consumer.sh --topic test-topic --group "east-1-consumer" --from-beginning --bootstrap-server kafka-east-1:9092
```

```bash
kafka-console-consumer.sh --topic test-topic --group "east-2-consumer" --from-beginning --bootstrap-server kafka-east-2:9092
```

```bash
kafka-console-consumer.sh --topic test-topic --group "west-1-consumer" --from-beginning --bootstrap-server kafka-west-1:9092
```

```bash
kafka-console-consumer.sh --topic test-topic --group "west-2-consumer" --from-beginning --bootstrap-server kafka-west-2:9092
```
