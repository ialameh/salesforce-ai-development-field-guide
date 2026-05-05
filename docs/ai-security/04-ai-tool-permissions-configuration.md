# S4. AI Tool Permissions Configuration

AI tools need permissions configured before they are used with Salesforce. The configuration prevents the five destruction scenarios by controlling what commands the AI tool can run without asking, what it must ask before running, and what it must never run. This chapter covers the specific configuration for Claude Code (the most configurable option) and the patterns that apply to other AI tools.

## Claude Code permissions structure

Claude Code uses `~/.claude/settings.json` (global) and `.claude/settings.json` (project-level) for permissions. Project-level settings override global settings. The permissions structure uses `allow`, `deny`, and `ask` patterns that match against command strings.

```json
{
  "permissions": {
    "deny": [
      "<pattern>"
    ],
    "ask": [
      "<pattern>"
    ]
  }
}
```

- `deny`: The command is blocked. Claude Code will refuse to run it without asking.
- `ask`: Claude Code will ask for confirmation before running, even if the command is allowed by other rules.

Default behavior (when no pattern matches): allow.

## Essential Salesforce deny patterns

These patterns should be in every global Claude Code settings file for a Salesforce developer:

```json
{
  "permissions": {
    "deny": [
      "Bash(sf org delete*)",
      "Bash(sf data delete*)",
      "Bash(sf project deploy start*--target-org production*)",
      "Bash(sf project deploy start*--target-org <prod-alias-1>*)",
      "Bash(sf project deploy start*--target-org <prod-alias-2>*)",
      "Bash(rm -rf force-app*)",
      "Bash(curl*--data*)",
      "Bash(wget*)"
    ]
  }
}
```

These patterns:
- Block all `sf org delete` commands (prevents org wipe)
- Block `sf data delete` (prevents data deletion)
- Block deploys to production org aliases specifically
- Block recursive delete of the force-app directory (metadata loss)
- Block raw curl with data (prevents data exfiltration via HTTP)
- Block wget (same as curl, alternative data exfiltration path)

## Essential Salesforce ask patterns

These patterns require confirmation before running:

```json
{
  "permissions": {
    "ask": [
      "Bash(sf project deploy start*)",
      "Bash(sf project deploy validate*)",
      "Bash(sf data query*)",
      "Bash(sf org list)",
      "Bash(sf org display)",
      "Bash(sf project retrieve start*)",
      "Bash(sf project start*)"
    ]
  }
}
```

These patterns:
- Ask before any deploy (including validate) so the developer is aware a deploy is happening
- Ask before data queries (prevents accidental data exfiltration via large queries)
- Ask before org display (shows what org is targeted)
- Ask before retrieve operations (pulling metadata from org)

## Project-level overrides

Project-level settings in `.claude/settings.json` can be more permissive for the specific project workspace while global settings remain restrictive:

```json
// In project .claude/settings.json
{
  "permissions": {
    "allow": [
      "Bash(sf project deploy start*--target-org ai-dev*)"
    ],
    "ask": [
      "Bash(sf project deploy start*)"
    ]
  }
}
```

This allows deploys to the project's designated scratch org without asking, while still asking for all other deploys. The project-specific allow does not override the global deny for production orgs.

## Alias-specific controls

If you have multiple scratch orgs and want to allow deploys to one but not others:

```json
// Global settings (~/.claude/settings.json)
{
  "permissions": {
    "deny": [
      "Bash(sf project deploy start*--target-org <any-prod-alias>*)"
    ],
    "ask": [
      "Bash(sf project deploy start*)"
    ]
  }
}

// Project settings (.claude/settings.json in project directory)
{
  "permissions": {
    "allow": [
      "Bash(sf project deploy start*--target-org ai-dev-scratch*)",
      "Bash(sf project deploy start*--target-org test-scratch*)"
    ]
  }
}
```

The project-level allow does not override the global deny for production aliases. This is the correct behavior.

## Other AI tools permissions

GitHub Copilot, Cursor, and CodeWhisperer have less granular permission controls. The Salesforce-specific controls available:

### GitHub Copilot

Copilot does not have a Salesforce-specific permission system. Use these organizational controls:
- Copilot Business/Enterprise: organization-level policies that can restrict certain types of suggestions
- VS Code settings: disable Copilot for specific file patterns (e.g., `*.cls` files)
- GitHub organization rules: require PR approval for Apex file changes (does not prevent AI from suggesting changes, but catches them at review)

### Cursor

Cursor's `.cursor/rules` file can enforce project-specific constraints:

```
// In .cursor/rules (cursor-specific)
- Never suggest sf project deploy to production orgs
- Never include real record IDs in generated code
- Always use selector pattern for SOQL queries
```

Cursor rules are not enforced at the CLI level, so they are advisory, not blocking.

### General tool settings

For any AI tool, find and configure:
- Whether prompts are logged or used for training (opt out if available)
- Whether the tool can make network calls (disable or restrict)
- Whether the tool can run shell commands (disable for production work)
- Custom instructions that encode Salesforce safety rules

## Testing the configuration

After setting up permissions:

```
1. Run sf org list and confirm only scratch orgs
2. Ask Claude Code: "deploy to production using sf project deploy start --target-org <prod-alias>"
3. Confirm Claude Code denies the command
4. Ask Claude Code: "run sf org delete scratch --target-org ai-dev"
5. Confirm Claude Code denies the command
6. Ask Claude Code: "deploy to scratch org using sf project deploy start --target-org ai-dev"
7. Confirm Claude Code asks for confirmation (ask pattern works)

If any test fails, the configuration is not correct.
```

## What this chapter covered

- Claude Code permissions structure (allow/deny/ask patterns)
- Essential deny patterns for Salesforce (org delete, data delete, production deploy, recursive rm, curl with data)
- Essential ask patterns (deploy, data query, org display, retrieve)
- Project-level overrides for scratch org allow with global production deny
- Other AI tool permissions limitations and controls
- How to test the permissions configuration

## References

- [Claude Code permissions documentation](https://docs.anthropic.com/en/docs/claude-code/permissions)
- [Salesforce CLI sf commands](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_guide/sfdx_cli_userguide_org_commands.htm)