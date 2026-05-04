# F3. Prompt Anatomy

Every good prompt for AI-assisted Salesforce development has four parts. Getting these right is the difference between code that works and code that passes CI but fails in production under load. This chapter breaks down the anatomy of a Salesforce-specific prompt and explains what each part does.

## The four parts of a Salesforce prompt

A complete prompt has:
1. **System context** — what the AI knows about your project and role
2. **Task framing** — what you want built and why
3. **Output constraints** — rules the code must follow
4. **Validation criteria** — how you know the output is correct

Most developers write only the task framing. They say "write a batch class" and get back a batch class that compiles but has SOQL in the execute method, no bulk handling, and no test coverage. The missing parts are why.

## System context

System context tells the AI what kind of project it is working in and what patterns to follow. Without this, an AI writing Apex will use Java or Python patterns, which do not map to Salesforce.

**What to include:**
- The Salesforce org type (scratch, sandbox, production)
- Your project architecture (selector/service/domain or simpler)
- Which AI role to use (architect-AI, developer-AI, etc.)
- The API version and any relevant limits (governor limits your org cares about)
- Any org-specific metadata the AI needs to know about

**Example:**
```
You are developer-AI working in a Salesforce project with:
- Selector layer: AccountSelector, OpportunitySelector, ContactSelector
- Service layer: AccountService, OpportunityService
- Trigger framework: TriggerHandler base with before/after hooks
- API version: 59.0
- Org: scratch org with Enterprise license

Your code must:
- Use selector classes for all SOQL queries
- Follow the existing TriggerHandler pattern
- Never hardcode IDs
- Use WITH USER_MODE for all DML operations
```

The selector layer reference is important. When you tell the AI to use `AccountSelector`, it knows to call a selector rather than writing raw SOQL in the service class. That single constraint prevents the most common AI-generated Salesforce failure: SOQL inside loops.

## Task framing

Task framing is the "what" and the "why." Be specific about what you want built. If the AI does not know the business context, it will make assumptions that may not hold.

**What to include:**
- The specific Apex class, trigger, LWC, or flow you want created
- The business context: what is this for, what triggers its use
- Any relevant record counts or data volumes: "handles up to 500 records per transaction"
- If this is a refactor: what the current code does and why it needs changing

**Example:**
```
Write OpportunityService.handleClosedWon(List<Opportunity> opportunities) that:
1. Called by OpportunityTriggerHandler when stage changes to 'Closed Won'
2. Must handle bulk triggers: up to 200 Opportunity records in a single transaction
3. Queries related OpportunityLineItem records for each Opportunity using OpportunitySelector
4. Calls ErpIntegrationService.notifyWonOpportunity(lineItems) to notify the ERP system
5. Uses Queueable to handle the ERP callout in a separate transaction context
6. Must be idempotent: if called twice with the same records, produces the same result
```

The idempotency constraint is doing real work here. It tells the AI to think about retry behavior, which most AI-generated code ignores.

## Output constraints

Output constraints are the rules the code must obey. These come from your architecture, your team's standards, and the Salesforce platform constraints.

**Key constraints for Apex:**
```
Must:
- Use selector layer for all SOQL queries (no raw SOQL in service classes)
- Use WITH USER_MODE for all DML operations
- Be bulk-safe: handle List<Opportunity>, not just Opportunity
- Use Database.insert(records, false) for partial success
- Follow trigger handler pattern: beforeInsert, afterInsert, etc.
- Add @TestVisible for any test seam needed

Must not:
- Hardcode IDs (use Custom Metadata or selector queries instead)
- Put SOQL or DML inside loops
- Use @future(callout=true) for synchronous callouts (use Queueable instead)
- Log sensitive data (PII, access tokens, record IDs in plain text)
- Use System.debug() in production code paths

Never:
- Deploy to production without human review
- Skip the security-AI review step for code that touches customer data
```

The "must not" and "never" sections are where you encode the safety constraints from your AI security policy. These are not stylistic preferences. They prevent production failures.

## Validation criteria

Validation criteria tell you how to know the code is correct before deploying. This is also where you tell the AI what tests to write.

**Example:**
```
Validate:
1. Governor limits: 0 SOQL queries inside execute(), no query loops, CPU < 10000ms
2. Bulk test: passes with 200 Opportunity records in a single transaction
3. Partial success: if ERP callout fails for 1 of 10 records, other 9 succeed
4. Idempotency: running handleClosedWon twice with same records produces no duplicate ERP notifications
5. Security: WITH USER_MODE used, no hardcoded credentials, FLS enforced on all field access

Test requirements:
- Test with 0 records (empty list)
- Test with 1 record
- Test with 200 records (bulk)
- Test with 3 records where 1 has no line items
- Test when ERP callout throws an exception (partial success path)
- Test as a user without FLS access to one field (error handling)
```

The empty list test is a Salesforce-specific validation that catches a common AI failure: the 0-record edge case where `scope[0]` causes an index out of bounds error.

## The prompt template in full

Combine the four parts into a reusable template:

```markdown
You are [architect-AI | developer-AI | reviewer-AI | security-AI] for a Salesforce project.

Project structure:
- Selectors: <list>
- Services: <list>
- Domain: <list>
- Trigger framework: <describe>
- API version: <version>

Constraint:
- <architectural constraints, governor limits, security rules>
- <what must not happen>

Task:
<specific description of what to build or review, including business context and data volumes>

Validate:
- <how to verify correctness, including test cases>
- <governor limit constraints>

Return: <what format to return: code, architectural decision, review report, etc.>
```

## Common mistakes in prompt anatomy

**Missing system context:** "Write a batch class" produces a batch class with no awareness of your selector layer, your trigger framework, or your API version. The batch class compiles but does not integrate with your existing code.

**Vague task framing:** "Create an Apex class to handle Opportunities" is not specific enough. The AI does not know the trigger context, the volume, or the idempotency requirements.

**No output constraints:** Without constraints like "no SOQL inside loops" and "must use selector layer," the AI defaults to patterns that look clean on a single-record example and fail under bulk load.

**No validation criteria:** Without explicit test requirements, the AI writes tests that assert coverage, not behavior. The test passes but does not prove the code works.

**Conflicting constraints:** Saying "make this fast" and "make this bulk-safe" can conflict on Salesforce. State which takes priority: correctness over speed is usually the right answer.

## What this chapter covered

- The four parts of a Salesforce AI prompt: system context, task framing, output constraints, validation criteria
- Why each part exists and what it prevents
- The complete prompt template
- Common mistakes and how to fix them

## References

- [Anthropic prompt engineering guide](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering)
- [Salesforce Apex Governor Limits](https://developer.salesforce.com/docs/atlas.en-us.226.0/apex_code犬.content/apex_code_execution_governor.htm)
- [Apex Bulk Best Practices](https://developer.salesforce.com/docs/atlas.en-us.226.0/apex_code犬.content/apex_code_bulk_best_practices.htm)