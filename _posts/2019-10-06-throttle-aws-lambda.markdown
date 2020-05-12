---
layout: post
title:  "Calling functions at a throttled rate in AWS Lambdas"
date:   2019-10-06 12:55:26 -0700
categories: AWS Lambda, Scala, Akka
---

One problem I frequently face is the need the call an external resource, such as a REST API, at a throttled rate. In Scala, I've used the `throttle` method in Akka Streams to tackle this problem, which allows events from a sources to be emitted at a specified rate. Here's an example in Scala where I might want to call an external resource (via the `callExternalResource` method) with a list of items at a throttled rate of one item every five seconds:


```
import akka.stream.scaladsl.Source
import akka.stream.ThrottleMode
import scala.concurrent.duration._

val inputListToThrottle = List(1, 2, 3)
Source(inputListToThrottle)
.throttle(1, 5.seconds, 1, ThrottleMode.shaping)
.map { nextItem =>
  callExternalResource(nextItem)
}
```

Recently, I've needed to implement this pattern in AWS Lambdas. A simple implementation in Python would be to use `time.sleep`, but that would entail paying for the compute time when the AWS Lambda function is sleeping. I thought about employing AWS Step Functions and using a method outlined [here](https://blog.scottlogic.com/2018/06/19/step-functions.html), but it turns out that AWS Step Functions are relatively expensive compared to AWS Lambdas. Instead, I found a solution by putting events on an SQS queue with a `delay` with a producer Lambda and a consumer Lambda that's triggered when the `delay` expires.

In the producer Lambda, simply put a JSON `record` to SQS using the `DelaySeconds` parameter:

```
import boto3
import json

sqs_client = boto3.client('sqs', region_name='us-west-2')

def put_throttled_request_on_queue(record, delay_seconds):
return sqs_client.send_message(
    QueueUrl='queue-name-here',
    MessageBody=json.dumps(record),
    DelaySeconds=delay_seconds
)
```

The only problem with this is that the maximum delay in SQS is fifteen minutes, so I specify an additional `delay` for the number of additional seconds beyond fifteen minutes that a record needs to be throttled:

```
def put_throttled_request_on_queue(record, delay_seconds):
    max_delay_seconds = 900
    if delay_seconds > max_delay_seconds:
        new_delay = delay_seconds - max_delay_seconds
        record['delay'] = new_delay
        return put_sqs_record(record, max_delay_seconds)
    else:
        return put_sqs_record(record, delay_seconds)
``` 

This means that an item will have to be re-queued every fifteen minutes. Configure a Lambda to be triggered based by SQS events to consume the throttled messages and call the external resourse (`call_external_resource`) or re-queue the item if additional wait time is needed:

```
for record in event['Records']:
    record_body = record['body']
    body_json = json.loads(record_body)
    additional_delay_for_throttling = body_json.get('delay', None)
    if additional_delay_for_throttling:
        put_customer_request_on_queue(customer_id, additional_delay_for_throttling)
    else:
    	call_external_resource(record)
```

This creates a recursive call to the consumer, which is something that AWS discourages because it could incur large compute costs if there are problems in the recursive logic. Additional constraints could be placed to prevent too many recursive calls, such as a `count` field on the queued item that keeps track of the number of times that the item has been re-queued.
