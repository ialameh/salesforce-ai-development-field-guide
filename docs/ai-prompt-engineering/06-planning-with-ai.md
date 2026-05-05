# P6. Planning with AI

AI is a capable planning partner for Salesforce architecture decisions, but only if you ask it the right questions and give it the right context. This chapter covers how to use architect-AI for architectural decisions, rollback planning, risk assessment, and multi-objective trade-off analysis.

## When to use architect-AI for planning

Use architect-AI when you face a non-trivial Salesforce decision:

- Building a new integration (ERP, payment processor, third-party API)
- Designing a batch processing architecture for high-volume data
- Choosing between Flow and Apex for a business process
- Designing a selector or service layer pattern for a new object domain
- Planning a metadata-driven architecture that will be extended by admins
- Assessing async architecture (Queueable vs Batch vs Platform Events vs Scheduled Apex)
- Making a security architecture decision (Named Credentials, Custom Permissions, Auth Providers)

Do not use architect-AI for straightforward implementation tasks (write a trigger handler, add a field to a Flow). Use it for decisions where there are multiple valid approaches and trade-offs to reason through.

## Planning prompt structure

```
You are architect-AI for a Salesforce project.

Project context:
- Org: <type: scratch/sandbox/production>
- Current architecture: <selector/service/domain or other pattern>
- Constraints: <governor limits, org limitations, team patterns>

Decision: <what you need to decide>

Options considered:
1. <option A>: <brief description>
2. <option B>: <brief description>
3. <option C>: <include any non-obvious option>

Evaluation criteria (weight these by importance):
- <e.g., correctness first (100%), performance second, maintainability third>)

Decision request: Recommend one option and explain why. Include:
- Why this over the alternatives
- What this means for the rest of the codebase
- What constraints must be satisfied for this to work
- What could go wrong and how to mitigate it
- Open questions that need more information before finalizing

Return format: Use the architect-AI output format (Decision, Rationale, Implications, Constraints, Open Questions).
```

## Example: batch or queueable decision

```
You are architect-AI for a Salesforce project processing 50,000 Opportunity records per night.

Project context:
- Current architecture: Batch process runs at midnight using a scheduled Apex class
- Org: Enterprise edition with 2 million records
- Constraint: ERP system can handle 100 callouts per minute

Decision: Should the nightly sync use Batch Apex or Queueable with Schedulable?

Options:
1. Batch Apex with Queueable chaining for ERP callouts
2. Scheduled Apex that enqueues a Queueable for each chunk of 100 records
3. Platform Event subscriber that processes records async

Constraints:
- Must not exceed ERP rate limit of 100 calls per minute
- Must complete within 4-hour window (scheduled window limited)
- Must be idempotent (can safely run twice without double-processing)

Recommendation: [choose one, explain trade-offs]
```

## Rollback planning

Every significant change to a Salesforce org needs a rollback plan. AI can help you design the rollback path before you deploy.

```
You are architect-AI. Plan the rollback for a deployment of <change description>.

Change: <what is being deployed>
Risk level: <high/medium/low based on scope of change>

Rollback requirements:
- Who can trigger rollback (admin, developer, CI/CD pipeline)
- How long before rollback must be triggered (minutes, hours, days)
- What state must be restored (metadata, data, or both)
- How to verify rollback completed correctly

Rollback options:
1. <metadata rollback>: sf project deploy cancel or re-deploy previous version
2. <data rollback>: Apex script or Data Loader export/import
3. <feature flags>: Custom Metadata setting that disables the feature without rollback

For each option:
- Trigger: <how to activate>
- Time to execute: <how long>
- Data integrity: <what happens to data created after deploy>

Return: Rollback plan with options ranked by safety and speed.
```

## Risk assessment prompts

```
You are architect-AI. Assess risks for <project or feature>.

Scope: <what is changing>

Risk categories to evaluate:
1. Data risk: what data could be corrupted or lost
2. Deployment risk: what could fail during deploy
3. Runtime risk: what could fail in production under load
4. Security risk: what authorization or data exposure gaps exist
5. Integration risk: what external system dependencies could fail

For each risk:
- Severity: Critical / High / Medium / Low
- Likelihood: Likely / Possible / Unlikely
- Mitigation: <what to do if this risk occurs>

Top 3 risks to address before go-live:
1. <risk with highest severity and likelihood>
2. <second highest>
3. <third highest>

Return: Risk matrix sorted by severity.
```

## Multi-objective tradeoff prompts

Salesforce architecture often requires balancing competing objectives: speed vs correctness, simplicity vs maintainability, time-to-market vs technical debt.

```
You are architect-AI. Resolve a trade-off decision for <feature>.

Trade-off: <the competing objectives, e.g., "Speed of development vs long-term maintainability">

Option A: <approach name>
- Pros: <list>
- Cons: <list>

Option B: <approach name>
- Pros: <list>
- Cons: <list>

Decision criteria weighted:
- <e.g., correctness > 50%, maintainability > 30%, speed > 20%>

Analysis:
- Under what conditions does Option A win?
- Under what conditions does Option B win?
- What is the cost of being wrong in each direction?

Recommendation: <one option with clear rationale>

Criteria that would change the recommendation: <what information or environment change would flip the decision>
```

## What this chapter covered

- When to use architect-AI (complex decisions with trade-offs)
- The planning prompt structure for architect-AI
- Rollback planning for Salesforce deployments
- Risk assessment using architect-AI
- Multi-objective tradeoff resolution framework

## References

- [Salesforce Architecture Patterns](https://developer.salesforce.com/docs/atlas.en-us.226.0/apex_code犬.content/apex_code_apex_patterns.htm)
- [Apex Trigger Frameworks](https://developer.salesforce.com/docs/atlas.en-us.226.0/apex_code犬.content/apex_code_best_practices.htm)