## Summary
Replace the local plan-to-capabilities lookup in the account service with a call to billing-microservice's `GetBillingDetails` gRPC endpoint.

## Current State
- `account/internal/grpc/account/get_account.go` calls `plans.Plan.GetCapabilities()` which uses a hardcoded map in `libraries/plans/plans.go`
- The plan is retrieved from the local `acc.Plan` database field
- `libraries/plans/plans.go` already has `GetCapabilityFromPlanCode()` which maps BMS `planCode` to capabilities

## Relevant BMS Interface
**`nav.billing.v2.BillingService.GetBillingDetails`**:
- Request: `GetBillingDetailsRequest { account_id: string }`
- Response: `GetBillingDetailsResponse { details: BillingDetails }` 
- `BillingDetails` contains `Subscription.plan_code` and `Plan.plan_code`

The existing `GetCapabilityFromPlanCode(planCode)` function in `libraries/plans` can convert the BMS plan_code to capabilities.

## Implementation Plan

### 1. Create BMS gRPC Client (`account/internal/clients/billingmicroservice/`)
- New client wrapping `nav.billing.v2.BillingServiceClient`
- `GetCapabilities(ctx, accountID) ([]navapi.Capability, error)` method that:
  - Calls `GetBillingDetails(accountID)`
  - Extracts `plan_code` from subscription/plan
  - Uses `plans.GetCapabilityFromPlanCode()` to convert to capabilities
  - Falls back gracefully if BMS unavailable or no subscription

### 2. Add Interface and Inject Client (`account/internal/grpc/account/server.go`)
- Define `BillingClient` interface with `GetCapabilities` method
- Add `BillingClient` field to `Server` struct
- Initialize client in `NewServer` using existing `cfg.BillingMicroservice` config

### 3. Update `buildAccountMessage` to Accept Capabilities
- Modify function signature to accept capabilities as a parameter
- Remove the call to `plan.GetCapabilities()`
- Caller will provide capabilities from the new source

### 4. Update `GetAccount` to Use BMS Client
- Call `s.BillingClient.GetCapabilities(ctx, accountID)` before building the account message
- Pass capabilities to `buildAccountMessage`
- Handle errors gracefully (fallback to local plan lookup if BMS fails)

### 5. Update Tests
- Add mock for new `BillingClient` interface
- Update `get_account_test.go` to test both BMS success and fallback paths

## Open Questions
1. **Fallback behavior**: Should we fall back to local `plans.GetCapabilities()` if BMS is unavailable, or return an error?
2. **Caching**: Should we cache BMS responses briefly to avoid per-request latency?
3. **Other endpoints**: `ListAccountIDs` and `SearchAccounts` also use capabilities - should they be updated too?