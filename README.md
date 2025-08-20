# Issue #1 - Unexpected behavior when Service Task that is marked as Async fails
## Description
The ```"hiring"``` BPMN workflow contains a Service Task, ```"Store Candidate Data"```, which is marked with isAsync=true. The Service Task executes the Java method ```org.kie.kogito.hr.servicetasks.DummyServiceTask.callDummy```that calls an HTTP service, and throws an exception in case of HTTP response code <> 2xx. When the exception is thrown, the job execution is retried 62 times and then it stops, saying that the retry limit was exceeded, and the process instance gets stuck in that node (in ACTIVE state and not in ERROR state).

The HTTP call is performed against a mocked service using Wiremock dev services. To modify the HTTP response code, edit the line #15 in the file ```src\main\resources\wiremock\dummy\mappings\dummy-api-stubs.json``` and replace ```500``` by ```200```.

## Steps to reproduce:
1. Start the Kogito App in dev mode: ```mvn "-Pbamoe-community" "-Pbamoe-persistence-postgresql" "-Pembedded-postgresql" clean quarkus:dev```. The runtime will use the last timestamped version of Kogito Community. 

2. Send this request to start a new process instance:
```
curl --location 'http://localhost:8080/hiring' \
--header 'Accept: application/json' \
--header 'Content-Type: application/json' \
--header 'x-kogito-correlationkey: 12345' \
--header 'Authorization: ••••••' \
--data-raw '{
  "candidateData": {
    "name": "Jon",
    "lastName": "Snow",
    "email": "jon@snow.org",
    "experience": 5,
    "skills": ["Java", "Kogito", "Fencing"]
  },
  "needMgmtApproval" : false,
  "expirationTime" : "PT1200S",
  "processExpirationTime" : "PT1100S",
  "throwException" : true,
  "workitemType" : "ST",
  "errorStrategy" : "ABORT"
}'
```
3. The logs will show how the Jobs Service retries the failed job 62 times

## Current behavior
Async node execution:
```
// NOT OK: async node timer NOT executed immediately
2025-08-20 09:46:33,238 DEBUG [org.kie.kog.app.job.imp.VertxJobScheduler] (vert.x-eventloop-thread-1) Executing timeout with timer Id 1 and jobId 362099a8-9fb9-4fa8-a02c-c031e1600d4a
2025-08-20 09:46:33,239 TRACE [org.kie.kog.app.job.imp.VertxJobScheduler] (Jobs-1) Timeout task 1 with jobId 362099a8-9fb9-4fa8-a02c-c031e1600d4a newTimeoutTask
2025-08-20 09:46:33,245 TRACE [org.kie.kog.app.job.imp.VertxJobScheduler] (executor-thread-1) addTimerInfo JobDetails[id='ac3de423-462b-4aae-8860-c4fb69c76e5a', correlationId='7ab62494-f384-478b-96c7-252f4d7d2e69', status=SCHEDULED, lastUpdate=null, retries=0, executionCounter=0, scheduledId='null', recipient=RecipientInstance{recipient=InVMRecipient [data=InVMPayloadData [data=ProcessInstanceJobDescription{id='ac3de423-462b-4aae-8860-c4fb69c76e5a', timerId=-1', expirationTime=org.kie.kogito.jobs.DurationExpirationTime@486db6fc, priority=5, processInstanceId='7ab62494-f384-478b-96c7-252f4d7d2e69', rootProcessInstanceId='null', processId='hiring', rootProcessId='null', nodeInstanceId='815406a4-69f6-467d-92b9-22a815a044d3'}]]}, trigger=org.kie.kogito.timer.impl.SimpleTimerTrigger@2c950a25, executionTimeout=1755705893212, executionTimeoutUnit=Millis, created=null]

2025-08-20 09:46:33,246 TRACE [org.kie.kog.app.job.imp.VertxJobScheduler] (executor-thread-1) doSchedule JobDetails[id='ac3de423-462b-4aae-8860-c4fb69c76e5a', correlationId='7ab62494-f384-478b-96c7-252f4d7d2e69', status=SCHEDULED, lastUpdate=null, retries=0, executionCounter=0, scheduledId='null', recipient=RecipientInstance{recipient=InVMRecipient [data=InVMPayloadData [data=ProcessInstanceJobDescription{id='ac3de423-462b-4aae-8860-c4fb69c76e5a', timerId=-1', expirationTime=org.kie.kogito.jobs.DurationExpirationTime@486db6fc, priority=5, processInstanceId='7ab62494-f384-478b-96c7-252f4d7d2e69', rootProcessInstanceId='null', processId='hiring', rootProcessId='null', nodeInstanceId='815406a4-69f6-467d-92b9-22a815a044d3'}]]}, trigger=org.kie.kogito.timer.impl.SimpleTimerTrigger@2c950a25, executionTimeout=1755705893212, executionTimeoutUnit=Millis, created=null]

2025-08-20 09:46:33,248 TRACE [org.kie.kog.app.job.imp.VertxJobScheduler] (Jobs-1) Timeout 1 with jobId 362099a8-9fb9-4fa8-a02c-c031e1600d4a won't run


// job is executed only after sync
2025-08-20 09:47:11,570 DEBUG [org.kie.kog.app.job.imp.VertxJobScheduler] (Jobs-2) Syncing jobs with job store
2025-08-20 09:47:11,597 TRACE [org.kie.kog.app.job.imp.VertxJobScheduler] (Jobs-2) Timeout task 3 with jobId 362099a8-9fb9-4fa8-a02c-c031e1600d4a newTimeoutTask
2025-08-20 09:47:11,599 DEBUG [org.kie.kog.app.job.imp.VertxJobScheduler] (Jobs-2) Timeout 3 with jobId 362099a8-9fb9-4fa8-a02c-c031e1600d4a will be executed

// retries:
// OK - number of retries reached:
2025-08-20 09:49:33,264 TRACE [org.kie.kog.app.job.imp.VertxJobScheduler] (Jobs-7) Do not retry JobDetails[id='362099a8-9fb9-4fa8-a02c-c031e1600d4a', correlationId='7ab62494-f384-478b-96c7-252f4d7d2e69', status=ERROR, lastUpdate=2025-08-20T15:48:33.294Z[UTC], retries=3,

// NOT OK - additional timer is created:
2025-08-20 09:49:33,264 TRACE [org.kie.kog.app.job.imp.VertxJobScheduler] (Jobs-7) Timeout 6 with jobId 362099a8-9fb9-4fa8-a02c-c031e1600d4a will be updated
2025-08-20 09:49:33,264 TRACE [org.kie.kog.app.job.imp.VertxJobScheduler] (Jobs-7) addTimerInfo JobDetails[id='362099a8-9fb9-4fa8-a02c-c031e1600d4a', correlationId='7ab62494-f384-478b-96c7-252f4d7d2e69', status=ERROR, lastUpdate=2025-08-20T15:48:33.294Z[UTC], retries=3, executionCounter=0, scheduledId='null', recipient=RecipientInstance{recipient=InVMRecipient [data=InVMPayloadData [data=ProcessInstanceJobDescription{id='362099a8-9fb9-4fa8-a02c-c031e1600d4a', timerId=-1', expirationTime=org.kie.kogito.jobs.ExactExpirationTime@33dd237d, priority=5, processInstanceId='7ab62494-f384-478b-96c7-252f4d7d2e69', rootProcessInstanceId='null', processId='hiring', rootProcessId='null', nodeInstanceId='b7eadd8f-93c1-4c19-8d21-823c06605b0c'}]]}, trigger=org.kie.kogito.timer.impl.SimpleTimerTrigger@17d7815d, executionTimeout=1755704973224, executionTimeoutUnit=Millis, created=2025-08-20T15:46:33.225Z[UTC]]

2025-08-20 09:49:33,265 TRACE [org.kie.kog.app.job.imp.VertxJobScheduler] (Jobs-7) doSchedule JobDetails[id='362099a8-9fb9-4fa8-a02c-c031e1600d4a', correlationId='7ab62494-f384-478b-96c7-252f4d7d2e69', status=ERROR, lastUpdate=2025-08-20T15:48:33.294Z[UTC], retries=3, executionCounter=0, scheduledId='null', recipient=RecipientInstance{recipient=InVMRecipient [data=InVMPayloadData [data=ProcessInstanceJobDescription{id='362099a8-9fb9-4fa8-a02c-c031e1600d4a', timerId=-1', expirationTime=org.kie.kogito.jobs.ExactExpirationTime@33dd237d, priority=5, processInstanceId='7ab62494-f384-478b-96c7-252f4d7d2e69', rootProcessInstanceId='null', processId='hiring', rootProcessId='null', nodeInstanceId='b7eadd8f-93c1-4c19-8d21-823c06605b0c'}]]}, trigger=org.kie.kogito.timer.impl.SimpleTimerTrigger@17d7815d, executionTimeout=1755704973224, executionTimeoutUnit=Millis, created=2025-08-20T15:46:33.225Z[UTC]]

```

* OK: Number of retries matches the configured retry value (default = 3)
* NOT OK: Process is in ACTIVE state (not ERROR)

## Expected behavior
The Service Task is executed and, if an exception is thrown, it's retried a **predefined number of times with a delay between retries predefined as well** (how to do this?) as in jBPM v7. If the **number of retries exceeds the limit** and the Service Task still fails, then the **process instance stops** the execution, the **instance status changes to ```"ERROR"```** and the **job can be retriggered manually** once the underlying problem is solved.