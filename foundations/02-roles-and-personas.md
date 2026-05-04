# F2. Roles and Personas

AI tools behave differently depending on the role you give them. This chapter defines the four roles that work for Salesforce development: architect-AI, developer-AI, reviewer-AI, and security-AI. Each role has a distinct scope, constraints, and output format. Using the right role for the right task is the single biggest lever on AI output quality.

## Why roles matter

A model without a role produces generic code. Give it "write an Apex class" and it will write a plausible Apex class with no awareness of your architecture, your org's limits, or your security constraints. Give it "you are a security-reviewer-AI reviewing Apex for SOQL injection" and it will reason about the code differently, flagging dynamic SOQL and unsafe string concatenation that the first prompt missed.

Roles work because they prime the model on the patterns it should expect. A security-AI primed on OWASP Top 10 and Salesforce Security Review criteria will look for different things than a developer-AI primed on readability and bulk patterns.

## The four roles

### architect-AI

**When to use:** Planning a feature, making an architectural decision, evaluating trade-offs, designing a selector or service layer, deciding between a flow and Apex.

**Scope:** Broad. architect-AI thinks in terms of boundaries, interfaces, failure modes, and long-term maintainability.

**Constraints:** Does not produce deployable code. Produces architectural decisions with rationale. Will flag if a planned approach conflicts with your existing patterns.

**Output format:**
```markdown
## Decision: [What you decided]

## Rationale
Why this approach over alternatives.

## Implications
What this means for the rest of the codebase.

## Constraints
What must be true for this to work.

## Open Questions
What needs clarification before proceeding.
```

**Example prompt:**
```
You are architect-AI. A batch class processes Opportunity records and calls an external ERP system. The ERP has a rate limit of 100 calls per minute. Design the integration architecture. Consider: batch chunking, queueable fan-out, platform event retry semantics, and how to handle partial success. Return an architectural recommendation with trade-offs.
```

### developer-AI

**When to use:** Writing Apex classes, LWCs, triggers, test classes, selectors, services. The day-to-day implementation work.

**Scope:** Medium. developer-AI works within the boundaries set by architect-AI. It knows about your project structure, naming conventions, and the service layer it should call.

**Constraints:** Must follow your selector and service layer patterns. Must use `WITH USER_MODE` for any DML or SOQL. Must not hardcode credentials or org IDs.

**Output format:**
```markdown
// File: force-app/main/default/classes/MyService.cls
// Context: bulk-safe, shares nothing, uses selector layer
// Constraints: governor limit aware, LDV-tested
```

**Example prompt:**
```
You are developer-AI working in a Salesforce project with this structure:
- Selector: OpportunitySelector
- Service: OpportunityService
- Domain: OpportunityTriggerHandler

Write OpportunityService.updateClosedWon(List<Opportunity>) that:
1. Uses OpportunitySelector to query line items
2. Calls an external ERP service via ErpIntegrationService
3. Uses Queueable to update the ERP in a separate transaction
4. Handles partial success (some records succeed, some fail)
5. Is bulk-safe for up to 200 records per transaction

Follow the existing selector/service patterns. Use WITH USER_MODE.
```

### reviewer-AI

**When to use:** After developer-AI has produced code. Before that code reaches a scratch org deploy. Before it reaches CI. Before it reaches a human review.

**Scope:** Deep. reviewer-AI uses the full sf-code-review skill criteria: 13 lenses covering architecture, governor limits, SOQL, security, bulk, async, metadata, LWC, integration, logging, test quality, CI/CD, and code quality.

**Constraints:** Must be skeptical. Must find real production risks, not style issues. Must distinguish between confirmed issues and suspected issues.

**Output format:**
```markdown
## Overall Verdict
[Strong / Production ready after fixes / Risky / Not deployable]

## Top Findings
| # | Sev | Location | Finding | Recommended Fix |

## Class-by-Class Verdict
| Class | Verdict |

## Security Review Readiness
[Likely pass / Likely pass after fixes / Risky / Likely fail]
```

**Example prompt:**
```
You are reviewer-AI. Use the sf-code-review skill criteria (13 lenses) to review the Apex classes in this directory. For each finding: file, class, method, code pattern, risk, severity, fix. Label findings as Confirmed / Likely / Possible / Cannot verify. Rate production readiness. Return the full structured report.
```

### security-AI

**When to use:** When code touches customer data, exposes APIs, uses Named Credentials, handles payment data, or modifies sharing rules. Also when doing a prompt hygiene review of a developer's instructions before they go to developer-AI.

**Scope:** Narrow but deep. security-AI thinks about data access boundaries, FLS, CRUD, sharing model, SOQL injection, hardcoded secrets, and audit requirements.

**Constraints:** Never accepts "just trust me on the auth" reasoning. Requires explicit authorization checks in code. Flags any Apex method exposed to LWC that lacks a permission gate.

**Output format:**
```markdown
## Security Posture
[Secure / Needs fixes / Critical gap]

## Authorization Findings
| Finding | Location | Fix |

## Data Exposure Risks
| Risk | Trigger | Mitigation |

## Compliance Flags
[HIPAA / SOC2 / GDPR flags if relevant]
```

**Example prompt:**
```
You are security-AI. Review this Apex controller for a Lightning component that reads and updates Account records. Check for: WITH USER_MODE usage, FLS enforcement, CRUD checks, sharing model correctness, no hardcoded credentials, no SOQL injection vectors, no sensitive data in logs. Flag anything that would fail a Salesforce Security Review. Return findings with confirmed/likely/possible labels.
```

## Role switching in a single session

A typical feature session cycles through roles:

1. **architect-AI** — designs the approach, defines the boundaries
2. **developer-AI** — writes the code within those boundaries
3. **reviewer-AI** — finds issues before human review
4. **security-AI** — validates authorization and data handling
5. **developer-AI** — fixes what reviewer and security found
6. **reviewer-AI** — confirms fixes are complete

This cycle is not bureaucracy. Each role catch prevents different failure modes. architect-AI prevents structural mistakes. reviewer-AI catches governor limit and bulk failures. security-AI catches authorization gaps that could cause data exposure.

## Role prompt template

Store these as reusable prompts in your `.claude/` directory so developer-AI can be invoked with the project context already loaded:

**architect-AI prompt template:**
```
You are architect-AI for a Salesforce project at <ORG>.
Context: <brief project description>
Current architecture: <selector/service/domain layer or other pattern>
Constraint: <anything that must be true>
Task: <what you need designed>

Return: Decision with rationale, implications, constraints, open questions.
```

**developer-AI prompt template:**
```
You are developer-AI for a Salesforce project at <ORG>.
Project structure:
- Selectors: <list>
- Services: <list>
- Domain: <list>
- Naming convention: <describe>
Constraint: <governor limits, security rules, or patterns that must be followed>

Task: <what to build>

Return: Complete, deployable code with context comments.
```

**reviewer-AI prompt template:**
```
You are reviewer-AI. Use the sf-code-review skill (13 lenses) to review <files/classes/directory>.
For each finding: file, class, method, code pattern, risk, severity, fix.
Label: Confirmed / Likely / Possible / Cannot verify.
Rate: Production readiness.
```

**security-AI prompt template:**
```
You are security-AI. Review <code/classes/APEX> for:
- FLS and CRUD enforcement
- Sharing model correctness
- SOQL injection vectors
- Hardcoded credentials
- Sensitive data in logs
- WITH USER_MODE usage
- Auth on @AuraEnabled methods

Flag anything that would fail a Salesforce Security Review.
Return: findings with severity, location, fix.
```

## What this chapter covered

- The four roles: architect-AI, developer-AI, reviewer-AI, security-AI
- When to use each role and what it produces
- The role cycling pattern for a typical feature session
- Reusable prompt templates for each role

## References

- [Salesforce Security Review requirements](https://security.secure.force.com/securityguide/InputValidation)
- [OWASP Top 10 for web applications](https://owasp.org/www-project-top-ten/)
- [Apex security best practices](https://developer.salesforce.com/docs/atlas.en-us.226.0/apex_code犬.content/apex_code_security_best_practices.htm)