# F1. The Salesforce AI Stack

This chapter maps the tools, models, and integrations that make up the modern Salesforce AI development stack. If you are new to AI-assisted Salesforce work, start here. If you have been working with AI tools on Salesforce already, use this to fill gaps in your mental model.

## What the stack looks like

The Salesforce AI development stack has three layers:

1. **The AI coding tool** — the interface you type into. Claude Code, GitHub Copilot, Cursor, Amazon CodeWhisperer all fit here. This is what most developers think of as "the AI."

2. **The model** — the language model powering the tool. Claude (Anthropic), GPT-4o (OpenAI), Gemini (Google), Codestral (Mistral), Llama (Meta). Each has different context windows, cost structures, and strengths for Salesforce work.

3. **The Salesforce integration layer** — how the tool connects to Salesforce. Salesforce-specific extensions in the IDE, sf CLI integration, scratch org authentication, and deployment pipelines.

Most developers spend all their time thinking about layer 1. The failures that burn you happen in layers 2 and 3.

## The AI coding tools

### Claude Code

Anthropic's CLI tool. Runs locally, authenticates to Salesforce via the sf CLI, and can deploy to scratch orgs directly. The strongest option for deep Salesforce work because it has a full filesystem context, can run shell commands, and has a permissions system that can be configured to block production deploys.

Configuration happens in `~/.claude/settings.json` (global) or `.claude/settings.json` (project-level). Permissions can be scoped to specific org aliases, which makes it the most controllable tool for Salesforce safety.

### GitHub Copilot

 Microsoft's offering, embedded in VS Code and GitHub. Works well for autocomplete and small refactors. Copilot Chat is better for explaining code. The team policies feature can enforce certain prompts, but org-level control is weaker than Claude Code's. Copilot does not have a native Salesforce permissions system, so blocking production deploys requires external controls.

### Cursor

Built on top of Claude and GPT-4o. Has a strong codebase-indexing feature that helps when you have a large Salesforce project. The privacy mode prevents training on your code, which matters if you work in regulated industries. Cursor's rule files can enforce project-specific constraints.

### Amazon CodeWhisperer

Included with AWS and JetBrains tooling. Less Salesforce-aware than Claude Code or Copilot. Useful if your Salesforce project has significant AWS integration.

## The models

A language model is characterized by context window, cost, and training data. For Salesforce work, the relevant characteristics are:

**Context window** — How much code you can feed in at once. Salesforce projects with selector layers, service layers, and domain layers can hit 50k+ tokens of context. If the model cannot hold your entire class structure, it loses cross-class awareness and starts generating single-file solutions that break your architecture.

**Knowledge cutoff** — When the model's training data ends. Models trained before Spring 2025 do not know about recent Apex features, Flow enhancements, or platform changes. They also do not know about recent security vulnerabilities or governor limit changes.

**Salesforce-specific knowledge** — Most models have seen Salesforce documentation and Stack Exchange posts in training, but the depth varies. Claude and GPT-4o tend to have the best Salesforce knowledge. CodeWhisperer is weaker on Apex patterns.

**Cost structure** — Claude Code pricing includes token costs. Copilot is a flat subscription. Per-token pricing matters if you run large refactors that generate 100k+ tokens.

## The integration layer

### Salesforce CLI (sf)

The center of the integration layer. Every AI tool that connects to Salesforce does so through `sf`. The key commands for AI-assisted work:

```bash
sf org list                              # verify only scratch orgs are connected
sf project deploy start                  # deploy to org
sf project deploy validate               # validate without deploying
sf data query                            # SOQL queries
sf org display                           # org info and auth details
```

AI tools that run `sf` commands directly need permissions configured. Without guardrails, an AI tool with sf CLI access can deploy to any org it is authenticated to, including production.

### VS Code Salesforce extensions

The extensions pack provides Apex language support, LWC development, and org browser. When combined with an AI coding tool running in the VS Code terminal, you get a full IDE-plus-AI workflow.

The limitation is that the VS Code terminal is a second-class citizen to the AI tool's own terminal. Claude Code runs its own shell session, which means it can call `sf` directly without routing through VS Code.

### GitHub Actions and CI/CD

When AI-generated code reaches CI, the pipeline itself is part of the stack. A pipeline with manual approval gates between staging and production prevents AI from pushing directly to production. A pipeline with no such gate does not.

## Token costs and context management

The biggest hidden cost in AI-assisted Salesforce development is context overflow. When a project grows past what the model can hold in context, two things happen:

1. The model starts forgetting earlier parts of the codebase. A refactor that spans five classes starts generating code that contradicts itself across files.

2. Costs spike. If you are paying per token, a large context window that overflows means you are feeding redundant context on every prompt.

Managing context for Salesforce projects:

- Keep trigger handlers under 200 lines so the model can reason about the full handler
- Use selector layers to abstract SOQL so the model sees a clean interface rather than raw queries
- Write unit tests that are short and targeted rather than sprawling integration tests
- Use the model for one class at a time, not for generating entire feature layers at once

## What this chapter covered

- The three-layer AI development stack: tool, model, integration
- The four main AI coding tools and their Salesforce suitability
- How the sf CLI ties everything together
- Token costs and context management
- Where failures happen when tools are misconfigured

## References

- [Claude Code documentation](https://docs.anthropic.com/en/docs/claude-code)
- [Salesforce CLI documentation](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_guide/sfdx_cli_userguide_org_commands.htm)
- [GitHub Copilot documentation](https://docs.github.com/en/copilot)
- [Anthropic model pricing](https://www.anthropic.com/pricing)