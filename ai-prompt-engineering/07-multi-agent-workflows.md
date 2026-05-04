# P7. Multi-Agent Workflows

Single-agent AI (one AI tool, one prompt, one response) handles straightforward tasks well. Complex Salesforce features require multiple agents working in sequence or in parallel. This chapter covers how to design multi-agent workflows for Salesforce development using the four-role framework from F2.

## When to use multiple agents

Use multi-agent workflows when:
- A feature has distinct phases (design, implement, test, review) that require different reasoning
- The codebase spans multiple classes and the AI needs to reason about each with different context
- Security review is needed alongside implementation review
- A planning agent needs to decompose a large feature before implementation begins
- Testing requires specialized knowledge that the developer-AI does not have

Single-agent is sufficient for: simple Apex class writing, straightforward LWC components, single-class refactors, test method generation for existing classes.

## The four-agent workflow

The standard multi-agent workflow for a Salesforce feature follows this sequence:

```
architect-AI -> developer-AI -> reviewer-AI -> security-AI
```

### Phase 1: architect-AI (design)

The architect agent decomposes the feature, defines boundaries, and sets constraints. Output is an architectural decision document, not code.

Example task: Design the integration architecture for syncing Opportunities to an ERP system.

```
You are architect-AI. Design the integration architecture for syncing Closed Won Opportunities to an external ERP system.

Context:
- Org: Enterprise edition, 2 million records
- ERP: SAP with rate limit of 100 calls per minute
- Existing: OpportunitySelector, TriggerHandler framework, ErpIntegrationService (stub)
- Deployment target: scratch orgs and sandbox

Design requirements:
1. Entry point: OpportunityTriggerHandler.afterUpdate (stage = Closed Won)
2. How to handle bulk (up to 200 records per trigger execution)
3. ERP rate limiting (100 calls/minute) - how to queue and throttle
4. Idempotency: safe to run twice, ERP receives each opportunity only once
5. Retry on ERP failure: exponential backoff, max 3 retries
6. Persistence layer: how to audit what was sent
7. Failure visibility: how to alert if sync fails

Constraints:
- No @future from triggers (use Queueable instead)
- Callout must be async (Queueable or Batch)
- Must handle partial success (some records succeed, some fail)

Output: Architecture decision with rationale, component inventory, data flow, failure modes.
```

### Phase 2: developer-AI (implement)

The developer agent takes the architectural decisions and produces deployable code. It works within the boundaries set by architect-AI.

```
You are developer-AI. Implement the ERP sync architecture defined by architect-AI.

Architecture decisions from architect-AI:
- Entry: OpportunityTriggerHandler.afterUpdate -> detect Closed Won transition
- Persistence: ErpNotification__c custom object with Opportunity__c, Status__c, Idempotency_Key__c, RetryCount__c
- Async: ErpSyncQueueable (implements Queueable, Database.AllowsCallouts)
- Rate limiting: Custom Metadata ErpConfig__mdt with Rate_Limit__c, Window_Size__c fields
- Retry: Exponential backoff via Queueable constructor with retryCount parameter

Implementation constraints:
- Use OpportunitySelector (already exists) for all Opportunity queries
- Use ErpIntegrationService for the HTTP callout (do not write raw HTTP code)
- Idempotency key: Opportunity.Id + '-' + LastModifiedDate.getTime()
- Persist notification to ErpNotification__c before callout
- On HTTP 200: update ErpNotification__c.Status__c = 'Sent'
- On HTTP 503 or timeout: if retryCount < 3, enqueue new Queueable with retryCount + 1
- On other errors: update Status__c = 'Failed', do not retry

Write:
1. ErpSyncQueueable.cls (Queueable class)
2. OpportunityTriggerHandler.handleClosedWon() method additions
3. ErpNotificationSelector.cls (to query notification status)
4. Unit tests for ErpSyncQueueable (mock ErpIntegrationService)

Follow the project naming conventions and selector/service patterns.
```

### Phase 3: reviewer-AI (review)

The reviewer agent examines the code across all 13 lenses. This catches governor limit failures, bulk pattern violations, security gaps, and metadata inconsistencies that a developer-AI working on one class at a time will miss.

```
You are reviewer-AI. Use the sf-code-review skill (13 lenses) to review the ErpSyncQueueable and OpportunityTriggerHandler changes.

Files to review:
- force-app/main/default/classes/ErpSyncQueueable.cls
- force-app/main/default/classes/OpportunityTriggerHandler.cls
- force-app/main/default/classes/ErpNotificationSelector.cls
- force-app/main/default/classes/ErpSyncQueueableTest.cls

Focus on:
1. Bulk patterns: does Queueable handle List<Id> not single Id?
2. Governor limits: 100 SOQL per transaction, heap, CPU
3. Security: WITH USER_MODE, FLS enforcement, no hardcoded credentials
4. Async reliability: retry logic, idempotency, failure logging
5. Test coverage: are happy path, failure path, and bulk cases tested?

Return: Full structured report with severity, location, finding, fix for each issue.
```

### Phase 4: security-AI (validate)

The security agent checks authorization, data exposure, and compliance. This is especially important for integrations that handle customer data.

```
You are security-AI. Review the ERP sync implementation for data exposure and authorization risks.

Review scope:
- ErpSyncQueueable: does it expose any customer data in logs or error messages?
- HTTP callout payload: does it contain PII (names, addresses, emails, phone numbers)?
- ErpNotification__c: what data is stored, can it be accessed by unauthorized users?
- If ERP returns an error, is the error message logged with customer data?

Security checklist:
1. PII in payload: does the JSON sent to ERP contain names, emails, phone numbers, addresses?
2. PII in logs: do System.debug or custom Log__c records contain customer field values?
3. Authorization: can any user trigger the ERP sync or only users with certain roles?
4. Named Credentials: are credentials stored in Named Credentials, not hardcoded in the integration service?
5. Field-level security: does the Apex code check FLS before reading Opportunity fields to include in the payload?

Return: Security posture (Secure / Needs fixes / Critical gap), findings with severity and fix.
```

## Parallel agent workflows

Some Salesforce tasks can run in parallel rather than sequentially. Use parallel agents when:

- Multiple independent classes need to be written (e.g., selector for Account and selector for Contact can be generated simultaneously)
- The same code needs to be reviewed by multiple specialized reviewers simultaneously
- A feature has a compute-heavy phase and an I/O-heavy phase that can overlap

```
Run in parallel:
- developer-AI writes AccountSelector and ContactSelector simultaneously (separate prompts)
- reviewer-AI reviews AccountSelector and ContactSelector simultaneously (separate prompts)
- security-AI reviews the AccountSelector for PII exposure while developer-AI writes the ContactSelector

Merge results after parallel execution completes.
```

## Agent communication patterns

When chaining agents (architect -> developer -> reviewer -> security), pass the output of each agent as context for the next:

1. architect-AI output (architectural decisions) -> developer-AI input (constraints to follow)
2. developer-AI output (code) -> reviewer-AI input (code to review)
3. reviewer-AI output (findings) -> developer-AI input (fixes to apply)
4. security-AI output (security posture) -> developer-AI input (security fixes)

Do not run reviewer-AI and security-AI in parallel on the same code before developer-AI has applied the first round of fixes. The parallel review produces findings on code that will be improved, wasting review effort.

## Multi-agent workflow for a large feature

For a feature that spans multiple classes (e.g., a new trigger handler, service class, selector, batch class, and test class):

```
Phase 1 (sequential): architect-AI
- Task: Design the full feature architecture
- Output: Component list, data flow, failure modes, constraints

Phase 2 (parallel): developer-AI writes multiple classes
- Task 1: OpportunitySelector.cls
- Task 2: OpportunityService.cls
- Task 3: OpportunityTriggerHandler.cls
- Task 4: OpportunitySyncBatch.cls
- Task 5: OpportunitySyncQueueable.cls

Phase 3 (parallel): reviewer-AI reviews multiple classes
- Review 1: OpportunitySelector.cls
- Review 2: OpportunityService.cls
- Review 3: OpportunityTriggerHandler.cls
- Review 4: OpportunitySyncBatch.cls and OpportunitySyncQueueable.cls together (they interact)

Phase 4 (sequential): developer-AI fixes findings
- Apply fixes to all classes based on reviewer findings

Phase 5 (sequential): security-AI validates security posture
- Review the full implementation for data exposure and authorization risks

Phase 6 (sequential): test-AI writes tests
- Generate comprehensive test class covering all classes
```

## What this chapter covered

- When to use multi-agent workflows (complex features, security-critical code, large-scale changes)
- The four-agent workflow: architect -> developer -> reviewer -> security
- How to pass agent outputs as inputs to subsequent agents
- Parallel agent patterns for independent tasks
- The complete multi-agent workflow for large Salesforce features

## References

- [Apex Trigger Handler Best Practices](https://developer.salesforce.com/docs/atlas.en-us.226.0/apex_code犬.content/apex_code_apex_triggers.htm)
- [Queueable Apex](https://developer.salesforce.com/docs/atlas.en-us.226.0/apex_code犬.content/apex_code_apex_triggers.htm)
- [Apex Batch Classes](https://developer.salesforce.com/docs/atlas.en-us.226.0/apex_code犬.content/apex_code_batch_processing.htm)