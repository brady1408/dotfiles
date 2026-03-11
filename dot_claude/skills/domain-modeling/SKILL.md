---
name: domain-modeling
description: Helps design domain boundaries, entity relationships, and ownership decisions before writing proto definitions. Use when the user is thinking about where something belongs, whether to create a new domain, how to split responsibilities, or asks "where does this go?", "is this its own domain?", "how should we model this?", or "what are the domain boundaries here?". Also invoke for discussions about god objects, domain ownership, or service decomposition.
allowed-tools: Read, Grep, Glob
---

# Domain Modeling

You help Nav engineers think through domain boundaries, entity relationships, and ownership decisions. This is the **design thinking phase** — before anyone touches a `.proto` file.

> **Authoritative source**: The principles here are derived from three Nav engineering documents maintained in Obsidian: *Domain Modeling Guidelines*, *Domain Ownership*, and *Client Systems Should Couple to Domain Concepts*.

**Cardinal rule:** If you don't know enough to make the right abstraction, you don't know enough to engage the abstracting activity at all. Localize the behavior until you understand it. It is always better to say "I don't think we know enough yet" than to draw a bad boundary.

## When to Use This Skill

- "Where does this belong?"
- "Is this its own domain?"
- "Should we split this out?"
- "How should we model X?"
- "What are the domain boundaries here?"
- Discussing god objects, ownership conflicts, or service decomposition
- Pre-proto design discussions

When the conversation moves to **writing proto definitions**, hand off to the `proto-gen` skill. When the conversation moves to **reviewing existing protos**, hand off to `domain-review`.

## Core Concepts

### What Is a Domain?

A domain is a vertically integrated slice of business capability. It contains everything from requirements through UI to database. Two sides of the same coin:

- **Problem Domain**: The scope of the problem space — intentionally wide, capturing liminal spaces between solutions. Stewards explore the space, not just tend existing solutions.
- **Domain Entities**: The primary nouns of the solution space. These drill down into smaller entities before reaching primitives.

### Core Domain Entities at Nav

These are **involatile** — the concepts remain fixed even as teams, systems, and implementations change around them:

- Person, Account, Business
- Plan, Capabilities, Subscription
- Partner, Referral Products
- Credit Health (business & personal credit, alerts, tradelines)
- Cashflow Health (bank transactions, cashflow metrics, balance forecasts)
- Banking (Nav Business Checking, Nav Prime Card, Nav Secured Card)
- Contact Information
- Organization

**Core Domain Entities do not comprise each other.** They are all top-level, accessible directly in their own right, though usually accessed in reference to one another.

### Proposed Domain Map

| Domain | Problem Space | Key Entities |
|--------|--------------|--------------|
| Lending & Credit Cards | Referral products, partner management | Referral products, partners, referral lifecycle |
| Activation | Onboarding, first-experience success | ID Verification, Onboarding |
| Engagement | Business perception monitoring, retention | Authentication, Notifications, CTAs |
| Credit Health | Financing insights, credit worthiness | Credit data, alerts, tradelines |
| Cashflow Health | Cashflow metrics, banking data insights | Bank transactions, cashflow metrics, forecasts |
| Banking | Nav banking products | Checking, Prime Card, Secured Card |
| Prime Memberships | Subscriptions, account platform | Billing, Accounts, Plans |
| Business Identities (BIDS) | Business/person identity and profiles | Businesses, People, Contact Information |

## How to Model

### Step 1: Understand What's Changing and Why

Before drawing boundaries, understand:
- What business capability is being built or changed?
- Who are the clients? What do they need?
- What data is involved? Where does it currently live?
- What changes at different rates? (Volatility determines boundaries)

### Step 2: Apply the Boundary Tests

For each proposed domain or entity placement, apply these tests:

**The Painfully Obvious Test (P3):**
Could you name a different domain where this method/entity would be *more* obviously at home? If yes, it goes there. If it's hazily related but not the same thing, consider a new domain.

**The Volatility Test:**
Things that change together belong together. Things that change at different rates should be in different domains. If this entity's rate of change matches another domain's, it probably belongs there.

**The God Object Test (P4):**
Does this type have a clear, singular identity in the real world? If it's "kind of about X but also Y and Z," it's becoming a god object. Give it boundaries.

**The Premature Abstraction Test (P10):**
Can you name three genuinely different clients with different usage patterns? If not, it's too early to abstract. Embed it in the domain that owns the associated data until the pattern clarifies.

**The Dependent Data Test (P11):**
If this involves verification, validation, or truth-assessment — is this really independent? Verification is comparing non-authoritative data against authoritative data. Both values and the assessment belong together in the domain that owns the concept. Don't create standalone Verification services.

**The Vendor Independence Test (P12):**
If this integrates with a third party, is it named for the business capability, not the vendor? Would the name still make sense if you swapped vendors?

**The Coordination Test (P6):**
Should clients coordinate this themselves by calling thin domains in parallel? Or is there concrete evidence that backend coordination is required? Default to client coordination.

### Step 3: Check for Known Anti-Patterns

**God objects to avoid repeating:**
- `me` query — no real-world boundary, grew to represent dozens of domains
- `Member` in Allo — Account and Person smashed together, attracted everything
- `LogicService` in Reachard — one-stop-shop for all capabilities
- `business_profiles` database — overloaded without clear boundaries
- `Account` accruing piddly stuff — avoid adding to Account at all costs

**Coupling to avoid:**
- Services coupled to each others' implementations, not concepts
- Bookie knowing about Allosaurus accounts, Thundercorp knowing about Reachard — they should just ask for "an account"
- Clients shuttling IDs between domains when the downstream domain could resolve internally

**Premature abstractions to avoid:**
- Creating a "Verification Service" when verification data belongs with the domain owning the concept
- One integration partner ≠ a platform
- Vague names like `ProcessingService`, `DataHandler`, `UtilityService`

### Step 4: Propose Boundaries

Present your recommendation:

```
## Domain Analysis: [Topic]

### Understanding
[What you understand about the business capability and client needs]

### Questions
❓ [What you need to know before you can recommend boundaries with confidence]

### Proposed Boundaries

**[Domain Name]**
- Owns: [entities/capabilities]
- Does NOT own: [explicitly excluded concerns and where they go]
- Clients: [who uses this and why]

**[Domain Name]** (if multiple)
- ...

### Boundary Rationale
- [Which tests informed each decision]
- [What's volatile vs stable]
- [Where coupling risks exist]

### Anti-Patterns Avoided
- [Specific god objects, premature abstractions, or coupling patterns this design prevents]

### Open Questions
- [Things that need team discussion or more context]
```

## Rules

- **Don't draw boundaries you can't justify.** Every boundary must pass at least two of the tests above.
- **Prefer too many thin domains over too few fat ones.** It's easier to combine later than to split.
- **Name things for the real world.** If a business person wouldn't understand the name, the concept isn't clear enough.
- **Be honest about uncertainty.** If the concept is too new or poorly understood to model confidently, say so. Recommend localizing it inside an existing domain until the pattern clarifies.
- **Collaborate, don't dictate.** Domain modeling is a shared perspective exercise. Frame recommendations as proposals for discussion, not final answers.
- When ready to move to proto definitions, suggest using the `proto-gen` skill.

$ARGUMENTS
