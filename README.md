# AI Slop Skill

**Find code that is bloated, shallow, semantically incomplete, architecturally confused, or technically correct only on the happy path.**

AI-generated code can fail in two opposite ways:

1. **It builds too much** — unnecessary abstractions, wrappers, dependencies, duplicate implementations, speculative infrastructure, and code that the platform or repository already provides.
2. **It understands too little** — CRUD-shaped implementations for workflows that actually require lifecycle states, invariants, concurrency rules, recovery, reconciliation, partial success, authorization boundaries, and secondary user paths.

AI Slop Skill is intended to catch **both**.

> **Completeness before minimalism. Minimize the implementation, never the domain contract.**

The project is a repository-quality skill/plugin for coding agents. It is not tied to a framework, language, product, or architecture. The goal is to make an agent reason like a senior engineer reviewing whether the system is both **necessary** and **complete** before it writes more code.

## Why this exists

A feature can compile, pass its happy-path test, and still be structurally incomplete.

Consider checkout. A simplistic implementation may support:

```
Cart → Pay → Order
```

That says almost nothing about the real contract.

A production checkout must answer questions such as:

- When may quantities, addresses, promotions, shipping methods, and payment methods change?
- Which mutations invalidate an existing quote or authorization?
- When must the cart become immutable?
- What prevents duplicate submission?
- What happens when the payment provider commits but the response is lost?
- Can multiple merchants succeed independently?
- How are taxes, shipping, duties, discounts, and totals recomputed?
- What happens when fulfillment is split, delayed, substituted, cancelled, or partially completed?
- Which system is authoritative when local and remote state disagree?
- How does the operation recover after network loss, process death, logout, or another device changing the same state?

Those are not exotic corner cases. They are part of the feature's semantics.

The same problem appears in authentication, subscriptions, uploads, browser automation, background jobs, notifications, returns, bookings, connectors, account deletion, AI-agent actions, and almost every workflow that crosses a trust or persistence boundary.

AI Slop Skill treats missing secondary paths as an architectural defect rather than waiting for production incidents to reveal them.

## Core model

The skill starts by understanding the actual repository and identifying the canonical owners of state, business rules, authorization, persistence, side effects, and external integrations.

It then models the feature as a lifecycle rather than a collection of screens or endpoints.

A useful generic lens is:

```
Intent
  ↓
Preconditions
  ↓
Prepared
  ↓
Awaiting Input / Approval
  ↓
Authorized
  ↓
Executing
  ↓
Indeterminate / Reconciliation
  ↓
Partially Completed / Completed / Failed / Cancelled
  ↓
Post-action lifecycle
```

This is a reasoning model, not a mandatory enum.

For every relevant state, the audit asks:

- What operations are permitted?
- What operations are mandatory?
- What operations are forbidden?
- What are the entry preconditions?
- What evidence proves the state?
- Which mutations invalidate it?
- Who may perform each transition?
- What requires approval or step-up authorization?
- What is the idempotency boundary?
- What happens under concurrent access?
- What happens on timeout?
- What can be retried?
- What must be reconciled instead of retried?
- What can be rolled back or compensated?
- What survives process death?
- What is actually terminal?

A mature implementation either makes illegal transitions impossible or rejects them at its authoritative boundary.

## What it audits

### Semantic completeness

Detect features that exist syntactically but cannot represent their real lifecycle: missing intermediate states, invalid transitions, post-commit lifecycle, cancellation, expiry, supersession, approval, compensation, reconciliation, partial outcomes, or recovery.

### Implicit edge cases

Derive edge cases from the feature's responsibilities rather than dumping a generic checklist. Trace concurrency, duplicate commands, stale revisions, retries, replay, remote success with a lost response, delayed/duplicated/reordered callbacks, process death, offline/reconnect behavior, account/device changes, external-system disagreement, irreversible side effects, and multi-party partial success.

### Architecture integrity

Detect when a repository has more than one answer to the same architectural question: duplicate sources of truth, parallel state owners, active legacy and replacement backends, UI state claiming success before the server, duplicate workflows, compatibility layers with no deletion condition, and documentation that contradicts live code.

Preserve verified invariants, not obsolete architecture merely because it already exists.

### Data-model expressiveness

A schema can itself be AI slop. Look for contradictory booleans, overloaded nullable fields, magic strings, lossy flattening, one-to-one models for inherently one-to-many relationships, fields that imply functionality that does not exist, and schemas incapable of representing partial or uncertain outcomes.

A persisted field is not proof of an implemented capability:

- A `reminderTime` is not a reminder engine.
- A `trackingNumber` is not a fulfillment model.
- An `idempotencyKey` field is not idempotent execution unless acquisition and replay behavior enforce it.

### Authorization and privacy

Authentication is not authorization. Audit resource ownership, operation scope, approval binding/expiry/replay, step-up or human approval, revocation during execution, cross-device approval, and the distinction between reading private data and disclosing it to an external party.

### Reconciliation

Distributed operations are not always simply success or failure. If a provider may have committed before a timeout or disconnect, blindly retrying can be destructive.

Where the domain requires it, model states such as:

```
UNKNOWN
INDETERMINATE
RECONCILING
```

and define the authoritative evidence that resolves them.

### Partial success

Look for all-or-nothing models hiding partial payment/refund/fulfillment/return, multi-merchant checkout, multi-file processing, batch imports, multi-recipient actions, and partially completed background work.

### Recovery and durability

Ask what happens after process death, app/browser close, network loss, provider outage, credential expiry, logout, account switching, worker failure, restart, or cancellation. Persisting a status is insufficient if nothing can safely resume, reconcile, or terminate the operation.

### Cross-layer consistency

Audit the real execution path:

```
schema ↔ domain model ↔ backend ↔ jobs/webhooks ↔ external provider ↔ client state ↔ UI ↔ tests
```

Do not declare a feature complete because one screen or endpoint looks correct.

### Code and abstraction slop

Also target the familiar form of AI-generated slop: needless abstractions, speculative generalization, unnecessary wrappers, duplicate helpers, reinvention of platform features, needless dependencies, parallel implementations, premature infrastructure, dead compatibility paths, excessive indirection, copy/paste logic, and abstractions with no semantic purpose.

The target is not the fewest lines. It is the **smallest implementation that satisfies the complete contract**.

## AI Slop vs. Ponytail

[Ponytail](https://github.com/DietrichGebert/ponytail) is an excellent example of attacking one side of this problem.

Ponytail focuses on implementation minimalism: ask whether code needs to exist, then prefer reuse, the standard library, native platform capabilities, installed dependencies, and finally the minimum new implementation. It explicitly protects validation, security, accessibility, and data-loss handling while simplifying.

AI Slop Skill is complementary rather than a reimplementation of Ponytail.

| | Ponytail | AI Slop Skill |
|---|---|---|
| Primary question | How little code should we write? | What is the minimum **complete** system? |
| YAGNI / overengineering | Core | Core |
| Reuse native/platform functionality | Core | Core |
| Unnecessary abstractions | Core | Core |
| Implicit edge-case discovery | Secondary | Core |
| Semantic lifecycle modeling | Not the primary model | Core |
| Permitted/forbidden transitions | Not the primary model | Core |
| Indeterminate outcomes/reconciliation | Not systematic | Core |
| Partial-success modeling | Not systematic | Core |
| Cross-device concurrency | Not systematic | Core |
| Schema expressiveness | Not systematic | Core |
| Architecture/source-of-truth conflicts | Some overlap | Core |
| Recovery/process-death semantics | Not systematic | Core |
| Pragmatic programming principles | Some overlap | Planned module |

The distinction:

```
Ponytail:
50-line problem → don't build 1,500 lines.

AI Slop Skill:
15-state problem → don't pretend it is 3-state CRUD.
```

These philosophies belong together.

AI Slop Skill therefore uses this ordering:

```
Understand the repository
        ↓
Understand the domain
        ↓
Discover implicit states and edge cases
        ↓
Define invariants and ownership
        ↓
Establish the complete semantic contract
        ↓
Find the simplest architecture that can represent it
        ↓
Reuse existing / standard / native capabilities
        ↓
Implement the minimum correct code
        ↓
Verify the complete contract
```

**Completeness comes before minimalism.** Otherwise simplification can merely delete requirements the implementation failed to notice.

Ponytail and AI Slop Skill can therefore be used together rather than treated as mutually exclusive tools.

## Planned skill architecture

The project is intentionally modular.

```
AI Slop
│
├── semantic-completeness
│   ├── implicit-edge-cases
│   ├── state-transitions
│   ├── concurrency
│   ├── idempotency
│   ├── reconciliation
│   ├── recovery
│   └── partial-success
│
├── architecture-integrity
│   ├── source-of-truth
│   ├── ownership
│   ├── boundaries
│   ├── legacy-path detection
│   └── cross-layer contracts
│
├── implementation-quality
│   ├── YAGNI
│   ├── reuse
│   ├── native-first
│   ├── unnecessary abstractions
│   └── minimum-correct-code
│
├── code-smells
│
└── pragmatic-programming
    ├── DRY
    ├── orthogonality
    ├── reversibility
    ├── decoupling
    ├── domain languages
    ├── contracts
    ├── assertions
    ├── resource ownership
    └── transformation over inheritance
```

The Pragmatic Programming section is a direction, not a claim that every principle is implemented today. Add modules only when they provide concrete auditing behavior.

## Finding vocabulary

Audits should classify concrete defects:

- **CONTRACT GAP** — required behavior is undefined.
- **STATE GAP** — a valid lifecycle state or transition cannot be represented.
- **AUTH GAP** — permission, approval, ownership, or disclosure semantics are incomplete.
- **CONCURRENCY GAP** — race, revision, lock, or ordering behavior is undefined.
- **IDEMPOTENCY GAP** — replay can duplicate or corrupt effects.
- **RECONCILIATION GAP** — uncertain or divergent outcomes cannot be resolved safely.
- **RECOVERY GAP** — interruption loses or corrupts progress.
- **PARTIAL-SUCCESS GAP** — all-or-nothing modeling hides legitimate mixed outcomes.
- **DATA-MODEL GAP** — the schema cannot represent valid real-world state.
- **UX GAP** — the user cannot understand or recover from canonical state.
- **OBSERVABILITY GAP** — insufficient evidence exists to diagnose or reconcile.
- **IMPLEMENTATION CONFLICT** — layers or implementations assert incompatible ownership or behavior.
- **SLOP** — unnecessary code, abstraction, dependency, duplication, or complexity with no required semantic purpose.

A finding should identify a concrete state, transition, invariant, race, recovery path, architectural conflict, or unnecessary implementation. Generic "best practices" are not findings.

## What this project should not become

AI Slop Skill should not become another giant prompt that mechanically demands every possible enterprise pattern.

It should not:

- manufacture edge cases the platform makes impossible;
- introduce distributed-systems machinery into a local pure function;
- demand abstractions merely because a design pattern exists;
- preserve legacy architecture without a bounded migration reason;
- confuse more states with better modeling;
- optimize line count at the expense of correctness;
- generate checklist findings without repository evidence;
- turn every feature into a framework.

The audit is evidence-driven.

**Complexity must be justified by the real domain. Simplicity must also be justified by the real domain.**

## Example

Suppose an order implementation contains:

```
status: PENDING | PAID | SHIPPED | DELIVERED
trackingNumber?: string
returnRequested: boolean
```

A shallow audit might conclude that ordering, tracking, and returns are implemented.

AI Slop Skill asks whether the product actually requires states such as authorization without capture, unknown payment outcome, partial capture, cancellation after authorization, split shipments, backorders, delivery exceptions, lost shipments, replacements, partial-quantity returns, return authorization, carrier receipt, inspection, rejected returns, partial refunds, exchanges, or disputes.

If valid product states are structurally impossible to represent, the feature is incomplete even though its UI works.

The remediation is **not automatically to build all of them**.

First establish which states genuinely belong to the product contract. Then choose the smallest model and implementation capable of representing them correctly.

That is the difference between eliminating AI slop and simply generating more code to handle edge cases.

## Status

Early development.

The first module is the **implicit-edge-case / semantic-completeness auditor**. Architecture integrity and implementation-quality rules are part of the initial scope. Pragmatic Programming principles will be expanded as independent, evidence-driven modules.

The repository is being designed for portable agent-skill/plugin packaging rather than for one application or one coding assistant.

## Principle

> **Don't ask only whether the code works. Ask whether it represents the whole problem — and whether every line needed to represent that problem deserves to exist.**
