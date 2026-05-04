# S2. Org Isolation Strategy

Org isolation is the foundation of AI-assisted Salesforce development. If you only do one thing to make AI-assisted development safer, make it this: ensure AI tools can never reach production. This chapter covers the specific technical controls and operational practices that achieve effective org isolation.

## The isolation principle

AI tools should only have access to scratch orgs and sanitized sandboxes. Production org credentials should never be present on a developer machine running AI tools. CI/CD systems that have production access should require manual approval for all production operations and should never run AI-generated code directly to production without review.

The reason for this strictness: the five destruction scenarios in S1 all have the same root cause. An AI tool had access to a production org and something went wrong. Org isolation removes the production org from the AI tool's reach entirely.

## Authentication isolation

### Developer machine setup

On a developer machine running Claude Code or another AI tool:

```
# Authenticated orgs should only include:
sf org list
# Output:
# - my-dev-hub (dev hub, no scratch orgs listed as default)
# - ai-dev-scratch (scratch org, default)
# - ai-test-scratch (scratch org, not default)

# Production org should NEVER appear in this list on a developer machine
```

Never authenticate a production org on a developer machine. If you need to test AI-generated code against production-like data, use a sanitized sandbox that has had PII removed and is designated as a test environment.

### CI/CD authentication

Production access in CI/CD is separate from developer machine access:

```
CI/CD Production Access:
- Authenticate to production using a CI secrets store (GitHub Secrets, Bitbucket Pipelines Secrets)
- The auth URL is stored as a secret, not in plaintext
- The pipeline uses this auth only for staging-to-production deploys
- Manual approval gate required before production deploy
- After production deploy, the auth session expires or is invalidated

Developer Machine Access:
- Only scratch orgs and sandboxes
- No production auth URL stored locally
- If production access is needed for a specific task, it is done via a separate isolated environment with full human oversight
```

### Auth alias naming convention

Use naming conventions that make production obvious:

```
# Good - obvious which orgs are non-production
sf org list
# alias: dev-scratch-01, dev-scratch-02, test-sandbox-01

# Bad - production alias looks like any other org
sf org list
# alias: dev-hub, staging, prod-us-01  <- "prod" should not be on a dev machine
```

For AI tool permissions to work (see S4), use aliases that clearly identify production orgs so the deny patterns can match them.

## Scratch org management for AI development

AI development happens against scratch orgs. Set up a dedicated scratch org for AI-assisted work:

```
# Create a scratch org specifically for AI development
sf org create scratch --alias ai-dev --definition-file config/project-scratch-def.json --set-default

# When the org expires or gets corrupted (AI can corrupt test orgs with bad deploys), reset it
sf org delete scratch --target-org ai-dev
sf org create scratch --alias ai-dev --definition-file config/project-scratch-def.json --set-default
```

AI tools can corrupt scratch orgs. Bad deploys, destructiveChanges applied incorrectly, or test configurations that leave the org in an inconsistent state are all possible. Treat scratch orgs as disposable. Have a script that can recreate them quickly.

## Sandbox vs scratch for AI work

Scratch orgs are preferred for AI work because they are:
- Disposable (easy to reset)
- Fresh (known starting state)
- Fast to create (minutes vs hours for sandbox)

Use sanitized sandboxes for:
- Integration testing that requires real data volumes
- Testing against specific metadata that is not in scratch org definitions
- Performance testing under realistic data conditions

Never use production for any AI-assisted development or testing.

## Network isolation

If your Salesforce org uses IP restrictions or Connected App policies:

```
AI tool running machine requirements:
- Use a machine that falls within the allowed IP range for the org
- If the org uses MFA + IP restrictions, ensure the authenticated session is from an allowed location
- Claude Code and similar tools run from the developer's terminal, which inherits the machine's network context
```

Network-level controls are a defense-in-depth measure, not a replacement for org isolation. They catch mistakes but do not prevent intentional misuse.

## Verification

Before every AI-assisted session:

```
1. Run sf org list
2. Confirm no production orgs in the list
3. Confirm only scratch orgs (or designated test sandboxes) are present
4. Confirm the scratch org is not pointing to anything sensitive
5. If production appears: run sf org logout --target-org <alias> immediately
```

Make this a habit. It takes 10 seconds and prevents the most common cause of the five destruction scenarios.

## What this chapter covered

- The isolation principle: AI tools only reach scratch orgs and sanitized sandboxes
- Authentication isolation for developer machines vs CI/CD
- Scratch org management (disposable mindset, reset scripts)
- When to use sandbox vs scratch for AI work
- Network isolation as defense-in-depth
- Pre-session verification checklist

## References

- [Salesforce CLI Authentication](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_guide/sfdx_cli_userguide_auth.htm)
- [Scratch Org Management](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_guide/sfdx_cli_userguide_scratch_orgs.htm)
- [Connected App OAuth Policies](https://help.salesforce.com/s/articleView?id=sf.connected_app_create_oauth.htm)