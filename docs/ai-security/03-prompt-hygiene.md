# S3. Prompt Hygiene

Prompt hygiene is the practice of controlling what goes into an AI tool prompt. In AI-assisted Salesforce development, what you paste into a prompt may end up in the AI provider's logs. PII, credentials, internal org structure, and real record IDs do not belong in prompts. This chapter covers the specific rules, the why behind them, and the techniques to maintain productivity while staying safe.

## Why prompt hygiene matters

When you paste a SOQL query into an AI tool prompt, the query is sent to the AI provider for processing. What the provider does with that data depends on their policy. Some providers log prompts for training. Some do not. Some retain logs for a period. The safest assumption is: anything you paste may be logged and may not be under your control after that point.

This is not a theoretical risk. Organizations have had customer data appear in AI provider logs because a developer pasted a query with real record IDs. The fix is not to be more careful with the data you paste. The fix is to never paste data that should not leave your org.

## The four categories of restricted content

### 1. PII (Personally Identifiable Information)

PII includes:
- Customer names, email addresses, phone numbers, physical addresses
- Employee names and internal HR data
- Financial account information
- Social security numbers, tax IDs, passport numbers
- Health records (HIPAA protected)

Prompt examples that contain PII (bad):
```
Query this: SELECT Name, Email, Phone FROM Contact WHERE AccountId = '0015g000006ABCDEFG'
```
```
My customer John Smith called about his case. He wants to change the status field on his Opportunity to Closed Won. How do I write the Apex?
```

Prompt examples without PII (good):
```
Query: SELECT Name, Email, Phone FROM Contact WHERE AccountId = :accountId
```
```
A customer wants to update their Opportunity stage to Closed Won. Write an Apex method that handles this transition with proper FLS checks.
```

### 2. Credentials and secrets

Never paste:
- API keys or tokens
- Passwords or secret answers
- Named Credential configurations with actual credentials
- OAuth client secrets
- Salesforce session IDs or auth tokens
- Private keys or certificates

Even when demonstrating a pattern, paste the structure, not the credential:

Bad: `req.setHeader('Authorization', 'Bearer sk_prod_1234567890abcdef');`
Good: `req.setHeader('Authorization', 'Bearer <API_KEY>');` (but Named Credentials is better)

### 3. Internal org structure

Information about your org that is not public:
- Internal team names and structure
- Custom object names that reveal internal business processes
- Org-specific field names that reveal internal naming conventions
- Internal workflow descriptions that are not in public documentation
- Custom metadata names that describe internal systems

This is a judgment call. Org structure in prompts is lower risk than PII, but it can reveal internal business logic to a third party. Use generic descriptions instead of org-specific ones.

### 4. Real record IDs

Record IDs in SOQL queries or in descriptions are the most common way PII leaks into prompts. An ID like `0015g000006ABCDEFG` does not look sensitive, but it uniquely identifies a specific Salesforce record that may contain customer data.

Always replace real record IDs with sample IDs or variable names:
- Real: `WHERE Id = '0015g000006ABCDEFG'`
- Safe: `WHERE Id = :recordId` (use a variable)
- Safe: `WHERE Name = 'Sample Account'` (use name, not ID)

## Safe prompting techniques

### Use sample data

When demonstrating a SOQL query pattern, use sample data:

```sql
-- Safe: sample data
SELECT Id, Name, Industry, AnnualRevenue
FROM Account
WHERE Industry = 'Technology'
LIMIT 100
```

```sql
-- Unsafe: real-looking data that could contain real customer names
SELECT Id, Name, Industry, AnnualRevenue
FROM Account
WHERE Name = 'Acme Corporation'
LIMIT 100
```

Use generic names like "Sample Account," "Test Company," "Example Corp."

### Describe patterns without pasting code

Instead of pasting a broken piece of code to ask for a fix, describe the pattern:

```
Instead of: pasting a SOQL query with real field names and WHERE clause with record ID

Say: "I have a query that fetches Account records by industry. It currently does a raw SOQL inside a for loop. How do I refactor it to use a selector pattern and move the SOQL outside the loop?"
```

The second approach gets you the same answer without exposing internal data.

### Use scratch org data

AI-assisted development should use scratch orgs (see S2). The data in scratch orgs is test data, not customer data. Paste freely from scratch org exports because they do not contain real PII.

But verify the data is actually test data and not production data that was loaded into the scratch org for testing purposes.

### Describe error messages without pasting full stack traces

Stack traces in error messages can contain file paths, internal server names, or internal business logic descriptions. Paste the relevant portion of the error, not the full stack trace.

```
Safe: "Getting a UNABLE_TO_LOCK_ROW error on Opportunity update. How do I handle this in bulk?"

Full stack trace: requires review to ensure no internal paths are visible
```

## Prompt hygiene checklist

Before pasting anything into an AI tool:

```
[ ] Does this contain customer PII (names, emails, phone numbers, addresses)?
[ ] Does this contain credentials or secrets (API keys, tokens, passwords)?
[ ] Does this contain real record IDs that could map to customer data?
[ ] Does this describe internal org structure that is not publicly known?
[ ] Am I pasting from a scratch org (safe) or from production/sandbox (needs review)?

If any answer is yes: sanitize before pasting. If unsure: do not paste.
```

## Team-level prompt hygiene

Prompt hygiene works when the whole team follows it. Establish team rules:

```
Team Prompt Hygiene Rules:
1. No production record IDs in prompts (use sample IDs or variables)
2. No credentials or secrets in prompts (use Named Credentials by name, not value)
3. No customer PII in prompts (use generic descriptions instead)
4. Scratch org data is safe to paste; sandbox and production data is not
5. If you are unsure whether something is safe to paste: ask first
```

Post these rules in the team channel. Reinforce them during onboarding. Review them when someone causes a prompt hygiene incident (even a near-miss).

## What this chapter covered

- Why prompt hygiene matters (data in prompts may end up in provider logs)
- The four restricted content categories: PII, credentials, org structure, real record IDs
- Safe prompting techniques (sample data, describe patterns, scratch org data)
- The prompt hygiene checklist
- Team-level rules for maintaining prompt hygiene

## References

- [Salesforce Data Privacy](https://developer.salesforce.com/docs/atlas.en-us.226.0/sfdc/sfdc_lx_data_privacy.htm)
- [OWASP Prompt Injection](https://owasp.org/www-project-web-security-testing/)
- [Anthropic data handling](https://www.anthropic.com/news/data-management-at-anthropic)