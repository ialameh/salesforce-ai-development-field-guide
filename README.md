# Salesforce AI Development Field Guide

A production-grade methodology for using AI coding tools in Salesforce development. Covers prompt engineering for Apex, LWC, Flows, and testing, plus a comprehensive security framework for AI-assisted Salesforce work.

This guide is for senior Salesforce developers, architects, and tech leads who use AI coding tools (Claude Code, Copilot, Cursor, or similar) in Salesforce projects. It assumes you know Salesforce development. The guide focuses on how to get AI to produce correct, safe, production-ready Salesforce code, and how to prevent AI from causing the categories of failures that are unique to AI-assisted development.

## What you get

The guide is organized into four major sections:

**AI Prompt Engineering** covers how to write prompts that produce correct Salesforce code. The core insight is that AI coding tools fail on Salesforce for predictable reasons: they default to single-record thinking, they do not know your org's specific metadata, they hallucinate Salesforce limits and APIs, and they treat Flows as simple when they are complex. Each chapter in the prompt engineering section addresses one of these failure modes with specific prompt patterns that work.

**AI Security** covers the five destruction scenarios that AI tools introduce to Salesforce work: org wipe, data exfiltration, credential exposure, metadata corruption, and production deploy without human review. This section gives you the org isolation strategy, the prompt hygiene rules, the permissions configuration, and the review checklist that prevents each scenario.

**Reference** contains the prompt template library, the anti-pattern catalog, and the AI tool settings reference. These are the practical tools you reach for during actual work.

**Worked Material** contains two case studies: one where AI-generated Apex passed a Salesforce Security Review, and one where AI-generated Apex caused a governor limit cascade in production.

## Chapter map

```text
Quickstart
  Q1. 15-Minute AI Safety Checklist
  Q2. AI Tool Setup for Salesforce

Foundations
  F1. The Salesforce AI Stack
  F2. Roles and Personas
  F3. Prompt Anatomy
  F4. What Salesforce Breaks AI

AI Prompt Engineering
  P1. Prompts for Apex
  P2. Prompts for LWC
  P3. Prompts for Flows
  P4. Prompts for Testing
  P5. Prompts for CI/CD
  P6. Planning with AI
  P7. Multi-Agent Workflows

AI Security
  S1. The Five Destruction Scenarios
  S2. Org Isolation Strategy
  S3. Prompt Hygiene
  S4. AI Tool Permissions Configuration
  S5. Reviewing AI-Generated Code
  S6. Secrets Management
  S7. Compliance Considerations
  S8. AI-Generated Code Audit Trail

Reference
  R1. Prompt Templates Library
  R2. Anti-Pattern Catalog
  R3. AI Tool Settings Reference
```

## 30-second version

- AI is a pair programmer, not an autopilot. Human review is mandatory before anything reaches production.
- Never connect an AI tool to production. Scratch orgs and sandboxes only.
- Salesforce breaks AI in predictable ways. Prompt for bulk, for limits, for your specific metadata.
- AI-generated code has the same security requirements as human-written code. FLS, sharing, SOQL injection: all apply.
- What you paste in a prompt may end up in a provider's logs. PII, credentials, internal org structure: keep those out.

## Status

Active development. Core chapters in progress.

## License

CC BY 4.0