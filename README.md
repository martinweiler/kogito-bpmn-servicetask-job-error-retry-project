# Issue #1 - Unexpected behavior when Service Task that is marked as Async fails
## Description
The ```"hiring"``` BPMN workflow contains a Service Task, ```"Store Candidate Data"```, which is marked with isAsync=true. The Service Task executes the Java method ```org.kie.kogito.hr.servicetasks.DummyServiceTask.callDummy```that calls an HTTP service, and throws an exception in case of HTTP response code <> 2xx. When the exception is thrown, the job execution is retried 62 times and then it stops, saying that the retry limit was exceeded, and the process instance gets stuck in that node (in ACTIVE state and not in ERROR state).

The HTTP call is performed against a mocked service using Wiremock dev services. To modify the HTTP response code, edit the line #15 in the file ```src\main\resources\wiremock\dummy\mappings\dummy-api-stubs.json``` and replace ```500``` by ```200```.

## Steps to reproduce:
1. Start the Kogito App in dev mode: ```mvn "-Pbamoe-community" "-Pbamoe-persistence-postgresql" "-Pembedded-postgresql" "-Pbamoe-audit" "-Pdevelopment" clean quarkus:dev```. The runtime will use the last timestamped version of Kogito Community. In order to use BAMOE 9.2.1 build, replace ```"-Pbamoe-community"``` with ```"-Pbamoe-enterprise"```.

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
3. The logs will show how the Jobs Service retries the failed job 1 time, based on the following configuration:
```
# Delay between retries when a job execution fails, and it must be retried. Default: 1000
kogito.jobs-service.backoffRetryMillis=2000
# Maximum amount of time the jobs service will be retrying to get a successful execution for a job. Default: 60000
kogito.jobs-service.maxIntervalLimitToRetryMillis=0
```

## Current behavior
The Service Task is executed and, if an exception is thrown, it's retried 1 time. After that, the retry limit is exceeded, the process instance execution stops and remains in ```"ACTIVE"``` state.

## Expected behavior
The Service Task is executed and, if an exception is thrown, it's retried 
1. a **predefined number of times with a delay between retries predefined as well** as in jBPM v7. -> **This is addressed by the config parameters above**
2. If the **number of retries exceeds the limit** and the Service Task still fails, then the **process instance stops** the execution, the **instance status changes to ```"ERROR"```** -> **Question: How can we get the process instance in ERROR state after failed async executions?**
3. and the **job can be retriggered manually** once the underlying problem is solved. -> **Question: What is the proper way to retrigger this async node?**

Here's what I tried to address #3:

- Execute a `ProcessInstanceUpdateVariables` mutation:
```
mutation {
  ProcessInstanceUpdateVariables(
    id: "571397e1-74cd-4d2b-9a88-7b4c1b048eb4", 
    variables:"{\"throwException\":false}")
}
```

The idea is that this variable is used in the `DummyServiceTask.callDummy` method:
```
    public String callDummy(CandidateDataRestDTO candidateDataRestDto, Boolean throwException) {
      ...
        catch(Exception ex) {
            logger.info("*** caught exception " + ex.getMessage() + ", rethrow: " + throwException);
            if(throwException) {
                throw new WorkItemHandlerRuntimeException(ex);
            }
        }  
```

- Execute a `NodeInstanceTrigger` mutation:
```
mutation {
  NodeInstanceTrigger(
    id: "571397e1-74cd-4d2b-9a88-7b4c1b048eb4", 
    nodeId: "_EC746C99-E104-4015-83D9-03AA6DC69B5A"
  )
}
```

However, this results in the following failure:
```
2025-08-01 14:07:56,665 mweiler-ibm-com INFO  [org.kie.kogito.quarkus.processes.devservices.DevModeWorkflowLogger:57] (executor-thread-1) Triggered node 'Store Candidate Data' for process 'hiring' (35444c62-e514-4200-b041-577cffcb9330)
java.lang.RuntimeException: Unable to execute Assignment
2025-08-01 14:07:56,666 mweiler-ibm-com ERROR [org.jbpm.workflow.instance.impl.NodeInstanceImpl:274] (executor-thread-1) Error executing node instance '5a3c6b35-30dc-4875-be80-7d77ca9390d0' (node 'Store Candidate Data' id: '_EC746C99-E104-4015-83D9-03AA6DC69B5A') in process instance '35444c62-e514-4200-b041-577cffcb9330' (process: 'hiring') in a transactional environment (Wrapping)
	at org.jbpm.workflow.core.impl.NodeIoHelper.handleAssignment(NodeIoHelper.java:116)
	at org.jbpm.workflow.core.impl.NodeIoHelper.lambda$processDataAssociation$2(NodeIoHelper.java:99)
	at java.base/java.util.Arrays$ArrayList.forEach(Arrays.java:4305)
	at org.jbpm.workflow.core.impl.NodeIoHelper.processDataAssociation(NodeIoHelper.java:98)
	at org.jbpm.workflow.core.impl.NodeIoHelper.processDataAssociations(NodeIoHelper.java:73)
	at org.jbpm.workflow.core.impl.NodeIoHelper.processInputs(NodeIoHelper.java:52)
	at org.jbpm.workflow.core.impl.NodeIoHelper.processInputs(NodeIoHelper.java:151)
	at org.jbpm.workflow.core.impl.NodeIoHelper.processInputs(NodeIoHelper.java:140)
	at org.jbpm.workflow.core.impl.NodeIoHelper.processInputs(NodeIoHelper.java:156)
	at org.jbpm.workflow.instance.node.WorkItemNodeInstance.createWorkItem(WorkItemNodeInstance.java:248)
	at org.jbpm.workflow.instance.node.WorkItemNodeInstance.internalTrigger(WorkItemNodeInstance.java:145)
	at org.jbpm.workflow.instance.impl.NodeInstanceImpl.lambda$trigger$0(NodeInstanceImpl.java:250)
	at org.jbpm.workflow.instance.impl.NodeInstanceImpl.captureExecutionException(NodeInstanceImpl.java:260)
	at org.jbpm.workflow.instance.impl.NodeInstanceImpl.trigger(NodeInstanceImpl.java:250)
	at org.jbpm.workflow.instance.impl.NodeInstanceImpl.lambda$triggerNodeInstance$1(NodeInstanceImpl.java:480)
	at org.jbpm.workflow.instance.impl.NodeInstanceImpl.captureExecutionException(NodeInstanceImpl.java:260)
	at org.jbpm.workflow.instance.impl.NodeInstanceImpl.triggerNodeInstance(NodeInstanceImpl.java:480)
	at org.jbpm.workflow.instance.impl.NodeInstanceImpl.triggerNodeInstance(NodeInstanceImpl.java:464)
	at org.jbpm.workflow.core.node.AsyncEventNodeInstance.triggerCompleted(AsyncEventNodeInstance.java:195)
	at org.jbpm.workflow.core.node.AsyncEventNodeInstance$AsyncExternalEventListener.signalEvent(AsyncEventNodeInstance.java:70)
	at org.jbpm.workflow.instance.impl.WorkflowProcessInstanceImpl.signalEvent(WorkflowProcessInstanceImpl.java:721)
	at org.kie.kogito.process.impl.AbstractProcessInstance.lambda$send$5(AbstractProcessInstance.java:371)
	at org.kie.kogito.process.impl.AbstractProcessInstance.lambda$executeInWorkflowProcessInstance$24(AbstractProcessInstance.java:595)
	at org.kie.kogito.process.impl.lock.ProcessInstanceAtomicLockStrategy.executeOperation(ProcessInstanceAtomicLockStrategy.java:98)
	at org.kie.kogito.process.impl.AbstractProcessInstance.executeInWorkflowProcessInstance(AbstractProcessInstance.java:588)
	at org.kie.kogito.process.impl.AbstractProcessInstance.executeInWorkflowProcessInstanceWrite(AbstractProcessInstance.java:561)
	at org.kie.kogito.process.impl.AbstractProcessInstance.send(AbstractProcessInstance.java:367)
	at org.kie.kogito.services.jobs.impl.TriggerJobCommand.lambda$execute$0(TriggerJobCommand.java:62)
	at java.base/java.util.Optional.map(Optional.java:260)
	at org.kie.kogito.services.jobs.impl.TriggerJobCommand.lambda$execute$1(TriggerJobCommand.java:61)
	at org.kie.kogito.services.uow.UnitOfWorkExecutor.executeInUnitOfWork(UnitOfWorkExecutor.java:40)
	at org.kie.kogito.services.jobs.impl.TriggerJobCommand.execute(TriggerJobCommand.java:59)
	at org.kie.kogito.jobs.embedded.EmbeddedJobExecutor.lambda$processJobDescription$4(EmbeddedJobExecutor.java:127)
	at org.kie.kogito.services.uow.UnitOfWorkExecutor.executeInUnitOfWork(UnitOfWorkExecutor.java:40)
	at org.kie.kogito.jobs.embedded.EmbeddedJobExecutor.lambda$processJobDescription$5(EmbeddedJobExecutor.java:125)
	at io.smallrye.context.impl.wrappers.SlowContextualSupplier.get(SlowContextualSupplier.java:21)
	at io.smallrye.mutiny.operators.uni.builders.UniCreateFromItemSupplier.subscribe(UniCreateFromItemSupplier.java:28)
	at io.smallrye.mutiny.operators.AbstractUni.subscribe(AbstractUni.java:35)
	at io.smallrye.mutiny.operators.uni.UniOnFailureTransform.subscribe(UniOnFailureTransform.java:31)
	at io.smallrye.mutiny.operators.AbstractUni.subscribe(AbstractUni.java:35)
	at io.smallrye.mutiny.operators.uni.UniOnItemTransform.subscribe(UniOnItemTransform.java:22)
	at io.smallrye.mutiny.operators.AbstractUni.subscribe(AbstractUni.java:35)
	at io.smallrye.mutiny.operators.uni.UniOnItemTransformToUni.subscribe(UniOnItemTransformToUni.java:25)
	at io.smallrye.mutiny.operators.AbstractUni.subscribe(AbstractUni.java:35)
	at io.smallrye.mutiny.operators.uni.UniOnFailureFlatMap.subscribe(UniOnFailureFlatMap.java:31)
	at io.smallrye.mutiny.operators.AbstractUni.subscribe(AbstractUni.java:35)
	at io.smallrye.mutiny.operators.uni.UniRunSubscribeOn.lambda$subscribe$0(UniRunSubscribeOn.java:27)
	at io.quarkus.vertx.core.runtime.VertxCoreRecorder$15.runWith(VertxCoreRecorder.java:638)
	at org.jboss.threads.EnhancedQueueExecutor$Task.doRunWith(EnhancedQueueExecutor.java:2675)
	at org.jboss.threads.EnhancedQueueExecutor$Task.run(EnhancedQueueExecutor.java:2654)
	at org.jboss.threads.EnhancedQueueExecutor.runThreadBody(EnhancedQueueExecutor.java:1627)
	at org.jboss.threads.EnhancedQueueExecutor$ThreadBody.run(EnhancedQueueExecutor.java:1594)
	at org.jboss.threads.DelegatingRunnable.run(DelegatingRunnable.java:11)
	at org.jboss.threads.ThreadLocalResettingRunnable.run(ThreadLocalResettingRunnable.java:11)
	at io.netty.util.concurrent.FastThreadLocalRunnable.run(FastThreadLocalRunnable.java:30)
	at java.base/java.lang.Thread.run(Thread.java:1583)
Caused by: [Error: unresolvable property or identifier: candidateData]
[Near : {... r.HiringProcessDtoUtils.fromCandidateDataVariable(candidateData) ....}]
                                                                              ^
[Line: 1, Column: 67]

	at org.mvel2.PropertyAccessor.getBeanProperty(PropertyAccessor.java:682)
	at org.mvel2.PropertyAccessor.getNormal(PropertyAccessor.java:178)
	at org.mvel2.PropertyAccessor.get(PropertyAccessor.java:145)
	at org.mvel2.PropertyAccessor.get(PropertyAccessor.java:125)
	at org.mvel2.ast.ASTNode.getReducedValue(ASTNode.java:187)
	at org.mvel2.MVELInterpretedRuntime.parseAndExecuteInterpreted(MVELInterpretedRuntime.java:112)
	at org.mvel2.MVELInterpretedRuntime.parse(MVELInterpretedRuntime.java:58)
	at org.mvel2.MVEL.eval(MVEL.java:416)
	at org.mvel2.PropertyAccessor.getMethod(PropertyAccessor.java:877)
	at org.mvel2.PropertyAccessor.getNormal(PropertyAccessor.java:181)
	at org.mvel2.PropertyAccessor.get(PropertyAccessor.java:145)
	at org.mvel2.PropertyAccessor.get(PropertyAccessor.java:125)
	at org.mvel2.ast.ASTNode.getReducedValue(ASTNode.java:187)
	at org.mvel2.MVELInterpretedRuntime.parseAndExecuteInterpreted(MVELInterpretedRuntime.java:112)
	at org.mvel2.MVELInterpretedRuntime.parse(MVELInterpretedRuntime.java:58)
	at org.mvel2.MVEL.eval(MVEL.java:142)
	at org.drools.mvel.util.RawMVELEvaluator.eval(RawMVELEvaluator.java:41)
	at org.jbpm.workflow.core.impl.InputExpressionAssignment.evalInput(InputExpressionAssignment.java:71)
	at org.jbpm.workflow.core.impl.InputExpressionAssignment.execute(InputExpressionAssignment.java:60)
	at org.jbpm.workflow.core.impl.NodeIoHelper.handleAssignment(NodeIoHelper.java:114)
	... 55 more
```

Any idea how to properly trigger this async node?