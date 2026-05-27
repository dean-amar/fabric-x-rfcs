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
(A) wrapping requests in typed message with a signed `common.Envelope`,
and (B) extracting client identity from the mTLS certificate at the transport layer.

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
A resource name is a short, structured identifier such as `query/GetTransactionStatus` or `blockquery/GetBlockByNumber`.
Each resource maps to a **policy** defined in the channel configuration,
such as `/Channel/Application/Readers` or `/Channel/Application/Writers`.

Operators declare these mappings in `configtx.yaml`:

```yaml
ACLs:
  # ACL policy for the Query "GetTransactionStatus" function.
  query/GetTransactionStatus: /Channel/Application/Readers
  # ACL policy for the BlockQuery "GetBlockByNumber" function.
  blockquery/GetBlockByNumber: /Channel/Application/Readers
```

When a client invokes one of these methods, 
a gRPC interceptor on the server resolves the method to its resource name,
looks up the policy in the current channel configuration bundle,
and evaluates the client's identity against that policy.
If the policy is satisfied, the call proceeds; otherwise the server returns `PermissionDenied`.

Two components back this up at runtime:

- **ACL Provider** — holds the resource-to-policy map and runs the evaluation. If the channel configuration does not define a mapping for a given resource (because the operator did not add the entry in configtx.yaml), the provider falls back to a hard-coded global default map. This guarantees that no exposed method is ever left without an applicable policy.
- **Bundle Manager** — holds the latest channel configuration bundle (MSP definitions, policy definitions, configuration sequence number). It plugs into Committer-X's existing dynamic root CA refresh path, which already monitors for configuration blocks. When a new configuration block arrives, the bundle reference is atomically swapped and later ACL checks evaluate against the new policies.

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
is given an `ACLProvider` instance composed of:

- A reference to the **global default resource-to-policy map** (hard-coded fallback, shared across all services in the process).
- A reference to a `BundleManager` that exposes the current channel configuration bundle.

The default map is global rather than per-service because resource names are namespaced by their service prefix (`query/`, `blockquery/`, `notify/`), and a single registry makes it easy to audit the full set of protected resources in one place.

The `BundleManager` is updated by the same dynamic root CA refresh path that already watches for configuration blocks. On a new configuration block:

1. A new bundle is constructed from the latest channel configuration.
2. The bundle reference held by the `BundleManager` is atomically swapped.
3. Active streams compare the configuration sequence on each later message and re-evaluate their cached identities if the sequence has advanced.

The system always evaluates against the latest bundle observed.
A stream that started milliseconds before a new configuration block arrived will,
on its next received message, detect the sequence advance and re-evaluate the identity against the new policy.

The bundle provides everything required for identity evaluation: the MSP definitions per organization, 
the policy definitions, and the configuration sequence number used for cache invalidation.

ACL enforcement is executed via gRPC interceptors—one unary and one streaming—installed per service.
The unary interceptor performs a full check on every call.
The streaming interceptor performs a full check on the first message,
caches the result keyed by session, and re-checks only when the bundle's configuration sequence advances.

## Bootstrap

ACL evaluation requires MSP definitions and policy definitions,
which come from the channel configuration bundle.
Before the running services have processed any configuration block,
the `BundleManager` has nothing to evaluate against.

Bootstrap is handled via the existing hard-coded genesis-block path. 
The sidecar loads the genesis block, the block is committed,
and the query service's bundle-construction mechanism then builds the initial bundle from that committed config block.
Until this sequence completes, the exposed APIs have no usable bundle and ACL-protected methods will reject calls.
This introduces a small, bounded delay in API availability between process start and first-bundle availability.
Although the first block should always be the configuration block.
Therefore, no useful information can be gained by the time it arrives.

## Configuration

Default mappings live in code.
Operator-visible mappings live in `sampleconfig/configtx.yaml` in the `fabric-x-common` repository,
under the `ACLs` section.
The lookup order at request time is:

1. The current channel configuration bundle's `ACLs` section.
2. The hard-coded global default map.

If neither contains an entry for the requested resource, the request is rejected with `PermissionDenied`.

## Identity Acquisition: Two Options

The two options below differ exclusively in how the client's identity is obtained at the server.
All policy lookups, evaluation, and response are identical.
Streams have a small difference between the two methods.


### Option A — Signed Envelope (`common.Envelope`) Wrapping

The gRPC request payload is replaced by a typed wrapper that contains a signed `common.Envelope`.
The envelope carries the request data, the client's serialized MSP signing identity,
and a signature over the payload computed with that identity's private key.
A typed wrapper—rather than a raw `common.Envelope` — is used so the gRPC layer does not erase the payload type.

**Breaking changes.** The following proto files in `fabric-x-common` are modified:

- `query.proto`
- `notify.proto`
- `block_query.proto`

**Pros.**

- The full envelope (data + signature) travels in a single structure, making the cryptographic provenance of each request unambiguous.
- The MSP signing identity is fully exercised, and the TLS certificate hash binding inside `ChannelHeader.TlsCertHash` prevents an envelope captured on one connection from being replayed on another.

**Cons.**

- Requires coordinated breaking changes to three proto files and to every client.
- Requires client-side boilerplate to wrap and sign each request.
- **Replay protection depends on mTLS.** The envelope is bound to a specific connection only via the TLS certificate hash inside `ChannelHeader.TlsCertHash`. If mTLS is disabled, that hash cannot be verified and a captured envelope becomes replayable from any connection. mTLS is the default production setting for Committer-X deployments, so in practice this is not a gap, but it should be acknowledged.

**Execution flow.**

1. The interceptor unwraps the typed message and extracts the `common.Envelope`.
2. The envelope payload is unmarshalled and the `ChannelHeader` is extracted.
3. If mTLS is enabled, the claimed TLS certificate hash inside `ChannelHeader.TlsCertHash` is compared with the actual TLS certificate hash extracted from the gRPC context via `util.ExtractCertificateHashFromContext(ctx)`. A mismatch returns `codes.Unauthenticated`.
4. `protoutil.EnvelopeAsSignedData()` extracts the serialized identity, payload, and signature into a `SignedData` structure.
5. The interceptor maps `info.FullMethod` to an ACL resource name and looks up the policy in the bundle's `PolicyManager`.
6. `policy.EvaluateSignedData(signedData)` verifies the signature and validates the identity against the channel's MSP definitions.
7. On success, the handler is invoked.

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

// Verify TLS cert hash binding.
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
return nil, status.Errorf(codes.Unauthenticated,
"TLS cert hash mismatch: claimed=%x, actual=%x",
claimedHash, actualHash)
}
}

// 2. Extract SignedData: serialized identity + signature + payload.
signedData, err := protoutil.EnvelopeAsSignedData(envelope)
if err != nil {
return nil, status.Errorf(codes.InvalidArgument,
"failed to extract signed data: %v", err)
}

// 3. Map the gRPC method to an ACL resource name and retrieve
//    the corresponding policy from the channel config bundle.
resource := methodToResource(info.FullMethod)
policyMgr := s.currentBundle.PolicyManager()
policy, exists := policyMgr.GetPolicy(resource)
if !exists {
return nil, status.Errorf(codes.PermissionDenied,
"no policy defined for resource: %s", resource)
}

// 4. Evaluate: verify the signature and validate the identity
//    against the channel's MSP definitions.
if err := policy.EvaluateSignedData(signedData); err != nil {
return nil, status.Errorf(codes.PermissionDenied,
"ACL check failed for [%s]: %v", resource, err)
}

return handler(ctx, req)
}
```

**Stream interceptor with sequence-based caching.**

Streaming services — `blockDelivery` and `OpenNotificationStream` —
require every message to carry a full signed envelope.
This is not optional:
on a bundle update we need to re-evaluate both the identity (against the updated MSP set)
and the policy decision (against the updated `ACLs` section),
and that re-evaluation needs the original signed data.

The resolved identity,
the configuration sequence observed at the first check, and the session's TLS certificate hash are cached.
Re-evaluation occurs
when the configuration sequence advances — catching mid-stream changes such as an organization
being removed from the channel.

```go
func (s *Server) ACLStreamInterceptor(
srv interface{},
ss grpc.ServerStream,
info *grpc.StreamServerInfo,
handler grpc.StreamHandler,
) error {
ctx := ss.Context()

// 1. Receive first envelope and validate TLS binding.
var firstEnvelope *common.Envelope
if err := ss.RecvMsg(&firstEnvelope); err != nil {
return status.Errorf(codes.InvalidArgument,
"failed to receive first envelope: %v", err)
}

payload, err := protoutil.UnmarshalPayload(firstEnvelope.Payload)
if err != nil {
return status.Errorf(codes.InvalidArgument,
"failed to unmarshal payload: %v", err)
}

chdr, err := protoutil.UnmarshalChannelHeader(payload.Header.ChannelHeader)
if err != nil {
return status.Errorf(codes.InvalidArgument,
"failed to unmarshal channel header: %v", err)
}

if s.mutualTLS {
claimedHash := chdr.TlsCertHash
actualHash := util.ExtractCertificateHashFromContext(ctx)

actualHash := util.ExtractCertificateHashFromContext(ctx)
if len(actualHash) == 0 {
return nil, status.Error(codes.Unauthenticated,
"client didn't send a TLS certificate")
}

if !bytes.Equal(actualHash, claimedHash) {
return status.Errorf(codes.Unauthenticated,
"TLS cert hash mismatch")
}
}

// 2. Extract identity for ACL check.
signedData, err := protoutil.EnvelopeAsSignedData(firstEnvelope)
if err != nil {
return status.Errorf(codes.InvalidArgument,
"failed to extract signed data: %v", err)
}

// 3. Initial ACL check.
resource := methodToResource(info.FullMethod)
currentSeq := s.currentBundle.ConfigtxValidator().Sequence()

policyMgr := s.currentBundle.PolicyManager()
policy, exists := policyMgr.GetPolicy(resource)
if !exists {
return status.Errorf(codes.PermissionDenied,
"no policy for resource: %s", resource)
}

if err := policy.EvaluateSignedData(signedData); err != nil {
return status.Errorf(codes.PermissionDenied,
"ACL check failed: %v", err)
}

// 4. Cache identity and sequence for subsequent messages.
session := &SessionAccessControl{
signedData:     signedData,
configSequence: currentSeq,
resource:       resource,
tlsCertHash:    chdr.TlsCertHash,
}

return handler(srv, &aclServerStream{
ServerStream: ss,
server:       s,
session:      session,
})
}
```

**Stream session caching behavior.**

| Situation | Behavior                                                                                                                                                                                                                               |
|---|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Initial handshake** | Full cryptographic verification (signature + policy evaluation). Identity, configuration sequence, and TLS certificate hash are cached.                                                                                                |
| **Configuration unchanged** | Sequence matches the cache and the identity did not changed. No cryptographic operation. Request proceeds immediately.                                                                                                                 |
| **Configuration changed** | `RecvMsg` detects the sequence increment and re-evaluates the cached identity against the updated policy set. If the identity no longer satisfies the policy (e.g., its organization was removed), the stream is terminated immediately. |

---

### Option B — Mutual TLS-Based ACL Enforcement

The client's identity is obtained from the mTLS certificate presented during the handshake.
No envelope wrapping is required and the existing proto APIs are unchanged.
A custom `TransportCredentials` wrapper resolves the certificate to an MSP identity at handshake time;
the gRPC interceptor then reads this pre-resolved identity from the connection's `AuthInfo` on each call
and evaluates it against the channel's Policy Manager.

For this to work,
the client's organizational (MSP)
identity must be embedded in the TLS certificate
so that the same certificate satisfies both the transport handshake and the MSP `DeserializeIdentity` path. 
A change in the `fabric-x-common`/`fabric-ca` CA certs providers is required to issue such certificates.

**Identity resolution at handshake.*
Identity resolution runs once during the mTLS handshake via a custom `TransportCredentials` wrapper. 
The wrapper iterates the channel MSPs to derive an `msp.Identity` from the client certificate
and attaches it to the connection's `AuthInfo`;
from that point on, this identity *is* the identity of the connection.
Every subsequent RPC reuses it and evaluates it against the current bundle's policy.
If the bundle has changed in a way that invalidates the identity—for example,
the client's organization was removed from the channel—the policy evaluation fails naturally, and the call is rejected.

**Precondition.
Option B requires mTLS.
Without a client certificate there is no identity to evaluate, and ACL enforcement cannot run at all.

**Pros.**

- No breaking changes to existing proto files.
- No client-side boilerplate; the client just presents its certificate as usual.

**Cons.**

- Collapses the conceptual separation between transport identity (TLS CA) and organizational identity (MSP CA).

**Server-side flow.**

1. During the mTLS handshake, the custom `TransportCredentials` wrapper resolves the client certificate to an `msp.Identity` by iterating the channel MSPs and attaches it to the connection's `AuthInfo` at resolution time. A certificate that no MSP recognizes causes the handshake to fail with `Unauthenticated`.
2. On each RPC, the interceptor reads the pre-resolved identity from `AuthInfo` and evaluates it against the current bundle's policy. If the bundle changed in a way that invalidates the identity, the evaluation fails naturally.
3. Map `info.FullMethod` to a resource name and look up the policy in the bundle's `PolicyManager`.
4. Invoke `policy.EvaluateIdentities([]msp.Identity{identity})` to perform the policy check.

> As with Option A, the code below is pseudocode.

**Unary interceptor.**

```go
func (s *Server) ACLInterceptor(
    ctx context.Context,
    req interface{},
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (interface{}, error) {

    p, ok := peer.FromContext(ctx)
    if !ok {
        return nil, status.Error(codes.Unauthenticated, "no peer found")
    }

    authInfo, ok := p.AuthInfo.(*MSPAuthInfo)
    if !ok {
        return nil, status.Error(codes.Unauthenticated,
            "connection did not complete MSP resolution")
    }

    // Identity is already resolved — no MSP iteration here.
    identity := authInfo.Identity

    resource := methodToResource(info.FullMethod)
    policy, exists := s.currentBundle.PolicyManager().GetPolicy(resource)
    if !exists {
        return nil, status.Errorf(codes.PermissionDenied,
            "no policy defined for resource: %s", resource)
    }

    if err := policy.EvaluateIdentities([]msp.Identity{identity}); err != nil {
        return nil, status.Errorf(codes.PermissionDenied,
            "ACL check failed for [%s]: %v", resource, err)
    }

    return handler(ctx, req)
}
```

**Streaming under Option B.**

The streaming model under Option B is materially simpler than under Option A.
The TLS certificate is fixed for the lifetime of the connection—gRPC negotiates it once at handshake time—and the resolved `msp.Identity` is attached to `AuthInfo` then and there.
As a result:

- **Identity does not need to be re-derived per message.** The MSP-iteration step runs once at handshake time, before the stream begins; the resolved identity is reused for the life of the connection.
- **Per-message work reduces to a policy re-evaluation, and only when the configuration sequence advances.** On each received message the interceptor compares the cached sequence with `s.currentBundle.ConfigtxValidator().Sequence()`. If equal, the call proceeds with no further work. If advanced, the cached identity is re-evaluated against the updated policy set and, if it no longer satisfies the policy, the stream is terminated.
- **Certificate expiration and revocation are handled by the gRPC/TLS layer.** A connection whose certificate becomes invalid is torn down at the transport layer.

**TLS certificate vs. MSP signing certificate.** Fabric distinguishes two certificates per client:

- The **TLS certificate** authenticates the transport connection. It is issued by the channel's TLS CA and proves network-level identity.
- The **MSP signing certificate** signs proposals and envelopes. It is issued by the organization's MSP CA and proves organizational identity.

Option B requires either embedding organizational identity into the TLS certificate or issuing the TLS certificate from a CA trusted by the MSP.

# Drawbacks
[drawbacks]: #drawbacks

- **Per-call cost.** Every unary RPC pays for a policy evaluation.
- **Option A specifically.** Breaking changes to three proto files force every existing client to be updated. Clients must implement envelope construction, signing, and TLS certificate hash inclusion.


# Prior art
[prior-art]: #prior-art

Hyperledger Fabric uses essentially the model proposed here (A).

**gRPC services and message types.
Fabric's ACL framework rests on two primary signed protobuf message types.
The first, `SignedProposal`, 
is used for chaincode execution and ledger interactions;
it is handled by the `Endorser.ProcessProposal()` gRPC service
and is the standard for invoking smart contracts or querying the ledger via system chaincodes.
The second, `common.Envelope`, is used for retrieving blocks from the network via the Delivery service,
implemented on both orderer and peer, which streams blocks to clients.

**Endorser service flow and ACL integration.**The lifecycle of a chaincode invocation begins at the Endorser Service.
Defined in `peer.proto`, the `ProcessProposal` gRPC method is the entry point for all client proposals.
When a `SignedProposal` reaches the gRPC handler, the system performs a multi-step validation:
the proposal is unpacked and validated for message integrity,
then enters the `preProcess` phase where validation and ACL checks occur.
The `SignedProposal` contains the serialized client identity in `SignatureHeader.Creator`,
the target chaincode name, and the `ChannelID`.

Fabric's design employs a two-tier ACL architecture:

- **Application chaincodes.** The ACL check occurs at the endorser level, validating against `/Channel/Application/Writers` before chaincode execution.
- **System chaincodes (QSCC, CSCC, lifecycle).** The endorser-level ACL check is skipped. Each system chaincode instead performs its own internal ACL check inside its `Invoke()` method. For example, QSCC extracts the `SignedProposal` from the stub and validates it against function-specific policies (e.g., `qscc/GetBlockByNumber` requires `/Channel/Application/Readers`).

**Policy mapping and configuration storage.
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

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- **Option A vs. Option B.** This RFC presents both options for team discussion and does not pre-select one. The trade-offs are spelled out per option above.
- **Resource naming convention.** The examples in this RFC use `service/method` (e.g., `query/GetTransactionStatus`). The alternative is `package/method`.
- **Behavior of in-flight streams on configuration change.** When a new configuration block arrives, streams established under the previous configuration are still alive. If the new block removes an organization, that organization's streams keep processing until the change is detected. Three paths:
    - **(a) Lazy re-authorization.** Re-authorize on the next received message; terminate if the client no longer passes. Trade-off: a bounded staleness window between block arrival and next message.
    - **(b) Tear down all streams.** Close every active stream on any configuration block; clients reconnect. Clean, but disrupts unaffected clients and risks a reconnect burst.
    - **(c) Proactive re-authorization.** Re-authorize active sessions in the background on a new bundle; terminate only those that fail. Cleanness of (b) without the disruption, at the cost of extra bookkeeping.

# Dependencies
[dependencies]: #dependencies

- The MSP, policy, and channel-configuration packages from `fabric-x-common` (`protoutil`, `msp`, `policies`, channel-config bundle construction).
- `sampleconfig/configtx.yaml` in `fabric-x-common` to add a sample `ACLs` section.
- **For Option A:** coordinated changes to `query.proto`, `notify.proto`, and `block_query.proto`, and corresponding client library updates.
- **For Option B:** changes to the `fabric-x-common`/`fabric-ca` CA certs provider to set the organizational identity in TLS certificates.
