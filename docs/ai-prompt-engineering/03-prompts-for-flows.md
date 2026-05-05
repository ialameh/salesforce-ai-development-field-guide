# P3. Prompts for Flows

Flows are the most powerful declarative tool in Salesforce and the hardest for AI to reason about correctly. AI will treat Flows as simple if-else trees when they are complex state machines with execution order dependencies, element-level sharing rules, and implicit recursion risks.

This chapter covers when to use Flows vs Apex, how to prompt for Flow design, and how to catch AI-generated Flow logic that belongs in Apex.

## When to prompt for Flow vs Apex

The first decision in any flow design task is whether a Flow is the right tool. AI defaults to Flow because it sounds simpler and less code-heavy. Sometimes it is. Often it is not.

**Use Flow for:**
- Straightforward data creation or update actions with clear inputs
- Screen flows that collect information from users and act on it
- Scheduled flows that run batch actions on a schedule (with awareness of volume limits)
- Simple approval processes with clear step assignments
- Auto-launched flows that respond to platform events or record changes (with caution on recursion)

**Use Apex for:**
- Complex multi-object transactions that require partial success handling
- Recursive logic or loops over large data sets
- HTTP callouts to external systems
- Logic that requires fine-grained control over transaction boundaries
- Anything that requires custom exception handling or retry logic
- Flow-to-Apex callbacks that must return data to the Flow after an Apex operation

**The recursion problem:**

Flows that launch on record changes can recurse. A Flow triggered by Opportunity updates updates the Opportunity, which triggers the Flow again. AI will not flag this without explicit instruction. If you prompt for a Flow that handles record updates, add:

```
Constraint: This Flow must not cause recursive triggers. If a Flow element updates the same record that triggered the Flow, add a Decision element to check if an update is needed before updating (compare old vs new field values). Never put a Get Records element that reads the same record being updated in the triggering trigger context.
```

## Flow design prompts

When you need a Flow designed, prompt architect-AI with the boundary conditions:

```
You are architect-AI. Design a Flow for <business requirement>.

Boundary:
- Entry criteria: <when does this Flow run: record change, screen load, schedule, platform event>
- Exit criteria: <what does success look like, what does failure look like>
- Data volume: <how many records expected per run: 10, 100, 10000>

What belongs in Flow (straightforward):
- <list simple actions: Create/Update records, Assign variable, Decision paths, screen elements>

What belongs in Apex (complex):
- <list anything requiring: loops, HTTP callouts, complex multi-object transactions, retry logic, custom exception handling>

Flow requirements:
1. Entry: <trigger type> on <Object>
2. Decision: check <field> before running logic (prevent unnecessary updates)
3. Get Records: fetch related <Object> records (note: Get Records respects sharing but NOT FLS by default - add FLS checks in Apex if needed)
4. Loop: iterate over <collection> (note: do not put Assignment elements inside the loop that modify the same record being iterated - move to Post-Loop Assignment)
5. Decision: <what to check at loop boundary>
6. Update Records: <which records to update>
7. Create Records: <if creating new records>

Design the Flow with these constraints and return a flow diagram description in text format.
```

## Flow element prompts

When you need a specific Flow element configured, be precise about the element configuration:

```
You are developer-AI. Configure a <element type> element in a Flow.

Element: <name and type: Decision, Assignment, Get Records, etc.>

Flow context: <what Flow this is part of>

Configuration requirements:
1. <specific field and value mappings for this element>
2. <any variable assignments or formulas>
3. <error handling or default values if data is missing>

Common Flow element mistakes to avoid:
- Assignment inside a loop: use Post-Loop Assignment to aggregate values, not individual assignments per loop iteration
- Get Records without filters: always add filter criteria to Get Records elements, do not fetch all records of an object
- Decision that does not cover null: add a default path for null values
- Update Records element that updates the same record that triggered the Flow: add a Decision to check if update is needed first

Write the element configuration as a structured description that can be implemented in Flow Builder.
```

## Screen Flow prompts

Screen flows have specific constraints that other Flow types do not.

```
You are developer-AI. Design a screen Flow for <requirement>.

Screen Flow requirements:
1. Screen elements: <what fields to display, what input types>
2. Navigation: next/previous/submit buttons
3. Validation: client-side validation in the screen (note: client-side validation is UI only, add server-side validation in any Apex action called from the Flow)
4. Record creation: use Create Records element after screen submission
5. Error handling: if Create/Update fails, display error message and allow retry

Screen Flow constraints:
- Do not put complex business logic in screen flow formulas (Flow formula syntax is limited and hard to debug)
- Screen flow latency: each screen element adds ~1-2 seconds of load time
- If you need more than 5 screen steps, consider splitting into two flows or moving to a Lightning component
- Do not store sensitive data in Flow variables that persist (Flow variables with Manually Assigned values persist until the Flow ends)

Write the screen flow structure and element list.
```

## What this chapter covered

- When to use Flows vs Apex and how to make that decision
- The recursion problem and how to prevent it
- How to prompt for Flow design (architect-AI) and Flow elements (developer-AI)
- Screen flow constraints
- Why Flows are not simple and how to catch AI-generated Flow logic that belongs in Apex

## References

- [Flow Best Practices](https://help.salesforce.com/s/articleView?id=sf.flow_reference_best_practices.htm)
- [Flow Governor Limits](https://developer.salesforce.com/docs/atlas.en-us.sfdc_cli_guide/sfdc_cli_userguide_flow_tools.htm)
- [Flow Recursion and Trigger Interaction](https://developer.salesforce.com/docs/atlas.en-us.226.0/apex_code犬.content/apex_code_flow_interaction.htm)