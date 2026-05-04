# P1. Prompts for Apex

Apex is where AI-assisted Salesforce development has the highest payoff and the highest failure risk. A good Apex prompt produces clean, bulk-safe, secure code that follows your architecture. A bad Apex prompt produces code that compiles, passes tests, and fails in production under load.

This chapter covers the prompt patterns that work for Apex, with examples for the main class types: service classes, selector classes, trigger handlers, batch classes, queueables, and REST endpoints.

## The non-negotiable Apex constraints

Every Apex prompt must include these constraints. They are not stylistic. They are the difference between code that works and code that causes a governor limit cascade in production.

```
Must:
- Use selector classes for all SOQL (no raw SOQL in service classes)
- Use WITH USER_MODE for all DML operations
- Be bulk-safe: accept List<T> not T, handle empty lists
- Use Database.insert(records, false) for partial success where appropriate
- Follow your trigger framework (TriggerHandler base class pattern)
- Add @TestVisible for test seams only
- Add error handling that returns meaningful messages via Database.SaveResult

Must not:
- Put SOQL or DML inside any loop (for, while, do-while)
- Hardcode IDs (use Custom Metadata or selector queries instead)
- Use System.debug() in production code paths (remove before deploy)
- Expose @AuraEnabled methods without feature management or custom permission gates
- Log record IDs or field values in plain text

Governor limits to respect:
- 100 SOQL queries per transaction
- 150 DML statements per transaction
- 10,000ms CPU time per transaction
- 6MB heap per transaction
- 100 callouts per transaction
```

## Service class prompts

Service classes are the core of Salesforce business logic. AI writes decent service classes if you give it the right context and constraints.

**Prompt template:**
```
You are developer-AI. Write a service class method in a Salesforce project with:
- Selectors: <list selector names and what they query>
- Service layer pattern: <describe naming and structure>
- API version: <e.g. 59.0>

Task: <specific business logic description>

Context:
- This method is called from <trigger handler / LWC / REST endpoint / Flow>
- Must handle <bulk / single record / partial success> scenarios
- Record volumes: <expected count range>

Code must:
- Use <SelectorName> for all queries
- Use WITH USER_MODE for DML
- Handle empty input: return early if list is empty
- Bulk-safe: iterate over the input collection once
- Return Database.SaveResult[] for DML operations so callers can handle failures

Write the complete class with comments on lines that need customization.
```

**Example:**
```
Write AccountService.updateBillingAddress(List<Account> accounts, String newStreet, String newCity, String newState) in a project with:
- AccountSelector with findByIds(Set<Id> ids) returning List<Account>
- AccountTriggerHandler extending TriggerHandlerBase
- API version 59.0

Requirements:
1. Accepts List<Account> (bulk-safe for up to 200 records per transaction)
2. Uses AccountSelector.findByIds to get current account records
3. Updates billing address fields on each account
4. Uses Database.update(accounts, false) to allow partial success
5. Returns Database.SaveResult[] so the caller (TriggerHandler) can log failures
6. Handles the case where newStreet/newCity/newState are null or empty (skip update for that field)
7. Uses WITH USER_MODE for all DML

The method is called from AccountTriggerHandler.beforeUpdate().
```

## Selector class prompts

Selectors are the cleanest pattern for AI to generate because they are essentially parameterized queries with a defined interface. The main risk is AI generating raw SOQL instead of using the selector.

**Prompt template:**
```
You are developer-AI. Write a selector class for <ObjectName> in a project with:
- Selector base class: <describe, e.g. SObjectSelector from fflib selector pattern>
- Naming: <e.g. OpportunitySelector.cls>
- Query builder: <if you use a query builder like SOQL Builder or QueryUtil>

Selector methods to implement:
- findByIds(Set<Id> ids): returns List<SObject> filtered by Id IN :ids
- findBy<Field>(<Type> value): returns List<SObject> filtered by indexed field
- findAll(): returns all records (use sparingly, with LIMIT)

Constraints:
- All queries must use SELECT fields explicitly (no SELECT *)
- Use WITH USER_MODE for security context
- Add WITH SECURITY_ENFORCED clause where appropriate
- Add LIMIT 1000 or higher only where pagination is not needed
- Use ORDER BY on indexed fields when sort order matters

Write the complete selector class.
```

**Example:**
```
Write OpportunitySelector for a project using the fflib Selector pattern.
Selector base: SObjectSelector
Naming convention: OpportunitySelector.cls

Methods:
1. selectById(Set<Id> ids): SELECT Id, Name, StageName, Amount, CloseDate, AccountId, OwnerId FROM Opportunity WHERE Id IN :ids WITH USER_MODE
2. selectByAccountId(Set<Id> accountIds): SELECT Id, Name, StageName, Amount FROM Opportunity WHERE AccountId IN :accountIds WITH USER_MODE ORDER BY CloseDate DESC
3. selectClosedWonBetweenDates(Date startDate, Date endDate): SELECT Id, Name, Amount, CloseDate, Account.Name FROM Opportunity WHERE StageName = 'Closed Won' AND CloseDate >= :startDate AND CloseDate <= :endDate WITH USER_MODE

Additional requirements:
- No SELECT * (explicit fields only)
- Use Schema.SObjectType.Opportunity.fields.getMap() for field metadata if needed at runtime
- The selector must work in both synchronous and async (Queueable) contexts
```

## Trigger handler prompts

Trigger handlers have more structure than service classes. AI needs to know your trigger framework to generate correct code.

**Prompt template:**
```
You are developer-AI. Write a TriggerHandler class for <SObject> in a project with:
- Trigger framework: <TriggerHandlerBase / TriggerDispatcher / fflib_TriggerHandler>
- Handler naming: <e.g. AccountTriggerHandler.cls>
- Trigger operation types: <before insert, before update, after insert, etc.>

Task: Handle <specific business requirement> when <trigger event>.

Constraints:
- Extend the base class properly (override the relevant method)
- Do not put business logic in the trigger file itself (force-app/main/default/triggers/<Object>Trigger.trigger should only call the handler)
- Use the operand variable names from the base class: Trigger.new, Trigger.old, Trigger.oldMap, Trigger.newMap
- For before triggers: modify Trigger.new records directly, do not DML
- For after triggers: do not modify Trigger.new
- Use appropriate methods: beforeInsert(), beforeUpdate(), afterInsert(), afterUpdate()
- For update triggers, check for changes using Trigger.oldMap before running logic (prevent unnecessary updates)

Bulk handling:
- Code must handle Trigger.new with multiple records
- Do not assume Trigger.new has only one record
- Use maps to group records by Id or field value when processing in bulk

Write:
1. The trigger file (one line that calls the handler)
2. The trigger handler class with the relevant methods
```

**Example:**
```
Write AccountTriggerHandler.cls and the trigger file for a project using TriggerHandlerBase.
TriggerHandlerBase defines: beforeInsert(), beforeUpdate(), afterInsert(), afterUpdate()
Trigger context variables: Trigger.new, Trigger.oldMap, Trigger.newMap

Business requirement: When an Account's BillingAddress changes, create a Task to follow up with the account owner if the address modification is > 50 miles from the previous address.

Handler requirements:
1. beforeUpdate() - detect address changes using Trigger.oldMap
2. For each account with an address change, calculate distance (simplified: just flag the record)
3. Store the address change flag on the Account record (add a custom field Address_Changed__c = true)
4. Use Trigger.new to modify the records (beforeUpdate, no DML needed)
5. Bulk-safe: process all records in Trigger.new

Trigger file requirements:
- One line only: AccountTriggerHandler.run() called before all operations
- No business logic in the trigger itself
```

## Batch class prompts

Batch Apex is one of the most complex Apex constructs. AI often generates batch classes that compile but fail in ways that are hard to diagnose. The key constraints are in the `execute()` method (no SOQL outside the query locator), the `finish()` method (chain behavior), and the scope sizing.

**Prompt template:**
```
You are developer-AI. Write a Batchable class for <use case> in a project with:
- Batch framework: <describe any custom batch base or use Database.Batchable>
- Scope size: <default 200 or custom>
- Chain behavior: <if batch chains to another batch, describe the pattern>

Class: <Name>Batch implements Database.Batchable<SObject>, Database.AllowsCallouts

Task: <describe what the batch processes and why>

Constraints:
- start(): use Database.getQueryLocator with a selective SOQL query. No aggregate queries, no joins that return > scope size
- execute(): receives List<sObject> scope (or a custom type). Process one scope at a time. No SOQL inside execute() except for query locators from start()
- finish(): handle partial success, chain to another Queueable or Batch if needed, log results to a custom Log__c object
- Must implement Database.AllowsCallouts if the batch makes HTTP callouts
- Must NOT call any @future methods from execute() (use Queueable instead)
- Keep state in instance variables (bulk-safe because each execution context is independent)

Governor limits specific to batching:
- Batch execute runs in its own governor limit context (100 SOQL limit still applies per execute)
- Chain depth: max 5 chained jobs (Batch -> Queueable -> Batch -> Queueable -> Batch is the limit)
- If this batch runs more than 5 times per day, use Scheduled Apex instead of batch chaining

Write the complete batch class with the interface methods.
```

**Example:**
```
Write OpportunityBatch for a project using Database.Batchable<sObject>.
Scope size: 200 records per execute() invocation.

Purpose: Daily batch to sync Closed Won opportunities to an external ERP system.
ERP integration: ErpIntegrationService.notifyOpportunityWon(opportunityIds) makes HTTP callouts.

Requirements:
1. start(): QueryLocator returns all Opportunities where StageName = 'Closed Won' AND Synced_To_ERP__c = false AND CreatedDate = LAST_N_DAYS:1
2. execute(): For each scope of up to 200 Opportunities, call ErpIntegrationService.notifyOpportunityWon with the Opportunity IDs. On success, set Synced_To_ERP__c = true. Use partial success: if 10 records succeed and 5 fail, commit the 10 successes.
3. finish(): If any records failed, enqueue a Queueable to retry failed records. Log the batch run to Batch_Log__c with record counts.
4. Database.AllowsCallouts: the batch can make HTTP callouts

Implementation notes:
- ErpIntegrationService.notifyOpportunityWon returns ErpSyncResult with success boolean and message
- On partial success, use Database.update(scopedRecords, false) to save successes
- Track failed record IDs to pass to the retry Queueable
```

## Queueable prompts

Queueables are simpler than batch classes but have their own pitfalls: the 50 Queueable limit per transaction, the fact that they cannot be chained from within the same transaction, and the lack of retry semantics.

**Prompt template:**
```
You are developer-AI. Write a Queueable class for <use case>.

Class: <Name>Queueable implements Queueable, Database.AllowsCallouts

Task: <describe what the Queueable does and when it is enqueued>

Constraints:
- Constructor must accept parameters needed for execute()
- If making callouts, implement Database.AllowsCallouts interface
- Enqueue from: trigger (use Database.enqueueJob), batch finish(), another Queueable finish(), or scheduled apex
- NEVER enqueue a Queueable from within a Queueable execute() in the same transaction (you will hit the chain depth limit). Use Database.update to persist data, then enqueue from trigger or scheduled apex instead.
- For retry logic: include retryCount in constructor, limit to 3 retries, use exponential backoff via Custom Metadata
- Make the class test-visible (@TestVisible) for test seams
- For idempotency: store the operation in a custom object with an External_Id__c or unique key to prevent duplicate processing

Write the complete Queueable class with constructor, execute(), and any helper methods.
```

**Example:**
```
Write ErpNotificationQueueable for a project where OpportunityService needs to notify an ERP system asynchronously.

Context:
- Enqueued from OpportunityService.handleClosedWon() after DML
- Uses Named Credentials: ErpIntegrationNamedCredential
- Callout to: POST https://erp.example.com/api/won-opportunities
- Payload: JSON with Opportunity Id, Name, Amount, CloseDate, Account Name

Queueable requirements:
1. Constructor takes List<Id> opportunityIds and Integer retryCount (default 0)
2. execute(): queries Opportunity records, builds JSON payload, makes HTTP callout
3. On HTTP 200: log success to Erp_Notification_Log__c with status 'Sent'
4. On HTTP 503 (retryable): if retryCount < 3, enqueue a new ErpNotificationQueueable with retryCount + 1
5. On other errors: log failure to Erp_Notification_Log__c with status 'Failed' and error message
6. Uses Database.AllowsCallouts interface
7. Idempotency: generate an Idempotency_Key__c per Opportunity based on Opportunity.Id + LastModifiedDate timestamp

Named Credentials: ErpIntegrationNamedCredential (credentials stored in Salesforce, not hardcoded)
```

## REST endpoint prompts

REST endpoints are the most exposed Apex code in a Salesforce org. Every `@AuraEnabled` method or `@RestResource` method is an attack surface. AI needs strong constraints when generating these.

**Prompt template:**
```
You are developer-AI. Write a REST endpoint class for <resource> in a project with:
- REST framework: <describe custom base or pure @RestResource>
- Auth: <what authentication mechanism: OAuth, session ID, custom permission>
- Naming: <URL mapping, e.g. /services/apexrest/v1/accounts/*>

Class: <Name>Resource

Constraints:
- Every public method must have explicit authorization check before executing
- Use FeatureManagement.checkPermission('CustomPermissionName') for custom permissions
- Use Schema.sObjectType.Account.isAccessible() for FLS checks on read
- Use Schema.sObjectType.Account.isCreateable() for FLS checks on write
- Strip inaccessible fields: Security.stripInaccessible(AccessType.READABLE, records) before returning to client
- Never return raw SOQL results (wrap in DTO classes)
- Validate all input parameters before processing
- Use LoggingUtil for structured logging, not System.debug()
- Return proper HTTP status codes: 200 for success, 400 for bad request, 401 for unauthorized, 500 for server error

DTO pattern: Define inner classes or separate DTO classes that shape the response. Never expose sObject records directly.

Write the complete REST resource class.
```

**Example:**
```
Write AccountResource@RestResource for a project using pure @RestResource (no custom framework).
URL mapping: /services/apexrest/v1/accounts/*

Purpose: Expose Account read/write for an external integration partner.

Security requirements:
1. Require custom permission: 'AccountIntegrationAccess'
2. Use Schema.sObjectType.Account.isAccessible() before SELECT
3. Use Schema.sObjectType.Account.isUpdateable() before UPDATE
4. Strip inaccessible fields from response: Security.stripInaccessible(AccessType.READABLE, accounts)
5. Validate that input JSON contains only whitelisted fields (reject unexpected keys)
6. Log all incoming requests with correlation ID (generate UUID if not provided)

Endpoints to implement:
- GET /v1/accounts/{id}: return Account by Id (use selector, check FLS, strip inaccessible)
- PATCH /v1/accounts/{id}: update Account fields from JSON body (validate, update with WITH USER_MODE)
- POST /v1/accounts: create Account from JSON body (validate, insert with WITH USER_MODE)

Response format: { "success": true, "data": {...}, "correlationId": "uuid" }
Error format: { "success": false, "error": { "code": "...", "message": "..." }, "correlationId": "uuid" }
```

## What this chapter covered

- The non-negotiable Apex constraints that every prompt must include
- Prompt templates for service classes, selector classes, trigger handlers, batch classes, queueables, and REST endpoints
- How to specify bulk safety, FLS enforcement, and partial success handling in prompts
- Why each class type needs its own specific constraints

## References

- [Apex Bulk Best Practices](https://developer.salesforce.com/docs/atlas.en-us.226.0/apex_code犬.content/apex_code_bulk_best_practices.htm)
- [Apex Governor Limits](https://developer.salesforce.com/docs/atlas.en-us.226.0/apex_code犬.content/apex_code_execution_governor.htm)
- [Selector Pattern](https://github.com/apex-enterprise-patterns/fflib-apex-common)
- [Apex REST Classes](https://developer.salesforce.com/docs/atlas.en-us.226.0/apex_code犬.content/apex_code_rest_code.htm)