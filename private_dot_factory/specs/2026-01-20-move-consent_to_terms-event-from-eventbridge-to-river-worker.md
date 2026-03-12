## Summary

Move the `consent_to_terms` event publishing from EventBridge to the existing River worker mechanism that syncs directly to Salesforce. 

## Current State

**Good news**: This work is **already largely done**! Looking at the codebase:

1. **`accept_terms.go`** - Already enqueues a `SyncContactToSalesforceArgs` job via River with `HasConsentTerms: &hasConsent` (lines 62-79)

2. **`post_registration.go`** - Already enqueues the same Salesforce sync job during registration via `enqueueSalesforceSync()` (lines 185-214)

3. **`salesforce_contact.go`** - The River worker already handles `HasConsentTerms` field and passes it to `sfsync.Contact.HasAcceptedTerms`

4. **`grpc.go`** - EventBridge is already disabled with a no-op `mockEB{}` and a comment: 
   > "EventBridge consent terms publishing has been replaced with River workers that sync directly to Salesforce API. Using no-op publisher for interface compatibility."

## What Remains (Cleanup)

The EventPublisher infrastructure is **no longer being used** but still exists. To complete the migration, we should remove the dead code:

### Files to Delete
- `internal/events/events.go` - EventBridge publisher (no longer called)
- `internal/events/events_test.go` - Tests for above
- `internal/events/event_publisher_adapter.go` - Adapter that's only used to satisfy interface
- `internal/events/BUILD.bazel` - Build file for events package

### Files to Modify
1. **`internal/grpc/grpc.go`**
   - Remove `mockEB` struct and its `Publish` method
   - Remove `events.NewPublisher()` and `EventPublisherAdapter` creation
   - Update `MustNewServer` call to not require EventPublisher

2. **`internal/grpc/account/server.go`**
   - Remove `EventPublisher` interface and field from `Server` struct
   - Remove `ep` parameter from `NewServer` and `MustNewServer`
   - Regenerate the mock file (or delete `event_publisher_moq_test.go`)

3. **Test files** (update to remove EventPublisher mocks)
   - `internal/grpc/account/account_test.go`
   - `internal/grpc/account/register_account_test.go`
   - `internal/grpc/account/event_publisher_moq_test.go` (delete)

4. **`internal/events/types.go`** (if exists) - Check if `Account` and `Partner` types are still needed elsewhere, otherwise delete

## Verification

After cleanup, run `./bin/validate account` to ensure:
- Code compiles
- Linting passes
- Tests pass
- BUILD files regenerate correctly

## No Functional Changes

This is purely a cleanup task. The River-based Salesforce sync is already working in production via `AcceptTermsOfService` and `RegisterAccount` endpoints.