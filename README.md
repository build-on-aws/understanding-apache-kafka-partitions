# Understanding Apache Kafka Partitions
Every technology has that key concept that people struggle to understand. With databases, is which join clause to use for fetching data from multiple tables. Containers are tricky when you have to pick a storage type given some persistence requirements. With Apache Kafka, the struggle is in understanding partittions. Partitions are at the heart that everything Kafka does. They govern aspects like parallelism, storage, and durability. They also affect how much load Kafka can handle. This repository contains the code and instructions for you to dig a little deeper into partitions in Kafka to fully understand how it works and how it affects clusters.

This repository was also used as the backup material for the session [The Right Number of Kafka Partitions](https://talks.riferrei.com/qX2fay) presented on the Devnexus conference in 2023.

![The Right Number of Partitions for a Kafka Topic](images/preso.png)

Alternatively, you can read [this detailed blog post](https://www.buildon.aws/posts/in-the-land-of-the-sizing-the-one-partition-kafka-topic-is-king/01-what-are-partitions) that explains everything you need to fully understand partitions in Kafka, and the effect they have on the cluster and clients.

## 🚀 Figuring out the # of partitions using load testing

For this exercise, you will execute some load testing to figure out the maximum capacity that one Kafka partition can handle in terms of events per second. You will execute a load test to measure the write throughput, and then another load test to measure the read throughput. Once you have this, you will apply your findings in the formula `NUM_PARTITIONS = MAX(T/W, T/R)`, where:

`T` = your desired number of events per second<br>
`W` = maximum write throughput of one partition<br>
`R` = maximum read throughput of one partition<br>

1. Start two Kafka brokers.
```bash
docker compose up -d
```

💡 By default, the Docker Compose file has only the brokers `broker-1` and `broker-2` uncommented. Make sure the brokers `broker-3` and `broker-4` are commented out before proceeding.

2. Create a topic with only one partition.
```bash
kafka-topics.sh --bootstrap-server localhost:9092 --create --topic load-test --partitions 1 --replication-factor 1
```

3. Execute a load test to measure write throughput.
```bash
kafka-producer-perf-test.sh --producer.config config.properties --throughput 100000 --num-records 1000000 --record-size 1024 --topic load-test
```

💡 In this example, you will be writing `1M` events into the topic, each one with a payload size of `1K`. The idea is to measure a desired write throughput of `100K` events per second.

4. Execute a load test to measure read throughput.
```bash
kafka-consumer-perf-test.sh --bootstrap-server localhost:9092 --messages 1000000 --topic load-test
```

💡 In this example, you will be reading `1M` events from the topic.

## 🔁 Moving data across brokers at the partitions level

For this exercise, you will create a Kafka topic with `8` partitions and write some data on it. You will do this using a cluster with only `2` brokers. Then, you will start two additional brokers, forming a cluster with `4` brokers. You will investigate if the partitions have been automatically moved to the new brokers, and if not, you will move them manually to keep the dataset workload evenly spread.

1. Start two Kafka brokers.
```bash
docker compose up -d
```

💡 By default, the Docker Compose file has only the brokers `broker-1` and `broker-2` uncommented. Make sure the brokers `broker-3` and `broker-4` are commented out before proceeding.

2. Create a topic with only eight partitions.
```bash
kafka-topics.sh --bootstrap-server localhost:9092 --create --topic test --partitions 8 --replication-factor 1
```

3. Write some random data into the topic.
```bash
kafka-producer-perf-test.sh --producer.config config.properties --throughput 1000 --num-records 10000 --record-size 1024 --topic test
```

4. Edit the [Docker Compose file](./docker-compose.yml) to uncomment the brokers `broker-3` and `broker-4`.

5. Start the two new brokers.
```bash
docker compose up -d
```

6. Describe the topic details to check the partition assignment.
```bash
kafka-topics.sh --bootstrap-server localhost:9092 --topic test --describe
```

💡 You should see an output like this:

```bash
Topic: test     TopicId: J-hfYNmHTpqUHGwG41-zJQ PartitionCount: 8       ReplicationFactor: 1    Configs: 
        Topic: test     Partition: 0    Leader: 2       Replicas: 2     Isr: 2
        Topic: test     Partition: 1    Leader: 1       Replicas: 1     Isr: 1
        Topic: test     Partition: 2    Leader: 2       Replicas: 2     Isr: 2
        Topic: test     Partition: 3    Leader: 1       Replicas: 1     Isr: 1
        Topic: test     Partition: 4    Leader: 1       Replicas: 1     Isr: 1
        Topic: test     Partition: 5    Leader: 2       Replicas: 2     Isr: 2
        Topic: test     Partition: 6    Leader: 1       Replicas: 1     Isr: 1
        Topic: test     Partition: 7    Leader: 2       Replicas: 2     Isr: 2
```

💡 As you can see in the leader's configuration, all eight partitions are still assigned to the brokers `broker-1` and `broker-2` even though the cluster now has four brokers.

7. Generate a new partition assignment suggestion.
```bash
kafka-reassign-partitions.sh --bootstrap-server localhost:9092 --broker-list 1,2,3,4 --topics-to-move-json-file partitions.json --generate
```

8. Copy the proposed partition reassignment into the file `suggestion.json`.

9. Execute the proposed partition reassignment file.
```bash
kafka-reassign-partitions.sh --bootstrap-server localhost:9092 --reassignment-json-file suggestion.json --execute
```

10. Describe the topic details to check the partition assignment.
```bash
kafka-topics.sh --bootstrap-server localhost:9092 --topic test --describe
```

💡 You should see an output like this:

```bash
Topic: test     TopicId: J-hfYNmHTpqUHGwG41-zJQ PartitionCount: 8       ReplicationFactor: 1    Configs: 
        Topic: test     Partition: 0    Leader: 3       Replicas: 3     Isr: 3
        Topic: test     Partition: 1    Leader: 4       Replicas: 4     Isr: 4
        Topic: test     Partition: 2    Leader: 1       Replicas: 1     Isr: 1
        Topic: test     Partition: 3    Leader: 2       Replicas: 2     Isr: 2
        Topic: test     Partition: 4    Leader: 3       Replicas: 3     Isr: 3
        Topic: test     Partition: 5    Leader: 4       Replicas: 4     Isr: 4
        Topic: test     Partition: 6    Leader: 1       Replicas: 1     Isr: 1
        Topic: test     Partition: 7    Leader: 2       Replicas: 2     Isr: 2
```

💡 As you can see in the leader's configuration, all eight partitions are now evenly assigned to all brokers.

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.
