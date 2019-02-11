# RESTProxyExamples
This is a simple repo of examples using the Confluent REST Proxy

The Confluent REST Proxy is a convenient way to use REST to communicate to your Kafka cluster.

An example of using the REST Proxy could be a client application using a language that does not have great native Avro support. In Java, Avro support is great. However, nearly all languages support JSON/HTTP. Enter REST proxy.

Be aware though:
- There is a performance hit using HTTP instead of Kafka's native protocol. So this is probably ok for a lot of use-cases, but if you are looking for maximum performance, re-consider your plan.
- It's up to the producing application to batch events to the REST Proxy.
- Use the new v2 API. If you see anyone using the older v1, have them migrate to v2.
- REST Proxy is stateless, but consumers are stateful - see https://docs.confluent.io/current/kafka-rest/docs/deployment.html#post-deployment
- Keep in mind that individual REST Proxy instances sitting behind a load balancer still have to be uniquely addressable and accessible for consumers (e.g. commits). This allows you to run multiple instances of REST Proxy, each with a unique ID upon startup, but sharing the workload of consumers.

You can produce to the REST Proxy using Binary data (base64 encoded strings), JSON (plain old JSON) and Avro (JSON encoded data). With the latter, you can configure REST Proxy (and should) to connect directly to the Confluent Schema Registry. If you want help encoding/decoding to/from base64, you can try this free utility: http://www.utilities-online.info/base64.

Pre-reqs:
No need to overcomplicate things, if you have a cluster great, if not just download Confluent platform from confluent.io and untar the bundle and run it on your workstation locally.
It will make it easier to connect to localhost (as these configs show) and allow things like producers to auto-create topics.

Reference the sub-directories in this repo for the details of each example.

<br/>

**Example 1**

**Producing and consuming binary data**

In this example, we will post this JSON binary (base64) data to the new binary topic we just created.
The first record has a key and will go to some partition. The second and third records have no key. The second record will write to partition 0 whereas the third partition will write to some partition.

<br/>

**Example 2**

**Producing and consuming JSON data**

JSON data does not need to be transformed like the example 1 above. JSON can be sent as is to the REST Proxy.

Everything is pretty much similar except for the header which will change from binary to JSON.

<br/>

**Example 3**

**Producing and consuming Avro data**

The interesting feature here is you can (and should) connect REST Proxy directly to the Confluent Schema Registry.

Everything is pretty much similar except that you send the schema and payload in JSON. Also, after you first produce you will receive a schema ID which you can re-use for future production.

<br/>

**That's it. I hope this summed it up a few quick examples for you nicely.**
