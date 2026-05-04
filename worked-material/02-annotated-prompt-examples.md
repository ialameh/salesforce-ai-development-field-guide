# WM2. Annotated Prompt Examples by Persona

This chapter contains annotated examples of prompts for each role: architect-AI, developer-AI, reviewer-AI, and security-AI. Each example shows the actual prompt, explains why each part is structured that way, and shows what the output looked like.

## architect-AI Example

### The prompt

```
You are architect-AI for a Salesforce project at <ORG>.

Project context:
- Org: Enterprise edition with 2 million records
- Current architecture: selector/service/domain layers, TriggerHandlerBase framework
- Existing: OpportunitySelector, AccountSelector, ContactSelector, ErpIntegrationService (HTTP callout stub)
- Constraint: Must use scratch orgs only for AI-assisted development

Decision: Design the integration architecture for syncing Closed Won Opportunities to an external SAP system.

Options considered:
1. Queueable-based: OpportunityTriggerHandler.afterUpdate enqueues ErpSyncQueueable for each closed won Opportunity
2. Batch-based: Scheduled Apex runs nightly, queries closed won Opportunities, uses ErpSyncBatch with ERP callouts via Queueable chaining
3. Platform Event-based: Publish OpportunityClosedWon__e platform event from trigger, subscriber processes via Platform Event-driven Flow or Apex

Constraints to satisfy:
- SAP has rate limit: 100 calls per minute
- Must be idempotent: running sync twice for the same Opportunity must not cause duplicate SAP records
- Must handle partial success: if SAP rejects 5 of 100 records, the other 95 succeed
- Retry on SAP downtime: exponential backoff, max 3 retries
- Monitoring: must be observable (know when sync succeeds, fails, or retries)

Evaluation criteria:
- Reliability: what happens if SAP is down for 2 hours during the sync window?
- Scalability: what happens at 10,000 closed won Opportunities per night?
- Maintainability: can a new developer understand and modify the sync logic?
- Cost: what is the async execution cost (Queueable vs Batch vs Platform Events)?

Return format: Decision with rationale, component inventory, data flow, failure modes, open questions.
```

### Annotation

The prompt is structured for the architect-AI role because:
- It includes project context (org type, architecture, existing components)
- It defines the decision clearly (integration architecture for ERP sync)
- It lists multiple options with names, not just "which is best"
- It gives specific constraints (rate limit, idempotency, partial success) that force real trade-offs
- It weights evaluation criteria (reliability > scalability > maintainability > cost)
- It specifies the return format expected

What this prompt prevents: The AI does not default to the simplest option (Queueable from trigger) without considering the rate limit, partial success, and retry requirements.

### Expected output

```
Decision: Use Batch-based architecture with Queueable fan-out for ERP callouts.

Rationale:
Option 1 (Queueable from trigger) fails the idempotency requirement under retry. If the trigger fires twice for the same Opportunity, two Queueables are enqueued. Option 2 (Batch) handles this better because the scheduled query can filter by sync status.

Option 3 (Platform Events) adds complexity for the retry semantics we need. Platform Events have at-least-once delivery, which helps idempotency, but the subscriber pattern makes partial success harder to track.

Batch with Queueable fan-out: The Scheduled Apex runs at midnight, queries Opportunities where StageName = 'Closed Won' AND Synced_To_ERP__c = false. The batch scopes to 50 records. For each scope, it enqueues an ErpSyncQueueable with the opportunity IDs. The Queueable makes the HTTP callout to SAP. On success, it updates Synced_To_ERP__c = true. On 503 or timeout, it retries with exponential backoff (max 3). On other errors, it marks the record as failed and alerts via EmailAlertQueueable.

Component inventory:
- OpportunitySyncScheduled: cron-based scheduled apex, runs at midnight
- OpportunitySyncBatch: processes Opportunities in batches of 50, enqueues Queueables
- ErpSyncQueueable: makes HTTP callout, handles retry, updates sync status
- ErpNotification__c: custom object tracking sync attempts (Opportunity__c, Status__c, RetryCount__c, Error__c)

Failure modes:
- SAP down: Queueables retry with exponential backoff. After 3 retries, status = 'Failed', alert sent.
- Queueable chain depth: batch enqueues Queueables (1 level). Queueable does not enqueue from itself (prevents chain depth violation).
- Governor limits: 50 records per batch scope prevents SOQL overflow in execute(). Queueable allows callouts.

Open questions:
- What is the SAP retry endpoint for idempotency key validation?
- Should failed syncs automatically retry on the next scheduled run, or require manual intervention?
```

## developer-AI Example

### The prompt

```
You are developer-AI for a Salesforce project at <ORG>.

Project structure:
- Selectors: OpportunitySelector (findByIds, findByStage)
- Services: OpportunityService
- Domain: OpportunityTriggerHandler extending TriggerHandlerBase
- API version: 59.0

Task: Write OpportunityService.handleClosedWon(List<Opportunity> opportunities) that:
1. Is called by OpportunityTriggerHandler.afterUpdate when stage changes to 'Closed Won'
2. Must handle bulk triggers: up to 200 Opportunity records in a single transaction
3. Uses OpportunitySelector.findByIds(Set<Id> ids) to get related Opportunity records
4. Queries related OpportunityLineItem records for each Opportunity using OpportunitySelector (do not write raw SOQL in this method)
5. Calls ErpIntegrationService.notifyWonOpportunity(List<OpportunityLineItem> items) to notify the ERP system
6. Uses Queueable to handle the ERP callout in a separate transaction context
7. Must be idempotent: if called twice with the same records, produces no duplicate ERP notifications
8. Uses Database.insert(notificationRecords, false) for partial success on notification persistence

Code constraints:
- Use OpportunitySelector for all queries (no raw SOQL in service)
- Use WITH USER_MODE for all DML
- Handle empty list: return early if opportunities is empty or null
- All DML outside loops
- No System.debug() in production code

Write the complete method with comments explaining the business logic.
```

### Annotation

The prompt uses the developer-AI role because the architect-AI has already defined the approach (batch/queueable for ERP sync), and this is implementation work within those boundaries.

Key specificity that prevents AI failures:
- "Uses OpportunitySelector" (prevents raw SOQL in service)
- "up to 200 Opportunity records" (forces bulk thinking)
- "idempotent: if called twice with the same records" (forces idempotency key logic)
- "Uses Queueable" (specifies async architecture)
- "Uses Database.insert(notificationRecords, false)" (specifies partial success handling)

What this prevents: The single-record assumption, the SOQL-in-loop failure, and the fire-and-forget async pattern that AI defaults to.

## reviewer-AI Example

### The prompt

```
You are reviewer-AI. Use the sf-code-review skill (13 lenses) to review this Apex code.

Files to review:
- force-app/main/default/classes/OpportunityService.cls (the handleClosedWon method you just generated)
- force-app/main/default/classes/ErpSyncQueueable.cls
- force-app/main/default/classes/OpportunityTriggerHandler.cls

Focus on:
1. Bulk patterns: does the code handle List<Opportunity>, not single Opportunity?
2. Governor limits: 100 SOQL per transaction, heap, CPU - what is the worst-case SOQL count?
3. Security: WITH USER_MODE, FLS enforcement, no hardcoded credentials, no PII in logs
4. Async reliability: retry logic, idempotency key, failure logging
5. Test coverage: are happy path, failure path, and bulk (200 records) cases tested?

For each finding:
- File: the file and line number (if available)
- Class/Method: which class and method contains the issue
- Finding: what specifically is wrong
- Risk: what happens in production if this is not fixed
- Severity: Critical / High / Medium / Low
- Fix: specific code change to fix the issue
- Label: Confirmed / Likely / Possible / Cannot verify

Return: Full structured report. Overall verdict: Production ready / Production ready after fixes / Risky / Not deployable.
```

### Annotation

The reviewer-AI prompt is structured to use the sf-code-review skill criteria (13 lenses). It identifies the specific files to review (the output from the developer-AI phase), focuses on the highest-risk areas (bulk patterns, governor limits, security), and specifies the output format (table with severity and fix).

What this catches: The single-record assumption, the missing idempotency key, the hardcoded credentials, the SOQL in loops that developer-AI might have missed.

## security-AI Example

### The prompt

```
You are security-AI. Review this Apex controller for a Lightning component that reads and updates Account records.

Review scope:
- force-app/main/default/classes/AccountController.cls
- force-app/main/default/classes/AccountService.cls

Security checklist:
1. WITH USER_MODE: is DML using WITH USER_MODE?
2. FLS enforcement: are fields checked for accessibility before reads (Schema.sObjectType.Account.isAccessible()) and creatability/updatability before writes (isCreateable, isUpdateable)?
3. Sharing model: is the class declared correctly (with sharing / without sharing / inherited sharing)?
4. @AuraEnabled authorization: do mutating methods (insert, update, delete) have FeatureManagement.checkPermission() or custom permission gates?
5. SOQL injection: are queries using static strings or dynamic concatenation with user input?
6. DML injection: is DML using robust type checking or string concatenation?
7. Named Credentials: are external callouts using Named Credentials by name, not hardcoded credentials?
8. Sensitive data in logs: do System.debug or custom Log__c records contain field values (names, emails, phone numbers)?
9. Security.stripInaccessible: are records stripped before returning to the LWC?

For each finding:
- Location: file, class, method
- Finding: what specifically is wrong
- Severity: Critical / High / Medium / Low
- Fix: specific code change to fix
- Compliance implication: does this fail Security Review / HIPAA / SOC2?

Return: Security posture (Secure / Needs fixes / Critical gap), findings table, compliance implications.
```

### Annotation

The security-AI prompt uses the security-review lens with specific Salesforce security requirements: WITH USER_MODE, FLS checks, sharing model, @AuraEnabled authorization, SOQL injection, Named Credentials, and PII in logs.

What this catches: Authorization gaps on @AuraEnabled methods (the most common security failure in AI-generated Apex), hardcoded credentials, and PII in logs.

## What this chapter covered

- architect-AI prompt with annotation explaining why each section is structured that way
- developer-AI prompt with annotation explaining how the constraints prevent AI failures
- reviewer-AI prompt using the sf-code-review skill for multi-lens review
- security-AI prompt with the security review checklist for Salesforce

## References

- [Apex Security Best Practices](https://developer.salesforce.com/docs/atlas.en-us.226.0/apex_code犬.content/apex_code_security_best_practices.htm)
- [SOQL Injection Prevention](https://developer.salesforce.com/docs/atlas.en-us.226.0/apex_code犬.content/apex_code_security_tips.htm)
- [Security Review Requirements](https://security.secure.force.com/securityguide/InputValidation)