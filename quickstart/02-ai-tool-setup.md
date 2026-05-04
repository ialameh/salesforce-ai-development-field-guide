# Q2. AI Tool Setup for Salesforce

This chapter sets up Claude Code specifically for Salesforce development. The same principles apply to other AI tools (Copilot, Cursor), but the configuration steps are for Claude Code.

## What to install

1. Salesforce CLI (`sf`) — authentication and org management
2. Claude Code — the AI coding tool
3. A code editor with Salesforce extensions (VS Code + Salesforce Extension Pack recommended)

Install Salesforce CLI:

```bash
npm install -g sf-cli
```

Install Claude Code:

```bash
npm install -g @anthropic-ai/claude-code
```

Or follow the official installation guide at https://docs.anthropic.com/en/docs/claude-code/install

## Authentication setup

Authenticate to scratch orgs only. Never to production.

```bash
# Authenticate to a dev hub
sf auth login web --alias my-dev-hub

# Create a scratch org
sf org create scratch --alias ai-dev --definition-file config/project-scratch-def.json --set-default

# Verify auth list shows only non-production orgs
sf org list
```

Confirm the output shows no production orgs. Every entry should be a scratch or sandbox.

## Claude Code permissions configuration

Create or edit `~/.claude/settings.json` to add Salesforce-specific permissions:

```json
{
  "permissions": {
    "deny": [
      "Bash(sf project deploy start*--target-org production*)",
      "Bash(sf data delete*)",
      "Bash(sf org delete*)",
      "Bash(rm -rf*)"
    ],
    "ask": [
      "Bash(sf project deploy start*)",
      "Bash(sf project deploy validate*)",
      "Bash(sf org list)"
    ]
  }
}
```

This configuration tells Claude Code to:
- Deny deploys to production automatically
- Ask before running any deploy (including validate)
- Ask before running data delete operations
- Always ask before org delete

## Project workspace setup

For each Salesforce project you work on, set up a dedicated workspace directory:

```text
~/projects/
  └── my-salesforce-project/
      ├── force-app/
      ├── config/
      ├── scripts/
      └── .claude/
          └── settings.json  (project-specific overrides)
```

Project-specific settings in `.claude/settings.json` override global settings. Use project-specific settings to add the project's scratch org alias to the allowed deploy targets.

## Verify the setup

Run through the Q1 checklist to confirm everything is configured correctly before using the tool on real Salesforce code.

A working setup means:
- `sf org list` shows only scratch orgs
- Claude Code starts without errors
- Deploy to scratch org works
- Deploy to production is denied or requires confirmation