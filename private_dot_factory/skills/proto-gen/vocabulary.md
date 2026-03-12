# Nav Canonical Vocabulary

Use these terms in proto definitions. When introducing new terms, prefer real-world nouns and business-process verbs. If you find yourself naming something after a system (Allo, Depot, Reachard, Bookie, etc.), stop — that's implementation language.

## Identifiers

| Use This | Not This | Why |
|----------|----------|-----|
| `person_id` | `member_id`, `user_id`, `owner_id` | A Person is a real thing — a human being |
| `account_id` | `allosaurus_account_id`, `reachard_account_id`, `member_id` | An Account is a real thing — a relationship with Nav |
| `business_id` | `enterprise_id`, `company_id` | Business is the canonical Nav domain entity |

## Entities

| Domain Term | Definition | Not This |
|-------------|------------|----------|
| Person | A human being | Member (blurs Person and Account) |
| Account | A relationship with Nav initiated by a Person | Allosaurus account, Reachard account |
| Business | A business entity — core identity facts | Business profile (implementation) |
| Contact Information | Address, phone, email — may be its own domain | Inline copies per domain |
| Plan | What plan a customer has | Subscription (ambiguous) |
| Capabilities | What a plan grants access to | Plan features (not part of Plan domain) |
| Partner | A lending/issuing partner | — |
| Registration | The singular action of creating an Account | CreateAccount + CreateAccountOwner + CreateOwnedBusiness |

## Methods — Name for Intentions

| Use This | Not This | Principle |
|----------|----------|-----------|
| `RegisterAccount` | `CreateAccount`, `CreateAccountOwner` | Business process, not DB operation |
| `UnfollowBusiness` | `DeleteFollow` | Client intention, not table mutation |
| `GetPartner` | `ListPartnerships` (when client needs the relevant one) | Domain answers the question directly |
| `GetCustomerPlan` | `GetPlanByAccountId` | Speaks to intention, not lookup key |

## Status Enums — Match the Audience

| Cross-Domain (coarse) | In-Domain (fine) |
|-----------------------|------------------|
| `should_apply` (bool or simple signal) | `application_not_started`, `application_started`, `application_submitted`, `application_in_review`, `application_accepted`, `application_rejected` |
| `application_started_at` (timestamp) | Full lifecycle status codes |

**Rule:** Use enums, not booleans, for states that might grow beyond two values. `PhotoType` beats `bool gif`. Booleans are only correct when the field is truly and permanently binary.

## Verification & Truth — Not Independent Domains

Verification is not a standalone concept. It is the comparison of non-authoritative data against authoritative data. Both values and the assessment belong together in the domain that owns the concept being described.

| Pattern | Where It Lives | Not This |
|---------|----------------|----------|
| Customer-reported employee count + bureau employee count + match result | Business domain (or Employment subdomain) | Separate "Verification Service" |
| Customer-provided address + USPS-validated address + validation status | Contact Information domain | Separate "Address Verification Service" |
| Self-reported identity + bureau-confirmed identity + verification result | Person domain (or Identity subdomain) | Standalone "ID Verification Service" |

**Test:** Would a client ever want the verification result without the data being verified? If not, they belong together.

## Service Naming — Name for Capability, Not Vendor

| Use This | Not This | Why |
|----------|----------|-----|
| Billing & Membership Service (BMS) | StripeService | BMS survived vendor evolution because it was named for what it does, not who it talks to |
| Credit Reporting | ExperianService, EquifaxService | The capability is "credit reporting", not the vendor |
| Payment Processing | StripePayments | Vendor is an implementation detail |

**BMS is the gold standard:** Originally built for Stripe, now vendor-agnostic. "I can derive capabilities from how you pay us — not good enough? I'll derive from any detail." Every third-party-facing service should aspire to this.

## Known Domain Boundaries

These are separate domains even when closely related:

- Plan ≠ Capabilities ≠ PlanRecommendation
- Cashflow Metrics ≠ Banking Transactions
- Person ≠ Account ≠ ID Verification (though verification data lives with Person, not as a standalone)
- Business ≠ AGR (AGR is Cashflow)
- Business ≠ Contact Information (Contact Info may be its own domain)

## System Names to Avoid in Interfaces

These are implementation details, never domain language:

- Allosaurus / Allo
- Reachard
- Depot
- Bookie
- Thundercorp
- Bass
- The Vig
- Zuul
- Pulir
- Pudge
- Singer
- LogicService
- BMS (acceptable as an internal service name, but the *interface* speaks in business terms)

If any of these names appear in a proto field, method, service, or enum value, it's a violation.

## Protobuf Technical Vocabulary

| Use This | Not This | Why |
|----------|----------|-----|
| `FOO_UNSPECIFIED = 0` | Meaningful zero-value | Zero-value must not carry business semantics (T5) |
| `reserved 2; reserved "old_field";` | Commented-out fields | Deleted fields must be reserved to prevent tag reuse (T4) |
| `google.protobuf.Timestamp` | Custom timestamp message | Use well-known types (T8) |
| `google.protobuf.Duration` | `int64 duration_ms` | Use well-known types (T8) |
| `google.type.Money` | Custom money message | Use well-known types (T8) |
| Enum for future-flexible states | `bool` for two-state fields | Booleans can't grow; enums can (P5) |
