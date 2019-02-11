# RESTProxyExamples

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
