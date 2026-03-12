# External RegisterAccount Implementation Plan

## Current State Analysis
I've analyzed the existing code and found:

1. **Internal RegisterAccount**: Fully implemented in `account/internal/grpc/account/register_account.go` with comprehensive business logic, validation, and integrations.

2. **External Service Structure**: Already set up with a `Server` struct that wraps an `InternalServer` interface and delegates to internal methods for `GetAccount` and `AcceptTermsOfService`.

3. **Current External RegisterAccount**: Simple stub that directly calls `s.internalServer.RegisterAccount(ctx, req)`.

4. **Proto Definitions**: The external service uses the same `RegisterAccountRequest` and `RegisterAccountResponse` types as the internal service - no conversion needed.

## Implementation Plan

### 1. Enhance External RegisterAccount Implementation
- **Current code**: Simple direct delegation
- **Enhancement needed**: Add proper tracing, logging, and error handling consistent with other external methods
- **Key difference**: Unlike `GetAccount`/`AcceptTermsOfService`, registration doesn't need header ID extraction (since no account exists yet)

### 2. Fix Bazel Configuration
- **Issue**: The `BUILD.bazel` file shows signs of being manually edited
- **Solution**: Update dependencies to ensure all required proto imports are available
- **Check**: Verify the bazel build works after implementation

### 3. Implementation Details
The external `RegisterAccount` method will:
- Add OpenTelemetry tracing with proper attributes
- Add structured logging with navlog
- Set `x-no-forward` header to prevent circular forwarding to Allosaurus
- Delegate to internal service
- Handle errors appropriately
- Return response directly (no conversion needed)

### 4. Key Code Pattern
Following the existing pattern from `get_account.go` and `accept_terms.go`:
```go
func (s *Server) RegisterAccount(ctx context.Context, req *acctService.RegisterAccountRequest) (*acctService.RegisterAccountResponse, error) {
    ctx, span := tracer.Start(ctx, "external.account.RegisterAccount")
    defer span.End()
    
    // Add tracing attributes
    // Set x-no-forward header to prevent circular calls
    // Delegate to internal service
    // Handle errors and return response
}
```

### 5. Testing Strategy
- Run `make bazel-test` or `bazel test //account/...` to verify build
- Check that the external service properly delegates to internal
- Verify no circular forwarding occurs

## Files to Modify
1. `account/internal/grpc/external/account/register_account.go` - Enhance implementation
2. `account/internal/grpc/external/account/BUILD.bazel` - Fix any dependency issues

This approach maintains consistency with existing external service patterns while ensuring the robust internal logic is properly utilized.