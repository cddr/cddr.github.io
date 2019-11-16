---
layout: post
title: Reporting on Kafka Connect Jobs
---

At the risk of diluting the brand message (i.e. testing kafka stuff
using Clojure), in this post, I'm going to introduce some code for
extracting a report on the status of Kafka Connect jobs. I'd argue
it's still "on-message", falling as it does under the
observability/metrics umbrella and since observability is an integral
part of [testing in
production](https://medium.com/@copyconstruct/testing-in-production-the-safe-way-18ca102d0ef1)
then I think we're on safe ground.

I know I promised a deep-dive on the test-machine journal but it's
been a crazy week and I needed to self-sooth by writing about
something simpler that was mostly ready to go.

## Kafka Connect API

The distributed version of Kafka Connect provides an HTTP API for
managing jobs and providing access to their configuration and current
status, including any errors that have caused the job to stop
working. It also provides metrics over JMX but that requires

 1. Server configuration that is not enabled by default
 2. Access to a port which is often only exposed inside the production
    stack and is intended to support being queried by a "proper"
	monitoring system

This is not to say that you shouldn't go ahead and setup proper
monitoring. You definitely should. But you needn't let the absence of
it prevent you from quickly getting an idea of overall health of your
Kafka Connect system.

For this script we'll be hitting two of the endpoints provided by
Kafka Connect

## GET /connectors

Here's the function that hits the `/connectors` endpoint. It uses Zach
Tellman's [aleph](https://github.com/ztellman/aleph) and
[manifold](https://github.com/ztellman/manifold) libraries. The
`http/get` function returns a deferred that allows the API call to be
handled asynchronously by setting up a "chain" of operations to deal
with the response when it arrives.

{% highlight clojure %}

(ns grumpybank.observability.kc
 (:require
   [aleph.http :as http]
   [manifold.deferred :as d]
   [clojure.data.json :as json]
   [byte-streams :as bs]))

(defn connectors
  [connect-url]
  (d/chain (http/get (format "%s/connectors" connect-url))
    #(update % :body bs/to-string)
    #(update % :body json/read-str)
    #(:body %)))

{% endhighlight %}

## GET /connectors/:connector-id/status

Here's the function that hits the `/connectors/:connector-id/status`
endpoint.  Again, we invoke the API endpoint and setup a chain to deal
with the response by first converting the raw bytes to a string, and
then reading the JSON string into a Clojure map. Just the same as
before.

{% highlight clojure %}
(defn connector-status
  [connect-url connector]
  (d/chain (http/get (format "%s/connectors/%s/status"
                             connect-url
                             connector))
    #(update % :body bs/to-string)
    #(update % :body json/read-str)
    #(:body %)))
{% endhighlight %}	
	
## Generating a report
	
Depending on how big your Kafka Connect installation becomes and how
you deploy connectors you might easily end up with 100s of connectors
returned by the request above. Submitting a request to the status
endpoint for each one in serial would take quite a while. On the
other-hand, the server on the other side is capable of handling many
requests in parallel. This is especially true if there are a few Kafka
Connect nodes co-operating behind a load-balancer.

This is why it is advantageous to use aleph here for the HTTP requests
instead of the more commonly used clj-http. Once we have our list of
connectors, we can fire off simultaneous requests for the status of
each connector, and collect the results asynchronously.

{% highlight clojure %}
(defn connector-report
  [connect-url]
  (let [task-failed? #(= "FAILED" (get % "state"))
        task-running? #(= "RUNNING" (get % "state"))
        task-paused? #(= "PAUSED" (get % "state"))]
    (d/chain (connectors connect-url)
      #(apply d/zip (map (partial connector-status connect-url) %))
      #(map (fn [s]
              {:connector (get s "name")
               :failed? (failed? s)
               :total-tasks (count (get s "tasks"))
               :failed-tasks (->> (get s "tasks")
                                  (filter task-failed?)
                                  count)
               :running-tasks (->> (get s "tasks")
                                   (filter task-running?)
                                   count)
               :paused-tasks (->> (get s "tasks")
                                  (filter task-paused?)
                                  count)
               :trace (when (failed? s)
                        (traces s))}) %))))
{% endhighlight %}

Here we first define a few helper predicates (`task-failed?`,
`task-running?`, and `task-paused?`) for classifying the status
eventually returned by `connector-status`. Then we kick off the
asynchronous pipeline by requesting a list of connectors using
`connectors`.

The first operation on the chain is to apply the result to `d/zip`
which as described above will invoke the status API calls concurrently
and return a vector with all the responses once they are all complete.

Then we simply map the results over an anonymous function which builds
a map out of with the connector id together with whether it has
failed, how many of its tasks are in each state, and when the connector
*has* failed, the stacktrace provided by the status endpoint.

If you have a huge number of connect jobs you might need to split the
initial list into smaller batches and submit each batch in
parallel. This can easily be done using Clojure's built-in `partition`
function but I didn't find this to be necessary on our fairly large
collection of kafka connect jobs.

Wrap these functions up in a simple command line script and run it
after making any changes to your kafka-connect configuration to make
sure everything is still hunky-dory.

Here's a [gist](https://gist.github.com/cddr/da5215ed83653872bee3febdbb435e65)
that wraps these functions up into a quick and dirty script that reports the
results to STDOUT. Feel free to re-use, refactor, and integrate with
your own script to make sure after making changes to your deployed Kafka
Connect configuration, everything remains hunky-dory.


