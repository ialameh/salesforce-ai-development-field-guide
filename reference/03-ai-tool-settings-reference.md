# R3. AI Tool Settings Reference

This chapter is the complete reference for configuring AI coding tools for Salesforce work. It covers Claude Code permissions in full detail, GitHub Copilot workspace rules, Cursor rules, and general settings that apply across tools.

## Claude Code Permissions

Claude Code is the most configurable AI tool for Salesforce work. Its permissions system uses glob-like patterns to match command strings.

### Settings file locations

```
Global: ~/.claude/settings.json
Project-level: <project-root>/.claude/settings.json
```

Project-level settings override global settings.

### Permission structure

```json
{
  "permissions": {
    "allow": [
      "<pattern>"
    ],
    "deny": [
      "<pattern>"
    ],
    "ask": [
      "<pattern>"
    ]
  }
}
```

- `allow`: Command runs without asking
- `deny`: Command is blocked
- `ask`: Claude Code asks for confirmation before running
- When no pattern matches: allow (default)

### Pattern syntax

Patterns match command strings that Claude Code would execute. The pattern format is:

```
<tool>(<command string>)
```

For example:
- `Bash(sf org list)` matches the command `sf org list`
- `Bash(sf project deploy start*)` matches any command starting with `sf project deploy start`
- `Bash(sf data delete*)` matches any command starting with `sf data delete`

### Complete Salesforce deny patterns

```json
{
  "permissions": {
    "deny": [
      "Bash(sf org delete*)",
      "Bash(sf data delete*)",
      "Bash(sf project deploy start*--target-org production*)",
      "Bash(sf project deploy start*--target-org <prod-alias-1>*)",
      "Bash(sf project deploy start*--target-org <prod-alias-2>*)",
      "Bash(sf project deploy start*--target-org prod*)",
      "Bash(rm -rf force-app*)",
      "Bash(rm -rf config*)",
      "Bash(rm -rf scripts*)",
      "Bash(curl*--data*)",
      "Bash(wget*--post-data*)"
    ]
  }
}
```

These patterns:
- Block all org delete commands (prevents org wipe)
- Block all data delete commands (prevents data loss)
- Block deploys to production orgs (use actual org alias names in place of `<prod-alias-1>`, `<prod-alias-2>`)
- Block recursive delete of source directories (prevents metadata loss)
- Block raw HTTP POST operations (prevents data exfiltration via HTTP)

### Complete Salesforce ask patterns

```json
{
  "permissions": {
    "ask": [
      "Bash(sf project deploy start*)",
      "Bash(sf project deploy validate*)",
      "Bash(sf data query*)",
      "Bash(sf data create*)",
      "Bash(sf data update*)",
      "Bash(sf org list)",
      "Bash(sf org display)",
      "Bash(sf project retrieve start*)",
      "Bash(sf project start*)",
      "Bash(sf data import*)"
    ]
  }
}
```

These patterns:
- Ask before any deploy (confirms developer intent)
- Ask before data queries (awareness of data access)
- Ask before data modifications (awareness of data changes)
- Ask before retrieve operations (awareness of metadata retrieval)

### Project-level allow for scratch orgs

```json
// Project-level .claude/settings.json
{
  "permissions": {
    "allow": [
      "Bash(sf project deploy start*--target-org ai-dev*)",
      "Bash(sf project deploy start*--target-org test-scratch*)"
    ],
    "ask": [
      "Bash(sf project deploy start*)"
    ]
  }
}
```

This allows deploys to the project's designated scratch orgs without asking, while still asking for all other deploys and never overriding the global deny for production.

### Full settings.json examples

**Global settings (~/.claude/settings.json):**

```json
{
  "permissions": {
    "deny": [
      "Bash(sf org delete*)",
      "Bash(sf data delete*)",
      "Bash(sf project deploy start*--target-org production*)",
      "Bash(sf project deploy start*--target-org prod*)",
      "Bash(sf project deploy start*--target-org <insert-prod-alias>*)",
      "Bash(rm -rf force-app*)",
      "Bash(rm -rf config*)",
      "Bash(curl*--data*)",
      "Bash(wget*--post-data*)"
    ],
    "ask": [
      "Bash(sf project deploy start*)",
      "Bash(sf project deploy validate*)",
      "Bash(sf data query*)",
      "Bash(sf data create*)",
      "Bash(sf data update*)",
      "Bash(sf org list)",
      "Bash(sf org display)",
      "Bash(sf project retrieve start*)",
      "Bash(sf project start*)",
      "Bash(sf data import*)"
    ]
  }
}
```

**Project-level settings (project/.claude/settings.json):**

```json
{
  "permissions": {
    "allow": [
      "Bash(sf project deploy start*--target-org ai-dev*)",
      "Bash(sf project deploy start*--target-org ci-test*)"
    ]
  }
}
```

## GitHub Copilot Settings

Copilot does not have Salesforce-specific permission controls like Claude Code. Use organizational policies and workspace rules instead.

### Workspace rules for Salesforce

Create `.github/copilot-instructions.md` or configure via GitHub organization settings:

```
# Salesforce project guidelines

## Security
- Never suggest hardcoded credentials or API keys
- Always use Named Credentials for external callouts
- Use WITH USER_MODE for all DML operations

## Apex patterns
- No SOQL or DML inside loops
- Use selector layer for all queries
- Follow trigger handler framework

## Commands to avoid suggesting
- sf org delete (requires human review)
- Direct production deploys

## Review requirements
- All AI suggestions must be reviewed before deployment
```

### VS Code settings for Copilot

In `.vscode/settings.json`:

```json
{
  "github.copilot.enable": {
    "*": true,
    "apex": true,
    "javascript": true
  },
  "github.copilot.inlineSuggest.enable": true
}
```

Copilot cannot be disabled per-file for Salesforce files specifically. Use organizational policies to enforce review requirements.

## Cursor Rules

Cursor uses `.cursor/rules` for project-specific constraints. Rules are not enforced at the CLI level (they influence the AI's suggestions but do not block commands).

### Salesforce rules file

Create `.cursor/rules` in the project root:

```
# Salesforce AI Development Rules

## Critical
- Never suggest sf project deploy to production orgs
- Never include real record IDs or PII in generated code
- Always use selector pattern for SOQL (no raw SOQL in service classes)
- Use WITH USER_MODE for all DML operations
- No hardcoded credentials (use Named Credentials by name)

## Apex requirements
- No SOQL or DML inside loops
- Bulk-safe: handle List<T> not just T
- Handle empty lists (early return)
- No System.debug() in production code

## LWC requirements
- Client-side authorization is not authorization (enforce in Apex)
- Use @wire only for cacheable methods
- Handle loading and error states in template

## Review
- All suggestions must pass human review before deployment
```

### Custom instructions in Cursor

In Cursor Settings -> Workspace -> Custom Instructions:

```
You are working on a Salesforce project.
- Only use scratch orgs (never production)
- Use selector/service/domain architecture
- No hardcoded credentials
- Follow Salesforce security best practices (WITH USER_MODE, FLS checks)
```

## General AI Tool Settings

### Prompt logging and training

```
Claude Code: Does not log prompts for training by default.
Copilot: Business/Enterprise plans have better data controls. Check your plan.
Cursor: Has privacy mode that prevents training on your code.
```

For compliance-sensitive work, verify your AI provider's data handling policy before using with real customer data.

### Disabling network access

If your AI tool has a network access setting, disable it for Salesforce work to prevent:
- Uploading code to external services without consent
- Calling external APIs during code generation
- Sending Salesforce metadata to AI providers outside your agreed scope

### Session management

For Claude Code sessions that span multiple tasks:

```
Start of session:
1. Run sf org list to confirm only scratch orgs are authenticated
2. Review current project .claude/settings.json permissions
3. Set context for the session: "We are working on the Opportunity trigger handler, scratch org only"

During session:
- If asked to deploy, verify target org in the ask confirmation
- If asked to run queries, verify the query is selective and scoped

End of session:
- Review what was generated
- Run reviewer-AI on new files before committing
```

## What this chapter covered

- Claude Code permissions structure (glob patterns, allow/deny/ask)
- Complete deny and ask patterns for Salesforce
- Project-level overrides for scratch org allow
- GitHub Copilot workspace rules and organizational policies
- Cursor rules and custom instructions
- General AI tool settings (prompt logging, network access, session management)

## References

- [Claude Code permissions documentation](https://docs.anthropic.com/en/docs/claude-code/permissions)
- [GitHub Copilot documentation](https://docs.github.com/en/copilot)
- [Cursor rules documentation](https://cursor.com/docs/rules)