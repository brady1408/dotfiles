# Implementation Plan: Add Reachard Plan Fallback to BMS GetCapabilities

## Current State Analysis

**BMS GetCapabilities Flow:**
1. `external/capabilities/capability_service.go` - External gRPC endpoint extracts account ID from headers
2. Calls `internal/capabilities/capability_service.go::GetCapabilities()`
3. Queries BMS database via `CustomerActiveSubscriptionsForAccountID()` for plan data
4. **Issue:** Returns error if customer/subscriptions not found in BMS database
5. Converts plan codes to capabilities using `plans.GetCapabilityFromPlanCode()`

**Reachard Service:**
- Has `GetAccountsData` gRPC endpoint that returns `Account` proto
- **Problem:** The `Account` proto in protorepo does NOT contain plan information (only id, email, program_code, active, personal_profile, user_id)
- The user's assumption that `GetAccount()` returns plan data appears incorrect based on proto definition

## Investigation Results

After examining the Reachard proto definition at `/Users/bcole/ws/src/git.nav.com/protorepo/protorepo/nav/reachard/account.proto`, the `Account` message contains:
- `id`, `email`, `program_code`, `active`, `partner_soft_pull_consent_at`, `personal_profile`, `user_id`
- **No plan field exists**

## Recommendation

Before proceeding with implementation, we need to clarify:

### Option A: Use program_code as plan fallback
If `program_code` maps to plan codes (e.g., "nav_direct", "credit_suite"), we can:
1. Add Reachard gRPC client config to BMS
2. When BMS has no subscriptions, call Reachard's `GetAccountsData` 
3. Use the `program_code` field from the response as the plan code
4. Convert to capabilities using existing `plans.GetCapabilityFromPlanCode()`

### Option B: Extend Reachard proto to include plan
If we need actual plan data from Reachard:
1. Modify Reachard's `Account` proto to add `plan_code` field
2. Update Reachard service to populate plan in response
3. Then implement fallback in BMS as described above

### Option C: Different approach
If the plan should come from a different source (Allosaurus, another service), we need to identify the correct service.

## Implementation Steps (Option A - if program_code works)

1. **Add Reachard client configuration to BMS**
   - Add to `pkg/config/config.go`:
     - `ReachardDial string env:"REACHARD_GRPC" default:"reachard-iii-grpc.default:2052"`
     - `ReachardAdminKey string env:"REACH_ADMIN_KEY" default:""`
   - Add to `Values.yaml` envcfg sections

2. **Create Reachard client package**
   - Create `pkg/clients/reachard/client.go` (similar to notifysvc pattern)
   - Implement `GetAccountsData()` method with admin API key header
   - Handle connection lifecycle

3. **Modify GetCapabilities with fallback logic**
   - In `internal/capabilities/capability_service.go`:
     - Add Reachard client to `CapabilityService` struct
     - When `CustomerActiveSubscriptionsForAccountID()` returns error
     - Call Reachard's `GetAccountsData()` with account ID
     - Extract `program_code` from response
     - Use it as fallback plan code
     - Continue with capability conversion

4. **Update tests**
   - Mock Reachard client
   - Add test cases for fallback scenarios
   - Test error handling paths

5. **Validation**
   - Run `./bin/validate billing-microservice`
   - Fix any issues

## Questions to Answer Before Implementation

1. **Does `program_code` from Reachard map to BMS plan codes?** Need to verify the values match (e.g., "nav_direct" vs "direct_free")
2. **Should we cache the Reachard response?** To avoid repeated calls
3. **What if Reachard also doesn't have the account?** Return error or default to free plan?
4. **Is there logging/metrics requirement** for when fallback is triggered?