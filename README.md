# URL Scheduler 

## Problem Statement

A user is provided CLI Application to schedule URL invocations. User should be able to CRUD a scheduled entry.   

Assumptions:
- Invocations are one time and not reoccurring. 
- Only GET method allowed — URL invocations do not allow specified headers or body requests. 
- No Auth implementation 
- No restriction on number of requests or specific actions per user
- Multiple executions of the scheduled job are okay
- Fixed Timestamp format/time zone
- Users can schedule requests however far in the future

**Note**: Job is used interchangeable with URL invocation scheduled.

---
# [System Design](url_scheduler.png)

## Walkthrough:

1) User will provide identifier, timestamp, and url to invoke. A new scheduled job will be created.

2) Once the job is saved, if the scheduled time falls within a 15 minute range after the current time, it will be added to the queue (scheduled) to be processed. Otherwise, background process will run every 15 minutes looking for upcoming jobs with a timestamp within the next 15 minutes of the current time.

3) The messages that are scheduled will then be processed by workers associated to the queue. The workers will report back to the queue if the event/s were successful.
    - If successful, the message will be removed from the queue.
    - Otherwise, the message will be retried a certain number of times and after so many retries, the event will be handed off to a dead letter queue, which will have its own set of workers to notify team of a failure.*Future feature to  later support allowing user to set up notification policy.*   

## Implementation: 

The solution discussed above and depicted in the diagram makes use of the following AWS resources and responsibilities:

- **Lambda** — A serverless function invoked on events to handle business logic. Compute size will be configured based on workload expectations.  
    - Events include:
        - API invocations where each function is associated to a specific method and path.
        - Database Actions (DynamoDB Streams) — leveraging the creation event to automatically invoke a Lambda function to hand off request to queue if request falls within certain time frame.
        - Queue (SQS) events being published and awaiting processing
    - Serverless functions allow for a lower cost since they only incur cost while running.
    - Provides the opportunity to scale and reach demand of dynamic workloads automatically.

- **API Gateway** — Managed Service that allows developers to configure API configurations. This will house our REST API.

- **DynamoDB** — A managed NoSQL database solution. This will allow us to achieve very quick reads and writes. Also we will be looking at leveraging Single Table Design to leverage the same table to capture invocation details (owner, time of execution, url) and user information. 

- **EventBridge** — A managed event bus service that will be used to create a scheduled process to query the database to return results within a specific time frame to be handed off to our job queue.

- **Simple Queue Service (SQS)** — A managed service that will allow the application to offload requests, orchestrate workers that process business logic and capture errors made. Errors will be retried and handed to a dead letter queue if exceeding retry policy. 

- **Simple Notification Service (SNS)** — A managed messaging service that is based on a Pub/Sub model and will be used to notify application team if processing URL resulted in an error and being sent to the dead letter queue.  

### Considerations:
- Recovering from points of failures and outages
    - Expected behavior should retry execution and lastly move over to a dead letter queue for additional analysis and intervention.
- Notifications of tasks process (Pub/Sub) 
    - This could include a development team, user (individual requesting invocation), or a service.
- How close to defined invocation time can we get? Is exact time invocation extremely important?
    - Understanding that certain services do not provide immediate invocation such as TTL offering in DynamoDb. Meaning that it may be more efficient to relay on a scheduled lookup and invocation assignment  
---

### Database Access Patterns

Job will have the following contents:

- Id — Unique Identifier 
- User — User associated to
- URL — URL to invoke
- Time to Invoke — EPOCH Timestamp 
- Status

User Access Patterns:

- Find specific job by identifier  — ID 
- Find jobs by user identifier — User PK and Sort Key TimeStamp (Between to support between a specific range - 1 month, week, days, hours)
- Find jobs by user identifier and status — User PK and Status and Sort by TimeStamp

Service Access Patterns:
- Find entries with a particular status a certain range (15 minutes, 1 day) — Status and Sort Key (Between)

Id — GUID HK & Timestamp SK => Primary key Combination
USERNAME HK & TIMESTAMP SK => Secondary Key (username and range)
STATUS HK & Timestamp SK => Secondary Key (service call)

Think about combination:
STATUS & TIMESTAMP#USERNAME

## API Design

***Bonus Points:***

Protected Endpoints
- Number invocations by status, date, and time range
- Validation: 
    - See if endpoint matches required criteria. 
    - Output list of criteria that needs to be satisfied
- Test: allow users to test payload against expected responses/payloads

Create scheduled job — user will provide identifier, url to invoke, timestamp to invoke job

Read scheduled job — user identifier and job identifier

Read scheduled jobs — user identifier and status (upcoming, succeeded, failed) 

Update scheduled job — user identifier, task identifier, timestamp change and url to invoke

Delete Scheduled job — user identifier, task identifier

---
## Gotchas

### DynamoDB

- TTL Attribute 
    
    If you are looking for near real time deletion of records, TTL attribute may not be something you can relay on. 

    According to [AWS](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/howitworks-ttl.html) 

    > The TTL deletion is meant more as a background process and happens within 48 hours of expiration. This is a great case for archiving information however not if you need to know if an item has reach the exact expiration time.

- DynamoDb Keys

    DynamoDb Global Secondary Indexes do not need to be unique. Meaning whether using solely a partition key or composite key values can be the same. However remember that Primary Keys (default key schema when creating table) needs to be guaranteed unique. 

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
