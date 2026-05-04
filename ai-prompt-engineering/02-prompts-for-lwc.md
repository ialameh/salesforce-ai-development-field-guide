# P2. Prompts for LWC

Lightweight Web Components (LWC) sit at the intersection of Apex and the browser. AI-generated LWC code fails in different ways than Apex code: reactive property mismatches, wire adapter misuse, security assumptions about client-side validation, and promise handling bugs. This chapter covers the prompt patterns that work for LWC components.

## LWC failure modes

LWC fails differently than Apex:

1. **Reactivity mismatches** — AI uses the wrong reactive decorator or mutates a tracked property in a way that does not trigger a re-render
2. **Wire vs imperative** — AI picks the wrong data fetch strategy, using @wire when imperative Apex would be simpler and more controllable
3. **Client-side authorization** — AI hides buttons or disables features client-side, treating UI hiding as authorization (it is not)
4. **Promise chain breakage** — AI generates async LWC code that loses errors or creates unhandled promise rejections
5. **LDS misuse** — AI over-fetches or under-fetches data by not understanding Lightning Data Service's caching behavior

## The non-negotiable LWC constraints

Every LWC prompt must include:

```
Must:
- Use @track for reactive properties that change component behavior or appearance
- Use @api for public properties and methods (parent-child contract)
- Use correct import paths: Lightning.MessageContext, etc.
- Call Apex via @wire(cacheable=true) only for data that does not change during the user session
- Use imperative Apex for data that must be fresh on every call
- Always handle loading and error states in the template with if:true={loading} and if:true={error}
- Validate all @api property values in connectedCallback or renderedCallback
- Sanitize any user input before sending to Apex (no direct innerHTML without sanitization)

Must not:
- Use document.querySelector() or direct DOM manipulation (LWC blocks this)
- Hide authorization decisions in the template (buttons should be hidden server-side, not just with class="slds-hide")
- Store sensitive data in localStorage (it is not encrypted)
- Use System.debug() in client-side JavaScript (it goes to browser console, not Salesforce logs)
- Trust the client for business logic validation (validate in Apex, not just in JS)

Security:
- Never trust client-side field-level visibility (FLS is enforced in Apex, not in JS)
- All Apex methods called from LWC must enforce FLS and CRUD in Apex, not in the component
- Sanitize data returned from Apex before displaying (Lightning Locker blocks some XSS but not all)
```

## Wire adapter prompts

The @wire decorator is the most misused LWC feature. AI often uses wire when imperative would be better, or uses wire incorrectly (e.g., calling a non-cacheable method as cacheable).

**Prompt template:**
```
You are developer-AI. Write an LWC component that uses @wire to fetch data from an Apex method.

Apex method constraints:
- Only use @AuraEnabled(cacheable=true) for data that genuinely does not change during the session
- If the Apex method performs DML or must return fresh data on every call, it must NOT be cacheable
- For cacheable methods: the method must be idempotent and must not have side effects

Component requirements:
1. Import CurrentPageReference from 'lightning/navigation' if needed
2. Import { LightningElement } from 'lwc'
3. Use @track for reactive state: error, loading, data
4. Wire an Apex method that returns <specific object type>
5. Handle the wire response: { data, error } destructuring
6. Show loading spinner while data is undefined
7. Show error message if error is present
8. Render data in the template once loaded

Template requirements:
- Use <template if:true={loading}> for loading state (do not just show a spinner without state tracking)
- Use <template if:true={error}> to show error message with error.body.message
- Use <template if:true={wiredData}> to render data (never assume data is present)

Write the JS file (only the .js, not the HTML or XML).
```

**Example:**
```
Write accountList.js for an LWC component that displays a list of Accounts.

Requirements:
1. Use @wire to call getAccounts() Apex method (which is @AuraEnabled(cacheable=true))
2. Track error and loading state separately
3. Handle the wire response: { data, error }
4. Show Account.Name, Account.Industry, Account.AnnualRevenue for each account
5. Handle empty state: if data returns empty array, show "No accounts found" message
6. Sort accounts alphabetically by Name in the JS (do not rely on SOQL ORDER BY)

Apex method signature: @AuraEnabled(cacheable=true) public static List<Account> getAccounts()
Import: import getAccounts from '@salesforce/apex/AccountController.getAccounts'
```

## Imperative Apex prompts

Use imperative Apex when you need fresh data on every call, when you need to pass parameters dynamically, or when the wire adapter would cause unnecessary re-renders.

**Prompt template:**
```
You are developer-AI. Write an LWC component that uses imperative Apex calls (not @wire) for data that must be fresh.

Use imperative when:
- Data changes during the user session (e.g., after a form submit)
- You need to pass dynamic parameters at runtime
- The Apex method is not idempotent or has side effects
- You need fine-grained control over loading states

Component requirements:
1. Import getRecord from '@salesforce/apex/AccountController.getAccount' (imperative, not wire)
2. Call the Apex method imperatively with .then()/.catch() or async/await
3. Track loading state manually (not via wire reactive)
4. Handle errors with .catch() and display user-friendly error messages
5. On success, update @track decorated properties

Common pattern:
```javascript
import getAccounts from '@salesforce/apex/AccountController.getAccounts';

@track accounts;
@track loading = false;
@track error;

fetchAccounts() {
    this.loading = true;
    this.error = null;
    getAccounts({ industry: this.selectedIndustry })
        .then(data => {
            this.accounts = data;
            this.loading = false;
        })
        .catch(error => {
            this.error = error.body.message;
            this.loading = false;
        });
}
```

Write the complete JS file.
```

**Example:**
```
Write searchAccounts.js for an LWC component with a search input field.

Requirements:
1. User types an industry in an <lightning-input> field
2. On button click, call searchAccounts(industry) imperatively
3. Show loading spinner while waiting for Apex response
4. Display results or "No results" message
5. Handle errors gracefully

Apex method: @AuraEnabled public static List<Account> searchAccounts(String industry)
Import: import searchAccounts from '@salesforce/apex/AccountController.searchAccounts'

Additional requirements:
- Debounce the search: do not call Apex on every keystroke (wait 300ms after last keystroke)
- Cancel previous promises if a new search is initiated before the previous one resolves
- Clear previous error when a new search starts
```

## Parent-child component prompts

LWC components communicate via a public API (@api properties and methods) and via events. AI often gets the event naming or the property passing wrong.

**Prompt template:**
```
You are developer-AI. Write an LWC parent component and child component pair.

Parent: containerComponent.js
Child: listItemComponent.js

Data flow:
- Parent passes data to child via @api properties
- Child communicates back to parent via custom events

Child component requirements:
1. @api property to receive a single record: @api record
2. Fire custom event on action: this.dispatchEvent(new CustomEvent('recordclick', { detail: { id: this.record.Id } }))
3. Custom event name format: use kebab-case, e.g., 'recordselect' not 'recordSelect'
4. Event detail must contain all data the parent needs to act on the event

Parent component requirements:
1. Child component in parent template: <c-list-item record={account} onrecordclick={handleRecordClick}></c-list-item>
2. Handler receives the event: handleRecordClick(event) { const recordId = event.detail.id; }
3. Parent does not need to know the internal structure of the child record (child exposes what parent needs via event detail)

Additional LWC rules:
- Use LightningMessageService for cross-component communication within a flexmap page
- Use LMS.publish() and LMS.subscribe() for pub/sub patterns
- Never use window.postMessage() (Locker Service blocks this)

Write both JS files.
```

**Example:**
```
Write a parent component accountDashboard.js and child component accountTile.js.

Child (accountTile):
- @api account (type: Object with Id, Name, Industry, AnnualRevenue)
- @api compactMode (boolean, defaults to false)
- Button click fires 'accountselect' event with { id: this.account.Id, name: this.account.Name }
- Display: tile view when compactMode=false, list view when compactMode=true

Parent (accountDashboard):
- Fetches accounts via imperative Apex call
- Passes each account to accountTile component
- Listens for 'accountselect' event on each child
- On event: navigates to Account detail page using Lightning Navigation
- Shows count of selected accounts in a badge

Import navigation: import { NavigationMixin } from 'lightning/navigation'
```

## Security in LWC prompts

AI needs explicit reminders that client-side LWC code is not a security boundary.

```
You are developer-AI. Write an LWC component that handles sensitive data.

Security requirements:
1. All authorization happens in Apex, not in the LWC component
2. The LWC component must not assume the user has access to a field just because the template renders it
3. Use Security.stripInaccessible in the Apex controller, not in JS
4. If a field is not accessible, Apex returns null or omits the field from the response (handle this in the LWC by checking if the field exists before displaying)
5. Never log sensitive data in console.log() statements (browser console is accessible to browser extensions and debugging tools)
6. Sanitize any HTML content before setting innerHTML (use a library or Lightning Locker's sanitization if available)
7. If the component reads from localStorage, validate that the data is not tampered (use a checksum or verify structure)
```

## What this chapter covered

- The three LWC failure modes and how to prevent them
- The non-negotiable constraints for every LWC prompt
- Prompt patterns for @wire, imperative Apex, parent-child components, and security-critical components
- Why client-side authorization is not authorization at all

## References

- [LWC Documentation](https://developer.salesforce.com/docs/component-library/documentation/en/lwc)
- [Lightning Data Service](https://developer.salesforce.com/docs/component-library/documentation/en/lwc/data)
- [Security in LWC](https://developer.salesforce.com/docs/atlas.en-us.226.0/lwc/docs_security)