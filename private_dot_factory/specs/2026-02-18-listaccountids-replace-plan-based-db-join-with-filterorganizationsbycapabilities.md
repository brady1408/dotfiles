## Summary

Replace the plan-based `WHERE plan = ANY(plans)` query in `ListAccountIDs` with the new `FilterOrganizationsByCapabilities` bidirectional streaming RPC, making Capabilities service the source of truth for capability-based filtering. Remove all plan-based filtering logic.

## Current Flow
1. Request comes in with capabilities
2. `plans.GetPlansWithAnyCapability()` maps capabilities to plan names
3. Repo queries `SELECT id FROM accounts WHERE plan = ANY(plans)` (with optional shard filter)
4. All account IDs loaded into memory, chunked, streamed to caller

## New Flow (when capabilities are requested)
1. Request comes in with capabilities
2. If `CapabilityClient` is nil, return `codes.Unavailable` error -- no fallback
3. **Repo** runs a new query: `SELECT id::text, organization_id FROM accounts` (with optional shard filter, **no plan filter**)
4. **Handler** builds an `orgID -> []accountID` map from the results
5. **Handler** opens `FilterOrganizationsByCapabilities` bidi stream:
   - Sends `init` message with the requested capabilities
   - Sends org IDs in chunks (using `NavAPIAccountFetchSize` as chunk size)
   - For each chunk sent, reads the response (matching org IDs that passed the filter)
   - Maps filtered org IDs back to account IDs and streams them to the caller
6. **When no capabilities are requested**, behavior is unchanged -- query all accounts directly from DB

## Files to Change

### 1. `account/internal/repo/reachard/list_account_ids.go`
- Add `AccountOrgPair` struct: `{ AccountID string; OrganizationID string }`
- Add `ListAccountIDsWithOrgs(ctx, params)` method -- returns `[]AccountOrgPair`, queries `SELECT id::text, organization_id FROM accounts` with optional shard filter only
- Remove the plan-based query variants (`accountsWithPlans`, `accountsWithShardAndPlan`) and the `Plans` field from `ListAccountIdsParams` since they are no longer needed
- Keep `ListAccountIDs` for the no-capabilities path (all accounts, optional shard only)

### 2. `account/internal/grpc/account/server.go`
- Expand `CapabilityClient` interface to add:
  ```go
  FilterOrganizationsByCapabilities(ctx context.Context, opts ...grpc.CallOption) (capabilityService.CapabilityService_FilterOrganizationsByCapabilitiesClient, error)
  ```
- Add `ListAccountIDsWithOrgs` to `AccountRepository` interface

### 3. `account/internal/grpc/account/list_account_ids.go`
- Remove `plans.GetPlansWithAnyCapability` usage and the `plans` import
- When capabilities are requested:
  1. If `CapabilityClient == nil`, return `codes.Unavailable` error
  2. Call `Repo.ListAccountIDsWithOrgs()` to get all (accountID, orgID) pairs (with shard filter if present)
  3. Build `map[string][]string` (orgID -> accountIDs)
  4. Open bidi stream, send init with capabilities
  5. Chunk the org IDs, for each chunk: send, recv, map back to account IDs, stream to caller
- When no capabilities requested: call `Repo.ListAccountIDs()` (all accounts, optional shard), chunk and stream as before

### 4. `account/internal/grpc/account/account_test.go`
- Update existing tests to remove plan-based expectations
- Add tests for the new capabilities-via-stream path with mock bidi stream client
- Add test for `CapabilityClient == nil` returning `codes.Unavailable`

### 5. `account/internal/repo/reachard/list_account_ids_test.go`
- Update tests: remove plan-related test cases, add cases for `AccountOrgPair` and `ListAccountIDsWithOrgs` params

### 6. Regenerate moq files
- Run `go generate` for the updated `CapabilityClient` and `AccountRepository` interfaces

## Key Design Decisions
- **No fallback**: If capabilities are requested and `CapabilityClient` is nil, return an error. No plan-based fallback code.
- **Remove dead code**: Plan-based query variants and `plans.GetPlansWithAnyCapability` usage are removed entirely from ListAccountIDs.
- **Memory management**: Send one chunk of org IDs, wait for response, map to account IDs, stream to caller, then next chunk. The org->account map is the main memory-resident structure.
- **No-caps path unchanged**: When no capabilities filter is specified, skip capabilities service entirely.
- **Chunk size**: Reuse `NavAPIAccountFetchSize` for both org ID chunks to capabilities service and account ID chunks to the caller.