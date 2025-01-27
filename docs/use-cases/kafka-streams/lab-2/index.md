# Kafka Streams Test Lab 2

## Overview

- This is a continuation of the previous [Lab 1](/use-cases/kafka-streams/lab-1/). You should complete Lab 1 first before you get started here.
- There's a few more pre-reqs (if you so choose to use them) outlined below.

## Scenario Prerequisites

**Java**

- For the purposes of this lab we suggest Java 8+

**Maven**

- Maven will be needed for bootstrapping our application from the command-line and running
our application.

**An IDE of your choice**

- Ideally an IDE that supports Quarkus (such as Visual Studio Code)

**OpenShift Container Platform, IBM Cloud Pak for Integration and IBM Event Streams**

- This is an optional portion of the lab for those who have access to an OCP Cluster where IBM Cloud Pak for Integration has been installed on top and an IBM Event Streams instance deployed.

- **The following are optional**
- **OpenShift Container Platform** v4.4.x
- **IBM Cloud Pak for Integration** CP4I2022.2
- **IBM Event Streams**: IBM Event Streams v10 or later preferrably. If you are using a previous version of IBM Event Streams, there are some differences as to how you would configure `application.properties` to establish the connection to IBM Event Streams.

## Adding in more Kafka Streams operators

In this section we are going to to add more functionality to our previous test class in order to see, work with and understand more Kafka Streams operators.

- Add the following definitions to the `TestFinancialMessage.java` Java class:

```java
private static String tradingTable = "tradingTable";
private static String tradingStoreName = "tradingStore";
private static TestInputTopic<String, String> tradingTableTopic;
```

- Add the following Store and KTable definitions inside the `buildTopology()` function to support the trading fuctionality we are adding to our application:

```java
KeyValueBytesStoreSupplier tradingStoreSupplier = Stores.persistentKeyValueStore(tradingStoreName);

KTable<String, String> stockTradingStore = builder.table(tradingTable,
            Consumed.with(Serdes.String(), Serdes.String()),
            Materialized.as(tradingStoreSupplier));
```

- Add the following import to your Java class so that you can use objects of type KTable:

```java
import org.apache.kafka.streams.kstream.KTable;
```

- Edit the `branch[1]` logic again to create new `KeyValue` pairs of `userId` and `stockSymbol`

```java
branches[1].filter(
            (key, value) -> (value.totalCost > 5000)
        )
        .map(
            (key, value) -> KeyValue.pair(value.userId, value.stockSymbol)
        )
        .to(
            tradingTable,
            Produced.with(Serdes.String(), Serdes.String())
        );
```

Notice that, previously, we wrote straight to `outTopic`. However, we are now writing to
a KTable which we can query in our tests by using the State Store it is materialised as.

- Before we create a test for the new functionality, remove or comment out the previous existing tests cases as these do no longer apply.

- Create a new test with the code below:

```java
    @Test
    public void filterAndMapNewPair() {

        FinancialMessage mock = new FinancialMessage(
            "1", "MET", "SWISS", 12, 1822.38, 21868.55, 94, 7, true
        );
        inTopic.pipeInput("1", mock);

        KeyValueStore<String,ValueAndTimestamp<String>> tableStore = testDriver.getTimestampedKeyValueStore(tradingStoreName);
        Assertions.assertEquals(1, tableStore.approximateNumEntries());
        Assertions.assertEquals("MET", tableStore.get("1").value());
    }
```

The first assertion checks whether the store has a record. The second assertion checks that the mock record that we
inserted has the correct value as our map function created new KeyValue pairs of type `<userId, stockSymbol>`.

- Test the application by running the following:

```shell
./mvnw clean verify
```

- You should see the tests pass with the following output:

```shell
[INFO] -------------------------------------------------------
[INFO]  T E S T S
[INFO] -------------------------------------------------------
[INFO] Running com.ibm.GreetingResourceTest
2021-01-16 20:47:06,478 INFO  [io.quarkus] (main) Quarkus 1.10.5.Final on JVM started in 2.086s. Listening on: http://localhost:8081
2021-01-16 20:47:06,479 INFO  [io.quarkus] (main) Profile test activated. 
2021-01-16 20:47:06,479 INFO  [io.quarkus] (main) Installed features: [cdi, kafka-streams, resteasy, resteasy-jsonb]
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 5.611 s - in com.ibm.GreetingResourceTest
[INFO] Running com.ibm.garage.cpat.lab.TestFinancialMessage
2021-01-16 20:47:08,248 INFO  [org.apa.kaf.str.pro.int.StateDirectory] (main) stream-thread [main] Deleting state directory 0_0 for task 0_0 as user calling cleanup.
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.276 s - in com.ibm.garage.cpat.lab.TestFinancialMessage
[INFO] Running com.ibm.garage.cpat.lab.TestLoadKtableFromTopic
C01:Health Care
C02:Finance
C03:Consumer Services
C04:Transportation
C05:Capital Goods
C06:Public Utilities
sector-types-store
2021-01-16 20:47:08,290 WARN  [org.apa.kaf.str.sta.int.RocksDBStore] (main) Closing 1 open iterators for store sector-types-store
2021-01-16 20:47:08,292 INFO  [org.apa.kaf.str.pro.int.StateDirectory] (main) stream-thread [main] Deleting state directory 0_0 for task 0_0 as user calling cleanup.
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.035 s - in com.ibm.garage.cpat.lab.TestLoadKtableFromTopic
2021-01-16 20:47:08,325 INFO  [io.quarkus] (main) Quarkus stopped in 0.026s
[INFO] 
[INFO] Results:
[INFO] 
[INFO] Tests run: 3, Failures: 0, Errors: 0, Skipped: 0
```

Now we are going to do something a little bit more advanced. We are going to join a KStream with a KTable. The Streams API
has an inner join, left join, and an outer join. [KStream-KTable joins](https://docs.confluent.io/platform/current/streams/developer-guide/dsl-api.html#kstream-ktable-join) are [non-windowed](https://docs.confluent.io/platform/current/streams/concepts.html#windowing) and asymmetric.
By asymmetric we mean that a join only gets triggered if the left input KStream gets a new record while the right KTable holds the latest input records materialized.

- Add the following new attributes:

```java
    private static String joinedTopicName = "joinedTopic";
    private static TestOutputTopic<String, String> joinedTopic;
    private static String joinedStoreName = "joinedStore";
```

- Replace the `buildTopology()` function for the following new one:

```java
public static void buildTopology() {
        final StreamsBuilder builder = new StreamsBuilder();
        KeyValueBytesStoreSupplier storeSupplier = Stores.persistentKeyValueStore(storeName);
        KeyValueBytesStoreSupplier tradingStoreSupplier = Stores.persistentKeyValueStore(tradingStoreName);
        KeyValueBytesStoreSupplier joinedStoreSupplier = Stores.persistentKeyValueStore(joinedStoreName);

        KStream<String, FinancialMessage> transactionStream =
            builder.stream(
                inTopicName,
                Consumed.with(Serdes.String(), financialMessageSerde)
            );

        KTable<String, String> stockTradingStore = builder.table(tradingTable,
            Consumed.with(Serdes.String(), Serdes.String()),
            Materialized.as(tradingStoreSupplier));

        KTable<String, String> joinedMessageStore = builder.table(joinedTopicName,
            Consumed.with(Serdes.String(), Serdes.String()),
            Materialized.as(joinedStoreSupplier));

        KStream<String, String> joinedStream = transactionStream.join(
            stockTradingStore,
            (financialMessage, companyName) -> "userId = " + financialMessage.userId + " companyName = " + companyName);

        joinedStream.to(
            joinedTopicName,
            Produced.with(Serdes.String(), Serdes.String()));

        testDriver = new TopologyTestDriver(builder.build(), getStreamsConfig());
        inTopic = testDriver.createInputTopic(inTopicName, new StringSerializer(), new JsonbSerializer<FinancialMessage>());
        tradingTableTopic = testDriver.createInputTopic(tradingTable, new StringSerializer(), new StringSerializer());
        joinedTopic = testDriver.createOutputTopic(joinedTopicName, new StringDeserializer(), new StringDeserializer());
    }
```

We can see that our `buildTopology()` function still contains the `transactionsStream` KStream that will contain the stream of `FinancialMessages` being received through the `inTopicName`. Then, We can see a KTable called `stockTradingStore`, which will get materialized as a State Store, that will contain the messages comming in through the input topic called `tradingTable`. This KTable will hold data about the tradding companies that will serve to enhance the incoming `FinancialMessages` with. The join between the KStream and the KTable is being done below and is called `joinedStream`. The result of this join is being outputed into a topic called `joinedTopicName`. Finally, this output topic is being store in a KTable called `joinedMessageStore`, and materialized in its respective State Store, in order to be able to query it later on in our tests. The inner join is performed on matching keys between the KStream and the KTable and the matched records
produce a new `<String, String>` pair with the value of `userId` and `companyName`.

In order to test the above new functionality of the `buildTopology()` function, remove or comment out the existing test and create the following new one:

```java
    @Test
    public void checkStreamAndTableJoinHasOneRecord() {

        tradingTableTopic.pipeInput("1", "Metropolitan Museum of Art");

        FinancialMessage mock = new FinancialMessage(
            "1", "MET", "SWISS", 12, 1822.38, 21868.55, 94, 7, true
        );
        inTopic.pipeInput("1", mock);

        KeyValueStore<String,ValueAndTimestamp<String>> joinedTableStore = testDriver.getTimestampedKeyValueStore(joinedStoreName);
        Assertions.assertEquals(1, joinedTableStore.approximateNumEntries());
        System.out.println(joinedTableStore.get("1").value());
    }
```

- Test the application by running the following:

```shell
./mvnw clean verify
```

- You should see the following output:

```shell
[INFO] -------------------------------------------------------
[INFO]  T E S T S
[INFO] -------------------------------------------------------
[INFO] Running com.ibm.GreetingResourceTest
2021-01-16 21:53:03,682 INFO  [io.quarkus] (main) Quarkus 1.10.5.Final on JVM started in 2.134s. Listening on: http://localhost:8081
2021-01-16 21:53:03,684 INFO  [io.quarkus] (main) Profile test activated. 
2021-01-16 21:53:03,685 INFO  [io.quarkus] (main) Installed features: [cdi, kafka-streams, resteasy, resteasy-jsonb]
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 5.574 s - in com.ibm.GreetingResourceTest
[INFO] Running com.ibm.garage.cpat.lab.TestFinancialMessage
userId = 1 companyName = Metropolitan Museum of Art
2021-01-16 21:53:05,432 INFO  [org.apa.kaf.str.pro.int.StateDirectory] (main) stream-thread [main] Deleting state directory 0_0 for task 0_0 as user calling cleanup.
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.294 s - in com.ibm.garage.cpat.lab.TestFinancialMessage
[INFO] Running com.ibm.garage.cpat.lab.TestLoadKtableFromTopic
C01:Health Care
C02:Finance
C03:Consumer Services
C04:Transportation
C05:Capital Goods
C06:Public Utilities
sector-types-store
2021-01-16 21:53:05,482 WARN  [org.apa.kaf.str.sta.int.RocksDBStore] (main) Closing 1 open iterators for store sector-types-store
2021-01-16 21:53:05,484 INFO  [org.apa.kaf.str.pro.int.StateDirectory] (main) stream-thread [main] Deleting state directory 0_0 for task 0_0 as user calling cleanup.
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.044 s - in com.ibm.garage.cpat.lab.TestLoadKtableFromTopic
2021-01-16 21:53:05,518 INFO  [io.quarkus] (main) Quarkus stopped in 0.026s
[INFO] 
[INFO] Results:
[INFO] 
[INFO] Tests run: 3, Failures: 0, Errors: 0, Skipped: 0
```

We can see that our test passes successfully as the `FinancialMessage` with `userId=1` and the KTable record with key `1` successfully joined to produce the output message: `userId = 1 companyName = Metropolitan Museum of Art`

We are now going to create yet another new test class. But first, we are going to create two new POJO classes to work with. These are `EnrichedMessage.java` and `AggregatedMessage.java`. Both of them should be placed where the previos POJO class is: `src/main/java/com/ibm/garage/cpat/domain`.

- Create the `EnrichedMessage.java` Java file with the following code:

```java
package com.ibm.garage.cpat.domain;


public class EnrichedMessage {

    public String userId;
    public String stockSymbol;
    public int quantity;
    public double stockPrice;
    public double totalCost;
    public double adjustedCost;
    public boolean technicalValidation;
    public String companyName;

    public EnrichedMessage (FinancialMessage message, String companyName) {
        this.userId = message.userId;
        this.stockSymbol = message.stockSymbol;
        this.quantity = message.quantity;
        this.stockPrice = message.stockPrice;
        this.totalCost = message.totalCost;
        this.companyName = companyName;

        if (message.technicalValidation)
        {
            this.technicalValidation = message.technicalValidation;
            this.adjustedCost = message.totalCost * 1.15;
        }

        else {
            this.technicalValidation = message.technicalValidation;
            this.adjustedCost = message.totalCost;
        }
    }
}
```

- Create the `AggregatedMessage.java` Java file with the following code:

```java
package com.ibm.garage.cpat.domain;

import java.math.BigDecimal;
import java.math.RoundingMode;


public class AggregatedMessage {

    public String userId;
    public String stockSymbol;
    public int quantity;
    public double stockPrice;
    public double totalCost;
    public double adjustedCost;
    public boolean technicalValidation;
    public String companyName;
    public int count;
    public double sum;
    public double average;

    public AggregatedMessage updateFrom(EnrichedMessage message) {
        this.userId = message.userId;
        this.stockSymbol = message.stockSymbol;
        this.quantity = message.quantity;
        this.stockPrice = message.stockPrice;
        this.totalCost = message.totalCost;
        this.companyName = message.companyName;
        this.adjustedCost = message.adjustedCost;
        this.technicalValidation = message.technicalValidation;

        this.count ++;
        this.sum += message.adjustedCost;
        this.average = BigDecimal.valueOf(sum / count)
                    .setScale(1, RoundingMode.HALF_UP).doubleValue();

        return this;
    }
}
```

- Now create the new test class named `TestAggregate.java` in the same path we have the other test classes (`src/test/java/com/ibm/garage/cpat`) and paste the following code:

```java
package com.ibm.garage.cpat.lab;

import java.util.Properties;

import org.apache.kafka.common.serialization.LongDeserializer;
import org.apache.kafka.common.serialization.Serde;
import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.apache.kafka.common.serialization.StringSerializer;
import org.apache.kafka.streams.KeyValue;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.StreamsConfig;
import org.apache.kafka.streams.TestInputTopic;
import org.apache.kafka.streams.TestOutputTopic;
import org.apache.kafka.streams.TopologyTestDriver;
import org.apache.kafka.streams.kstream.Consumed;
import org.apache.kafka.streams.kstream.KGroupedStream;
import org.apache.kafka.streams.kstream.KStream;
import org.apache.kafka.streams.kstream.KTable;
import org.apache.kafka.streams.kstream.Materialized;
import org.apache.kafka.streams.kstream.Produced;
import org.apache.kafka.streams.kstream.Windowed;
import org.apache.kafka.streams.kstream.WindowedSerdes;
import org.apache.kafka.streams.processor.StateStore;
import org.apache.kafka.streams.state.KeyValueBytesStoreSupplier;
import org.apache.kafka.streams.state.KeyValueIterator;
import org.apache.kafka.streams.state.KeyValueStore;
import org.apache.kafka.streams.state.Stores;
import org.apache.kafka.streams.state.ValueAndTimestamp;
import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.Assertions;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;

import io.quarkus.kafka.client.serialization.JsonbDeserializer;
import io.quarkus.kafka.client.serialization.JsonbSerde;
import io.quarkus.kafka.client.serialization.JsonbSerializer;
import io.quarkus.test.junit.QuarkusTest;

import com.ibm.garage.cpat.domain.*;


@QuarkusTest
public class TestAggregate {

    private static TopologyTestDriver testDriver;
    private static String inTopicName = "financialMessages";
    private static String outTopicName = "enrichedMessages";
    private static String storeName = "financialStore";
    private static String aggregatedTopicName = "aggregatedMessages";

    private static String companyTable = "companyTable";
    private static String companyStoreName = "companyStore";

    private static TestInputTopic<String, FinancialMessage> inTopic;
    private static TestOutputTopic<String, EnrichedMessage> outTopic;
    private static TestOutputTopic<String, AggregatedMessage> aggregatedTopic;
    private static TestInputTopic<String, String> companyTableTopic;

    private static final JsonbSerde<FinancialMessage> financialMessageSerde = new JsonbSerde<>(FinancialMessage.class);
    private static final JsonbSerde<EnrichedMessage> enrichedMessageSerde = new JsonbSerde<>(EnrichedMessage.class);
    private static final JsonbSerde<AggregatedMessage> aggregatedMessageSerde = new JsonbSerde<>(AggregatedMessage.class);


    public static Properties getStreamsConfig() {
        final Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "kstream-lab3");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "dummmy:3456");
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        //props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, financialMessageSerde);
        return props;
    }

    @BeforeAll
    public static void buildTopology() {
        final StreamsBuilder builder = new StreamsBuilder();
        KeyValueBytesStoreSupplier storeSupplier = Stores.persistentKeyValueStore(storeName);
        KeyValueBytesStoreSupplier companyStoreSupplier = Stores.persistentKeyValueStore(companyStoreName);

        // create a KStream for financial messages.
        KStream<String, FinancialMessage> financialStream =
            builder.stream(
                inTopicName,
                Consumed.with(Serdes.String(), financialMessageSerde)
            );

        // create a KTable from a topic for companies.
        KTable<String, String> companyStore = builder.table(companyTable,
            Consumed.with(Serdes.String(), Serdes.String()),
            Materialized.as(companyStoreSupplier));

        // join KStream with KTable and use aggregate.
        KStream<String, EnrichedMessage> enrichedStream = financialStream.join(
                companyStore,
                //(financialMessage, companyName) -> financialMessage.userId,
                (financialMessage, companyName) -> {
                    return new EnrichedMessage(financialMessage, companyName);
                }
            );

        enrichedStream.groupByKey()
            .aggregate(
                AggregatedMessage::new,
                (userId, value, aggregatedMessage) -> aggregatedMessage.updateFrom(value),
                Materialized.<String, AggregatedMessage> as(storeSupplier)
                                .withKeySerde(Serdes.String())
                                .withValueSerde(aggregatedMessageSerde)
            )
            .toStream()
            .to(
                aggregatedTopicName,
                Produced.with(Serdes.String(), aggregatedMessageSerde)
            );

        testDriver = new TopologyTestDriver(builder.build(), getStreamsConfig());
        inTopic = testDriver.createInputTopic(inTopicName, new StringSerializer(), new JsonbSerializer<FinancialMessage>());
        //outTopic = testDriver.createOutputTopic(outTopicName, new StringDeserializer(), new JsonbDeserializer<>(EnrichedMessage.class));
        companyTableTopic = testDriver.createInputTopic(companyTable, new StringSerializer(), new StringSerializer());
        aggregatedTopic = testDriver.createOutputTopic(aggregatedTopicName, new StringDeserializer(), new JsonbDeserializer<>(AggregatedMessage.class));
    }

    @AfterAll
    public static void close(){
        testDriver.close();
    }
}

```

Even though the code above might seem a completely new one at first glance, it is just an extra step of the previous code we have been working with. We can see that we still have both our `financialStream` KStream receiving `FinancialMessage` from the `inTopicName` topic and our `companyStore` KTable holding the latest being received from the `companyTable` input topic. Then, we also still have the join of the previous two KStream and KTable in our `enrichedStream` KStream that will be a KStream of `EnrichedMessage`. What is new in this code is the following aggregation we are doing on the resulting `enrichedStream` KStream. We are grouping `EnrichedMessage` objects by key and doing an aggregation of that grouping. The aggregation result will be an `AggregatedMessage` which will get materialized as `storeSupplier` so that we can query it later on. We are also converting that KTable to a KStream that will get outputed to `aggregatedTopicName`. The aggregate logic will simply work out some average and count of `EnrichedMessage` you can check out in the `EnrichedMessage.java` Java file.


- Now add a test that will make sure the functionality we have implemented in the `buildTopology()` function works as desired:

```java
    @Test
    public void aggregatedMessageExists() {

        companyTableTopic.pipeInput("1", "Metropolitan Museum of Art");

        FinancialMessage mock = new FinancialMessage(
            "1", "MET", "SWISS", 12, 1822.38, 21868.55, 94, 7, true
        );
        FinancialMessage mock2 = new FinancialMessage(
            "1", "MET", "SWISS", 12, 1822.38, 6634.56, 94, 7, true
        );
        inTopic.pipeInput("1", mock);
        inTopic.pipeInput("1", mock2);

        KeyValueStore<String,ValueAndTimestamp<AggregatedMessage>> aggregatedTableStore = testDriver.getTimestampedKeyValueStore(storeName);
        Assertions.assertEquals(2, aggregatedTableStore.approximateNumEntries());
        System.out.println("Average = " + aggregatedTableStore.get("1").value().average);
        Assertions.assertEquals(16389.3, aggregatedTableStore.get("2").value().average);
    }
```

- Test the application by running the following:

```shell
./mvnw clean verify
```

- You should see the following output:

```shell
[INFO] -------------------------------------------------------
[INFO]  T E S T S
[INFO] -------------------------------------------------------
[INFO] Running com.ibm.GreetingResourceTest
2021-01-17 13:13:02,533 INFO  [io.quarkus] (main) Quarkus 1.10.5.Final on JVM started in 2.178s. Listening on: http://localhost:8081
2021-01-17 13:13:02,535 INFO  [io.quarkus] (main) Profile test activated. 
2021-01-17 13:13:02,535 INFO  [io.quarkus] (main) Installed features: [cdi, kafka-streams, resteasy, resteasy-jsonb]
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 5.441 s - in com.ibm.GreetingResourceTest
[INFO] Running com.ibm.garage.cpat.lab.TestAggregate
Average = 16389.3
2021-01-17 13:13:04,216 INFO  [org.apa.kaf.str.pro.int.StateDirectory] (main) stream-thread [main] Deleting state directory 0_0 for task 0_0 as user calling cleanup.
[ERROR] Tests run: 1, Failures: 0, Errors: 1, Skipped: 0, Time elapsed: 0.339 s <<< FAILURE! - in com.ibm.garage.cpat.lab.TestAggregate
[ERROR] aggregatedMessageExists  Time elapsed: 0.122 s  <<< ERROR!
java.lang.NullPointerException
	at com.ibm.garage.cpat.lab.TestAggregate.aggregatedMessageExists(TestAggregate.java:145)

[INFO] Running com.ibm.garage.cpat.lab.TestFinancialMessage
userId = 1 companyName = Metropolitan Museum of Art
2021-01-17 13:13:04,283 INFO  [org.apa.kaf.str.pro.int.StateDirectory] (main) stream-thread [main] Deleting state directory 0_0 for task 0_0 as user calling cleanup.
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.055 s - in com.ibm.garage.cpat.lab.TestFinancialMessage
[INFO] Running com.ibm.garage.cpat.lab.TestLoadKtableFromTopic
C01:Health Care
C02:Finance
C03:Consumer Services
C04:Transportation
C05:Capital Goods
C06:Public Utilities
sector-types-store
2021-01-17 13:13:04,333 WARN  [org.apa.kaf.str.sta.int.RocksDBStore] (main) Closing 1 open iterators for store sector-types-store
2021-01-17 13:13:04,335 INFO  [org.apa.kaf.str.pro.int.StateDirectory] (main) stream-thread [main] Deleting state directory 0_0 for task 0_0 as user calling cleanup.
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.043 s - in com.ibm.garage.cpat.lab.TestLoadKtableFromTopic
2021-01-17 13:13:04,372 INFO  [io.quarkus] (main) Quarkus stopped in 0.029s
[INFO] 
[INFO] Results:
[INFO] 
[ERROR] Errors: 
[ERROR]   TestAggregate.aggregatedMessageExists:145 NullPointer
[INFO] 
[ERROR] Tests run: 4, Failures: 0, Errors: 1, Skipped: 0
```

We see the test fails, but that is expected. In this test we have two different `financialMessage` inserted with a key of `"1"` and there is only one entry
in the KTable `("1", "Metropolitan Museum of Art")`. Those two `financialMessage` will get enriched with the only record in the KTable. Later on, those two `EnrichedMessage` should get grouped by as they have the same key and result in an `AggregatedMessage`. There are two assertions in this test. The first one passes as the `aggregatedTableStore` contains two entries: The first `EnrichedMessage` that eas aggregated with a new empty `AggregatedMessage` as the initializer and then the second `EnrichedMessage` that it was aggregated with the resulting `AggregatedMessage` of the previous `EnrichedMessage`. However, the second second assertion fails. The reason for this is that even though there are two records in the `aggregatedTableStore`, this store is key based and will return, as a result, the latest `AggregatedMessage` it holds for a particular key. Then, if we want to retrieve the latest `AggregatedMessage` for out key `1` we need to change the assetion:

```java
Assertions.assertEquals(16389.3, aggregatedTableStore.get("2").value().average);
```

to use the appropriate key:

```java
Assertions.assertEquals(16389.3, aggregatedTableStore.get("1").value().average);
```

- Test the application by running the following:

```shell
./mvnw clean verify
```

- You should see the following output:

```shell
[INFO] -------------------------------------------------------
[INFO]  T E S T S
[INFO] -------------------------------------------------------
[INFO] Running com.ibm.GreetingResourceTest
2021-01-17 13:14:40,370 INFO  [io.quarkus] (main) Quarkus 1.10.5.Final on JVM started in 2.025s. Listening on: http://localhost:8081
2021-01-17 13:14:40,371 INFO  [io.quarkus] (main) Profile test activated. 
2021-01-17 13:14:40,372 INFO  [io.quarkus] (main) Installed features: [cdi, kafka-streams, resteasy, resteasy-jsonb]
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 5.495 s - in com.ibm.GreetingResourceTest
[INFO] Running com.ibm.garage.cpat.lab.TestAggregate
Average = 16389.3
2021-01-17 13:14:42,114 INFO  [org.apa.kaf.str.pro.int.StateDirectory] (main) stream-thread [main] Deleting state directory 0_0 for task 0_0 as user calling cleanup.
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.33 s - in com.ibm.garage.cpat.lab.TestAggregate
[INFO] Running com.ibm.garage.cpat.lab.TestFinancialMessage
userId = 1 companyName = Metropolitan Museum of Art
2021-01-17 13:14:42,177 INFO  [org.apa.kaf.str.pro.int.StateDirectory] (main) stream-thread [main] Deleting state directory 0_0 for task 0_0 as user calling cleanup.
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.051 s - in com.ibm.garage.cpat.lab.TestFinancialMessage
[INFO] Running com.ibm.garage.cpat.lab.TestLoadKtableFromTopic
C01:Health Care
C02:Finance
C03:Consumer Services
C04:Transportation
C05:Capital Goods
C06:Public Utilities
sector-types-store
2021-01-17 13:14:42,227 WARN  [org.apa.kaf.str.sta.int.RocksDBStore] (main) Closing 1 open iterators for store sector-types-store
2021-01-17 13:14:42,229 INFO  [org.apa.kaf.str.pro.int.StateDirectory] (main) stream-thread [main] Deleting state directory 0_0 for task 0_0 as user calling cleanup.
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.042 s - in com.ibm.garage.cpat.lab.TestLoadKtableFromTopic
2021-01-17 13:14:42,272 INFO  [io.quarkus] (main) Quarkus stopped in 0.035s
[INFO] 
[INFO] Results:
[INFO] 
[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0
```

## Producing to and Consuming from a Kafka Topic on Event Streams

When using the Kafka Streams API, we usually do it against a Kafka instance containing data already in, at least, one of its topics. In order to build up that scenario, we are going to use the MicroProfile Reactive Messaging library to send messages to a topic.

- For using the MicroProfile Reactive Messaging library, we first need to add such dependency to our `pom.xml` file:

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-smallrye-reactive-messaging-kafka</artifactId>
</dependency>
```

- Create the `MockProducer.java` Java file in `src/main/java/com/ibm/garage/cpat/infrastructure` with the following code:

```java
package com.ibm.garage.cpat.infrastructure;

import javax.enterprise.context.ApplicationScoped;

import org.eclipse.microprofile.reactive.messaging.Outgoing;

import io.reactivex.Flowable;
import io.smallrye.reactive.messaging.kafka.KafkaRecord;

import java.util.concurrent.TimeUnit;
import java.util.Random;

import com.ibm.garage.cpat.domain.*;


@ApplicationScoped
public class MockProducer {

    private Random random = new Random();

    FinancialMessage mock = new FinancialMessage(
        "1", "MET", "SWISS", 12, 1822.38, 21868.55, 94, 7, true
        );

    @Outgoing("mock-messages")
    public Flowable<KafkaRecord<String,FinancialMessage>> produceMock() {
        return Flowable.interval(5, TimeUnit.SECONDS)
                       .map(tick -> {
                            return setRandomUserId(mock);
                        });
    }

    public KafkaRecord<String, FinancialMessage> setRandomUserId(FinancialMessage mock) {
        mock.userId = String.valueOf(random.nextInt(100));

        return KafkaRecord.of(mock.userId, mock);
    }
}
```

The producer code above produces a `FinancialMessage` every 5 seconds to the `mock-messages` channel
with a random userId (out of 100). We will see later on how the `mock-messages` channel relates to a Kafka topic through configuration.

- Next, create the topology that we are going to build for processing the messages sent by our previous producer. Create a `FinancialMessageTopology.java` Java file in `src/main/java/com/ibm/garage/cpat/domain` with the following code:

```java
package com.ibm.garage.cpat.domain;

import javax.enterprise.context.ApplicationScoped;
import javax.enterprise.inject.Produces;

import org.eclipse.microprofile.config.inject.ConfigProperty;

import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.Topology;
import org.apache.kafka.streams.kstream.Consumed;
import org.apache.kafka.streams.kstream.Produced;

import io.quarkus.kafka.client.serialization.JsonbSerde;


@ApplicationScoped
public class FinancialMessageTopology {

    @ConfigProperty(name = "START_TOPIC_NAME")
    private String INCOMING_TOPIC;

    @ConfigProperty(name = "TARGET_TOPIC_NAME")
    private String OUTGOING_TOPIC;


    @Produces
    public Topology buildTopology() {

        StreamsBuilder builder = new StreamsBuilder();

        JsonbSerde<FinancialMessage> financialMessageSerde = new JsonbSerde<>(FinancialMessage.class);

        // Stream reads from input topic, filters it by checking the boolean field on the message.
        // If the boolean is true, it gets passed to the mapValues function which will then send that record
        // to an outgoing topic.

        builder.stream(
            INCOMING_TOPIC,
            Consumed.with(Serdes.String(), financialMessageSerde)
        )
        .filter (
            (key, message) -> checkValidation(message)
        )
        .mapValues (
            checkedMessage -> adjustPostValidation(checkedMessage)
        )
        .to (
            OUTGOING_TOPIC,
            Produced.with(Serdes.String(), financialMessageSerde)
        );

        return builder.build();
    }

    public boolean checkValidation(FinancialMessage message) {
        return (message.technicalValidation);
    }

    public FinancialMessage adjustPostValidation(FinancialMessage message) {
        message.totalCost = message.totalCost * 1.15;

        return message;
    }

}
```

The code above builds a KStream from the messages in `INCOMING_TOPIC`. It will filter the `FinanacialMessage` based on a boolean property of them. Then, it will do some adjustment of the cost attribute within that `FinancialMessage` post message validation and send that out to the `OUTGOING_TOPIC`.

Now, we have seen the code for both the mock producer and the topology we want to build are dependant on certain configuration variables such as the input and output topics they are meant to produce to and consume from. In
a Quarkus application, all the configuration settings is done through a properties file, which makes the application portable across different environments. This property file, which we already had to configure in previous labs is called `application.properties` and is located in `src/main/resources`. 

- In order to configure the application to work with an IBM Event Streams v10 or later, paste the following configuration in the `application.properties` file:

```properties
quarkus.http.port=8080
quarkus.log.console.enable=true
quarkus.log.console.level=INFO

# Base ES Connection Details
mp.messaging.connector.smallrye-kafka.bootstrap.servers=${BOOTSTRAP_SERVERS}
mp.messaging.connector.smallrye-kafka.security.protocol=SASL_SSL
mp.messaging.connector.smallrye-kafka.ssl.protocol=TLSv1.2
mp.messaging.connector.smallrye-kafka.sasl.mechanism=SCRAM-SHA-512
mp.messaging.connector.smallrye-kafka.sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
                username=${SCRAM_USERNAME} \
                password=${SCRAM_PASSWORD};
mp.messaging.connector.smallrye-kafka.ssl.truststore.location=${CERT_LOCATION}
mp.messaging.connector.smallrye-kafka.ssl.truststore.password=${CERT_PASSWORD}
mp.messaging.connector.smallrye-kafka.ssl.truststore.type=PKCS12


# Initial mock JSON message producer configuration
mp.messaging.outgoing.mock-messages.connector=smallrye-kafka
mp.messaging.outgoing.mock-messages.topic=${START_TOPIC_NAME}
mp.messaging.outgoing.mock-messages.value.serializer=io.quarkus.kafka.client.serialization.JsonbSerializer



# Quarkus Kafka Streams configuration settings
quarkus.kafka-streams.bootstrap-servers=${BOOTSTRAP_SERVERS}
quarkus.kafka-streams.application-id=financial-stream
quarkus.kafka-streams.application-server=localhost:8080
quarkus.kafka-streams.topics=${START_TOPIC_NAME},${TARGET_TOPIC_NAME}
quarkus.kafka-streams.health.enabled=true

quarkus.kafka-streams.security.protocol=SASL_SSL
quarkus.kafka-streams.ssl.protocol=TLSv1.2
quarkus.kafka-streams.sasl.mechanism=SCRAM-SHA-512
quarkus.kafka-streams.sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
                username=${SCRAM_USERNAME} \
                password=${SCRAM_PASSWORD};
quarkus.kafka-streams.ssl.truststore.location=${CERT_LOCATION}
quarkus.kafka-streams.ssl.truststore.password=${CERT_PASSWORD}
quarkus.kafka-streams.ssl.truststore.type=PKCS12

# pass-through options
kafka-streams.cache.max.bytes.buffering=10240
kafka-streams.commit.interval.ms=1000
kafka-streams.metadata.max.age.ms=500
kafka-streams.auto.offset.reset=latest
kafka-streams.metrics.recording.level=DEBUG
```

- If using a previous IBM Event Streams version (such as v2019.4.2) or on IBM Cloud, use the following configuration in the `application.properties` file:

```properties
quarkus.http.port=8080
quarkus.log.console.enable=true
quarkus.log.console.level=INFO

# Base ES Connection Details
mp.messaging.connector.smallrye-kafka.bootstrap.servers=${BOOTSTRAP_SERVERS}
mp.messaging.connector.smallrye-kafka.security.protocol=SASL_SSL
mp.messaging.connector.smallrye-kafka.ssl.protocol=TLSv1.2
mp.messaging.connector.smallrye-kafka.sasl.mechanism=PLAIN
mp.messaging.connector.smallrye-kafka.sasl.jaas.config=org.apache.kafka.common.security.scram.PlainLoginModule required \
                username="token" \
                password=${API_KEY};
# If connecting to Event Streams on IBM Cloud the following truststore options are not needed.
mp.messaging.connector.smallrye-kafka.ssl.truststore.location=${CERT_LOCATION}
mp.messaging.connector.smallrye-kafka.ssl.truststore.password=password


# Initial mock JSON message producer configuration
mp.messaging.outgoing.mock-messages.connector=smallrye-kafka
mp.messaging.outgoing.mock-messages.topic=${START_TOPIC_NAME}
mp.messaging.outgoing.mock-messages.value.serializer=io.quarkus.kafka.client.serialization.JsonbSerializer



# Quarkus Kafka Streams configuration settings
quarkus.kafka-streams.bootstrap-servers=${BOOTSTRAP_SERVERS}
quarkus.kafka-streams.application-id=financial-stream
quarkus.kafka-streams.application-server=localhost:8080
quarkus.kafka-streams.topics=${START_TOPIC_NAME},${TARGET_TOPIC_NAME}
quarkus.kafka-streams.health.enabled=true

quarkus.kafka-streams.security.protocol=SASL_SSL
quarkus.kafka-streams.ssl.protocol=TLSv1.2
quarkus.kafka-streams.sasl.mechanism=PLAIN
quarkus.kafka-streams.sasl.jaas.config=org.apache.kafka.common.security.scram.PlainLoginModule required \
                username="token" \
                password=${API_KEY};
# If connecting to Event Streams on IBM Cloud the following truststore options are not needed.
quarkus.kafka-streams.ssl.truststore.location=${CERT_LOCATION}
quarkus.kafka-streams.ssl.truststore.password=password

# pass-through options
kafka-streams.cache.max.bytes.buffering=10240
kafka-streams.commit.interval.ms=1000
kafka-streams.metadata.max.age.ms=500
kafka-streams.auto.offset.reset=latest
kafka-streams.metrics.recording.level=DEBUG
```

There are some environment variables our `application.properties` file depends on:

- `START_TOPIC_NAME`: The Kafka topic the mock producer will produce `FinancialMessage` to and the topology consume messages from for processing. **IMPORTANT:** Use a topic name with some unique identifier if you are sharing the IBM Event Streams instance with other lab students. Also, you **must** create this topic in IBM Event Streams. See [here](/use-cases/overview/pre-requisites) for more details as to how to create a topic.
- `TARGET_TOPIC_NAME`: The Kafka topic the topology will produce the processed `FinancialMessage` to. **IMPORTANT:** Use a topic name with some unique identifier if you are sharing the IBM Event Streams instance with other lab students. Also, you **must** create this topic in IBM Event Streams. See [here]((/use-cases/overview/pre-requisites) for more details as to how to create a topic
- `BOOTSTRAP_SERVERS`: Your IBM Event Streams bootstrap server. See [here](/use-cases/overview/pre-requisites) for more details as to how to obtain these.
- `CERT_LOCATION`: The location where the PKCS12 certificate for the SSL connection to the IBM Event Streams instance is. See [here](/use-cases/overview/pre-requisites) for more details as to how to obtain these.
- `CERT_PASSWORD`: The password of the PKCS12 certificate. See [here](/use-cases/overview/pre-requisites) for more details as to how to obtain these.
- `API_KEY` if you are using an IBM Event Streams instance in IBM Cloud or
- `SCRAM_USERNAME` and `SCRAM_PASSWORD`: The SCRAM credentials for your application to get authenticated and authorized to work with IBM Event Streams. See [here](/use-cases/overview/pre-requisites) for more details as to how to obtain these.

Export the variables and values on the terminal you will run the application from:

- IBM Event Strams v10 or later:

```shell
export BOOTSTRAP_SERVERS=your-bootstrap-server-address:443 \
export START_TOPIC_NAME=name-of-topic-to-consume-from \
export TARGET_TOPIC_NAME=name-of-topic-to-produce-to \
export CERT_LOCATION=/path-to-pkcs12-cert/es-cert.p12 \
export CERT_PASSWORD=certificate-password \
export SCRAM_USERNAME=your-scram-username \
export SCRAM_PASSWORD=your-scram-password
```

- Previous IBM Event Streams versions:

```shell
export BOOTSTRAP_SERVERS=your-bootstrap-server-address:443 \
export START_TOPIC_NAME=name-of-topic-to-consume-from \
export TARGET_TOPIC_NAME=name-of-topic-to-produce-to \
export CERT_LOCATION=/path-to-jks-cert/es-cert.jks \
export API_KEY=your-api-key
```

- Local Kafka Cluster:

```shell
export BOOTSTRAP_SERVERS=your-bootstrap-server-address:443 \
export START_TOPIC_NAME=name-of-topic-to-consume-from \
export TARGET_TOPIC_NAME=name-of-topic-to-produce-to 
```

- You can now test the Quarkus application

```shell
./mvnw quarkus:dev
```

- You should now be able to see `FinancialMessage` events in the input topic and those processed messages in the output topic you have specified above in the IBM Event Streams user interface.