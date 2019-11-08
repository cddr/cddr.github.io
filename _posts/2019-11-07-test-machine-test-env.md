---
layout: post
title: A Test Environment for Kafka Applications
---

In [Testing Event Driven
Systems](https://www.confluent.io/blog/testing-event-driven-systems),
I introduced the test-machine, (a Clojure library for testing kafka
applications) and included a simple example for demonstration
purposes. I made the claim that however your system is implemented, as
long as its input and output can be represented in Kafka, the
test-machine would be an effective tool for testing it. Now we’ve had
some time to put that claim to the...ahem test, I thought it might be
interesting to explore some actual use-cases in a bit more detail.

Having spent a year or so of using the test-machine, I can now say
with increased confidence that it is an effective tool for
testing a variety of Kafka based systems. However with the benefit of
experience, I'd add that you might want to define your own domain
specific layer of helper functions on top so that your tests may bear
some resemblance to the discussion that happens in your sprint
planning meetings. The raw events represent a layer beneath what we
typically discuss with product owners.

Hopefully the use-cases described in this forthcoming mini-series
will help clarify this concept and get you thinking about
how you might be able to apply the test-machine to solve your own
testing problems.

Before getting into the actual use-cases though, let’s get a test environment
setup so we can quickly run experiments locally without having to deploy
our code to a shared testing environment.

## Service Composition

For each of these tests, we’ll be using docker-compose to setup the
test environment. There are other ways of providing a test-environment
but the nice thing about docker-compose is that when things go awry
you can blow away all test state and start again with a clean
environment. This makes the process of acquiring a test-environment
*repeatable*, and at least after the first time you do it, pretty
fast. On my machine, `docker-compose down && docker-compose up -d`
doesn’t usually take more than 5-10 seconds or so. If you have not
used the confluent images before, it might take a while to download
the images if you're not on the end of a fat internet pipe.

Ideally you should be able to run your tests against a
test-environment with existing data. Your tests should create all the
data they need themselves and ignore any data that has been already
entered so acquiring a fresh test environment is not something you
should be doing for each test-run. Sometimes while developing a test,
it can help avoid confusing behavior to have a completely clean environment
but I wouldn't consider the test to be complete until it can be run
against a test environment with old data.

Below is a base docker-compose file containing the core services from
Confluent that will be required to run these tests. Depending on what’s being
tested, we will need additional services to fully exercise the system
under test. The configuration choices are made with a view to minimizing
the memory required by the collection of services. This is tailored
for the use-case of running small tests on a local laptop that
typically has zoom, firefox and chrome all clamoring for their share
of RAM. It is not intended for production workloads.

{% highlight yaml %}
version: '3'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:5.1.0
    expose:
      - "2181"
    ports:
      - "2181:2181"
    environment:
      KAFKA_OPTS: '-Xms256m -Xmx256m'
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  broker:
    image: confluentinc/cp-kafka:5.1.0
    depends_on:
      - zookeeper
    expose:
      - "9092"
    ports:
      - "9092:9092"
      - "19092:19092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://broker:9092,PLAINTEXT_HOST://localhost:19092
      KAFKA_ADVERTISED_HOST_NAME: localhost
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "false"
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_OFFSETS_TOPIC_NUM_PARTITIONS: 1
      KAFKA_OPTS: '-Xms256m -Xmx256m'
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_AUTO_OFFSET_RESET: "latest"
      KAFKA_ENABLE_AUTO_COMMIT: "false"

  schema-registry:
    image: confluentinc/cp-schema-registry:5.1.0
    depends_on:
      - zookeeper
      - broker
    expose:
      - "8081"
    ports:
      - "8081:8081"
    environment:
      KAFKA_OPTS: '-Xms256m -Xmx256m'
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_CONNECTION_URL: 'zookeeper:2181'
{% endhighlight %}

## Test Environment Healthchecks

It’s always a good idea to make sure the composition of services is
behaving as expected before trying to write tests against
them. Otherwise you might spend hours scratching your head wondering
why your system isn’t working when the problem is actually
mis-configuration of the test environment.

The most basic health-check you can do is to run `docker-compose ps`. This
will show at least that the services came up without exiting
immediately due to mis-configuration. In the happy case, the state of
all services should be “Up”. This command also shows which ports are
exposed by each service which will be important information when it comes
to configuring the system under test.

![docker-compose ps]({{ site.baseurl }}/images/docker-compose-ps.png)

### Accessing the logs

When something goes wrong there is often a clue in the logs although it
will take a bit of experience with them before you'll know what to look
for. Familiarizing yourself with them will payoff eventually though
both in “dev mode” when you’re trying to figure out why the code you’re
writing doesn’t work, and also in “ops mode” when you’re trying to
figure out what’s gone wrong in a deployed system. Getting access to
them in the test environment described here is the same as any other
docker-compose based system. The snippets below demonstrate a few of
the common use-cases and the full documentation is available
at [docs.docker.com](https://docs.docker.com/compose/reference/logs/)

{% highlight sh%}

# get all the logs
$ docker-compose logs

# get just the broker logs
$ docker-compose logs broker

# get the schema-registry logs and print more as they appear
$ docker-compose logs -f schema-registry

{% endhighlight%}

### Testing Connectivity

Another diagnostic tool that helps when debugging connectivity
issues is telnet. Experienced engineers will probably know this already
but for example, to ensure that you can reach kafka from your system under
test (assuming the system you're testing runs on the host OS), you can try
to reach the port exposed by the docker-compose configuration.

{% highlight sh%}

telnet localhost 19092

{% endhighlight %}

If the problem is more gnarly than basic connectivity issues, then Julia Evans'
[debugging zine](https://jvns.ca/debugging-zine.pdf) contains very useful advice
about debugging *any* problem you have with Linux based systems.

That's all for now. In the next article, I'll use this test environment
together with the [test-machine](https://github.com/FundingCircle/jackdaw/blob/master/doc/test-machine.md)
library to build a helper function for testing Kafka Connect JDBC Sinks.
