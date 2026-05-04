# S1. The Five Destruction Scenarios

AI coding tools introduce five failure categories that are specific to AI-assisted Salesforce development. None of these exist in traditional manual development. They are not edge cases. Every team using AI tools for Salesforce has encountered at least one of them, and the ones that reach production cause real damage. This chapter names each scenario, explains how it happens, and gives the prevention mechanism.

## The five scenarios

1. **Org wipe** — AI deletes everything in a production org
2. **Data exfiltration** — AI extracts sensitive data from Salesforce and sends it somewhere
3. **Credential exposure** — AI writes credentials to logs, code, or external systems
4. **Metadata corruption** — AI introduces breaking changes that corrupt org configuration
5. **Production deploy without review** — AI pushes code directly to production, bypassing human review

Each scenario has a specific cause. They are not random failures. They are predictable patterns that happen when AI tools are misconfigured, when prompts include sensitive data, or when review workflows are bypassed.

## Scenario 1: Org wipe

### What it looks like

`sf org delete scratch --target-org production` runs successfully. The AI tool has sf CLI access, the authenticated org list includes a production org alias, and the AI issues the delete command without asking for confirmation.

Or: `sf org delete scratch --target-org my-production-alias` where `my-production-alias` was set as the default org at some point during a session.

### How it happens

The root cause is org alias confusion. During a session, a developer authenticates to a scratch org, does some work, then authenticates to a sandbox or production org for a different task. The AI tool is running in the same terminal session. The production org alias is in the org list. The AI, following a prompt like "clean up old scratch orgs," deletes everything with an org in the list, including production.

### Prevention

- **Deny production orgs in AI tool permissions** — Configure the AI tool to deny any `sf org delete` command against production org aliases. Use the alias-based deny pattern in `~/.claude/settings.json`.
- **Never authenticate production to a machine running AI tools** — Use a dedicated CI/CD machine with manual approval gates for production access. Developer machines should only have scratch org authentication.
- **Use different auth mechanisms for scratch vs production** — Scratch orgs via sf CLI on developer machines. Production only via CI/CD with manual approval.
- **Verify org list before every AI session** — Run `sf org list` and confirm only scratch orgs are present before starting an AI-assisted session.

## Scenario 2: Data exfiltration

### What it looks like

A developer pastes a SOQL query into an AI tool prompt. The query includes record IDs that map to real customer data. The AI tool, trained on large language model architecture, may send that query to its provider's servers for processing. The customer data in the query is now in the AI provider's logs.

Or: A developer asks an AI tool to "export all Account records with emails and phone numbers to a CSV." The AI generates an Apex script that executes the query and formats the output. The developer runs it against a sandbox with real customer data. The output file contains PII that was not supposed to leave the org.

### How it happens

The prompt contains data that should not leave the org. Developers paste SOQL with real record IDs, error logs with customer names, or org schema descriptions that include internal business structure. Anything in a prompt may end up in the AI provider's logs, depending on the provider's data handling policy.

### Prevention

- **Prompt hygiene rules** — The team rule is: no PII, no credentials, no internal org structure in prompts. Ever. SOQL queries in prompts use sample data, not production record IDs.
- **Sanitize before pasting** — Before pasting any Salesforce content into an AI tool, remove real record IDs, real names, real email addresses, and any data that identifies customers or internal systems.
- **Use scratch orgs for AI-assisted development** — Real customer data stays in sandbox and production. AI tools only see sanitized scratch org data.
- **Verify AI tool data handling policy** — Understand whether your AI tool's provider logs prompts and for how long. Claude Code by default does not log prompts to training. Other tools may.

## Scenario 3: Credential exposure

### What it looks like

AI generates an Apex class that calls an external system. The prompt said "use Named Credentials," but the AI produces this instead:

```apex
req.setEndpoint('https://api.example.com');
req.setHeader('Authorization', 'Bearer sk_prod_1234567890abcdef');
```

The API key is in the code. It gets committed to version control. It is now exposed.

Or: AI generates a callout configuration with the password in plaintext in the code or in a comment.

### How it happens

The AI does not know that the credential should come from Named Credentials unless you explicitly tell it. When you say "call the ERP system," the AI defaults to putting the endpoint and auth in the code because that is what appears in most training data examples. The credential is not redacted by default.

### Prevention

- **Always specify Named Credentials in prompts** — "Use Named Credentials: ErpApiCredential. Do not hardcode any URL, token, or password."
- **Verify before running** — reviewer-AI and security-AI reviews catch hardcoded credentials before they reach version control.
- **Secret scanning in CI/CD** — Add a secret scanner to the pipeline (GitHub Secret Scanning, Snyk, or similar) that blocks commits containing potential credentials.
- **Never prompt with actual credentials** — If you paste a credential into a prompt to show the AI what to use, that credential is now in the AI provider's logs. Always describe credentials, never paste them.

## Scenario 4: Metadata corruption

### What it looks like

AI generates a destructive deploy that removes several metadata components. The developer runs it against a sandbox to test. The destructiveChanges.xml removes a Flow that was critical to a business process. The Flow is not recoverable from the local source (it was created in the sandbox directly). The org needs a Flow restore from a backup.

Or: AI generates metadata that conflicts with existing configuration. A trigger handler that has the same logic as an existing trigger handler runs twice, causing double-processing of records.

### How it happens

AI does not know what metadata is already in your org. It generates new metadata based on what you describe, not based on what exists. If you describe a Flow that should replace an existing Flow, the AI may generate an additional Flow instead of a replacement. If you run a destructive deploy without checking what it removes, you may remove things that were not tracked in source.

### Prevention

- **Source tracking for all metadata** — Everything in the org should be in version control. If a Flow was created in the org directly (not via source), retrieve it with `sf project retrieve start` before AI-assisted work.
- **Review destructiveChanges.xml with a human** — Destructive changes require explicit human review before applying. AI tool permissions should block auto-application of destructive changes.
- **AI tool deny pattern for destructive deploys** — Configure the AI tool to ask before running `sf project deploy start` with a `destructiveChanges.xml` file.
- **Metadata diff before deploy** — Use `sf project diff` to compare local source to target org before deploying. Review what will change.

## Scenario 5: Production deploy without review

### What it looks like

AI generates an Apex class. The developer tests it in a scratch org, commits it to a feature branch, and opens a PR. The CI/CD pipeline runs tests and deploys to staging. A human approves staging-to-production. But: the AI was also used to configure the pipeline itself, and the pipeline was configured to auto-deploy to production on merge to main. The feature branch PR merges, tests pass, and the code goes straight to production with no human review.

Or: A developer uses an AI tool on their local machine to deploy directly to a sandbox that happens to be aliased as the production org in their sf auth list.

### How it happens

The review workflow has a gap. AI-generated code is treated the same as human-written code, but the review workflow was designed for human-written code and assumes the human knows what they are deploying. An AI generating code does not mean the code was reviewed.

### Prevention

- **Require human review for all AI-generated code** — This is a policy, not a technical control. AI-generated code always goes through human review before deployment. The review checklist in S5 guides the review.
- **Pipeline approval gates** — Staging-to-production requires human approval in the CI/CD pipeline. No auto-deploy to production regardless of test results.
- **AI tool ask confirmation for production deploy** — Configure the AI tool to ask before any deploy to a production org alias, even if the CI/CD pipeline is the one running it.
- **Branch protection** — Main branch requires PR with at least one approval. No direct push to main. This does not guarantee the approver reviewed AI-generated code, so train the team to scrutinize AI-generated PRs specifically.

## What this chapter covered

- The five destruction scenarios: org wipe, data exfiltration, credential exposure, metadata corruption, production deploy without review
- How each scenario happens (the root cause pattern)
- The specific prevention mechanism for each scenario

## References

- [Salesforce CLI org management](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_guide/sfdx_cli_userguide_org_commands.htm)
- [Named Credentials](https://developer.salesforce.com/docs/atlas.en-us.226.0/apex_code犬.content/apex_code_named_credentials.htm)
- [sf project deploy destructive](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_guide/sfdx_cli_userguide_deploy_commands.htm)