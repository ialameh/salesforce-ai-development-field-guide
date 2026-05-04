# S5. Reviewing AI-Generated Code

AI-generated code must go through human review before deployment. The review is not optional and is not the same as reviewing human-written code. AI generates code that compiles, passes tests, and looks correct, but has failure modes that are invisible to a casual review. This chapter covers the specific things to check when reviewing AI-generated Salesforce code.

## Why AI-generated code needs special review

Human-written code carries the author's intent. A reviewer can ask "why did you write it this way?" and get an answer that makes the approach clear. AI-generated code has no author intent. The code was produced by pattern-matching against training data. The AI does not know why a pattern is appropriate; it knows that the pattern appeared in similar contexts in its training data.

This means:
- AI generates code that looks correct but has subtle governor limit violations
- AI generates code that works for a single record and fails under bulk load
- AI generates code that follows an old pattern that is no longer best practice
- AI generates code that looks like the right architecture but misses critical security checks

The review must be technical, skeptical, and evidence-driven. A review that only checks "does this compile" is not sufficient.

## The AI code review checklist

Use this checklist for every AI-generated Apex class, LWC component, or Flow before it reaches production.

### Architecture and structure

```
[ ] Does the code follow the project's selector/service/domain architecture?
[ ] Are concerns separated (no business logic in triggers, no raw SOQL in service classes)?
[ ] Is the class cohesive (does one thing, does it well)?
[ ] Are there any god classes that do too much?
[ ] Is dependency injection present where tests need seams?
[ ] Is the code consistent with existing patterns in the codebase?
```

### Governor limits and bulk safety

```
[ ] Is SOQL outside all loops (including before/after trigger handlers)?
[ ] Is DML outside all loops?
[ ] Does the code handle List<T> not just T?
[ ] Does the code handle empty lists?
[ ] Does the code handle duplicate records in a list?
[ ] Is heap usage under control (no large JSON serialization in a loop)?
[ ] Does any describe call happen inside a loop?
[ ] If this is a batch class: does start() use a QueryLocator, not a query inside execute()?
[ ] Is the code tested with 200 records, not just 1 or 5?
```

### Security and authorization

```
[ ] Is DML using WITH USER_MODE?
[ ] Are FLS checks present on all field access (isAccessible(), isCreateable(), etc.)?
[ ] Are @AuraEnabled methods gated with FeatureManagement.checkPermission() or custom permissions?
[ ] Are @RestResource methods using proper authorization?
[ ] Is Security.stripInaccessible() used when returning records to LWC?
[ ] Are there any SOQL injection vectors (dynamic SOQL without escaping)?
[ ] Are credentials in Named Credentials, not hardcoded?
[ ] Are no secrets or credentials logged in System.debug or custom logs?
[ ] Is sharing model declared correctly (with sharing, without sharing, inherited sharing)?
```

### Async and transaction safety

```
[ ] If @future or Queueable is used, is the implementation correct (no @future from triggers, proper retry logic)?
[ ] Does the code handle partial success in DML (Database.insert(records, false))?
[ ] Does the code handle record locking errors?
[ ] If Platform Events are used, is retry behavior defined?
[ ] Does the code handle mixed DML scenarios correctly?
```

### Tests

```
[ ] Are there tests for 0 records, 1 record, and 200 records (bulk)?
[ ] Are FLS and CRUD tests present (System.runAs with restricted user)?
[ ] Are error path tests present (what happens when external callout fails)?
[ ] Are assertions about outcomes, not just execution?
[ ] Is seeAllData=false (unless explicitly required and documented)?
[ ] Are mocks used for callouts (HttpCalloutMock)?
[ ] Is there a test for the idempotency requirement?
```

### Metadata and configuration

```
[ ] If the code uses Custom Metadata, is the metadata type and field name spelled correctly?
[ ] Are hardcoded IDs absent (use selector queries or Custom Metadata instead)?
[ ] If the code reads from Custom Settings, is it using the correct type (Custom Settings vs Custom Metadata)?
[ ] Is API version current?
```

## Review process for AI-generated code

### Step 1: Run the code through reviewer-AI

Before human review, run the code through the reviewer-AI with the sf-code-review skill criteria (13 lenses). reviewer-AI catches governor limit issues, bulk pattern violations, and security gaps that are easy to miss in manual review.

The reviewer-AI report identifies findings. The human reviewer then:
1. Verifies each finding is real (not a false positive)
2. Prioritizes findings by severity
3. Determines what must be fixed before deployment

### Step 2: Manual review against the checklist

The human reviewer examines the code against the checklist. The reviewer-AI catches technical issues. The human reviewer catches:
- Whether the AI understood the business context correctly
- Whether the code makes sense in the full context of the feature
- Whether the test cases are realistic and sufficient
- Any edge cases the AI may have missed

### Step 3: Security review

Run the code through security-AI for a focused security review. This is separate from the technical review because security issues (credential exposure, FLS bypass, SOQL injection) are often subtle and require specific security knowledge to catch.

### Step 4: Human sign-off

After reviewer-AI findings are addressed and the human reviewer confirms the checklist is satisfied, a human signs off. The sign-off includes:
- The person reviewing (name)
- The date
- What was reviewed (file names, version)
- Any known limitations or concerns
- Explicit statement: "I have reviewed this AI-generated code and confirm it is safe to deploy to [org]"

## What to look for in AI-specific failures

### The single-record assumption trap

AI often generates code that uses `scope[0]` or assumes `inputRecords.size() == 1`. This fails under bulk triggers. Look for:
- Array indexing with `[0]` without bounds checking
- Loops that assume only one record will be processed
- `@future` methods that take a single Id instead of `List<Id>`

### The selector avoidance trap

AI often writes raw SOQL instead of using selector classes. Look for:
- `[SELECT ... FROM Account WHERE ...]` inside a service class
- No selector class reference in the code
- Query in a loop (the classic failure)

### The security空白 trap

AI often misses authorization on `@AuraEnabled` methods. Look for:
- Methods that mutate data and are `@AuraEnabled` but have no permission check
- Methods that read data and are `@AuraEnabled` but do not use `WITH USER_MODE` or `Security.stripInaccessible`

### The test coverage illusion

AI generates tests with high coverage that assert nothing meaningful. Look for:
- `System.assert(true)` anywhere in the test
- Tests that only test the happy path
- Tests with no actual assertions (just try/catch that swallows exceptions)
- `seeAllData=true` without explicit justification

## What this chapter covered

- Why AI-generated code needs special review (no author intent, subtle failure modes)
- The full AI code review checklist (architecture, governor limits, security, async, tests, metadata)
- The review process: reviewer-AI -> manual checklist review -> security-AI -> human sign-off
- Specific AI failure patterns to look for: single-record assumption, selector avoidance, security gaps, test coverage illusion

## References

- [sf-code-review skill criteria (13 lenses)](https://docs.anthropic.com/en/docs/claude-code)
- [Salesforce Security Review checklist](https://security.secure.force.com/securityguide/InputValidation)
- [Apex Governor Limits](https://developer.salesforce.com/docs/atlas.en-us.226.0/apex_code犬.content/apex_code_execution_governor.htm)