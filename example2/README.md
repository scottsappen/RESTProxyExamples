# RESTProxyExamples

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
