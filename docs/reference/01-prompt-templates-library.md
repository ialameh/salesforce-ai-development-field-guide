# R1. Prompt Templates Library

This chapter contains reusable prompt templates for the most common Salesforce development tasks. Each template is ready to use with minor customization for your specific project context. Templates are organized by use case.

## How to use these templates

1. Copy the template for your use case
2. Replace `<placeholders>` with your project-specific values
3. Add any additional constraints specific to your org
4. Pass to developer-AI or architect-AI as appropriate

## Apex Class Generation

### Service Class Template

```
You are developer-AI working in a Salesforce project.

Project structure:
- Selector layer: <list selector class names>
- Service layer pattern: <describe naming, e.g., <Object>Service.cls>
- Trigger framework: <TriggerHandlerBase / custom framework>

Task: Write <ObjectName>Service.<methodName> in a project with:
- Selectors: <list selector names and what they query>
- API version: <e.g. 59.0>

Requirements:
1. <specific business logic description>
2. Accepts: <List<T> or single T>
3. Returns: <void / Database.SaveResult[] / specific type>
4. Uses <SelectorName> for all SOQL queries
5. Uses WITH USER_MODE for all DML
6. Bulk-safe: handles up to 200 records per transaction
7. Partial success: <Database.insert(records, false) / update(records, false) where appropriate>
8. Error handling: <what to return or throw on failure>
9. <any additional requirements>

Non-negotiable constraints:
- No SOQL or DML inside loops
- No hardcoded IDs
- No System.debug() in production code
- Use selector layer for all queries

Write the complete service class method.
```

### Selector Class Template

```
You are developer-AI. Write a selector class for <ObjectName> using the <fflib / custom> selector pattern.

Selector: <ObjectName>Selector.cls
Methods to implement:
1. selectById(Set<Id> ids): SELECT <field list> FROM <Object> WHERE Id IN :ids WITH USER_MODE
2. selectBy<FieldName>(<Type> value): SELECT <field list> FROM <Object> WHERE <Field> = :value WITH USER_MODE
3. <additional methods as needed>

Constraints:
- Explicit field list (no SELECT *)
- WITH USER_MODE for security
- Add LIMIT only where pagination is not needed
- Use ORDER BY on indexed fields when sort matters

Selector base class: <SObjectSelector / custom base>
Query builder: <if used in this project>

Write the complete selector class.
```

### Trigger Handler Template

```
You are developer-AI. Write a TriggerHandler class for <SObjectName>.

Project trigger framework: <TriggerHandlerBase / fflib_TriggerHandler / custom>
Handler: <SObjectName>TriggerHandler.cls

Business requirement: <describe what the trigger should do>

Trigger event: <before insert / before update / after insert / after update / etc.>

Requirements:
1. Extend <base class> properly
2. Override relevant method: <e.g., beforeUpdate()>
3. Use Trigger.new, Trigger.oldMap as appropriate
4. Bulk-safe: process all records in Trigger.new
5. Before triggers: modify Trigger.new directly, no DML
6. After triggers: do not modify Trigger.new
7. For update triggers: check Trigger.oldMap to detect changes before running logic

Code must:
- Handle Trigger.new with multiple records
- Use maps for grouping records when processing in bulk
- Be bulk-safe for up to 200 records per execution context

Write:
1. The trigger file (one line, calls handler.run())
2. The trigger handler class with relevant method overrides

Trigger file location: force-app/main/default/triggers/<SObjectName>Trigger.trigger
Handler location: force-app/main/default/classes/<SObjectName>TriggerHandler.cls
```

### Batch Class Template

```
You are developer-AI. Write a Batchable class for <use case>.

Class: <Name>Batch implements Database.Batchable<sObject>, Database.AllowsCallouts (if needed)

Purpose: <what the batch processes>

Requirements:
1. start(): Use Database.getQueryLocator with a selective SOQL query. No aggregates, no joins that return > scope size.
2. execute(): Process List<sObject> scope. No SOQL inside execute() (use query locator from start). Call external systems via Queueable if needed.
3. finish(): Handle partial success, chain if needed, log results.

Scope size: <default 200 or custom>

Governor limits:
- Batch execute has its own 100 SOQL limit per invocation
- Chain depth: max 5 chained jobs
- If > 5 chains needed, use Scheduled Apex instead

Interface requirements:
- Database.Batchable<sObject>
- Database.AllowsCallouts (if HTTP callouts needed)
- Instance variables for state (bulk-safe, each execute runs independently)

Write the complete batch class.
```

### Queueable Class Template

```
You are developer-AI. Write a Queueable class for <use case>.

Class: <Name>Queueable implements Queueable, Database.AllowsCallouts (if needed)

Task: <describe what the Queueable does>

Requirements:
1. Constructor accepts parameters needed for execute() (List<Id>, retryCount, etc.)
2. execute(): implements the business logic
3. On success: log to ErpNotification__c or custom log
4. On retryable error (HTTP 503, timeout): if retryCount < 3, enqueue new Queueable with retryCount + 1
5. On non-retryable error: log failure, do not retry

Interface: Queueable, Database.AllowsCallouts (if callouts needed)

Idempotency: Generate External_Id__c or Idempotency_Key__c to prevent duplicate processing.
Persistence: Store operation in custom object before callout to enable retry.

Constraints:
- Never enqueue from within a Queueable execute() in the same transaction (chain limit)
- Use Named Credentials for external callouts (do not hardcode credentials)

Write the complete Queueable class.
```

### REST Endpoint Template

```
You are developer-AI. Write a REST resource class for <resource>.

Class: <Name>Resource @RestResource(urlMapping='/services/apexrest/v1/<resource>/*')

Authorization: Require custom permission '<PermissionName>' via FeatureManagement.checkPermission()

Endpoints:
- GET /v1/<resource>/{id}: retrieve record
- PATCH /v1/<resource>/{id}: update record
- POST /v1/<resource>: create record

Security requirements:
1. FeatureManagement.checkPermission('CustomPermissionName') on all methods
2. Schema.sObjectType.<Object>.isAccessible() for FLS on read
3. Schema.sObjectType.<Object>.isCreateable()/isUpdateable() for FLS on write
4. Security.stripInaccessible(AccessType.READABLE, records) before returning
5. Validate input JSON (reject unexpected fields)
6. Log requests with correlation ID

Response format:
{ "success": true, "data": {...}, "correlationId": "uuid" }
Error format:
{ "success": false, "error": { "code": "...", "message": "..." }, "correlationId": "uuid" }

DTO pattern: Use inner classes or separate DTO classes. Never expose sObject records directly.

Write the complete REST resource class.
```

## LWC Component Templates

### Wire Adapter Template

```
You are developer-AI. Write an LWC component JS file that uses @wire to fetch data.

Component: <componentName>.js

Apex method: @AuraEnabled(cacheable=true) public static <ReturnType> <methodName>()
Import: import <methodName> from '@salesforce/apex/<Controller>.<methodName>'

Requirements:
1. Import { LightningElement } from 'lwc'
2. @track for reactive state: loading, error, data
3. Wire the Apex method: @wire(<methodName>, { <param>: '$<property> })
4. Handle { data, error } from wire response
5. Show loading spinner when data is undefined
6. Show error message if error is present
7. Render data in template using <template if:true={wiredData}>

Template requirements:
- <template if:true={loading}>: loading spinner
- <template if:true={error}>: error message with error.body.message
- <template if:true={data}>: render data

Write the complete JS file.
```

### Imperative Apex Template

```
You are developer-AI. Write an LWC component JS file that uses imperative Apex calls.

Component: <componentName>.js

Apex method: @AuraEnabled public static <ReturnType> <methodName>(<Type> param)
Import: import <methodName> from '@salesforce/apex/<Controller>.<methodName>'

Requirements:
1. @track for reactive state: loading, error, data, <any other state>
2. Call Apex imperatively with .then()/.catch() or async/await
3. Track loading state manually
4. Handle errors with .catch() and display user-friendly error
5. On success: update @track properties

Common pattern:
fetchData() {
    this.loading = true;
    this.error = null;
    <methodName>({ <param>: this.<property> })
        .then(data => {
            this.data = data;
            this.loading = false;
        })
        .catch(error => {
            this.error = error.body.message;
            this.loading = false;
        });
}

Debounce if needed: Use setTimeout to debounce rapid calls.

Write the complete JS file.
```

### Parent-Child Component Template

```
You are developer-AI. Write parent and child LWC component JS files.

Child component: <childName>.js
- @api record: receives a single record object
- @api <additional public properties>
- Fires custom event: this.dispatchEvent(new CustomEvent('<kebab-event-name>', { detail: { <data> } }))
- Custom event name: kebab-case (e.g., 'recordchange' not 'recordChange')

Parent component: <parentName>.js
- Parent passes data to child via @api property binding
- Parent listens for event: on<kebab-event-name>={handler}
- Handler receives event: handleEvent(event) { const data = event.detail; }

Template communication:
- Child to parent: custom events with detail payload
- Parent to child: @api property binding
- Cross-component: Lightning Message Service (LMS) for flexipage communication

Write both JS files.
```

## Test Class Templates

### Service Method Test Template

```
You are developer-AI. Write test methods for <ClassName>.<methodName>.

Class under test: <ClassName>
Method: <methodName>(<params>)

Test methods required:

1. test<MethodName>HappyPath:
   - Setup: create <Object> test records
   - Execute: call <ClassName>.<methodName>(<test data>)
   - Assert: verify <specific outcome> using System.assertEquals / System.assertNotEquals
   - Assert on outcomes, not implementation details

2. test<MethodName>EmptyInput:
   - Execute: call method with empty list
   - Assert: verify early return or graceful handling (no exceptions)

3. test<MethodName>PartialSuccess:
   - Setup: mock partial DML failure
   - Execute: call method
   - Assert: verify success records processed, failed records handled gracefully

4. test<MethodName>Bulk200Records:
   - Setup: create 200 <Object> records
   - Execute: call method with 200-record list
   - Assert: all processed, no governor limit failures

5. test<MethodName>UserWithoutFLS:
   - Setup: create user without FLS access to a field
   - Execute: System.runAs(user) { call method }
   - Assert: code handles FLS gracefully

6. test<MethodName>ExternalCalloutFailure:
   - Setup: mock HttpCalloutMock returning error
   - Execute: call method
   - Assert: exception caught and handled appropriately

Mock strategy: Use HttpCalloutMock for callouts. Use Test.startTest/Test.stopTest for async.

Data creation: Use @TestVisible factory methods, not hardcoded IDs.

Write the complete test class.
```

## Flow Design Template

```
You are architect-AI. Design a Flow for <business requirement>.

Entry criteria: <record change / screen load / schedule / platform event>
Exit criteria: <success outcome / failure outcome / next step>

Data volume: <expected records per run: 10 / 100 / 10000>

Flow elements (in order):
1. Entry: <trigger type> on <Object>
2. Decision: check <field> before running logic (prevent recursion or unnecessary processing)
3. Get Records: fetch related records (note: Get Records respects sharing but NOT FLS)
4. Assignment: <assign values / aggregate data>
5. Loop: <iterate over collection> (note: do not put Assignment inside loop)
6. Decision: <loop boundary checks>
7. Create/Update Records: <modify records>
8. <any Flow-specific elements>

What belongs in Flow:
- Simple data create/update
- Screen flows
- Decision trees on existing data
- Scheduled batch actions (with awareness of volume)

What belongs in Apex:
- HTTP callouts
- Complex multi-object transactions
- Recursive logic or loops over large data sets
- Custom exception handling or retry logic

Flow constraints:
- Do not put complex logic in Flow formulas
- Screen flow: each screen element adds ~1-2 seconds
- If > 5 screens, split into multiple flows
- Recursive Flow: add Decision to check if update is needed before updating

Return: Flow diagram description in text format with element names and connections.
```

## References

- [Anthropic prompt engineering](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering)
- [Salesforce Apex patterns](https://developer.salesforce.com/docs/atlas.en-us.226.0/apex_code犬.content/apex_code_apex_patterns.htm)
- [LWC Documentation](https://developer.salesforce.com/docs/component-library/documentation/en/lwc)