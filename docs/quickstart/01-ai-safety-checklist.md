# Q1. 15-Minute AI Safety Checklist

Before you use any AI coding tool with Salesforce, run through this checklist. Each item prevents one of the five destruction scenarios.

If you cannot check an item, do not connect the AI tool to Salesforce until you can.

## The checklist

### 1. Org isolation confirmed

[ ] The AI tool has never been authenticated to a production org.
[ ] Only scratch orgs and sandboxes are in the AI tool's auth list.
[ ] Production credentials are stored in CI secrets only, not on developer machines.
[ ] The AI tool has a token that cannot reach production.

How to verify: Run `sf org list` and confirm every org in the list is a scratch or sandbox. If production appears, use `sf org logout` to remove it.

### 2. Prompt hygiene rules established

[ ] Your team has a written rule: no PII, no credentials, no internal org structure in prompts.
[ ] Developers know that SOQL with real record IDs, error logs with customer names, and org schema diagrams do not belong in prompts.
[ ] A reminder is posted in the team channel: what goes in a prompt may end up in the AI provider's logs.

### 3. Permissions configured

[ ] The AI tool is configured to ask before running `sf project deploy start`.
[ ] The AI tool is configured to deny `sf org delete`, `sf data delete`, and raw `curl` commands.
[ ] Production org names or aliases are on the deny list.

### 4. Review workflow exists

[ ] AI-generated code always goes through human review before deployment.
[ ] The review checklist (Chapter S5) is known to the team.
[ ] There is no path from AI suggestion to production that bypasses human review.

### 5. Destructive change review is explicit

[ ] Any `destructiveChanges.xml` or metadata deletion is reviewed by a human, not auto-applied.
[ ] The AI tool does not have permission to run destructive deploys without confirmation.
[ ] The CI pipeline is configured to require approval for destructive changes.

## What to do if you find a gap

If you cannot check an item, stop and fix it before continuing.

The most common gap is an AI tool that was initially connected to production for testing and never disconnected. Fix it by running `sf org logout --target-org production` and then re-authenticating only to scratch orgs.

The second most common gap is missing permissions configuration. Configure the `.claude/settings.json` (see Chapter S4) before using the tool with Salesforce code.

## What this chapter covered

- The five-item safety checklist
- How to verify each item
- What to do when you find a gap

## References

- [Salesforce CLI org management](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_guide/sfdx_cli_userguide_org_commands.htm)
- [Claude Code permissions](https://docs.anthropic.com/en/docs/claude-code/permissions)