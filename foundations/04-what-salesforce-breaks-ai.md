# F4. What Salesforce Breaks AI

AI coding tools default to patterns that work in most programming environments. Salesforce breaks every one of those defaults. The platform has governor limits, bulk requirements, metadata dependencies, and async complexity that do not exist in other environments. Understanding what breaks AI on Salesforce is the prerequisite for writing prompts that produce correct code.

## The five failure categories

AI fails on Salesforce in five predictable ways:

1. **Single-record thinking** — AI generates code that works on one record and fails on 200
2. **Metadata blindness** — AI does not know your org's specific metadata, Custom Metadata, or permission sets
3. **Limit hallucination** — AI invents APIs, limits, or features that do not exist
4. **Async complexity underestimation** — AI treats async as an afterthought rather than a critical design decision
5. **Flow ignorance** — AI treats Flows as simple when they are among the most complex configuration artifacts in the platform

Each category has specific patterns. When you know what the AI will do wrong, you can write prompts that correct for it.

## Single-record thinking

The most common failure. AI writes code that works in a unit test with one record and fails in production under bulk load. This happens because most AI training data shows small examples, not enterprise-scale batch operations.

**What it looks like in Apex:**

```apex
// AI-generated (broken)
public void processOpportunity(Opportunity opp) {
    List<Contact> contacts = [SELECT Id FROM Contact WHERE AccountId = :opp.AccountId];
    for (Contact c : contacts) {
        c.Status__c = 'Processed';
        update c;  // DML inside loop - will fail at 200 records
    }
}
```

```apex
// Correct
public void processOpportunities(List<Opportunity> opps) {
    Set<Id> accountIds = new Set<Id>();
    for (Opportunity opp : opps) {
        accountIds.add(opp.AccountId);
    }
    Map<Id, List<Contact>> contactsByAccount = new Map<Id, List<Contact>>();
    for (Contact c : [SELECT Id, AccountId FROM Contact WHERE AccountId IN :accountIds]) {
        if (!contactsByAccount.containsKey(c.AccountId)) {
            contactsByAccount.put(c.AccountId, new List<Contact>());
        }
        contactsByAccount.get(c.AccountId).add(c);
    }
    List<Contact> toUpdate = new List<Contact>();
    for (Opportunity opp : opps) {
        List<Contact> contacts = contactsByAccount.get(opp.AccountId);
        if (contacts != null) {
            for (Contact c : contacts) {
                c.Status__c = 'Processed';
                toUpdate.add(c);
            }
        }
    }
    if (!toUpdate.isEmpty()) {
        update toUpdate;  // Single DML outside loop
    }
}
```

**How to prompt against this:**

Always specify the record count in your prompt. "handle up to 200 records in a single transaction" primes the AI to think about bulk patterns. Without this, it will generate single-record code.

```
Write a method that handles List<Opportunity> (up to 200 records per transaction).
Must use AccountSelector to query contacts (no raw SOQL in the service).
All DML must be outside loops. Use Database.update with allOrNothing=false for partial success.
```

The "up to 200 records" language is specific and forces bulk thinking.

**What else fails under bulk:**

- SOQL queries inside `for` loops (most common)
- DML statements inside `for` loops (second most common)
- Describe calls inside loops ( Schema.sObjectType.Account.fields.getMap() called per record)
- JSON.serialize inside a loop (heap spike on large records)
- Individual callouts per record (HTTP callout limit is 100 per transaction)

## Metadata blindness

AI does not know your org's specific Custom Metadata records, permission sets, or configuration. It also does not know which fields exist on objects in your org unless you show it.

**What this causes:**

AI will generate code that references fields that do not exist in your org. It will assume a field called `Status__c` exists on `Opportunity` because the training data had `Status__c` on `Opportunity` in some other org. Your org might use `Stage__c` or `Deal_Status__c`.

AI will also miss Custom Metadata-driven logic. If your pricing logic reads from `PricingConfig__mdt`, AI will hardcode the values instead of querying the metadata.

**How to prompt against this:**

```
Use these exact field names (from my org's schema):
- Opportunity.Approval_Status__c (not Stage)
- Opportunity.Deal_Value__c (not Amount)
- Custom Metadata: PricingConfig__mdt with fields Region__c, Tier__c, Multiplier__c

Do not assume fields exist. Use Schema.sObjectType.Opportunity.fields.getMap() to verify field existence at runtime if the field might not be present in all orgs.
```

Include the Custom Metadata type name and field names explicitly. AI can reason about metadata-driven design if you tell it the metadata exists.

**The Custom Metadata exception:**

Custom Metadata queries are faster than Custom Settings and do not count against SOQL limits. If your AI prompt says "read pricing from Custom Metadata," it may use Custom Settings instead because that is what appears in older training data. Specify the type:

```
Query PricingConfig__mdt (not PricingConfig__c custom setting).
PricingConfig__mdt records are available in subscriber orgs and do not count against SOQL limits.
```

## Limit hallucination

AI invents Salesforce features, limits, and APIs that do not exist. This is distinct from not knowing something. Hallucination means the AI confidently states something that is false.

**Common hallucinations:**

- "Salesforce has a built-in rate limiter for API calls" (it does not)
- "You can use @future to make synchronous callouts" (@future is inherently async)
- "SOQL queries return up to 50,000 records by default" (the limit is 50,000 but without a LIMIT clause you get all matching rows, which can timeout)
- "Database.saveResult works like a transaction" (it does not automatically roll back on partial failure)
- "You can call HttpRequest from a batch constructor" (you cannot)

**How to prompt against this:**

```
Constraints for this integration:
- Use Named Credentials for the ERP endpoint (do not hardcode the URL)
- HTTP callouts must be made from a Queueable or @future method, not from a trigger
- The ERP rate limit is 100 calls per minute; implement a simple throttle using Custom Metadata
- Do not assume Platform Cache is available (check if FeatureManagement.checkPermission returns false for 'PlatformCache')
- SOQL LIMIT: do not assume more than 1000 records without explicit pagination
```

Naming the specific limits and constraints prevents the AI from inventing solutions to problems you have already solved.

**When AI invents features:**

The more specific your domain is, the more likely the AI is to hallucinate. A generic Apex trigger on Account will be mostly correct. An integration with SAP that uses custom objects and real-time streaming will have more hallucinated APIs.

## Async complexity underestimation

AI treats async as an implementation detail rather than a core architectural decision. It will suggest `@future` methods without considering retry behavior, chaining limits, or transaction context.

**What AI typically generates:**

```apex
// AI-generated (incomplete)
@future(callout=true)
public static void notifyErp(Id opportunityId) {
    Opportunity opp = [SELECT Id, Name, Amount FROM Opportunity WHERE Id = :opportunityId];
    HttpRequest req = new HttpRequest();
    req.setEndpoint('https://erp.example.com/api/opportunity');
    req.setBody(JSON.serialize(opp));
    req.setMethod('POST');
    Http http = new Http();
    HttpResponse res = http.send(req);
}
```

**Problems with this:**

- No retry logic if the ERP callout fails
- No idempotency key to prevent duplicate notifications
- No handling of partial success if called from a batch
- No way to track if the notification was processed
- No distinction between "ERP received it" and "ERP accepted it"

**What correct async looks like:**

```apex
// Queueable with retry and idempotency
public class ErpNotificationQueueable implements Queueable, Database.AllowsCallouts {
    @TestVisible private List<Id> opportunityIds;
    @TestVisible private Integer retryCount;

    public ErpNotificationQueueable(List<Id> opportunityIds, Integer retryCount) {
        this.opportunityIds = opportunityIds;
        this.retryCount = retryCount;
    }

    public void execute(QueueableContext ctx) {
        List<Opportunity> opps = [SELECT Id, Name, Amount, External_Id__c
            FROM Opportunity WHERE Id IN :opportunityIds];
        List<ErpNotification__c> notifications = new List<ErpNotification__c>();

        for (Opportunity opp : opps) {
            notifications.add(new ErpNotification__c(
                Opportunity__c = opp.Id,
                External_Id__c = opp.External_Id__c,
                Payload__c = JSON.serialize(new ErpPayload(opp)),
                Status__c = 'Pending',
                Idempotency_Key__c = opp.Id + '-' + String.valueOf(opp.LastModifiedDate.getTime())
            ));
        }

        Database.saveResult[] results = Database.insert(notifications, false);
        // Handle partial success, implement retry logic in finish()
    }
}
```

**How to prompt for async:**

```
The ERP notification must:
1. Be idempotent: if called twice with the same Opportunity, produce one notification with matching External_Id__c
2. Use ErpNotification__c custom object as the persistence layer for outbound notifications
3. Implement retry: if ERP returns 503, re-enqueue with exponential backoff (max 3 retries)
4. Include an idempotency key in the payload so ERP can deduplicate
5. Use Named Credentials for the endpoint URL
6. Never call HttpRequest from a trigger or batch execute() method
```

The persistence layer requirement is critical. AI will generate fire-and-forget async code. You need the persistence layer to audit what was sent and handle retries.

## Flow ignorance

AI treats Flows as simple configuration. They are not. Flows have execution order issues, element-level sharing rules, screen元素 visibility conditions, decision framework complexity, and post-case scenarios that trip up even experienced admins.

AI will suggest putting business logic in Flows that belongs in Apex. It will not know which Flows are slow, which are in recursive loops, or which ones cannot be activated until certain records exist.

**What AI gets wrong about Flows:**

- "Add a Decision element to check if the user has permission" (Decisions do not enforce FLS)
- "Use a Get Records element to fetch Account data" (Get Records respects sharing but not FLS by default)
- "Put the logic in a Flow so admins can maintain it" (admin-maintainable Flows have very limited logic expression capability)
- "Flows run synchronously by default" (certain Flow types have async paths)

**How to prompt for Flows:**

```
For this business requirement: [describe what the Flow should do]
Boundary: [what the Flow must not do]

Prompt for Flow design:
- Entry criteria: [when does this Flow run]
- Exit criteria: [what success/failure looks like]
- What must happen in Apex ( Flows cannot handle: recursive loops, HTTP callouts in loops, complex multi-object transactions)
- What can stay in Flow ( straightforward assignments, screen flows, email alerts)
- Pause and resume behavior ( if using Schedule-Triggered Flow, what happens if the org has 100k records to process)

Do not suggest putting complex validation or multi-step logic in a Flow. Suggest Apex for anything requiring: loops, conditions based on related records, callouts, or transactional consistency.
```

The "what must happen in Apex" line is the key boundary. AI that understands this will not try to put a multi-object transaction in a Flow.

## What this chapter covered

- The five failure categories: single-record thinking, metadata blindness, limit hallucination, async complexity underestimation, Flow ignorance
- How each failure manifests in AI-generated Salesforce code
- Prompt patterns that correct for each failure mode
- Why knowing these failures lets you write better prompts

## References

- [Salesforce Governor Limits Reference](https://developer.salesforce.com/docs/atlas.en-us.226.0/apex_code犬.content/apex_code_execution_governor.htm)
- [Apex Bulk Best Practices](https://developer.salesforce.com/docs/atlas.en-us.226.0/apex_code犬.content/apex_code_bulk_best_practices.htm)
- [Queueable Apex Best Practices](https://developer.salesforce.com/docs/atlas.en-us.226.0/apex_code犬.content/apex_code_apex_triggers.htm)
- [Flow Best Practices](https://help.salesforce.com/s/articleView?id=sf.flow_reference_best_practices.htm)