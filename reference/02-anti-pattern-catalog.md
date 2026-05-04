# R2. Anti-Pattern Catalog

AI generates Salesforce code that fails in predictable ways. This catalog documents the most common anti-patterns, explains why they fail, and gives the correct pattern to use instead. Use this catalog to review AI-generated code and to write better prompts that prevent these failures.

## Apex anti-patterns

### SOQL inside a loop

The most common AI-generated failure. The AI puts a SOQL query inside a `for` loop because training data shows query-and-process patterns that are correct in other languages but broken in Salesforce.

```apex
// AI-generated (broken)
public void processOpportunities(List<Opportunity> opps) {
    for (Opportunity opp : opps) {
        Account acct = [SELECT Id, Name FROM Account WHERE Id = :opp.AccountId LIMIT 1];
        opp.AccountName__c = acct.Name;
        update opp;
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
    Map<Id, Account> accountsById = new Map<Id, Account>(
        [SELECT Id, Name FROM Account WHERE Id IN :accountIds]
    );
    List<Opportunity> toUpdate = new List<Opportunity>();
    for (Opportunity opp : opps) {
        Account acct = accountsById.get(opp.AccountId);
        if (acct != null) {
            opp.AccountName__c = acct.Name;
            toUpdate.add(opp);
        }
    }
    update toUpdate;
}
```

**Why it fails:** Each iteration of the loop executes a SOQL query. With 200 records in the trigger, you get 200 SOQL queries and hit the 100-SOQL limit.

**Prevention in prompts:** "No SOQL inside loops. Use a Set to collect all IDs first, then query once outside the loop."

### DML inside a loop

```apex
// AI-generated (broken)
public void processContacts(List<Contact> contacts) {
    for (Contact c : contacts) {
        c.Status__c = 'Active';
        update c;  // DML in loop
    }
}
```

```apex
// Correct
public void processContacts(List<Contact> contacts) {
    List<Contact> toUpdate = new List<Contact>();
    for (Contact c : contacts) {
        c.Status__c = 'Active';
        toUpdate.add(c);
    }
    update toUpdate;  // Single DML outside loop
}
```

**Why it fails:** Each update is a separate DML statement. 200 records = 200 DML statements, hitting the 150-DML limit. Also causes mixed DML errors if the update triggers other DML.

### Single-record assumption

```apex
// AI-generated (broken)
public void handleOpportunity(Opportunity opp) {
    List<Contact> contacts = [SELECT Id FROM Contact WHERE AccountId = :opp.AccountId];
    contacts[0].Level__c = 'Primary';  // .size() == 0 causes index error
    update contacts[0];
}
```

```apex
// Correct
public void handleOpportunities(List<Opportunity> opps) {
    // Handle any number of records including 0
    if (opps.isEmpty()) return;

    Set<Id> accountIds = new Set<Id>();
    for (Opportunity opp : opps) {
        if (opp.AccountId != null) {
            accountIds.add(opp.AccountId);
        }
    }
    if (accountIds.isEmpty()) return;

    Map<Id, List<Contact>> contactsByAccount = new Map<Id, List<Contact>>();
    for (Contact c : [SELECT Id, AccountId, Level__c FROM Contact WHERE AccountId IN :accountIds]) {
        if (!contactsByAccount.containsKey(c.AccountId)) {
            contactsByAccount.put(c.AccountId, new List<Contact>());
        }
        contactsByAccount.get(c.AccountId).add(c);
    }

    List<Contact> toUpdate = new List<Contact>();
    for (Opportunity opp : opps) {
        List<Contact> contacts = contactsByAccount.get(opp.AccountId);
        if (contacts != null && !contacts.isEmpty()) {
            contacts[0].Level__c = 'Primary';
            toUpdate.add(contacts[0]);
        }
    }
    if (!toUpdate.isEmpty()) update toUpdate;
}
```

**Why it fails:** When `contacts` is empty, `contacts[0]` causes an index out of bounds exception. The code also only processes one contact per account, which may not be the intent.

### Missing WITH USER_MODE

```apex
// AI-generated (broken - no sharing enforcement)
public void updateAccountRating(List<Id> accountIds) {
    List<Account> accounts = [SELECT Id, Rating FROM Account WHERE Id IN :accountIds];
    for (Account a : accounts) {
        a.Rating = 'Hot';
    }
    update accounts;  // No sharing enforcement
}
```

```apex
// Correct
public void updateAccountRating(List<Id> accountIds) {
    List<Account> accounts = [
        SELECT Id, Rating
        FROM Account
        WHERE Id IN :accountIds
        WITH USER_MODE
    ];
    List<Account> toUpdate = new List<Account>();
    for (Account a : accounts) {
        a.Rating = 'Hot';
        toUpdate.add(a);
    }
    Database.update(toUpdate, false);  // Partial success
}
```

**Why it fails:** Without `WITH USER_MODE`, the code runs in system mode and can modify records the user does not have access to. This is a security and compliance violation.

### Hardcoded credentials

```apex
// AI-generated (broken)
req.setHeader('Authorization', 'Bearer sk_prod_1234567890abcdef');
req.setEndpoint('https://api.example.com');
```

```apex
// Correct
req.setEndpoint('callout:ErpIntegrationNamedCredential/orders');
// Named Credential handles auth automatically
```

**Why it fails:** API key is in code, committed to version control, visible in AI provider logs when code is pasted in prompts.

## LWC anti-patterns

### Client-side authorization

```javascript
// AI-generated (broken)
if (userProfile !== 'Admin') {
    this.showDeleteButton = false;  // UI hiding = authorization (wrong)
}
```

```javascript
// Correct: authorization happens in Apex
// LWC just displays what Apex returns
// Apex method checks CRUD/FLS before returning data
```

**Why it fails:** Client-side button hiding is not authorization. A user can inspect the DOM or call the Apex method directly if they know the API name.

### Wire with non-cacheable method

```javascript
// AI-generated (broken)
@wire(getOpportunity, { opportunityId: '$opportunityId' })
opportunity;  // If getOpportunity is not cacheable, this is wrong
```

```javascript
// Correct: use imperative call for non-cacheable methods
// @wire(cacheable=true) only for idempotent reads with no side effects
imperativeGetOpportunity() {
    getOpportunity({ opportunityId: this.opportunityId })
        .then(data => this.opportunity = data)
        .catch(error => this.error = error);
}
```

**Why it fails:** `cacheable=true` on a method that has side effects or returns fresh data causes stale data display and incorrect behavior.

### Unhandled promise rejection

```javascript
// AI-generated (broken)
fetchData() {
    getAccounts().then(data => this.accounts = data);
    // No .catch() - unhandled rejection if call fails
}
```

```javascript
// Correct
fetchData() {
    getAccounts()
        .then(data => this.accounts = data)
        .catch(error => {
            this.error = error.body.message;
            this.showError = true;
        });
}
```

**Why it fails:** Unhandled promise rejections cause silent failures in the browser. The user sees nothing, and the error is not logged.

## Flow anti-patterns

### Get Records without filters

```
// AI-generated (broken Flow element)
Get Records: Fetch all Account records (no filter criteria)
```

**Why it fails:** Fetching all records of an object can return millions of rows, timing out the Flow and causing governor limit exceptions.

**Correct pattern:** Always add filter criteria. "Fetch Account records where Industry equals '{!var_Industry}' AND AnnualRevenue greater than 0"

### Assignment inside loop

```
// AI-generated (broken Flow)
Loop: Iterate over accountList
  Assignment: Set variable for each iteration (inside the loop)
```

**Why it fails:** Assignments inside loops cause unexpected behavior because the assignment happens per iteration, not after collection. The Flow may not behave as expected.

**Correct pattern:** Use Assignment elements after the Loop to aggregate or set values based on loop results.

### Recursive Flow without guard

```
// AI-generated (broken Flow)
Entry: Record update on Account
-> Update Account fields
-> Entry: Record update on Account (triggered again)  // Infinite recursion
```

**Why it fails:** The Flow triggers on Account update, updates the Account, which triggers the Flow again, creating an infinite loop that causes governor limit exceptions.

**Correct pattern:** Add a Decision element before the Update Records element: "Has Account.Processing_Flag__c changed from false to true? If no, skip update."

## Test anti-patterns

### Fake assertions

```apex
// AI-generated (broken test)
System.assert(true);  // Asserts nothing
```

```apex
// Correct test
System.assertEquals('Hot', accounts[0].Rating, 'Rating should be Hot');
System.assertNotEquals(null, accounts[0].Id, 'Account should have Id');
```

**Why it fails:** `System.assert(true)` always passes. The test proves nothing.

### Missing bulk test

```apex
// AI-generated (broken test - only tests single record)
testUpdateAccount() {
    Account a = new Account(Name = 'Test');
    insert a;
    service.updateAccountRating(new List<Id>{ a.Id });
    Account result = [SELECT Rating FROM Account WHERE Id = :a.Id];
    System.assertEquals('Hot', result.Rating);
}
```

```apex
// Correct test - tests bulk
testUpdateAccountRatingBulk200Records() {
    List<Account> accounts = new List<Account>();
    for (Integer i = 0; i < 200; i++) {
        accounts.add(new Account(Name = 'Test ' + i));
    }
    insert accounts;

    List<Id> ids = new List<Id>();
    for (Account a : accounts) ids.add(a.Id);

    service.updateAccountRating(ids);

    List<Account> results = [SELECT Id, Rating FROM Account WHERE Id IN :ids];
    System.assertEquals(200, results.size());
    for (Account r : results) {
        System.assertEquals('Hot', r.Rating, 'Rating should be Hot for all records');
    }
}
```

**Why it fails:** The single-record test passes, but the code fails under bulk load with governor limit exceptions.

### seeAllData=true without justification

```apex
// AI-generated (broken test)
@IsTest(seeAllData=true)  // Makes test dependent on org state
```

**Why it fails:** `seeAllData=true` makes tests dependent on existing org data. Tests pass in one org and fail in another. They also slow down test execution.

**Correct pattern:** `seeAllData=false` (default) and create all test data in the test class using `@TestVisible` factory methods.

## What this chapter covered

- Apex anti-patterns: SOQL in loops, DML in loops, single-record assumption, missing WITH USER_MODE, hardcoded credentials
- LWC anti-patterns: client-side authorization, wire with non-cacheable method, unhandled promise rejection
- Flow anti-patterns: Get Records without filters, Assignment inside loop, recursive Flow without guard
- Test anti-patterns: fake assertions, missing bulk tests, seeAllData=true without justification

## References

- [Apex Governor Limits](https://developer.salesforce.com/docs/atlas.en-us.226.0/apex_code犬.content/apex_code_execution_governor.htm)
- [Apex Bulk Best Practices](https://developer.salesforce.com/docs/atlas.en-us.226.0/apex_code犬.content/apex_code_bulk_best_practices.htm)
- [LWC Best Practices](https://developer.salesforce.com/docs/component-library/documentation/en/lwc)