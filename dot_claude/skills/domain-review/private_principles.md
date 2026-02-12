# Nav Domain API Principles

These principles are derived from Nav's domain design philosophy. Apply all applicable principles during every review.

**Cardinal rule:** If you don't understand enough to know whether the design is right, say so and ask questions. It is always better to stop and clarify than to proceed with a bad abstraction. An API is inherently an abstraction — if you don't know enough to make the right abstraction, you don't know enough to engage the abstracting activity at all. Localize the behavior until you understand it.

---

## Part 1: Domain Design Principles

### P1: No Database Leaks

The API must not leak storage details. The lowest level you are allowed to think about is the existing API specification.

**Watch for:**
- Field names that mirror DB columns but don't match domain language (e.g., `allosaurus_member_id` instead of `person_id`)
- Methods named after CRUD operations on tables rather than business intentions (e.g., `DeleteFollow` instead of `UnfollowBusiness`, `CreateAccountOwner` instead of `RegisterAccount`)
- Response shapes that look like `SELECT *` — too many fields, internal IDs, audit columns exposed without clear client need
- Foreign keys from internal schemas surfaced to clients (e.g., `partner_id` in Plan responses when clients should get partner info from the Partner domain)
- Never assume the best interface just ships the DB schema — they evolve at different rates
- **Do NOT reuse the same message types for RPC APIs and storage.** The needs of long-term storage and live RPC services diverge over time. Separate types let storage format change without impacting clients.

**Key test:** If someone who only understood the business domain (not the database) read this proto, would any field or method name confuse them or reference something they wouldn't know about?

---

### P2: Reflect Client Intentions

Every method and field must have a clear answer to "what is the client trying to accomplish?" Clients are naturally closer to the customer and therefore closer to the real world. Design for the client's usage patterns, not the abstraction.

**Watch for:**
- Methods named for data mutations instead of business actions (`DeleteFollow` vs `UnfollowBusiness`, `CreateAccount + CreateAccountOwner + CreateOwnedBusiness` vs `RegisterAccount`)
- Methods that return data the client must filter or transform to answer their real question — the domain should answer the question directly (e.g., `GetPartner` returning the best-fit partner, not `ListPartnerships` with a flag to filter by)
- Status codes/enums that expose too much internal granularity for the audience. Cross-domain clients rarely need `application_not_started, application_started, application_submitted...` — they may just need `should_apply` or `application_started_at`
- Contracts that speak in terms of business processes rather than underlying entities are often better (`RegisterAccount` verb rather than coordinating `CreateAccount`, `CreateAccountOwner`, `CreateOwnedBusiness`)

**Key test:** "I need X" should always be met with "What do you really need out of X?" — if the method answers the proxy question instead of the real intention, it's leaking.

---

### P3: Domains Are Thin and Standalone

Each domain is a narrow, well-bounded slice of business capability. We demand finer-grained domains rather than domains with lots of hazily related behavior. The root domain is broad, not deep — we are not designing a graph.

Don't assume your project is the center of the universe. Think and design for conservative possibility.

**Watch for:**
- A service definition covering multiple loosely related concepts that should be split (e.g., Plan and Capabilities are separate domains even though related)
- Methods without a "painfully obvious" home in this domain — if it's not painfully obvious, it's wrong
- Meta-layers disguised as part of the domain (e.g., plan recommendations are NOT part of Plan — they're a PlanRecommendation domain)
- Domains embedding other domains (Plan should not embed Capabilities — clients call each in parallel)
- Core Domain Entities do not comprise each other — they are all top-level, accessible directly in their own right

**Key test:** Could you name a different domain where this method would be *more* obviously at home? If yes, it goes there. If it's hazily related but not exactly the same thing, consider a new domain.

**Examples from Nav:**
- Plan ≠ Capabilities ≠ PlanRecommendation (three separate domains)
- Cashflow Metrics ≠ Banking Transactions (probably separate)
- `id_verified_state` outgrew Person into its own domain
- Business is shallow — AGR belongs in Cashflow, not Business
- Contact Information may warrant its own domain if it's truly the same thing across Person, Business, and Bureau

---

### P4: No God Objects

Watch for types or services that accumulate unrelated concerns. These are natural places that are ambiguously defined and therefore attract garbage-dumping.

**Watch for:**
- Message types with fields spanning multiple domain concepts (e.g., Member with account info, person info, bureau IDs, billing codes, plan codes)
- Services that are a one-stop-shop for everything a system can do internally
- Account accruing piddly stuff because no domain is adequately designed for it — avoid adding stuff to Account at all costs
- Catch-all query objects like `me` in GraphQL that don't correspond to anything in the real world
- **Messages with hundreds of fields.** In C++ every field adds ~65 bits to the in-memory object size whether populated or not. Java has hard method size limits. If a message is growing large, it's a design smell — split it.

**Known Nav god objects to avoid repeating:**
- `me` query — no real-world boundary, grew to represent details from a dozen domains
- `Member` in Allo — Account and Person smashed together, attracted bureau IDs, billing codes, plan codes
- `LogicService` in Reachard — one-stop-shop for all internal capabilities
- `business_profiles` database — overloaded without clear domain boundaries

**Key test:** Does this type have a clear, singular identity in the real world? If it's "kind of about X but also Y and Z," it's becoming a god object.

---

### P5: Keep It Minimal

Minimalism, purity, and intentionality is the dividing line between good and bad API design. Only surface data that has well-understood use cases.

**Watch for:**
- Fields without a known client use case — for each field, you should be able to name who uses it and why
- `created_at`, `updated_at`, or similar audit fields exposed without concrete display need (these invite coupling to domain-internal details instead of intention-revealing interfaces like "is this recent enough?")
- Returning data from other domains because the implementation happens to have it (e.g., Plan returning `partner_id`)
- Overly comprehensive responses when thinner interfaces would serve clients better
- **Booleans for states that might grow.** If a field has two states *now* but could have more later, use an enum. `PhotoType` beats `bool gif`. Future-proof the vocabulary.

**Key test:** If you removed this field, which client would break and why? If you can't answer that specifically, the field shouldn't be there.

---

### P6: Clients Coordinate, Not the Backend

Clients should coordinate their own needs for their own use cases. Clients decide how to coordinate thin slices — we don't tightly couple to our assumptions about anything they might need. Shifting coordination downward is a performance optimization — apply only with concrete evidence.

**Watch for:**
- Methods bundling multiple domain concepts into one response to "save the client a call"
- `GetEverything` or `GetXWithY` endpoints combining unrelated domains
- Cross-domain aggregation endpoints — every one adds a layer to the call stack, more complexity, more single points of failure
- Backend services coupled to ephemeral UI requirements

**What to do instead:**
- Expose independent, parallelizable calls. Business Contact Info and Ownership History should be separate calls.
- It's far better for a UI to call 5 domains in parallel than to build 1 aggregated backend call that does the same fanout
- Exception: concrete performance evidence (not speculation), or the backend is rendering HTML

**Key test:** Is this endpoint serving a specific UI page's layout? If yes, it's probably over-fitted to ephemeral requirements.

---

### P7: Domains Own Their Logic

If a question belongs to a domain, the answer should come from behind its interface. Domain logic should be hidden from clients.

**Watch for:**
- Clients fetching from Domain A just to pass it to Domain B — if Domain B needs Domain A's data, Domain B fetches it internally (e.g., clients don't fetch Partners to pass to Plan and Capabilities)
- Business logic that belongs behind the domain interface but is pushed to clients (e.g., "which partner is primary?" logic being done client-side instead of behind `GetPartner`)
- IDs that clients must shuttle between domains — consider whether the downstream domain can resolve them internally
- Domains should not require sequential calls from clients — translate IDs internally or change upstream to make them available innately

**Key test:** Is the client doing logic that is really about this domain's business rules? If yes, move it behind the interface.

---

### P8: Use Domain Language, Not Implementation Language

Interfaces should speak in terms of Nav's understanding of reality and business processes, not implementation details.

**Canonical Nav vocabulary:**
- `person_id` — not `member_id`, `user_id`, `owner_id`
- `account_id` — not `allosaurus_account_id`, `reachard_account_id`, `member_id`
- `business_id` — not `enterprise_id`, `company_id`
- Person — a human being (real thing)
- Account — a relationship with Nav initiated by a Person (real thing)
- Member — NOT a real thing, blurs Person and Account

**Watch for:**
- Field names referencing specific systems (Allo, Depot, Reachard, Bookie, etc.)
- Service names that reflect the backing service name rather than the domain concept
- Enum values that describe system states rather than business states
- Details like `allosaurus_member_id` that betray implementation thinking

**Key test:** Does this name make sense to someone who understands Nav's business but has never seen our codebase?

---

### P9: Migration Permissiveness (Requests Only)

Acknowledge that clients are currently coupled to legacy implementations. Be permissive on input, strict on output.

**Rules:**
- Deprecated request paths accepting implementation-specific IDs are acceptable (e.g., `get_business_by_allosaurus_account_id`) alongside the preferred domain path (`get_business_by_account_id`)
- Mark legacy paths as deprecated with comments explaining the preferred alternative
- Response shapes must ALWAYS be domain-pure regardless of which request path was used
- New API contracts can include deprecated details where it helps illustrate future intent

**Key test:** Is the response contaminated by the legacy request path? If a deprecated request field changes the response shape, that's a violation.

---

### P10: Don't Abstract What You Don't Understand

It's always the right time to define your abstraction correctly — if you're going to make one. But if you don't know enough to make the right abstraction, you don't know enough to engage the abstracting activity. Localize the behavior until you understand it.

This principle is about intellectual honesty. The two right options when facing uncertainty are:
1. **Don't offer an API at all** — this is not "a thing" at the company level until we know how to generalize it. Work harder on understanding what's actually volatile.
2. **Make it an embedded component** of the domain that owns the associated data, until the pattern clarifies.

**Watch for:**
- New domains or services being created around concepts that are poorly understood or have only one use case so far
- APIs that try to generalize before the problem space has been explored (one integration partner ≠ a platform)
- Service boundaries drawn around implementation structure rather than business understanding
- Naming that reveals the designer doesn't yet know what this thing *is* — vague names like `ProcessingService`, `DataHandler`, `UtilityService`

**What to do instead:**
- Ask: "Can we name three genuinely different clients with different usage patterns for this?" If no, it's too early to abstract.
- Embed the capability inside the domain that owns the data, and let it grow until the pattern is clear.
- Build the tight coupling intentionally and clearly (see P13), with a plan to revisit.

**Key test:** Can you explain what this abstraction is *for* without referencing the current project? If the only justification is "we need it for X right now," it's premature.

---

### P11: Dependent Data Lives Together

Verification, validation, enrichment, and truth-assessment are not independent domains. They are properties of the data they assess and belong alongside that data, under the same domain roof.

The pattern: a data point is a value from some source. That source is either authoritative or it is not.
- If authoritative, "verification" is meaningless — the data just *is*.
- If non-authoritative, verification is the result of comparing the non-authoritative data point against an authoritative one.

The non-authoritative value, the authoritative value (if available), and the assessment result all belong together — under whatever domain owns the concept being described.

**Example:** A customer says they have 50 employees. A bureau says 47. Most clients want the truth; a settings page wants what the customer said; sometimes the discrepancy matters. Splitting this into a "Truth Service", a "Maybe Lies Service", and a "Verification Service" creates enormous integration complexity for every consumer. Simpler: offer all three together under the Business domain (or its subdomain, like Employment).

**Watch for:**
- Standalone "Verification" or "Validation" domains that are really just comparing data from two sources
- Services whose only job is to say "is X true?" about data owned by another domain
- Clients forced to call multiple services to assemble what is conceptually one idea (the customer-reported value, the authoritative value, and whether they match)
- ID Verification, Address Verification, Business Verification proposed as independent services when the verified data already has a natural home

**What to do instead:**
- Keep the non-authoritative data, the authoritative data, and the verification result together in the domain that owns the concept
- The domain can internally call authoritative sources — clients shouldn't have to

**Key test:** Would a client ever want the "verification" without also wanting the data being verified? If not, they belong together.

---

### P12: Serve Client Goals, Not Third-Party Structure

Design services around what clients need, not around the third parties you integrate with. A service named after a vendor is a service that will need to be renamed or replaced.

**The BMS principle:** BMS (Billing & Membership Service) was originally built to couple to Stripe. But it was named for its *business purpose*, not its vendor — and when requirements grew beyond Stripe, BMS adapted. "I can derive capabilities from how you pay us — oh, that's not good enough anymore? I'll derive capabilities from any detail. I don't care, just here to get you what you need." It serves client requirements exclusively, built with flexibility that lets Nav break out growing requirements into dedicated services without rebuilding the ecosystem.

Every third-party-facing service needs to adopt this moral. The alternative is Allosaurus.

**Watch for:**
- Services named after vendors or third parties (`StripeService`, `ExperianService`, `PlaidService`)
- APIs whose method signatures mirror a vendor's API rather than describing what the client needs
- Assumptions that the current vendor or data source is permanent — design for the *capability*, not the *provider*
- Service boundaries drawn around "we talk to X" rather than "we answer question Y for clients"

**What to do instead:**
- Name the service for the business capability it provides
- Design the interface around client questions, not vendor responses
- Accept that you might swap vendors — if the interface is right, that's an implementation detail

**Key test:** If you replaced the underlying vendor tomorrow, would the service name and interface still make sense? If not, the abstraction is wrong.

---

### P13: Tight Coupling Is Valid — If Temporary and Intentional

There is a principle at Nav: "if you're going to tightly couple to a service, actually do it clearly." The existence of Reachard and Allo protos demonstrates this. Tight coupling is valid when:
1. You know the coupling is temporary
2. The coupling is explicit and clearly marked (not hidden behind a façade pretending to be general)
3. There is an active follow-up to figure out what's needed more generally, so you can trash the tight coupling and build the sustainable thing

**Watch for:**
- Tight coupling disguised as a general abstraction — this is worse than honest tight coupling because it's harder to find and replace later
- "Temporary" interfaces that have no plan or timeline for generalization
- Tight coupling that has quietly become permanent (flag it — it's time to revisit)

**What to do instead:**
- If tight coupling is necessary, make it obvious: use clear naming, put it in a dedicated package, document what the generalized replacement should look like
- Set a concrete trigger for when to revisit (e.g., "when a second client needs this", "when we add a second vendor")

**Key test:** Is this tight coupling honest and planned, or is it accidental and growing? Honest coupling with a plan is fine. Accidental coupling is a fire.

---

## Part 2: Protobuf Technical Rules

These rules come from protobuf.dev best practices and protect wire compatibility, serialization safety, and long-term schema evolution. Violations here cause data loss and deserialization failures.

### T1: Never Reuse a Tag Number

Once a tag number has been used, it is permanently claimed — even if the field is deleted. Old serialized data may still exist in logs, caches, and other services. Reusing a tag causes silent data corruption on deserialization.

**When deleting a field:** Reserve the tag number AND the field name.
```protobuf
reserved 2, 3;
reserved "old_field_name", "another_removed_field";
```

### T2: Never Change a Field's Type

Changing a field's type breaks deserialization the same way tag reuse does. Limited exceptions exist for compatible numeric types (int32/uint32/int64/bool), but changing between message types, strings, and bytes is not safe.

### T3: Never Add Required Fields

Proto3 removed `required` for good reason. Use `// required` comments to document API contracts. Adding a `required` field to an existing message breaks all existing clients that don't populate it.

### T4: Reserve Deleted Fields and Enum Values

When removing a field or enum value, always reserve its number and name. This prevents future developers from accidentally reusing a tag that has serialized data in the wild.

```protobuf
enum MyEnum {
  MY_ENUM_UNSPECIFIED = 0;
  // MY_ENUM_OLD_VALUE = 1;  // Removed — do NOT reuse
  reserved 1;
  reserved "MY_ENUM_OLD_VALUE";
  MY_ENUM_NEW_VALUE = 2;
}
```

### T5: Enums Must Have an UNSPECIFIED Default

Every enum's first value (tag 0) must be `FOO_UNSPECIFIED`. This is the zero-value default and must not carry business semantics. If the zero-value means something specific, you can't distinguish "explicitly set to the default" from "not set at all."

Prefix enum values with the enum name to avoid C++ namespace collisions:
```protobuf
enum ActivationStatus {
  ACTIVATION_STATUS_UNSPECIFIED = 0;
  ACTIVATION_STATUS_ACTIVE = 1;
}
```

### T6: Don't Go From Repeated to Scalar

Changing a repeated field to scalar (or vice versa, in proto3 packed) causes data loss. In JSON, the entire message may be lost. Plan field cardinality correctly from the start.

### T7: One Top-Level Entity Per File (1-1-1 Rule)

Each `.proto` file should contain one top-level message, enum, or service definition. This makes refactoring easier, reduces transitive dependency bloat, and keeps build targets lean. Exceptions: cyclic dependencies or tightly coupled message pairs.

### T8: Use Well-Known Types

Don't reinvent standard types. Use `google.protobuf.Timestamp`, `google.protobuf.Duration`, `google.type.Date`, `google.type.Money`, `google.protobuf.FieldMask`, etc. Custom timestamp or money types are a maintenance burden and interop hazard.

### T9: Don't Rely on Serialization Stability

Proto serialization order is NOT guaranteed across builds or binaries. Never use serialized protos as cache keys, checksums, or content-addressable identifiers.

### T10: Don't Use Text Format for Interchange

Text format and JSON serialize field names as strings. If you rename a field, text format deserialization breaks. Use binary serialization for machine-to-machine interchange. Reserve text format for human debugging and configuration.

### T11: Don't Change Default Values

Changing a field's default value causes version skew between clients and servers reading the same data. Proto3 eliminated custom defaults for this reason — don't fight it.

### T12: Don't Cargo-Cult Proto Options

When creating new proto definitions, don't copy option settings from existing files unless you understand why they're there. Options like editions features typically signal experimental or deprecated behaviors. A new proto should be clean by default.

---

## Part 3: When to Stop and Ask

These principles are only useful if applied with intellectual honesty. When reviewing or generating proto definitions, **stop and ask the user** if any of the following are true:

- **The domain boundaries are unclear.** If you can't determine with confidence where a method belongs, say so. Don't guess — ask what the business capability actually is.
- **The client intention is ambiguous.** If "what is the client trying to accomplish?" doesn't have a clear answer from the available context, do not proceed. Ask.
- **The abstraction feels premature.** If the service or domain being designed has only one known use case and no clear generalization, flag it. Ask whether this should be localized first.
- **Verification/truth questions arise.** If the design involves comparing authoritative vs. non-authoritative data, ask whether this is truly independent or should be embedded in the domain that owns the data.
- **A service is being named after a vendor or system.** Stop and ask what business capability this serves, independent of the vendor.
- **You're not sure if tight coupling is intentional.** Ask whether there's a plan to generalize, or if this is accidental coupling that should be addressed.
- **Field or method naming feels forced.** If you can't find a name that a business person would understand, the concept itself may not be well-enough understood to build an API for.

The cost of asking is low. The cost of a bad interface is years of coupling pain.

---

## Relationships Between Principles

These principles reinforce each other:
- P1 (no DB leaks) + P2 (reflect intentions) → methods named for business actions, not table operations
- P3 (thin domains) + P4 (no god objects) → many small focused domains, never dump stuff into Account
- P5 (minimal) + P7 (domains own logic) → only expose what clients need, hide everything else behind the interface
- P6 (clients coordinate) + P7 (domains own logic) → clients call domains in parallel, domains fetch their own dependencies
- P8 (domain language) + P9 (migration permissive) → responses always use domain language, requests may temporarily accept implementation language
- P10 (don't abstract prematurely) + P11 (dependent data lives together) → don't create a "Verification" domain, embed it until you understand the pattern
- P12 (serve client goals) + P6 (clients coordinate) → services built for client needs, not vendor structure or backend convenience
- P13 (honest tight coupling) + P10 (don't abstract prematurely) → if you must couple, do it clearly with a plan to revisit
- T1-T12 (protobuf rules) protect the wire — domain principles protect the architecture, technical rules protect the bytes
