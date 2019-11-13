---
layout: post
title: Generating Test Data
---

In [A Test Helper for JDBC Sinks](/test-machine-test-jdbc-sink/) one
part of the testing process that I glossed over a bit was the line
"Generate some example records to load into the input topic". I said
this like it was no big deal but actually there are a few moving parts
that all need to come together for this to work and it's something I
struggled to get to grips with at the beginning of our journey and
have seen other experienced engineers struggle with too. Part of the
problem I think is that a lot of the Kafka eco-system is made up of
folks using statically typed languages like Scala, Kotlin etc. It does
all work with dynamically typed languages like Clojure but there are
just fewer of us around which makes it all the more important to share
what we learn. So here's a quick guide to generating
test-data and getting it into Kafka using the test-machine from Jackdaw

## Basic Data Generator

You may recall the fields enumerated in the whitelist from the example
sink config. They were as follows:-

 * customer-id
 * current-balance
 * updated-at

So a nice easy first step is to write a function to generate a map
with these fields

{% highlight clojure %}
(ns io.grumpybank.generators
  (:require
    [java-time :as t]))

(defn gen-customer-balance
  []
  {:customer-id (str (java.util.UUID/randomUUID))
   :current-balance (rand-int 1000)
   :updated-at (t/to-millis-from-epoch (t/instant))})
{% endhighlight %}

## Schema Definition

However this is not enough on it's own. The target database has a schema
which is only implicit in the function above. The JDBC sink connector
will create and evolve the schema for us if we allow it, but in
order to do that, we need to write the data using the Avro serialization
format. Here is Jay Kreps from Confluent [making the case for Avro](https://www.confluent.io/blog/avro-kafka-data/)
and much of the confluent tooling leverages various aspects of this particular
serialization format so it's a good default choice unless you have a good
reason to choose otherwise.

So let's assume the app that produces the customer-balances topic has
already defined a Avro schema. The thing we're trying to test is a
consumer of that topic but as a tester, we have to wear the producer
hat for for a while so we take a copy of the schema from the upstream
app and make it available to our connector test.

{% highlight JSON %}
{
  "type": "record",
  "name": "CustomerBalance",
  "namespace": "io.grumpybank.tables.CustomerBalance",
  "fields": [
    {
      "name": "customer_id",
      "type": "string"
    },
    {
      "name": "updated_at",
      "type": {
        "type": "long",
        "logicalType": "timestamp-millis"
      }
    },
    {
      "name": "current_balance",
      "type": ["null", "long"],
      "default": null
    }
  ]
}
{% endhighlight %}

We can use the schema above to create an Avro
[Serde](https://www.apache.org/dist/kafka/2.3.0/javadoc/org/apache/kafka/common/serialization/Serde.html).
Serde is just the name given to the composition of the Serialization
and Deserialization operations. Since one is the opposite of the other
it has become a strong convention that that they are defined together
and the Serde interface captures that convention.

The Serde will be used by the KafkaProducer to serialize the message
value into a ByteArray before sending it off to the broker to be
appended to the specified topic and replicated as per the topic
settings. Here's a helper function for creating the Serde for a schema
represented as JSON in a file using jackdaw.

{% highlight clojure %}
(ns io.grumpybank.avro-helpers
  (:require
    [jackdaw.serdes.avro :as avro]
    [jackdaw.serdes.avro.schema-registry :as reg]))
	
(def schema-registry-url "http://localhost:8081")
(def schema-registry-client (reg/client schema-registry-url 32))

(defn value-serde
  [filename]
  (avro/serde {:avro.schema-registry/client schema-registry-client
               :avro.schema-registry/url schema-registry-url}
              {:avro/schema (slurp filename)
               :key? false}))
{% endhighlight %}

The Avro Serdes in jackdaw ultimately use the KafkaAvroSerializer/KafkaAvroDeserializer
which share schemas via the Confluent Schema Registry and optionally
checks for various levels of compatability. The Schema Registry is yet
another topic worthy of it's own blog-post but fortunately Gwen
Shapira has already [written
it](https://www.confluent.io/blog/schema-registry-kafka-stream-processing-yes-virginia-you-really-need-one/).
The Jackdaw avro serdes convert clojure data structures like the one
output by `gen-customer-balance` into an [Avro
GenericRecord](https://avro.apache.org/docs/1.8.2/api/java/org/apache/avro/generic/GenericRecord.html)
I'll get into more gory detail about this some other time but for now,
let's move quickly along and discuss the concept of "Topic Metadata".

## Topic Metadata

In Jackdaw, the convention adopted for associating Serdes with
topics is known as "Topic Metadata". This is just a Clojure map so you
can put all kinds of information in there if it helps fulfill some
requirement. Here are a few bits of metadata that jackdaw will act upon

### When creating a topic
 * `:topic-name`
 * `:replication-factor`
 * `:partition-count`
 
### When serializing a message 
 * `:key-serde`
 * `:value-serde`
 * `:key-fn`
 * `:partition-fn`

{% highlight clojure %}
(ns io.grumpybank.connectors.test-helpers
  (:require
    [jackdaw.serdes :as serde]
    [io.grumpybank.avro-helpers :as avro]))

(defn topic-config
  [topic-name]
  {:topic-name topic-name
   :replication-factor 1
   :key-serde (serde/string-serde)
   :value-serde (avro/value-serde (str "./test/resources/schemas/"
                                       topic-name
									   ".json"))})

{% endhighlight %}

## Revisit the helper

Armed with all this new information, we can revisit the helper defined
in the previous post and understand a bit more clearly what's going on
and how it all ties together. For illustrative purposes, we've
explicitly defined a few variables that were a bit obscured in the
original example.

{% highlight clojure %}

(def kconfig {"bootstrap.servers" "localhost:9092"})
(def topics {:customer-balances (topic-config "customer-balances")})
(def seed-data (repeatedly 5 gen-customer-balance))
(def topic-id :customer-balances)
(def key-fn :id)

(fix/with-fixtures [(fix/topic-fixture kconfig topics)]
  (jdt/with-test-machine (jdt/kafka-transport kconfig topics)
    (fn [machine]
      (jdt/run-test machine (concat
                              (->> seed-data
                                   (map (fn [record]
                                          [:write! topic-id record {:key-fn key-fn}])))
                              [[:watch watch-fn {:timeout 5000}]])))))
{% endhighlight %}

The vars `kconfig` and `topics` are used by both the `topic-fixture` (to create the
required topic before starting to write test-data to it), and the `kafka-transport`
which teaches the test-machine how read and write data from the listed topics. In
fact the test-machine will start reading data from all listed topics straight
away even before it is instructed to write anything.

Finally we write the test-data to kafka by supplying a list of commands to the
`run-test` function. The `:write!` command takes a topic-identifier (one of the
keys in the topics map), the message value, and a map of options in this case
specifying that the message key can be derived from the message by invoking
`(:id record)`. We could also specify things like the `:partition-fn`,
`:timestamp` etc. When the command is executed by the test-machine, it looks up
the topic-metadata for the specified identifier and uses it to build a ProducerRecord
and send it off to the broker.

Next up will be a deep-dive into the test-machine journal and the watch command.
