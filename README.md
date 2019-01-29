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

<br/>

**Example 1**

**Producing and consuming binary data**

In this example, we will post this JSON binary (base64) data to the new binary topic we just created.
The first record has a key and will go to some partition. The second and third records have no key. The second record will write to partition 0 whereas the third partition will write to some partition.

Create your topic and verify it worked.
```commandline
$ ./kafka-topics --zookeeper localhost:2181 --create --topic climbinggym-binary --partitions 3 --replication-factor 1
$ ./kafka-topics --zookeeper localhost:2181 --describe --topic climbinggym-binary
$ curl http://127.0.0.1:8082/topics/climbinggym-binary | jq
```

```JSON
{
  "records": [
    {
      "key": "YXVuaXF1ZWtleQ==",
      "value": "SW5uZXIgUGVha3MgU291dGggRW5k"
    },
    {
      "value": "SW5uZXIgUGVha3MgQ3Jvd24gUG9pbnRl",
      "partition": 0
    },
    {
      "value": "QXRsYW50YSBSb2Nrcw=="
    }
  ]
}
```

Post your data to the topic.
```commandline
curl -X POST -H "Content-Type: application/vnd.kafka.binary.v2+json" \
      -H "Accept: application/vnd.kafka.v2+json" \
      --data '{"records": [{"key": "YXVuaXF1ZWtleQ==","value":"SW5uZXIgUGVha3MgU291dGggRW5k"},{"value": "SW5uZXIgUGVha3MgQ3Jvd24gUG9pbnRl","partition": 0},{"value": "QXRsYW50YSBSb2Nrcw=="}]}' "http://localhost:8082/topics/climbinggym-binary"
```
Create a consumer that will start at the beginning of the topic log. Make note of the returned base_url as that will be used to connect to in order to keep consuming from the same REST Proxy instance. The topics is a list of topics that you want your consumer to subscribe t. You can try to play with settings like auto.commit.enable=false to manually update your offset commits.
```commandline
curl -X POST -H "Content-Type: application/vnd.kafka.v2+json" \
      --data '{"name": "my_consumer_instance", "format": "binary", "auto.offset.reset": "earliest"}' \
      http://localhost:8082/consumers/my_binary_consumer

curl -X POST -H "Content-Type: application/vnd.kafka.v2+json" --data '{"topics":["climbinggym-binary"]}' \
      http://localhost:8082/consumers/my_binary_consumer/instances/my_consumer_instance/subscription
```
Consume from the topic. You can play with additional query parameters on the url such as "?timeout=3000"
```commandline
curl -X GET -H "Accept: application/vnd.kafka.binary.v2+json" \
      http://localhost:8082/consumers/my_binary_consumer/instances/my_consumer_instance/records
```
You should see results that look somewhat like the following. See if some of your records made it to other partitions.
```commandline
[{"topic":"climbinggym-binary","key":null,"value":"SW5uZXIgUGVha3MgQ3Jvd24gUG9pbnRl","partition":0,"offset":0},{"topic":"climbinggym-binary","key":null,"value":"QXRsYW50YSBSb2Nrcw==","partition":0,"offset":1},{"topic":"climbinggym-binary","key":"YXVuaXF1ZWtleQ==","value":"SW5uZXIgUGVha3MgU291dGggRW5k","partition":1,"offset":0}]
```
Finally, delete your consumer to make it exit the group and clean up resources.
```commandline
curl -X DELETE -H "Content-Type: application/vnd.kafka.v2+json" \
      http://localhost:8082/consumers/my_binary_consumer/instances/my_consumer_instance
```

<br/>

**Example 2**

**Producing and consuming JSON data**

JSON data does not need to be transformed like the example 1 above. JSON can be sent as is to the REST Proxy.

Everything is pretty much similar except for the header which will change from binary to JSON. Please take note of that below.

Create your topic and verify it worked.
```commandline
$ ./kafka-topics --zookeeper localhost:2181 --create --topic climbinggym-json --partitions 3 --replication-factor 1
$ ./kafka-topics --zookeeper localhost:2181 --describe --topic climbinggym-json
$ curl http://127.0.0.1:8082/topics/climbinggym-json | jq
```

```JSON
{
  "records": [
    {
      "key": "auniquekey",
      "value": "Inner Peaks South End"
    },
    {
      "value": "Inner Peaks Crown Pointe",
      "partition": 0
    },
    {
      "value": "Atlanta Rocks"
    }
  ]
}
```

Post your data.
```commandline
curl -X POST -H "Content-Type: application/vnd.kafka.json.v2+json" \
      -H "Accept: application/vnd.kafka.v2+json" \
      --data '{"records": [{"key": "auniquekey","value": "Inner Peaks South End"},{"value": "Inner Peaks Crown Pointe","partition": 0},{"value": "Atlanta Rocks"}]}' "http://localhost:8082/topics/climbinggym-json"
```
Create a consumer.
```commandline
curl -X POST -H "Content-Type: application/vnd.kafka.v2+json" \
      --data '{"name": "my_consumer_instance", "format": "json", "auto.offset.reset": "earliest"}' \
      http://localhost:8082/consumers/my_json_consumer

curl -X POST -H "Content-Type: application/vnd.kafka.v2+json" --data '{"topics":["climbinggym-json"]}' \
      http://localhost:8082/consumers/my_json_consumer/instances/my_consumer_instance/subscription
```
Consume from the topic.
```commandline
curl -X GET -H "Accept: application/vnd.kafka.json.v2+json" \
      http://localhost:8082/consumers/my_json_consumer/instances/my_consumer_instance/records
```
Look at your results.
```commandline
[{"topic":"climbinggym-json","key":"auniquekey","value":"Inner Peaks South End","partition":0,"offset":0},{"topic":"climbinggym-json","key":null,"value":"Inner Peaks Crown Pointe","partition":0,"offset":1},{"topic":"climbinggym-json","key":null,"value":"Atlanta Rocks","partition":1,"offset":0}]
```
Delete your consumer.
```commandline
curl -X DELETE -H "Content-Type: application/vnd.kafka.v2+json" \
      http://localhost:8082/consumers/my_json_consumer/instances/my_consumer_instance
```

<br/>

**Example 3**

**Producing and consuming Avro data**

Again, the interesting feature here is you can (and should) connect REST Proxy directly to the Confluent Schema Registry.

Everything is pretty much similar except that you send the schema and payload in JSON. Also, after you first produce you will receive a schema ID which you can re-use for future production.

Create your topic and verify it worked.
```commandline
$ ./kafka-topics --zookeeper localhost:2181 --create --topic climbinggym-avro --partitions 3 --replication-factor 1
$ ./kafka-topics --zookeeper localhost:2181 --describe --topic climbinggym-avro
$ curl http://127.0.0.1:8082/topics/climbinggym-avro | jq
```

Notice this time your Avro schema has to be stringified JSON (translation: you need to escape it).
```Avro
{
  "value_schema": "{\"type\": \"record\", \"name\": \"ClimbingGym\", \"fields\": [{\"name\": \"gym_name\", \"type\": \"string\"}, {\"name\" :\"gym_nickname\",  \"type\": \"string\"}]}",
  "records": [
    {
      "value": {"gym_name": "Inner Peaks", "gym_nickname": "Inner Peaks South End" }
    },
    {
      "value": {"gym_name": "Inner Peaks", "gym_nickname": "Crown Pointe" },
      "partition": 0
    }
  ]
}
```

Post your data. Make note of the returned schema_id.
```commandline
curl -X POST -H "Content-Type: application/vnd.kafka.avro.v2+json" \
      -H "Accept: application/vnd.kafka.v2+json" \
      --data '{"value_schema": "{\"type\": \"record\", \"name\": \"ClimbingGym\", \"fields\": [{\"name\": \"gym_name\", \"type\": \"string\"}, {\"name\" :\"gym_nickname\",  \"type\": \"string\"}]}","records": [{"value": {"gym_name": "Inner Peaks", "gym_nickname": "Inner Peaks South End" }},{"value": {"gym_name": "Inner Peaks", "gym_nickname": "Crown Pointe" },"partition": 0}]}' "http://localhost:8082/topics/climbinggym-avro"
```
Let's try doing another POST, but this time replacing your value_schema with value_schema_id. In my case, it would look something like this with a value_schema_id of 41. Once you post it, your schema id returned should not have changed. In my case, it's still 41 for this new POST. Schema Registry in action!
```commandline
curl -X POST -H "Content-Type: application/vnd.kafka.avro.v2+json" \
      -H "Accept: application/vnd.kafka.v2+json" \
      --data '{"value_schema_id": 41,"records": [{"value": {"gym_name": "Atlanta Rocks", "gym_nickname": "Best gym in Atlanta?" }}]}' "http://localhost:8082/topics/climbinggym-avro"
```
Create a consumer.
```commandline
curl -X POST -H "Content-Type: application/vnd.kafka.v2+json" \
      --data '{"name": "my_consumer_instance", "format": "avro", "auto.offset.reset": "earliest"}' \
      http://localhost:8082/consumers/my_avro_consumer

      curl -X POST -H "Content-Type: application/vnd.kafka.v2+json" --data '{"topics":["climbinggym-avro"]}' \
            http://localhost:8082/consumers/my_avro_consumer/instances/my_consumer_instance/subscription      
```
Consume from the topic.
```commandline
curl -X GET -H "Accept: application/vnd.kafka.avro.v2+json" \
      http://localhost:8082/consumers/my_avro_consumer/instances/my_consumer_instance/records
```
Look at your results. It should contain a couple of records that look something like this:
```commandline
[{"topic":"climbinggym-avro","key":null,"value":{"gym_name":"Inner Peaks","gym_nickname":"Inner Peaks South End"},"partition":2,"offset":0},{"topic":"climbinggym-avro","key":null,"value":{"gym_name":"Inner Peaks","gym_nickname":"Crown Pointe"},"partition":0,"offset":0}]
```
Delete your consumer.
```commandline
curl -X DELETE -H "Content-Type: application/vnd.kafka.v2+json" \
      http://localhost:8082/consumers/my_avro_consumer/instances/my_consumer_instance
```

<br/>
**That's it. I hope this summed it up a few quick examples for you nicely.**
