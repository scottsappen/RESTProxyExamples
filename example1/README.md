# RESTProxyExamples

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
