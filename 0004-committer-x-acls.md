---
layout: default
title: Access Control Lists for Committer-X Exposed APIs
nav_order: 3
---

- Feature Name: committer_x_acl_enforcement
- Start Date: 2026-05-26
- Fabric-X Component: committer-x (query service, block query service, block deliver, notification service)
- Fabric-X Issue: https://github.com/hyperledger/fabric-x-committer/issues/592

# Summary
[summary]: #summary

This RFC proposes an Access Control List (ACL) enforcement layer for the gRPC APIs exposed by Committer-X services.
Every exposed RPC is considered as a resource,
and every resource is mapped to a policy that defines which client identities may invoke it.
Both the policy definitions and the identities they evaluate are derived from the channel configuration bundle—treated as the single source of truth,
and are refreshed dynamically when new configuration blocks arrive.
A hard-coded default resource-to-policy map serves as a fallback
when the channel configuration mapping for a specific resource is missing.
Two implementation options are presented for discussion:
(A) wrapping every request in a typed message with a signed `common.Envelope`,
and (B) a dedicated `Authorize` RPC that acquires the identity once per connection
and binds it to the gRPC connection for all subsequent calls.

# Motivation
[motivation]: #motivation

Committer-X services currently expose their gRPC APIs without any authorization check.
A connection that successfully completes the mTLS handshake can invoke any exposed method — transaction queries,
block deliveries/queries,
and notification streams—regardless of which organization
the client belongs to or which role it has been granted in the channel.
This is a meaningful security gap.

Mutual TLS proves who a client is in the transport layer; it does not decide what that client is allowed to do.
Without an ACL layer, the system has no way to express that,
for example,
only members of organization A may read blocks while members of organization B may only subscribe to notifications.

The expected outcome is an enforcement layer in which (1) every exposed method has an explicit, named resource; (2) every resource has an explicit policy backed by the channel configuration; (3) policy decisions follow channel configuration updates automatically, including for in-flight streams; and (4) a default map exists for resources not yet defined in the channel configuration.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Every gRPC method exposed by a Committer-X service is governed by an **ACL resource name**.
A resource name is a structured identifier defined by the following pattern: `{proto-directory}.{service-name}/{rpc-name}`.
Each resource maps to a **policy** defined in the channel configuration,
such as `/Channel/Application/Readers` or `/Channel/Application/Writers`.

Operators declare these mappings in `configtx.yaml`:

```yaml
ACLs:
  # ACL policy for the Query "GetTransactionStatus" function.
  /servicepb.QueryService/GetTransactionStatus: /Channel/Application/Readers
  # ACL policy for the BlockQuery "GetBlockByNumber" function.
  /servicepb.BlockQueryService/GetBlockByNumber: /Channel/Application/Readers
```

When a client invokes one of these methods,
a gRPC interceptor on the server resolves the method to its resource name,
looks up the policy in the current channel configuration bundle,
and evaluates the client's identity against that policy.
If the policy is satisfied, the call proceeds; otherwise the server returns `PermissionDenied`.

One main component back this up at runtime:

- **ACL Provider** — holds the latest channel configuration bundle (MSP definitions, policy definitions, configuration sequence number). It utilizes the current Committer-X's existing dynamic root CA refresh path, which already monitors for configuration blocks. When a new configuration block arrives, the bundle reference is atomically swapped, and later ACL checks evaluate against the new policies.

For unary RPCs,
evaluation runs once per call.
For streaming RPCs (`blockDelivery`, `OpenNotificationStream`),
the client is authorized
when the stream is established and re-authorized
whenever the channel configuration changes during the stream's lifetime.
If a re-check fails—for example,
because the client's organization has been removed from the channel—the stream is terminated immediately.

When the ACL check fails,
the server returns a gRPC `PermissionDenied` status with a message identifying the resource.
For example:

```
rpc error: code = PermissionDenied desc = ACL check failed for [query/GetTransactionStatus]: identity is not a member of any organization with the Readers role
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Architecture

Each service that exposes APIs
(currently the query service, and the sidecar's notification service, block query and block deliver)
is given an `ACLProvider` as described above.

The `ACLProvider` is updated by the same dynamic root CA refresh path that already watches for configuration blocks. On a new configuration block:

1. A new bundle is constructed from the latest channel configuration.
2. The bundle reference held by the `ACLProvider` is atomically swapped.
3. Active streams compare the configuration sequence on each subsequent send or receive and re-evaluate their cached identities if the sequence has advanced.

The system always evaluates against the latest bundle observed.
A stream that started milliseconds before a new configuration block arrived will,
on its next sent or received message, detect the sequence advance and re-evaluate the identity against the new policy.

ACL enforcement is executed via gRPC interceptors—one unary and one streaming—installed per service.
The unary interceptor performs a full check on every call.
The streaming interceptor performs a full check when the stream is established,
caches the result in a session object bound to the stream, and re-evaludate the identity only when the bundle's configuration sequence advances.

## Bootstrap

ACL evaluation requires MSP definitions and policy definitions,
which come from the channel configuration bundle.
Before the running services have processed any configuration block,
the `ACLProvider` has nothing to evaluate against.

Bootstrap is handled via the existing hard-coded genesis-block path.
The sidecar receive the genesis block, and the block is committed.
Then, the services' bundle-construction mechanism builds the initial bundle from that committed config block.
Until this sequence completes, the exposed APIs have no usable bundle and ACL-protected methods will reject calls.
This introduces a small, bounded delay in API availability between process start and first-bundle availability.
Since the first block is always the configuration block, no useful information is exposed before enforcement becomes active.

## Configuration

Default mappings live in code.
Operator-visible mappings live in `{config-dir}/configtx.yaml` in the `fabric-x-common` repository,
under the `ACLs` section.
The lookup order at request time is:

1. The current channel configuration bundle's `ACLs` section.
2. The hard-coded global default map.

If neither contains an entry for the requested resource, the request is rejected with `PermissionDenied`.

## Identity Acquisition: Two Options

The two options below differ exclusively in how the client's identity is obtained at the server and how it is cached.
All policy lookups, evaluation, and responses are identical.
Option A carries a signed identity proof on every request;
Option B acquires it once per connection through a dedicated `Authorize` RPC and binds it at the gRPC connection level.

### Option A — Signed Envelope (`common.Envelope`) Wrapping

The gRPC request payload is replaced by a typed wrapper that contains a signed `common.Envelope`.
The envelope carries the request data, the client's serialized MSP signing identity,
and a signature over the payload computed with that identity's private key.
A typed wrapper—rather than a raw `common.Envelope` — is used so the gRPC layer does not erase the payload type.

**Breaking changes.** The following proto files in `fabric-x-common` are modified:

- `query.proto`
- `notify.proto`
- `block_query.proto`

**Replay prevention.**
Each envelope's `ChannelHeader` carries two pieces of information that together prevent replay attacks,
even when mTLS is not enabled:

- **Timestamp.** The `ChannelHeader.Timestamp` records when the signed envelope was created. The server validates it against the current time within a freshness window, so a captured envelope goes stale and cannot be replayed later — regardless of the TLS mode.
- **TLS certificate hash.** When mTLS is enabled, `ChannelHeader.TlsCertHash` is compared against the actual TLS certificate hash of the presenting connection. A captured envelope therefore cannot be replayed from any other connection.

**Pros.**

- The full envelope (data + signature) travels in a single structure, making the cryptographic provenance of each request unambiguous.
- The MSP signing identity is fully exercised on every call.
- Replay is prevented by two independent mechanisms: timestamp freshness (always) and TLS certificate hash binding (under mTLS).

**Cons.**

- Requires coordinated breaking changes to three proto files and to every client.
- Requires client-side boilerplate to wrap and sign each request.
- The same connection repeatedly re-sends its identity proof to the server, call after call, even though the connection itself has not changed.

**Execution flow (unary)**

1. The interceptor unwraps the typed message and extracts the `common.Envelope`.
2. The envelope payload is unmarshalled and the `ChannelHeader` is extracted. It contains the TLS certificate hash and the timestamp at which the signed envelope was created.
3. The timestamp is checked against the current time to ensure the envelope is recent and not a replay of an old request.
4. If mTLS is enabled, the claimed TLS certificate hash inside `ChannelHeader.TlsCertHash` is compared with the actual TLS certificate hash extracted from the gRPC context via `util.ExtractCertificateHashFromContext(ctx)`. A mismatch returns `codes.Unauthenticated`.
5. `protoutil.EnvelopeAsSignedData()` extracts the serialized identity, payload, and signature into a `SignedData` structure.
6. The interceptor reads the resource-to-policy map from the latest bundle exposed by the bundle provider, maps `info.FullMethod` to its policy reference, and retrieves the policy from the bundle's `PolicyManager`.
7. `policy.EvaluateSignedData(signedData)` verifies the signature and validates the identity against the channel's MSP definitions.
8. If every step succeeds, the RPC handler is invoked, allowing the request to proceed.

> The code snippets below are pseudocode illustrating the intended flow.

**Unary interceptor.**

```go
func (s *Server) ACLInterceptor(
    ctx context.Context,
    req interface{},
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (interface{}, error) {

    // 1. The request must contain a signed common.Envelope.
    typedMessage, ok := req.(*common.TypedMessage)
    if !ok {
        return nil, status.Error(codes.InvalidArgument,
            "request must be a typed common.TypedMessage")
    }

    envelope, ok := getEnvelopeFromMsg(typedMessage)
    if !ok {
        return nil, status.Error(codes.InvalidArgument,
            "request must be a signed common.Envelope")
    }

    payload, err := protoutil.UnmarshalPayload(envelope.Payload)
    if err != nil {
        return nil, status.Errorf(codes.InvalidArgument,
            "failed to unmarshal payload: %v", err)
    }

    chdr, err := protoutil.UnmarshalChannelHeader(payload.Header.ChannelHeader)
    if err != nil {
        return nil, status.Errorf(codes.InvalidArgument,
            "failed to unmarshal channel header: %v", err)
    }

    // 2. Verify envelope freshness (replay prevention, independent of mTLS).
    if err := validateTimestamp(chdr.Timestamp, s.envelopeFreshnessWindow); err != nil {
        return nil, status.Errorf(codes.Unauthenticated,
            "envelope timestamp outside freshness window: %v", err)
    }

    // 3. Verify TLS cert hash binding (replay prevention across connections).
    if s.mutualTLS {
        claimedHash := chdr.TlsCertHash
        if len(claimedHash) == 0 {
            return nil, status.Error(codes.Unauthenticated,
                "client didn't include TLS cert hash")
        }

        actualHash := util.ExtractCertificateHashFromContext(ctx)
        if len(actualHash) == 0 {
            return nil, status.Error(codes.Unauthenticated,
                "client didn't send a TLS certificate")
        }

        if !bytes.Equal(actualHash, claimedHash) {
            return nil, status.Error(codes.Unauthenticated,
                "TLS cert hash mismatch")
        }
    }

    // 4. Extract SignedData: serialized identity + signature + payload.
    signedData, err := protoutil.EnvelopeAsSignedData(envelope)
    if err != nil {
        return nil, status.Errorf(codes.InvalidArgument,
            "failed to extract signed data: %v", err)
    }

    // 5. Read the resource-to-policy map from the latest bundle.
    bundle := s.bundleProvider.GetBundle()
    appConfig, exists := bundle.ApplicationConfig()
    if !exists {
        return nil, status.Error(codes.Internal, "no application config in bundle")
    }

    policyRef := appConfig.APIPolicyMapper().PolicyRefForAPI(info.FullMethod)

    policy, exists := bundle.PolicyManager().GetPolicy(policyRef)
    if !exists {
        return nil, status.Errorf(codes.PermissionDenied,
            "no policy defined for resource: %s", info.FullMethod)
    }

    // 6. Evaluate: verify the signature and validate the identity
    //    against the channel's MSP definitions.
    if err := policy.EvaluateSignedData(signedData); err != nil {
        return nil, status.Errorf(codes.PermissionDenied,
            "ACL check failed for [%s]: %v", info.FullMethod, err)
    }

    return handler(ctx, req)
}
```

**Stream interceptor with a session-bound cache.**

When dealing with a stream, the method is a bit different.
The stream interceptor is activated only at the start of the stream, when the first envelope is received.
The first envelope is processed exactly as in the unary interceptor:
timestamp freshness, TLS certificate hash binding, signature verification, and policy evaluation.

After the first message passes full identity evaluation, a custom stream wrapper is created
and the validated state is saved in a **session object bound to the stream**.
Replay is irrelevant from this point on:
the stream is already established, and the first message was fully validated,
meaning the client has already proven its permission to access the stream.

The session object stores:

- the resolved **identity** (the original `SignedData`, so it can be re-evaluated later),
- the **latest accepted bundle**,
- the **bundle provider**, for loading the latest bundle on a sequence change,
- the **resource name** of the stream.

On every stream receive **or** send, the wrapper compares the bundle provider's current
configuration sequence against the cached one.
If they are the same, identity evaluation is skipped and the message proceeds immediately.
If the sequence has advanced, the latest bundle is loaded and the cached identity is
re-evaluated against the new policy set — catching mid-stream changes such as an
organization being removed from the channel. On failure, the stream is terminated immediately.

```go
func (s *Server) ACLStreamInterceptor(
    srv interface{},
    ss grpc.ServerStream,
    info *grpc.StreamServerInfo,
    handler grpc.StreamHandler,
) error {
    ctx := ss.Context()

    // 1. Receive the first envelope and validate it exactly as a unary call:
    //    freshness, TLS binding, signed-data extraction, policy evaluation.
    var firstEnvelope *common.Envelope
    if err := ss.RecvMsg(&firstEnvelope); err != nil {
        return status.Errorf(codes.InvalidArgument,
            "failed to receive first envelope: %v", err)
    }

    signedData, err := s.validateEnvelope(ctx, firstEnvelope) // steps 1-4 of the unary flow
    if err != nil {
        return err
    }

    bundle := s.bundleProvider.GetBundle()
    appConfig, exists := bundle.ApplicationConfig()
    if !exists {
        return status.Error(codes.Internal, "no application config in bundle")
    }

    policyRef := appConfig.APIPolicyMapper().PolicyRefForAPI(info.FullMethod)
    policy, exists := bundle.PolicyManager().GetPolicy(policyRef)
    if !exists {
        return status.Errorf(codes.PermissionDenied,
            "no policy defined for resource: %s", info.FullMethod)
    }

    if err := policy.EvaluateSignedData(signedData); err != nil {
        return status.Errorf(codes.PermissionDenied,
            "ACL check failed for [%s]: %v", info.FullMethod, err)
    }

    // 2. Bind the validated state to the stream in a session object.
    session := &SessionAccessControl{
        signedData:    signedData,        // identity, kept for re-evaluation
        currentBundle: bundle,            // latest accepted bundle
        provider:      s.bundleProvider,  // for loading the latest bundle
        resource:      info.FullMethod,   // resource name
    }

    return handler(srv, &aclServerStream{
        ServerStream: ss,
        session:      session,
    })
}

// aclServerStream re-checks the configuration sequence on every message,
// in both directions.
func (s *aclServerStream) RecvMsg(m interface{}) error {
    if err := s.session.checkConfigAndRevalidate(); err != nil {
        return err
    }
    return s.ServerStream.RecvMsg(m)
}

func (s *aclServerStream) SendMsg(m interface{}) error {
    if err := s.session.checkConfigAndRevalidate(); err != nil {
        return err
    }
    return s.ServerStream.SendMsg(m)
}
```

**Stream session caching behavior.**

| Situation | Behavior |
|---|---|
| **Stream establishment** | Full cryptographic verification of the first envelope (freshness + TLS binding + signature + policy evaluation). Identity, bundle, bundle provider, and resource name are saved in the session object. |
| **Configuration unchanged** | The provider's sequence matches the cached bundle's sequence. No cryptographic operation; the message proceeds immediately. This check runs on every receive and every send. |
| **Configuration changed** | The sequence advance is detected on the next receive or send. The latest bundle is loaded and the cached identity is re-evaluated against the new policy set. If the identity no longer satisfies the policy (e.g., its organization was removed), the stream is terminated immediately. |

---

### From per-request to per-connection

Option A requires a lot of client boilerplate and breaking changes in three proto files.
It also requires the same connection to send its identity proof to the server over and over again,
even though nothing about the connection has changed between calls.

If we want to avoid that — send a signed envelope only once and cache the connection's identity —
some RPC still has to receive a signed envelope for the initial identity acquisition.
Which RPC should it be?

Rather than electing one of the existing business RPCs, we dedicate one:
a common authenticator service with a single `Authorize` RPC.
This is Option B.

### Option B — Authorize RPC with Connection-Level Identity Binding

The client first calls a dedicated `Authorize(signedEnvelope)` unary RPC on a new `AuthService`,
presenting its signed envelope there.
The verified identity is bound to the gRPC connection itself,
and every subsequent RPC on that connection — unary or streaming — reuses the bound identity.

This requires introducing the `AuthService` and a corresponding proto file, but crucially,
it does not introduce breaking changes to any existing service APIs.
The service is simply registered on the sidecar and query service gRPC servers.
The same gRPC connection can then be used to create an auth client,
bind the identity at the connection level,
and be reused by the query and sidecar clients with the bound information.

**Replay prevention.**
The `Authorize` envelope is processed with the same checks as Option A's envelopes:
the `ChannelHeader` timestamp is validated for freshness,
and the TLS certificate hash is compared against the presenting connection's certificate under mTLS.
A captured `Authorize` envelope therefore expires quickly and cannot be bound from another connection.
After authorization, no further envelopes travel on the connection, so there is nothing left to capture and replay.

**Execution flow.**

1. **Custom handshake.** A custom `TransportCredentials` wrapper delegates the TLS handshake to the underlying TLS credentials and then injects a custom, mutable `MSPAuthInfo` struct as the connection's `AuthInfo`. This struct lives for the duration of the connection and initially carries only the TLS information.
2. **Identity resolution.** The `AuthService` is registered on the same gRPC server. When a client invokes the `Authorize` RPC, a dedicated `AuthorizeInterceptor` validates the signed envelope, extracts the client's identity, and safely binds it (together with the current configuration sequence) to the connection's `MSPAuthInfo`.
3. **Evaluation.** Because the identity is now bound to the physical connection, for every subsequent RPC the ACL interceptors simply read the pre-resolved identity from `MSPAuthInfo` and evaluate it against the channel's policies.

**Pros.**

- No breaking changes to existing proto files; only a new `AuthService` proto is added.
- The identity proof (envelope construction + signing) is paid once per connection, not once per call.
- Unary and streaming RPCs share the same enforcement path: read bound identity, evaluate policy.

**Cons.**

- Introduces a new service and connection-scoped mutable state.
- Clients must call `Authorize` before any other RPC on the connection.


> The code below reflects the intended implementation.

**Custom AuthInfo and TransportCredentials wrapper.**

```go
// MSPAuthInfo implements credentials.AuthInfo and holds MSP authentication state.
// This struct is attached to the gRPC connection during the TLS handshake and
// lives for the duration of the connection.
type MSPAuthInfo struct {
    mu             sync.RWMutex
    MSPIdentity    msp.Identity
    ConfigSequence uint64
    TLSCert        *x509.Certificate
    TLSCertHash    []byte

    TLSInfo credentials.AuthInfo
}

func (a *MSPAuthInfo) GetIdentity() (msp.Identity, uint64) {
    a.mu.RLock()
    defer a.mu.RUnlock()
    return a.MSPIdentity, a.ConfigSequence
}

// SetIdentity binds an MSP identity to this connection.
func (a *MSPAuthInfo) SetIdentity(identity msp.Identity, sequence uint64) {
    a.mu.Lock()
    defer a.mu.Unlock()
    a.MSPIdentity = identity
    a.ConfigSequence = sequence
}

type CustomCredentials struct {
    tlsCreds credentials.TransportCredentials
}

// NewCustomCredentials creates new custom credentials that wrap existing TLS credentials.
// If tlsCreds is nil, it will use insecure credentials (for testing only).
func NewCustomCredentials(tlsCreds credentials.TransportCredentials) credentials.TransportCredentials {
    if tlsCreds == nil {
        tlsCreds = insecure.NewCredentials()
    }
    return &CustomCredentials{tlsCreds: tlsCreds}
}

// ClientHandshake delegates to TLS credentials, then adds custom auth info.
func (c *CustomCredentials) ClientHandshake(
    ctx context.Context, authority string, rawConn net.Conn,
) (net.Conn, credentials.AuthInfo, error) {
    conn, tlsAuthInfo, err := c.tlsCreds.ClientHandshake(ctx, authority, rawConn)
    if err != nil {
        return nil, nil, fmt.Errorf("TLS handshake failed: %w", err)
    }
    return conn, &MSPAuthInfo{TLSInfo: tlsAuthInfo}, nil
}

// ServerHandshake delegates to TLS credentials, then adds custom auth validation.
func (c *CustomCredentials) ServerHandshake(rawConn net.Conn) (net.Conn, credentials.AuthInfo, error) {
    logger.Infof("Performing TLS handshake from: %s", rawConn.RemoteAddr().String())

    conn, tlsAuthInfo, err := c.tlsCreds.ServerHandshake(rawConn)
    if err != nil {
        return nil, nil, fmt.Errorf("TLS handshake failed: %w", err)
    }

    return conn, &MSPAuthInfo{TLSInfo: tlsAuthInfo}, nil
}
```

**Authorize interceptor.**

```go
// AuthorizeInterceptor creates a gRPC interceptor specifically for the Authorize RPC.
// This interceptor validates the signed envelope and binds the MSP identity to the connection.
//
// Behavior:
//   - Services with registered DynamicTLSUpdater: Processes authorization
//   - Services without updater (internal services): Returns error (should not call Authorize)
//   - Missing bundle when updater is registered: Returns error (configuration problem)
func AuthorizeInterceptor(provider BundleProvider) grpc.UnaryServerInterceptor {
    return func(
        ctx context.Context,
        req interface{},
        info *grpc.UnaryServerInfo,
        handler grpc.UnaryHandler,
    ) (interface{}, error) {
        // Only intercept the Authorize RPC.
        if !strings.EqualFold(info.FullMethod, AuthenticationResource) {
            return handler(ctx, req)
        }

        p, ok := peer.FromContext(ctx)
        if !ok {
            return &committerpb.AuthorizeResponse{
                Success: false,
                Message: ErrNoPeerInfo.Error(),
            }, nil
        }

        authInfo, ok := p.AuthInfo.(*MSPAuthInfo)
        if !ok {
            return &committerpb.AuthorizeResponse{
                Success: false,
                Message: ErrNoMSPAuthInfo.Error(),
            }, nil
        }

        bundle, err := provider.GetBundle()
        if errors.Is(err, ErrNoUpdater) {
            return &committerpb.AuthorizeResponse{
                Success: false,
                Message: "Authorization not available for internal services",
            }, nil
        }
        if err != nil {
            return &committerpb.AuthorizeResponse{
                Success: false,
                Message: "Channel configuration not available: " + err.Error(),
            }, nil
        }

        authReq, ok := req.(*committerpb.AuthorizeRequest)
        if !ok {
            return &committerpb.AuthorizeResponse{
                Success: false,
                Message: "Invalid request type",
            }, nil
        }

        signedEnvelope := authReq.GetSignedEnvelope()
        if signedEnvelope == nil {
            return &committerpb.AuthorizeResponse{
                Success: false,
                Message: "Signed envelope is required",
            }, nil
        }

        // Validates freshness + TLS binding, verifies the signature, and
        // resolves the identity against the channel MSPs.
        identity, mspID, _, err := ExtractIdentityFromEnvelope(signedEnvelope, bundle)
        if err != nil {
            return &committerpb.AuthorizeResponse{
                Success: false,
                Message: "Failed to extract identity: " + err.Error(),
            }, nil
        }

        logger.Infof("Binding identity to connection: identity=%s, mspID=%s",
            identity.GetIdentifier(), mspID)
        authInfo.SetIdentity(identity, bundle.ConfigtxValidator().Sequence())

        return handler(ctx, req)
    }
}
```

**Unary enforcement interceptor.**

```go
// MSPUnaryServerInterceptor creates a gRPC interceptor for MSP-based access control on unary RPCs.
func MSPUnaryServerInterceptor(provider BundleProvider) grpc.UnaryServerInterceptor {
    return func(
        ctx context.Context,
        req interface{},
        info *grpc.UnaryServerInfo,
        handler grpc.UnaryHandler,
    ) (interface{}, error) {
        // Skip authorization check for the Authorize RPC itself.
        if strings.EqualFold(info.FullMethod, AuthenticationResource) {
            return handler(ctx, req)
        }

        p, ok := peer.FromContext(ctx)
        if !ok {
            return nil, status.Error(codes.Unauthenticated, ErrNoPeerInfo.Error())
        }

        authInfo, ok := p.AuthInfo.(*MSPAuthInfo)
        if !ok {
            return nil, status.Error(codes.Internal, ErrNoMSPAuthInfo.Error())
        }

        bundle, err := provider.GetBundle()
        if errors.Is(err, ErrNoUpdater) {
            // No updater = internal service = bypass MSP auth check.
            return handler(ctx, req)
        }
        if err != nil {
            // ErrNoBundle or any other error = FAIL (enforce ACL).
            return nil, status.Error(codes.Internal,
                "channel configuration not available: "+err.Error())
        }

        // Bundle exists = public service = ENFORCE MSP auth.
        identity, _ := authInfo.GetIdentity()
        if identity == nil {
            return nil, status.Error(codes.Unauthenticated,
                "connection not authorized: call Authorize first")
        }

        // Evaluate policy on every unary call (no caching needed for short-lived RPCs).
        if err := evaluatePolicy(bundle, identity, info.FullMethod); err != nil {
            return nil, err
        }

        return handler(ctx, req)
    }
}
```

**Stream enforcement interceptor and wrapped stream.**

```go
// MSPStreamServerInterceptor creates a gRPC stream interceptor for MSP-based access control.
//
// The wrapped stream checks for config sequence changes on every RecvMsg/SendMsg call.
// If the config changed, it re-evaluates the identity against the new policy.
func MSPStreamServerInterceptor(provider BundleProvider) grpc.StreamServerInterceptor {
    return func(
        srv interface{},
        ss grpc.ServerStream,
        info *grpc.StreamServerInfo,
        handler grpc.StreamHandler,
    ) error {
        ctx := ss.Context()
        p, ok := peer.FromContext(ctx)
        if !ok {
            return status.Error(codes.Unauthenticated, ErrNoPeerInfo.Error())
        }

        authInfo, ok := p.AuthInfo.(*MSPAuthInfo)
        if !ok {
            return status.Error(codes.Internal, ErrNoMSPAuthInfo.Error())
        }

        bundle, err := provider.GetBundle()
        if errors.Is(err, ErrNoUpdater) {
            // Internal service - bypass authentication.
            return handler(srv, ss)
        }
        if err != nil {
            // Public service with missing config - fail.
            return status.Error(codes.Internal,
                "channel configuration not available: "+err.Error())
        }

        // Public service - enforce ACL.
        identity, _ := authInfo.GetIdentity()
        if identity == nil {
            return status.Error(codes.Unauthenticated,
                "connection not authorized: call Authorize RPC first")
        }

        // Initial policy evaluation.
        if err := evaluatePolicy(bundle, identity, info.FullMethod); err != nil {
            return err
        }

        // Wrap the stream for config change detection and re-evaluation.
        wrappedStream := &authServerStream{
            ServerStream:  ss,
            authInfo:      authInfo,
            provider:      provider,
            fullMethod:    info.FullMethod,
            currentBundle: bundle, // cached bundle to detect changes
        }

        return handler(srv, wrappedStream)
    }
}

// authServerStream wraps grpc.ServerStream to add config change detection
// and identity re-evaluation on every message.
type authServerStream struct {
    grpc.ServerStream
    authInfo      *MSPAuthInfo
    provider      BundleProvider
    fullMethod    string
    currentBundle *channelconfig.Bundle
}

// RecvMsg intercepts incoming messages to perform config change detection.
func (s *authServerStream) RecvMsg(m interface{}) error {
    if err := s.checkConfigAndRevalidate(); err != nil {
        return err
    }
    return s.ServerStream.RecvMsg(m)
}

// SendMsg intercepts outgoing messages to perform config change detection.
func (s *authServerStream) SendMsg(m interface{}) error {
    if err := s.checkConfigAndRevalidate(); err != nil {
        return err
    }
    return s.ServerStream.SendMsg(m)
}
```

# Drawbacks
[drawbacks]: #drawbacks
- **Option A specifically.** Breaking changes to three proto files force every existing client to be updated. Clients must implement envelope construction, signing, timestamping, and TLS certificate hash inclusion — on every request.
- **Option B specifically.** Introduces a new service, a proto file, and connection-scoped mutable state. Clients must call `Authorize` before any other RPC on a connection.

# Prior art
[prior-art]: #prior-art

Hyperledger Fabric uses essentially the model proposed here (A).

**gRPC services and message types.**
Fabric's ACL framework rests on two primary signed protobuf message types.
The first, `SignedProposal`,
is used for chaincode execution and ledger interactions;
it is handled by the `Endorser.ProcessProposal()` gRPC service
and is the standard for invoking smart contracts or querying the ledger via system chaincodes.
The second, `common.Envelope`, is used for retrieving blocks from the network via the Delivery service,
implemented on both orderer and peer, which streams blocks to clients.

**Endorser service flow and ACL integration.**
The lifecycle of a chaincode invocation begins at the Endorser Service.
Defined in `peer.proto`, the `ProcessProposal` gRPC method is the entry point for all client proposals.
When a `SignedProposal` reaches the gRPC handler, the system performs a multi-step validation:
the proposal is unpacked and validated for message integrity,
then enters the `preProcess` phase where validation and ACL checks occur.
The `SignedProposal` contains the serialized client identity in `SignatureHeader.Creator`,
the target chaincode name, and the `ChannelID`.

Fabric's design employs a two-tier ACL architecture:

- **Application chaincodes.** The ACL check occurs at the endorser level, validating against `/Channel/Application/Writers` before chaincode execution.
- **System chaincodes (QSCC, CSCC, lifecycle).** The endorser-level ACL check is skipped. Each system chaincode instead performs its own internal ACL check inside its `Invoke()` method. For example, QSCC extracts the `SignedProposal` from the stub and validates it against function-specific policies (e.g., `qscc/GetBlockByNumber` requires `/Channel/Application/Readers`).

**Policy mapping and configuration storage.**
ACL rules in Fabric are governed by a two-level system:
hardcoded default resource-to-policy mappings and dynamic policy definitions within the channel configuration.
The `defaultACLProviderImpl` is the central logic gate for the peer.
It maps the requested resource (such as `peer/Propose` or `qscc/GetBlockByNumber`) to a specific policy
(such as `/Channel/Application/Writers` or `/Channel/Application/Readers`).
A key feature of this provider is its ability
to handle multiple input types—including `SignedProposal` and `common.Envelope` —
converting them into a uniform `SignedData` structure for final policy evaluation.

The `defaultACLProviderImpl` maintains two maps:

- **Peer-level resources** (`pResourcePolicyMap`): operations like lifecycle management, checked without channel context.
- **Channel-level resources** (`cResourcePolicyMap`): operations like QSCC queries and endorsement, checked with channel context.

For example, all QSCC functions — such as `GetBlockByNumber` or `GetTransactionByID` — are hardcoded to require
`CHANNELREADERS` (`/Channel/Application/Readers`).
The resource names act as keys that map to a policy name within the channel configuration.

The actual policy definitions reside in the channel configuration (`configtx.yaml`).
Policies such as `Readers`, `Writers`, and `Admins` are expressed using `ImplicitMeta` rules; for instance,
a `Readers` policy defined as "ANY Readers"
is satisfied by a signature from any organization member with the Reader role.\
Committer-X will reuse these same policy definitions directly from the channel configuration bundle.

# Decisions to be made
[unresolved-questions]: #unresolved-questions

- **Option A vs. Option B.** This RFC presents both options for team discussion and does not pre-select one. The trade-offs are spelled out per option above.

# Dependencies
[dependencies]: #dependencies

- The MSP, policy, and channel-configuration packages from `fabric-x-common` (`protoutil`, `msp`, `policies`, channel-config bundle construction).
- `sampleconfig/configtx.yaml` in `fabric-x-common` to add a sample `ACLs` section.
- **For Option A:** coordinated changes to `query.proto`, `notify.proto`, and `block_query.proto`, and corresponding client library updates.
- **For Option B:** a new `AuthService` proto (`AuthorizeRequest`/`AuthorizeResponse`) and registration of the service on the sidecar and query service gRPC servers.
