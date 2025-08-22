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
3. The logs will show the Jobs Service retries the failed job 3 times - OK

4. NOT OK: Process is in ACTIVE state (not ERROR)

## Expected behavior
The Service Task is executed and, if an exception is thrown, it's retried a **predefined number of times with a delay between retries predefined as well** (how to do this?) as in jBPM v7. If the **number of retries exceeds the limit** and the Service Task still fails, then the **process instance stops** the execution, the **instance status changes to ```"ERROR"```** and the **job can be retriggered manually** once the underlying problem is solved.