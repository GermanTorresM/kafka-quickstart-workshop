# Part 4 - Processing data with Kafka Streams

In this part of the workshop, we will learn about Kafka Streams and how it can be used to process streams of data in real time. Make sure that you've completed Part 3, since this part follows on from [Part 3](../part3/README.md).

## Kafka Streams

[Kafka Streams](https://kafka.apache.org/documentation/streams/) is a client library for building applications and microservices, where the input and output data are stored in Kafka. It combines the simplicity of writing and deploying standard Java and Scala applications on the client-side with the benefits of Kafka's server-side cluster technology.

## Kafka Streams APIs

Kafka Streams exposes 2 APIs:

- **DSL**: This Domain Specific Language API makes it easy to express processing operations in just a few lines of code. This API is designed to allow most common use cases, making it especially easy to get started.

- **Processor**: This Processor API is lower level than the DSL API. This API gives developers full control and allows defining custom processors and accessing data stores.

Kafka Streams performs 1 record-at-a-time processing operations which enables near real-time processing of data. It also offers exactly-once processing semantics to ensure processing results are correct even in case of failure from the Kafka cluster or Kafka Streams itself.

## Processing Operations

There are 2 types of processing operations:

- **Stateless**: These transformations don't require Kafka Streams to keep state in order to be performed. Stateless operations include: filtering, branching, switching between KStreams and KTable abstractions, and so on.

- **Stateful**: These operations require Kafka Streams to maintain state in order to be performed. These include: joining, aggregating, and windowing.

A Kafka Streams application is a combination of processing operations linked together to accomplish a task. The logical flow of data is called a topology.

## WordCountDemo sample application

One of the built-in sample Kafka Streams application is [WordCountDemo](https://github.com/apache/kafka/blob/2.5/streams/examples/src/main/java/org/apache/kafka/streams/examples/wordcount/WordCountDemo.java).

This sample application consumes an input topic, counts how many times words appear, and writes the results back into another topic.

Let's use `WordCountDemo` to count words in our topic `streams-plaintext-input`.

### Updating WordCountDemo

Unfortunately, `WordCountDemo` is not currently configurable so we will need to make a few small code changes.

1. In a text editor, open `streams/examples/src/main/java/org/apache/kafka/streams/examples/wordcount/WordCountDemo.java` and make these changes:

- Add the following imports after line 32:

    ```java
    import java.io.FileInputStream;
    import java.io.IOException;
    ```
- Replace line 78 by `final Properties props = getStreamsConfig(args);`

- Replace the `getStreamsConfig()` method from line 50 to 63 by the following block:

    ```java
    static Properties getStreamsConfig(final String[] args) throws IOException {
        final Properties props = new Properties();
        if (args != null && args.length > 0) {
            try (final FileInputStream fis = new FileInputStream(args[0])) {
                props.load(fis);
            }
            if (args.length > 1) {
                System.out.println("Warning: Some command line arguments were ignored. This demo only accepts an optional configuration file.");
            }
        }
        props.putIfAbsent(StreamsConfig.APPLICATION_ID_CONFIG, "streams-wordcount");
        props.putIfAbsent(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.putIfAbsent(StreamsConfig.CACHE_MAX_BYTES_BUFFERING_CONFIG, 0);
        props.putIfAbsent(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass().getName());
        props.putIfAbsent(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass().getName());

        // setting offset reset to earliest so that we can re-run the demo code with the same pre-loaded data
        // Note: To re-run the demo, you need to use the offset reset tool:
        // https://cwiki.apache.org/confluence/display/KAFKA/Kafka+Streams+Application+Reset+Tool
        props.putIfAbsent(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        return props;
    }
    ```

2. After making these updates done, we need to recompile it. You can do that by running:

```sh
./gradlew assemble
```

### Running WordCountDemo

Start `WordCountDemo` by using `kafka-run-class.sh` and specify our configuration file.

```sh
> bin/kafka-run-class.sh org.apache.kafka.streams.examples.wordcount.WordCountDemo ${CONFIG_FILE}
```

The Kafka Streams application will run until interrupted, such as by pressing `CTRL+C`.

### Checking the result

By default, `WordCountDemo` writes its output in the `streams-wordcount-output` topic. We can use a consumer to check the result:

```sh
> bin/kafka-console-consumer.sh --bootstrap-server ${BOOTSTRAP_SERVERS} \
  --consumer.config ${CONFIG_FILE} \
  --topic streams-wordcount-output \
  --from-beginning \
  --formatter kafka.tools.DefaultMessageFormatter \
  --property print.key=true \
  --property print.value=true \
  --property key.deserializer=org.apache.kafka.common.serialization.StringDeserializer \
  --property value.deserializer=org.apache.kafka.common.serialization.LongDeserializer
```

While the Kafka Streams application is running, you can keep adding lines to our file and see new counts being emitted.

```sh
> echo "another line" >> /tmp/file-source.txt
```

You can also use a producer to directly write records into the input topic:

```sh
> bin/kafka-console-producer.sh --bootstrap-server ${BOOTSTRAP_SERVERS} \
  --producer.config ${CONFIG_FILE} --topic streams-plaintext-input
line from the producer
```

### Looking at the code

This sample application is implemented with the DSL API. The core of the logic lies in the following few lines of codes:

```java
final KStream<String, String> source = builder.stream(INPUT_TOPIC);

final KTable<String, Long> counts = source
    .flatMapValues(value -> Arrays.asList(value.toLowerCase(Locale.getDefault()).split(" ")))
    .groupBy((key, value) -> value)
    .count();

counts.toStream().to(OUTPUT_TOPIC, Produced.with(Serdes.String(), Serdes.Long()));
```

The first line defines the input stream. In this case, it is taking records from our input topic `streams-plaintext-input` and giving us a [KStream](http://kafka.apache.org/25/javadoc/org/apache/kafka/streams/kstream/KStream.html) instance.

The middle statement performs the processing operations. For each record (which is a line in our input file):

    - First, the record value is split by spaces to retrieve all the words.
    - Then, words are grouped together.
    - Finally, it counts how many times each word appear and emit a record `(word, count)` per word.

The last line defines the output stream. Here, we are sending it to our output topic `streams-wordcount-output` and we have to specify a serializer for both the key (words) and the value (current count).

The same logic is also implemented using the Processor API, see [WordCountProcessorDemo.java](https://github.com/apache/kafka/blob/trunk/streams/examples/src/main/java/org/apache/kafka/streams/examples/wordcount/WordCountProcessorDemo.java)

## Next Steps

Continue to [workshop summary](../part4/summary.md).
