---
layout: post
title: A Test Helper for JDBC Sinks
---

The Confluent JDBC Sink allows you to configure Kafka Connect to take
care of moving data reliably from Kafka to a relational database. Most
of the usual suspects (e.g. PostgreSQL, MySQL, Oracle etc) are
supported out the box and in theory, you could connect your data to
any database with a JDBC driver.

This is great because Kafka Connect takes care of

 * Splitting the job between a [configurable number of Tasks](https://kafka.apache.org/documentation/#connect_connectorsandtasks)
 * Keeping track of tasks' progress using [Kafka Consumer Groups](https://kafka.apache.org/documentation/#intro_consumers)
 * Making the current status of workers available over an [HTTP API](https://kafka.apache.org/documentation/#connect_rest)
 * Publishing [metrics](https://kafka.apache.org/documentation/#connect_monitoring) that facilitate the monitoring of all connectors in
   a standard way
 
Assuming your infrastructure has an instance of Kafka Connect up and
running, all you need to do as a user of this system is submit a JSON
HTTP request to register a "job" and Kafka Connect will take care of
the rest.

To make things concrete, imagine we're implementing an event-driven
bank and we have some process (or at scale, a collection of processes)
that keeps track of customer balances by applying a transaction
log. Each time a customer balance is updated for some transaction, a
record is written to the customer-balances topic and we'd like to sink
this topic into a database table so that other systems can quickly
look up the current balance for some customer without having to apply
all the transactions themselves.

The configuration for such a sink might look something like this...

{% highlight JSON %}
{
  "name": "customer-balances-sink",
  "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
  "table.name.format": "customer_balances",
  "connection.url": "jdbc:postgresql://DB_HOST:DB_PORT/DB_NAME",
  "connection.user": "DB_USER",
  "connection.password": "DB_PASSWORD",
  "key.converter": "org.apache.kafka.connect.storage.StringConverter",
  "value.converter": "io.confluent.connect.avro.AvroConverter",
  "value.converter.schema.registry.url": "SCHEMA_REGISTRY_URL",
  "topics": "customer-balances",
  "auto.create": "true",
  "auto.evolve": "true",
  "pk.mode": "record_value",
  "pk.fields": "customer_id",
  "fields.whitelist": "customer_id,current_balance,updated_at",
  "insert.mode": "upsert",
}
{% endhighlight %}

It may be argued that since this is all just configuration, there is no
need for testing. Or if you try to test this, aren't you just testing
Kafka Connect itself? I probably would have agreed with this sentiment until
the 2nd or 3rd time I had to reset the UAT environment after deploying a
slightly incorrect kafka connect job.

It is difficult to get these things perfectly correct first time and
an error can be costly to fix even if they happen in a test
environment (especially if the test environment is shared by other
developers and needs to be fixed or reset before trying again). For
this reason, it's really nice to be able to quickly test it out in
your local environment and/or run some automated tests as part of your
continuous integration flow before any code gets merged.

So how *do* we test such a thing? Here's a list of some of the steps we
could take. We could go further but this seems to catch most of the
errors that I've seen go wrong in practice.

 * Create the "customer-balances" topic from which data will be fed
   into the the sink
 * Register the "customer-balance-sink" connector with a kafka-connect
   instance provided by the test environment (and wait until it gets
   into the "RUNNING" state)
 * Generate some example records to load into the input topic
 * Wait until the last of the generated records appears in the sink
   table
 * Check that all records written to the input topic made it into the
   sink table

## Top-down, meet Bottom-up

As an aside, and to provide a bit of background to my thought
processes, many years ago, I came across the web.py project by the
late Aaron Swartz. The philosophy for that framework was

> Think about the ideal way to write a web app. Write the code to make it happen.
>
> -- Aaron Swartz ([http://webpy.org/philosophy](http://webpy.org/philosophy))

This was one of many things he wrote that has stuck with me over the
years and it always comes to mind whenever I'm attempting to solve a new problem.
So when I thought about "the ideal way to write a test for a kafka connect sink",
something like the following came to mind. This is the Top-down part of the
development process.

{% highlight clojure %}

(deftest ^:connect test-customer-balances
  (test-jdbc-sink {:connector-name "customer-balances-sink"
                   :config (config/load-config)
                   :topic "customer-balances"
                   :spec ::customer-balances
                   :size 2
                   :poll-fn (help/poll-table :customer-balances :customer-id)
                   :watch-fn (help/found-last? :customer-balances :customer-id)}
    (comp
     (help/table-counts? {:customer-balances 2})
     (help/table-columns? {:customer-balances
                           #{:customer-id
                             :current-balance
                             :updated-at}}))))

{% endhighlight %}

The first parameter to this function is simply a map that provides
information to the test helper about things like

 * How to identify the connector so that it can be found and loaded into the test environment
 * Where to write the test data
 * How to generate the test data (and how much test data to generate)
 * How to find the data in the database after the connect job has loaded it
   into the database
 * How to decide when the all data has appeared in the sink

The second parameter is a function that will be invoked with all the
data that has been collected by the test-machine journal during the
test run (specifically the generated seed data, and the data retrieved
from the sink table by periodically polling the database with the
test-specific query defined by the `help/poll-table` helper).

For this, we use regular functional composition to build a single
assertion function from any number of single purpose assertion
functions like `help/table-counts?` and `help/table-columns?`. Each
assertion helper returns a function that receives the journal, runs
some assertions, and then returns the journal so that it may be
composed with other helpers. If any new testing requirements are
identified they can be easily added independently of the existing
assertion helpers.

With these basic testing primitives in mind we now need to "write the
code to make it happen". i.e. The Bottom-up part of the development
process. With a bit of luck, they will meet in the middle.

## Test Environment Additions

In addition to the base docker-compose config included in the
[previous post](https://grumpyhacker.com/test-machine-test-env/), we
need a couple of extra services. We can either put those in their own
file and combine the two compose files using the `-f` option of
docker-compose, or we can just bundle it all up into a single compose
file. Each option has it's trade-offs. I don't feel too strongly
either way. Use whichever option fits best with your team's workflow.
This will also depend on the particular database you use. We use PostgreSQL
here because it's awesome.

{% highlight yaml %}
version: '3'
services:
  connect:
    image: confluentinc/cp-kafka-connect:5.1.0
    expose:
      - "8083"
    ports:
      - "8083:8083"
    environment:
      KAFKA_HEAP_OPTS: "-Xms256m -Xmx512m"
      CONNECT_REST_ADVERTISED_HOST_NAME: connect
      CONNECT_GROUP_ID: jdbc-sink-test
      CONNECT_BOOTSTRAP_SERVERS: broker:9092
      CONNECT_CONFIG_STORAGE_TOPIC: docker-connect-configs
      CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR: 1
      CONNECT_OFFSET_FLUSH_INTERVAL_MS: 10000
      CONNECT_OFFSET_STORAGE_TOPIC: docker-connect-offsets
      CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR: 1
      CONNECT_STATUS_STORAGE_TOPIC: docker-connect-status
      CONNECT_STATUS_STORAGE_REPLICATION_FACTOR: 1
      CONNECT_KEY_CONVERTER: org.apache.kafka.connect.storage.StringConverter
      CONNECT_VALUE_CONVERTER: io.confluent.connect.avro.AvroConverter
      CONNECT_VALUE_CONVERTER_SCHEMA_REGISTRY_URL: http://schema-registry:8081
      CONNECT_INTERNAL_KEY_CONVERTER: "org.apache.kafka.connect.json.JsonConverter"
      CONNECT_INTERNAL_VALUE_CONVERTER: "org.apache.kafka.connect.json.JsonConverter"
      CONNECT_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      CONNECT_PLUGIN_PATH: '/usr/share/java'

  pg:
    image: postgres:9.5
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_PASSWORD=yolo
      - POSTGRES_DB=jdbc_sink_test
      - POSTGRES_USER=postgres
{% endhighlight %}

## Implementing the Test Helpers

The test helpers are a collection of higher-order functions that
allow the `test-jdbc-sink` function to pass control back to the test
author in order to run test-specific tasks. Let's look at those
before delving into `test-jdbc-sink` itself which is a bit more
involved. The helpers are all fairly straight-forward so hopefully
the docstrings will be enough to understand what's going on.

{% highlight clojure %}

(defn poll-table
  "Returns a function that will be periodically executed by the `test-connector`
   to fetch data from the sink table. The returned function is invoked with the
   generated seed-data as a parameter so that it can ignore any data added by
   different test runs."
  [table-name key-name]
  (fn [seed-data db]
    (let [result (let [query (format "select *
                                             from %s
                                            where %s in (%s)"
                                     (if (keyword? table-name)
                                       (underscore table-name)
                                       (format "\"%s\"" table-name))
                                     (if (keyword? key-name)
                                       (underscore key-name)
                                       (format "\"%s\"" key-name))
                                     (->> seed-data
                                          (map key-name)
                                          (map #(format "'%s'" %))
                                          (string/join ",")))]
                   (try
                     (jdbc/query db query {:identifiers hyphenate})
                     (catch Exception e
                       (log/error "failed: " query))))]
      (log/info (format "%s rows: %s" table-name (count result)))
      result)))

(defn found-last?
  "Builds a watch function that is invoked whenever the test-machine journal
   is updated (the journal is updated whenever the poll function successfully finds
   data). When the watch function returns `true`, that denotes the completion of
   the test and the current state of the journal is passed to the test assertion
   function"
  [table-name key-name]
  (fn [seed-data journal]
    (let [last-id (:id (last seed-data))]
      (->> (get-in journal [:tables table-name])
           (filter #(= last-id (:id %)))
           first
           not-empty))))

(defn table-counts?
  "Builds an assertion function that checks whether the journal contains
   the expected number of records in the specified table. `m` is a map
   of table-ids to expected counts. The returned function returns the
   journal so that it can be composed with other assertion functions"
  [m]
  (fn [journal]
    (doseq [[k exp-count] m]
      (testing (format "count %s" k)
        (is (= exp-count (-> (get-in journal [:tables k])
                             count)))))
    journal))

(defn table-columns?
  "Builds an assertion function that checks whether the sink tables logged in
   test-machine journal contain the expected columns"
  [m]
  (fn [journal]
    (doseq [[k field-set] m]
      (testing (format "table %s has columns %s"
                       k field-set)
        (is (= field-set
               (->> (get-in journal [:tables k])
                    last
                    keys
                    set)))))
    journal))
	
(defn- load-seed-data
  "This is where we actually use the test-machine. We use the seed-data to generate
   a list of :write! commands, and just tack on a :watch command at the end that uses
   the `watch-fn` provided by the test-author. When the watch function is satisfied,
   this will return the test-machine journal that has been collecting data produced
   by the poller which we can then use as part of our test assertions"
  [machine topic-id seed-data
   {:keys [key-fn watch-fn]
    :or {key-fn :id}}]
  (jdt/run-test machine (concat
                         (->> seed-data
                              (map (fn [record]
                                     [:write! topic-id record {:key-fn key-fn}])))
                         [[:watch watch-fn {:timeout 5000}]])))	
	
{% endhighlight %}

Finally, here is the annotated code for `test-jdbc-sink`. This has not yet
been properly extracted from the project which uses these tests so it
contains a bit of accidental complexity but hopefully I'll be able to get
some version of this into [jackdaw](https://github.com/FundingCircle/jackdaw)
soon. In the meantime I'm hoping it serves as a nice bit of
documentation for using the test-machine outside of contrived
examples.

{% highlight clojure %}
(defn test-jdbc-sink
  {:style/indent 1}
  [{:keys [connector-name config topic spec size watch-fn poll-fn key-fn]} test-fn]
  
  ;; `config` is a global config map loaded from an EDN file. We fetch the
  ;; configured schema-registry url and create a schema-registry-client and assign
  ;; them to dynamic variables which are used when "resolving" the avro serdes that
  ;; are to be associated with the input topic
  (binding [t/*schema-registry-url* (get-in config [:schema-registry :url])
            t/*schema-registry-client* (reg/client (get-in config [:schema-registry :url]) 100)]
            
    ;; You may have noticed in the JSON configuration above that there were placeholders for
    ;; database paramters (e.g. DB_USER, DB_NAME etc). These are expanded using a "mustache"
    ;; template language renderer. That's all `load-connector` is doing here
    (let [connector (load-connector config connector-name)
    
          ;; `spec` represents a clojure.spec "entity map"
          seed-data (gen/sample (s/gen spec) size)

          ;; `topic-config` takes the topic specified as a string, and finds the corresponding
          ;; topic-metadata in the project configuration. topic-metadata is where we specify things
          ;; like how to create a topic, how to serialize a record, how to generate a key from
          ;; a record value
          topics    (topic-config topic)

          ;; `topic-id` is just a symbolic id representing the topic
          topic-id (-> topics
                       keys
                       first)

          ;; here we fetch the name of the sink table from the connector config
          sink-table (-> (get connector "table.name.format")
                         hyphenate
                         keyword)
                         
          ;; the kafka-config tells us where the kafka bootstrap.servers are. This is required
          ;; to connect to kafka in order to create the test topic and write our example test
          ;; data
          kconfig (kafka-config config)]

      ;; This is just the standard way to acquire a jdbc connection in Clojure. We're getting
      ;; the connection parameters from the same global project config we got the schema-registry
      ;; parameters from
      (jdbc/with-db-connection [db {:dbtype "postgresql"
                                    :dbname (get-in config [:jdbc-sink-db :name])
                                    :host "localhost"
                                    :port (get-in config [:jdbc-sink-db :port])
                                    :user (get-in config [:jdbc-sink-db :username])
                                    :password (get-in config [:jdbc-sink-db :password])}]

        ;; `with-fixtures` is one of the few macros used. It takes a vector of fixtures each of
        ;; which is a function that performs some setup before invoking a test function. The
        ;; test function ends up being defined by the body of the macro. The fixtures here
        ;; create the test topic, wait for kafka-connect to be up and running (important when
        ;; the tests are running in CircleCI immediately after starting kafka-connect), then
        ;; load the connector, 
        (fix/with-fixtures [(fix/topic-fixture kconfig topics)
                            (fix/service-ready? {:http-url "http://localhost:8083"})
                            (tfx/connector-fixture {:base-url "http://localhost:8083"
                                                    :connector {"config" connector}})]

          ;; Finally we acquire a test-machine using the kafka-config and the topic-metadata we
          ;; derived earlier. This will be used to write the test data and record the results
          ;; of polling the target table
          (jdt/with-test-machine (jdt/kafka-transport kconfig topics)
            (fn [machine]
            
              ;; Before writing any test-data, we setup the db-poller. This uses Zach Tellman's
              ;; manifold to periodically invoke the supplied function on a fixed pool of threads.
              ;; The `poll-fn` is actually provided as a parameter to `test-connector` so at this
              ;; point we're passing control back to the caller. They need to provide a polling
              ;; function that takes the seed-data we generated, and the db handle, and execute
              ;; a query that will find the records that correspond with the seed data. We take
              ;; the result, and put it in the test-machine journal which will make it available
              ;; to both the `watch-fn` and the test assertions.
              (let [db-poller (mt/every 1000
                                        (fn []
                                          (let [poll-result (poll-fn seed-data db)]
                                            (send (:journal machine)
                                                  (fn [journal poll-data]
                                                    (assoc-in journal [:tables sink-table] poll-data))
                                                  poll-result))))]
                (try
                  ;; All that's left now is to write the example data to the input topic and
                  ;; wait for it to appear in the sink table. That's what `load-seed-data` does.
                  ;; Note how again we're handing control back to the test author by using their
                  ;; `watch-fn` (again passing in the seed data we generated for them so they can
                  ;; figure out what to watch for).
                  (log/info "load seed data" (map :id seed-data))
                  (load-seed-data machine topic-id seed-data
                                  {:key-fn key-fn
                                   :watch-fn (partial watch-fn seed-data)})

                  ;; Now the test-machine journal contains all the data we need to verify that the
                  ;; the connector is working as expected. So we just pass the current state of the
                  ;; journal to the `test-fn` which is expected to run some test assertions against
                  ;; the data
                  (test-fn @(:journal machine))
                  (finally
                    ;; Manifold's `manifold.time/every` returns a function that can be invoked in
                    ;; the finally clause to cancel the polling operation when the test is finished
                    ;; regardless of what happens during the test
                    (db-poller)))))))))))
{% endhighlight %}					

And that's it for now! Thanks for reading. Look forward to hearing
your thoughts and questions about this on Twitter. I tried to keep it
as short as possible so let me know if there's anything I glossed over
which you'd like to see explained in more detail in subsequent posts.

