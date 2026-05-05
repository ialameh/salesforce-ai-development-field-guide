# WM1. Case Studies

This chapter contains two anonymized case studies of AI-generated Salesforce code: one where AI-generated Apex passed the Salesforce Security Review, and one where AI-generated Apex caused a governor limit cascade in production. These are real incidents with details changed to protect the organizations involved.

## Case Study 1: AI-Generated Apex Passed Security Review

### Background

A team building a managed package for the Salesforce AppExchange used AI assistance (developer-AI with security-AI review) to generate an Apex controller for a Lightning component that managed Opportunity line items.

The component allowed sales reps to add, remove, and modify line items on Opportunities. The controller was responsible for querying line items, performing calculations, updating the Opportunity, and handling error cases.

### What was generated

The developer-AI prompt specified:
- Use selector layer for all queries
- Use WITH USER_MODE for DML
- Bulk-safe for up to 200 line items per Opportunity
- FLS enforcement on all field reads and writes
- Security.stripInaccessible on returned records

The security-AI review confirmed:
- WITH USER_MODE on all DML
- FLS checks via Schema.sObjectType.ProductItem.isAccessible() before reads
- FLS checks via Schema.sObjectType.ProductItem.isCreateable()/isUpdateable() before writes
- Security.stripInaccessible(AccessType.READABLE, lineItems) before returning to LWC
- No hardcoded credentials
- No SOQL injection vectors (static queries only)
- @AuraEnabled method with FeatureManagement.checkPermission('LineItemEdit') gate

### The review outcome

The code was submitted for Salesforce Security Review. The reviewer found:
- No SOQL injection vulnerabilities
- Proper FLS enforcement
- Proper sharing model declaration (with sharing on the controller)
- No hardcoded credentials
- Proper input validation
- Error messages did not leak internal system details

The submission passed Security Review on first submission, which is unusual. The security-AI review before human review had caught and fixed all the issues that typically cause Security Review failures.

### Key factors in the success

1. **Specific prompts**: The prompts described the exact security requirements, not just "make it secure"
2. **security-AI review**: The dedicated security review phase before human review caught FLS issues that the developer-AI had not addressed
3. **selector pattern**: Using selectors for all queries prevented SOQL injection by keeping queries static
4. **Named Credentials reference**: The prompt explicitly said to use Named Credentials by name, not by value
5. **bulk-safe from the start**: The prompt specified bulk requirements, so the code handled 200-record scenarios correctly

### Lessons

- AI can generate Security-Review-ready code if the prompts are specific about security requirements
- The security-AI review phase is not optional for code going to Security Review
- Selector pattern is not just an architectural preference; it is a security control (prevents SOQL injection)

## Case Study 2: AI-Generated Apex Caused Governor Limit Cascade

### Background

A team used AI assistance to generate a batch class that processed Order records and updated related inventory. The batch was designed to run nightly and handle up to 50,000 Order records.

The developer-AI prompt specified:
- Use Database.Batchable
- Handle up to 200 records per scope
- Call external inventory service via HTTP callout
- Retry on failure with exponential backoff

### What was generated

The AI-generated batch class compiled and passed tests. The tests used a small set of test records and passed. The batch was deployed to production.

At 2 AM, the batch job started. At 2:47 AM, the batch job failed with `QueryException: too many SOQL queries`. The error triggered a monitoring alert. The on-call engineer discovered that:
1. The batch had processed 47,000 records successfully
2. The batch then hit the governor limit and failed
3. The failure left 3,000 Order records in an inconsistent state (partially processed, not rolled back)

The root cause was in the `execute()` method. The AI had generated code that queried the Order's line items inside the execute method (against the batch rule that says no SOQL in execute), but had wrapped it in a way that appeared correct:

```apex
// AI-generated (broken)
public void execute(Database.BatchableContext ctx, List<Order> orders) {
    Set<Id> orderIds = new Set<Id>();
    for (Order o : orders) {
        orderIds.add(o.Id);
    }
    // This query happens once per scope, but scopes can be large
    List<OrderItem> items = [SELECT Id, Quantity, Product2Id FROM OrderItem WHERE OrderId IN :orderIds];
    // Process items...
}
```

The problem: when the scope contained 200 Orders, the query returned up to 2,000 OrderItems. The AI did not account for OrderItems with many line items per order, nor did it account for the SOQL limit within the execute context.

The AI had been instructed to put all SOQL outside loops, but had not been instructed about SOQL inside batch execute methods. The prompt had said "no SOQL inside loops," which the AI followed correctly inside the business logic, but the AI treated the batch execute method itself as a different context.

### The failure sequence

1. Batch started at 2 AM with 50,000 Orders (250 scopes of 200)
2. Scopes 1-235 processed correctly (47,000 records)
3. Scope 236 contained Orders with many line items (one Order had 400+ OrderItems)
4. The execute() method's query returned 400+ items, plus additional processing queries inside the same execute context
5. Single execute hit 102 SOQL queries (100 limit + 2 from retry logic)
6. Governor limit exception thrown
7. Transaction rolled back for scope 236, but scopes already processed were committed
8. 3,000 records in inconsistent state (inventory updated for some, not all Orders)

### Key factors in the failure

1. **Missing bulk test**: Tests used 10 records per scope, never tested the edge case of Orders with many line items
2. **Missing scope analysis**: The AI did not reason about what a worst-case scope looked like (200 Orders with varying line item counts)
3. **No governor limit context**: The prompt did not specify that the execute() method is a separate governor limit context with the same 100-SOQL limit
4. **No chunking strategy**: The AI did not propose chunking large scopes or handling orders with many items differently

### The fix

The batch was redesigned with a chunking approach inside execute():
- Group Orders by complexity (number of line items)
- Process low-complexity Orders in the current scope
- Queue high-complexity Orders (>50 line items) to a separate Queueable
- Add retry logic specifically for the governor limit case

### Lessons

- AI-generated batch classes need specific governor limit context: execute() runs in its own 100-SOQL-limit context
- Test with worst-case data, not typical-case data (one Order with 400 line items is more realistic than 200 Orders with 2 line items each)
- Bulk tests must use realistic data volumes and realistic data shapes
- The "no SOQL in loops" rule does not cover "no SOQL in execute() methods"; batch-specific constraints must be stated explicitly

## What this chapter covered

- Case Study 1: AI-generated Apex that passed Security Review (what made it succeed)
- Case Study 2: AI-generated Apex that caused a governor limit cascade (what went wrong and why)

## References

- [Salesforce Governor Limits](https://developer.salesforce.com/docs/atlas.en-us.226.0/apex_code犬.content/apex_code_execution_governor.htm)
- [Batch Apex Best Practices](https://developer.salesforce.com/docs/atlas.en-us.226.0/apex_code犬.content/apex_code_batch_processing.htm)
- [Security Review Requirements](https://security.secure.force.com/securityguide/InputValidation)