# Glossary

**AI-assisted development**: Using AI coding tools (Claude Code, Copilot, Cursor, etc.) to generate, review, or refactor Salesforce code with human oversight and review before production deployment.

**Architect-AI**: The AI role focused on design decisions, architectural trade-offs, and planning. Produces architectural decision documents, not deployable code.

**Bulk-safe**: Code that handles collections of records (List<T>) correctly, without making per-record SOQL queries or DML statements. A bulk-safe method works for 1 record and 200 records.

**DML**: Data Manipulation Language. The Apex statements that modify database records: insert, update, delete, undelete. Governed by the 150-DML-statement limit per transaction.

**FLS**: Field-Level Security. The Salesforce feature that controls whether a user can read, create, or modify specific fields on a record, regardless of object-level access.

**Governor limits**: The limits Salesforce imposes on Apex code execution: 100 SOQL queries per transaction, 150 DML statements, 10,000ms CPU time, 6MB heap. AI code often violates these limits under bulk load.

**LDV**: Large Data Volume. An org with millions of records. LDV changes how SOQL queries perform (selective vs non-selective) and how governor limits are hit.

**Multi-agent workflow**: Using multiple AI agents (architect, developer, reviewer, security) in sequence or parallel to develop a feature with appropriate review at each stage.

**Named Credentials**: A Salesforce metadata type that stores endpoint URLs and authentication credentials securely. Apex references Named Credentials by name, keeping credentials out of code.

**Org isolation**: The practice of ensuring AI tools can only access scratch orgs and sanitized sandboxes, never production orgs. The foundation of safe AI-assisted Salesforce development.

**Prompt hygiene**: The discipline of not pasting PII, credentials, real record IDs, or internal org structure into AI tool prompts.

**Queueable**: An Apex interface for async processing that allows callouts and has more governor limit headroom than @future methods. Used when async work needs to make HTTP callouts.

**Reviewer-AI**: The AI role focused on multi-lens code review across 13 specialist areas (architecture, governor limits, SOQL, security, bulk, async, metadata, LWC, integration, logging, test quality, CI/CD, code quality).

**Scratch org**: A temporary Salesforce org created from a definition file, used for development and testing. Disposable and isolated. The only org type that AI tools should access.

**Security-AI**: The AI role focused on authorization, FLS, CRUD, SOQL injection, credential exposure, and compliance. Reviews code for Salesforce Security Review readiness.

**Selector pattern**: An architectural pattern where SOQL queries are encapsulated in selector classes (e.g., OpportunitySelector). Prevents SOQL in service classes and makes queries easier to test and maintain.

**SOQL**: Salesforce Object Query Language. The Salesforce-specific query language, similar to SQL but with governor limits. SOQL inside loops is the most common AI-generated failure.

**With USER_MODE**: An Apex security feature that causes SOQL and DML to run in the context of the current user, respecting FLS and sharing rules. Required for all AI-generated DML operations.

**With sharing**: The Apex class declaration that enforces record sharing rules. Classes without this declaration run in system mode and can access records the user does not have access to.

**@AuraEnabled**: The Apex annotation that exposes a method to Lightning Web Components. Methods annotated with @AuraEnabled that mutate data or access sensitive fields must have explicit authorization checks.

**@future**: An Apex annotation for async processing that cannot make callouts. Use Queueable instead for async callouts.

**@IsTest**: The Apex annotation that marks a class or method as a test. Tests with @IsTest do not count against governor limits.

**@TestVisible**: An Apex annotation that makes private members accessible to test methods. Used for test seams without exposing members to production code.

**seeAllData=true**: A test attribute that gives tests access to existing org data. Creates test dependencies on org state. Avoid unless explicitly required. Default is seeAllData=false.