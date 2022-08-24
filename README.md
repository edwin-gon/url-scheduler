# URL Scheduler 

## Problem Statement

A user is provided CLI Application to schedule URL invocations. User should be able to CRUD a scheduled entry.   

Assumptions:
- Only GET method allowed — URL invocations do not allow specified headers or body requests. 
- No Auth implementation 
- No restriction on number of requests or specific actions per user
- Multiple executions of the scheduled job are okay
- Fixed Timestamp format/time zone
- Users can schedule requests however far in the future
---
# [System Design](url_scheduler.png)

## Walkthrough:

1) User will provide identifier, timestamp, and url to invoke. A new scheduled job will be created.

2) Once the job is saved, if the scheduled time falls within a 15 minute range after the current time, it will be added to the queue (scheduled) to be processed. Otherwise, background process will run every 15 minutes looking for upcoming jobs with a timestamp within the next 15 minutes of the current time.

3) The messages that are scheduled will then be processed by workers associated to the queue. The workers will report back to the queue if the event/s were successful.
    - If successful, the message will be removed from the queue.
    - Otherwise, the message will be retried a certain number of times and after so many retries, the event will be handed off to a dead letter queue, which will have its own set of workers to notify team of a failure. *Future feature to  later support allowing user to set up notification policy.*   

The solution discussed above and depicted in the diagram makes use of the following AWS resources and responsibilities:

- **Lambda** — A serverless function invoked on events to handle business logic. 
    - Events include:
        - API invocations where each function is associated to a specific method and path.
        - Database Actions (DynamoDB Streams) — leveraging the TTL attribute to automatically trigger a deletion event to invoke a Lambda function to hand off request to queue to be 
        processed.
        - Queue (SQS) events being published and awaiting processing
    - Serverless functions allow for a lower cost since they only incur cost while running.
    - Provides the opportunity to scale and reach demand of dynamic workloads automatically.

- **API Gateway** — Managed Service that allows developers to configure API configurations. This will house our REST API.

- **DynamoDB** — A managed NoSQL database solution. This will allow us to achieve very quick reads and writes. Also we will be looking at leveraging Single Table Design to leverage the same table to capture invocation details (owner, time of execution, url) and current status. 

Additional Considerations:
- Recovering from points of failures and outages
    - Expected behavior should retry execution and lastly move over to a dead letter queue 
- Notifications of tasks process (Pub/Sub)
- How close to invocation time can we get? Is exact time invocation extremely important

Create scheduled job — user will provide identifier, url to invoke, timestamp to invoke job

Read scheduled job — user identifier and job identifier

Read scheduled jobs — user identifier and status (upcoming, succeeded, failed) 

Update scheduled job — user identifier, task identifier, timestamp change and url to invoke

Delete Scheduled job — user identifier, task identifier

**Access Patterns** 
- Find object by user identifier and identifier
- Find object by user identifier and status
- Find object by user identifier and ran between certain time range (1 month, week, days, hours) 
- Get object with upcoming status and timestamp 

## API Design

***Bonus Points:***

Protected Endpoints
- Number invocations by status, date, and time range
- Validation: 
    - See if endpoint matches required criteria. 
    - Output list of criteria that needs to be satisfied
- Test: allow users to test payload against expected responses/payloads


---
## Gotchas

### DynamoDB

- TTL Attribute 
    
    If you are looking for near real time deletion of records, TTL attribute may not be something you can relay on. 

    According to [AWS](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/howitworks-ttl.html) 

    > The TTL deletion is meant more as a background process and happens within 48 hours of expiration. This is a great case for archiving information however not if you need to know if an item has reach the exact expiration time.

- DynamoDB Stream Events
    
    Dynamo Streams if enabled will create events on modifications made to all items in the table. Meaning all creates, updates, and deletes will create an event. Developer can specify what information from the item is to be recorded in an event (Keys Only, New Image, Old Image, or both old and new images) For more information see there docs [here](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Streams.html).

    If only interested in particular events, say CREATE. You need to apply an [event filter](https://docs.aws.amazon.com/lambda/latest/dg/invocation-eventfiltering.html) if events are being processed by Lambda. This is can be defined as IaC, CLI or within the AWS Console and associated to Lambda.

## SQS

- A separate queue needs to be defined as a dead letter queue and not part of the deployment process of the initial queue.

- Workers associated to SQS can receive one to many events. If one event fails in the batch all events are retried if policy specified.


# Resources

## DynamoDB

- [How it work TTL](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/howitworks-ttl.html)


## SQS

- [SQS Retries](https://docs.aws.amazon.com/lambda/latest/operatorguide/sqs-retries.html)
- [Using Lambda with SQS](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html)
- [SQS Delay Queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-delay-queues.html)
