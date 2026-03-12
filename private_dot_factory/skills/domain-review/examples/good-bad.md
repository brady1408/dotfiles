# Domain API Examples

Real examples from Nav's protorepo illustrating good and bad domain design.

---

## Good Examples

### Good: ActivationStatus enum with guarded scope

**Principles:** P2, P5, P3

**Source:** `nav/navapi/account/v1/account.proto`

```protobuf
// Answers _only_ "Can the user actually use this account after logging in?".
// Only the UX needs to know what to do with this answer since this indicates
// what next steps a customer must take to use the account.
//
// DO NOT use this as a replacement for Capabilities. "Can system X do this
// action for this customer?" must be a capability, not hinge on whether the
// account is fundamentally accessible by the customer.
//
// DO NOT add statuses to this that reflect semantics other than whether the
// account is usable=true/false. Partial usability, billing matters, plan
// matters are all different dimensions for other enums or domains.
enum ActivationStatus {
  ACTIVATION_STATUS_UNSPECIFIED = 0;
  ACTIVATION_STATUS_ACTIVE = 1;
  ACTIVATION_STATUS_DEACTIVATED_BY_ACCOUNT_OWNER = 2;
  ACTIVATION_STATUS_DEACTIVATED_BY_ORGANIZATION = 3;
  ACTIVATION_STATUS_ANONYMIZED = 4;
  ACTIVATION_STATUS_DISABLED_BY_ADMIN = 5;
}
```

**Why:** The comments are as important as the code. They explicitly fence what this enum is and is NOT for — preventing it from becoming a dumping ground for unrelated statuses. Each value describes a business state the customer would understand. The warning against conflating this with Capabilities shows architectural discipline (P3 — keeping domains thin).

---

### Good: Organization entity — thin and standalone

**Principles:** P3, P4, P5

**Source:** `nav/navapi/organization/v1/organization.proto`

```protobuf
// Organization represents an abstract entity in the real world which may have
// multiple employees, may have multiple businesses which they manage, which in
// turn enables multiple logins to the same Nav entity.
// The many-to-many relationship between Account and Business are permissioned
// through the Role domain.
message Organization {
  string id = 1;
  string owner_account_id = 2;
  nav.navapi.account.v1.ActivationStatus activation_status = 3;
}
```

**Why:** Three fields. That's it. The message describes the entity and explicitly delegates permissions to the Role domain instead of embedding them. No god object tendencies — it knows what it is and what it isn't. The comment explains the real-world concept without referencing any system internals.

---

### Good: OrganizationService — methods named for business intentions

**Principles:** P2, P8

**Source:** `nav/navapi/services/organization/v1/external/organization.proto`

```protobuf
service OrganizationService {
  rpc GetOrganization(GetOrganizationRequest) returns (GetOrganizationResponse);
  rpc ChangeOwner(ChangeOwnerRequest) returns (ChangeOwnerResponse);
  rpc CreateInvitation(CreateInvitationRequest) returns (CreateInvitationResponse);
  rpc RevokeInvitation(RevokeInvitationRequest) returns (RevokeInvitationResponse);
  rpc ResendInvitationEmail(ResendInvitationEmailRequest) returns (ResendInvitationEmailResponse);
  rpc DeactivateAccount(DeactivateAccountRequest) returns (DeactivateAccountResponse);
  rpc ReactivateAccount(ReactivateAccountRequest) returns (ReactivateAccountResponse);
}
```

**Why:** Every method describes what the business actor is doing — `ChangeOwner`, `RevokeInvitation`, `DeactivateAccount`. None are CRUD operations on tables. A business person reading this list would understand every action without knowing anything about the implementation. No system names appear anywhere.

---

### Good: SubscriptionStatus — business states, not system states

**Principles:** P2, P8, P5

**Source:** `nav/navapi/subscription/v1/subscription.proto`

```protobuf
enum SubscriptionStatus {
  SUBSCRIPTION_STATUS_UNSPECIFIED = 0;
  SUBSCRIPTION_STATUS_CURRENT = 1;
  SUBSCRIPTION_STATUS_DUNNING = 2;
  SUBSCRIPTION_STATUS_CANCELED = 4;
  SUBSCRIPTION_STATUS_CREATED = 5;
  SUBSCRIPTION_STATUS_UNPAID = 6;
}
```

**Why:** These are billing lifecycle states that a business person would recognize. The comment on the message explains that CURRENT or DUNNING means "active" for capability purposes — this is the right level of granularity for cross-domain consumers. No Stripe-specific states or implementation details leak through.

---

## Bad Examples

### Bad: Allosaurus Member — the canonical god object

**Principles violated:** P1, P4, P8, P5

**Source:** `nav/allosaurus/allosaurus.proto`

```protobuf
message Member {
  string id = 1;
  optional string plan_id = 3;
  optional string account_guid = 6;
  optional string account_state = 9;
  optional string id_verification_state = 12;
  optional string email = 13;
  optional string first_name = 14;
  optional string last_name = 15;
  optional string encrypted_social = 16 [deprecated=true];
  optional string encrypted_date_of_birth = 17 [deprecated=true];
  optional Address address = 18;
  optional google.protobuf.Timestamp id_verified_at = 31;
  optional nav.PhoneNumber phone = 34;
  // ... plus dozens of commented-out fields like:
  // optional google.protobuf.Timestamp locked_at = 19;
  // optional google.protobuf.Timestamp cancelled_at = 22;
  // optional string stripe_customer_id = 39;
}
```

**Why:** `Member` spans at minimum five domain concepts: identity (Person), account state, plan info, ID verification, and PII. The term "Member" itself is implementation language — it blurs Person and Account (P8). Fields like `account_guid`, `plan_id`, and `id_verification_state` belong in separate domains. The commented-out fields (`stripe_customer_id`, `locked_at`, `cancelled_at`) reveal this was a direct DB schema dump (P1). Even the proto's own header comment admits: "THIS INTERFACE IS FOR REACHARD! IT IS NOT A VALID ABSTRACTION FOR OTHER CLIENTS!"

---

### Bad: Reachard Account — DB columns as comments, system coupling

**Principles violated:** P1, P8, P7

**Source:** `nav/reachard/account.proto`

```protobuf
message Account {
  // Enterprise UUID (accounts.id)
  string id = 1;
  // email associated with this account (accounts.email)
  string email = 2;
  // program_code associated with this account (programs.program_code)
  string program_code = 3;
  // active whether or not the account is active
  // (this IS NOT accounts.active, which is sparsely set & unreliable)
  bool active = 4;
  // user_id (accounts.user_id) this is also the member_id in allosaurus
  string user_id = 7;
}
```

**Why:** Every comment parenthetically cites the DB table and column: `(accounts.id)`, `(accounts.email)`, `(programs.program_code)`. The proto was clearly designed by looking at `SELECT *` and mapping columns to fields (P1). The comment on `user_id` explicitly couples to Allosaurus by saying "this is also the member_id in allosaurus" (P8). The field `user_id` should be `person_id` in domain language. The `program_code` references an internal table (`programs`) that clients shouldn't know exists.

---

### Bad: BillingDetails — cross-domain aggregation god object

**Principles violated:** P4, P6, P3

**Source:** `nav/billing/v2/billing_details.proto`

```protobuf
message BillingDetails {
  Customer customer = 1;
  Subscription subscription = 2;
  BillingAddress billing_address = 3;
  optional Plan plan = 4;
}
```

**Why:** Four separate domain concepts bundled into one response. Customer identity, subscription lifecycle, address, and plan are each their own concern. The `Customer` message inside this same package exposes `stripe_customer_id` — a third-party implementation detail leaking to clients. A client needing only the subscription status is forced to receive (and potentially depend on) customer, address, and plan data. Clients should call each domain independently and coordinate themselves (P6).

---

### Bad: ReportRefresh service — implementation-named methods

**Principles violated:** P2, P8, P1

**Source:** `nav/pudge/reports/grpc_service.proto`

```protobuf
service ReportRefresh {
  rpc BusinessReportRequestRaw (ReportRequest) returns (ReportReply) {}
  rpc BusinessReportRequestDandbParsed (ReportRequest) returns (dandb.ParsedReport) {}
  rpc BusinessReportRequestXPBizParsed (ReportRequest) returns (xpbiz.ParsedReport) {}
  rpc EquifaxBusinessRequest (equifax.Request) returns (equifax.Response) {}
  rpc RawEquifaxReportByID (equifax.RawEquifaxReportByIDRequest) returns (equifax.RawEquifaxReportByIDResponse) {}
  rpc ListEquifaxReportPulls (equifax.ListEquifaxReportPullsRequest) returns (equifax.EquifaxReportPullList) {}
  rpc DownloadXPBizReport (xpbiz.XPBizPDFRequest) returns (xpbiz.XPBizPDFReply) {}
}
```

**Why:** Methods are named after the data operation and implementation system, not the client intention. `BusinessReportRequestRaw` tells you nothing about what the client is trying to accomplish — contrast with something like `GetCurrentBusinessCreditReport`. The `XPBiz` prefix is an internal abbreviation for Experian Business that clients shouldn't need to know. `ListEquifaxReportPulls` exposes the implementation concept of a "pull" rather than describing what the client wants (e.g., `GetReportHistory`). The service itself is in the `nav.pudge.reports` package — `pudge` is a system name, not a domain concept.

---

### Bad: Allosaurus GetMember — response spans domains

**Principles violated:** P4, P6, P3

**Source:** `nav/allosaurus/allosaurus.proto`

```protobuf
message GetMemberRequest {
  string reachard_account_id = 1;
}

message GetMemberResponse {
  Member member = 1;
  repeated Business owned_businesses = 2;
}

service MemberService {
  rpc GetMember(GetMemberRequest) returns (GetMemberResponse);
}
```

**Why:** The request takes a `reachard_account_id` — coupling the caller to knowledge of Reachard's ID system (P8). The response bundles a Member (Person + Account god object) with owned Businesses — crossing at least three domain boundaries in a single response (P6). The client should call Person, Account, and Business domains independently. The service name `MemberService` uses implementation language — "Member" is not a real-world concept at Nav.

---

### Bad: Billing Customer — vendor leak in domain interface

**Principles violated:** P12, P1

**Source:** `nav/billing/v2/customer.proto`

```protobuf
message Customer {
  int64 id = 1;
  optional string stripe_customer_id = 2;
  CustomerState customer_state = 3;
  optional string name = 4;  // Full name of the customer from stripe
  optional string email = 5; // Email associated with the customer from stripe
  optional string phone = 6; // Phone number associated with the customer from stripe
}
```

**Why:** `stripe_customer_id` exposes the payment vendor as a first-class field in the domain interface (P12). The comments reinforce this — "from stripe" appears three times. If Nav switched payment providers, this field name and every comment would be wrong. Contrast with BMS, which was designed for client goals and survived vendor evolution. The `name`, `email`, and `phone` fields also likely duplicate data owned by the Person or Contact Information domains (P1, P5).

---

## Examples for Newer Principles

### Good (P12): BMS — named for capability, not vendor

**Principles:** P12, P2

BMS (Billing & Membership Service) is Nav's gold standard for P12. It was originally built to integrate with Stripe, but was named for its business purpose — billing and membership. When requirements grew beyond Stripe, the interface didn't need to change because it was designed around client questions ("what capabilities does this customer have?"), not vendor structure ("what does Stripe say about this customer?"). Every third-party-facing service should aspire to this model.

---

### Bad (P12): Pudge ReportRefresh — vendor names as method names

**Principles violated:** P12, P8, P2

**Source:** `nav/pudge/reports/grpc_service.proto` (also cited above for P2/P8)

The `ReportRefresh` service uses `BusinessReportRequestXPBizParsed`, `EquifaxBusinessRequest`, `DunBradstreetReport` — method names that are vendor names, not client intentions. If the client's question is "what is this business's credit report?", the method should answer that question regardless of which bureau supplies it. The package `nav.pudge.reports` compounds this — `pudge` is a system name.

---

### Good (T5): ActivationStatus enum — correct UNSPECIFIED default

**Principles:** T5

**Source:** `nav/navapi/account/v1/account.proto`

```protobuf
enum ActivationStatus {
  ACTIVATION_STATUS_UNSPECIFIED = 0;
  ACTIVATION_STATUS_ACTIVE = 1;
  ACTIVATION_STATUS_DEACTIVATED_BY_ACCOUNT_OWNER = 2;
  ...
}
```

**Why:** Tag 0 is `ACTIVATION_STATUS_UNSPECIFIED` — it carries no business semantics. Values are prefixed with the enum name (`ACTIVATION_STATUS_`) to avoid C++ namespace collisions. This is exactly what T5 requires.

---

### Bad (T5): SubscriptionStatus enum — skipped tag numbers without reservation

**Principles violated:** T4

**Source:** `nav/navapi/subscription/v1/subscription.proto`

```protobuf
enum SubscriptionStatus {
  SUBSCRIPTION_STATUS_UNSPECIFIED = 0;
  SUBSCRIPTION_STATUS_CURRENT = 1;
  SUBSCRIPTION_STATUS_DUNNING = 2;
  // tag 3 is missing — was it deleted?
  SUBSCRIPTION_STATUS_CANCELED = 4;
  SUBSCRIPTION_STATUS_CREATED = 5;
  SUBSCRIPTION_STATUS_UNPAID = 6;
}
```

**Why:** Tag 3 is missing with no `reserved` declaration. If tag 3 was previously used and removed, it should be reserved to prevent accidental reuse. If it was never used, the gap is confusing. Either way, `reserved 3;` should be present to make the intent explicit (T4).
