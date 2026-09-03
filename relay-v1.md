# AgentPulse Relay v1

Relay v1 is an opaque bidirectional public path for Native Transport v3 and the required
transport for QR-only first pairing. A successful QR pairing stores and selects
its authenticated Relay endpoint. A paired client may still explicitly select
LAN later; there is no silent route fallback.

Relay terminates a publicly trusted outer TLS connection, authenticates a Host
registration and a paired-client route, then becomes an opaque byte pump. The
existing Host-CA TLS and Native v3 WebSocket run inside the pump. Consequently,
Relay can observe connection metadata and byte counts, but it cannot decode the
inner Native bearer Token, Session/Event messages, or Host certificate identity.

## Endpoint and framing

- The endpoint is the canonical lowercase ASCII DNS authority `host:port`. IP
  literals, schemes, paths, user info, fragments, trailing dots, and port zero
  are invalid. This exact UTF-8 authority is a KDF input.
- Outer TLS validates the public DNS name with normal platform trust roots.
- Before tunneling, each message is a 4-byte unsigned big-endian length followed
  by strict UTF-8 JSON. Length must be 1..16384 bytes.
- Every JSON value is `{"relay_version":1,"message":...}`. Unknown fields,
  unknown message types, and non-v1 versions fail closed.
- `TunnelReady` is the final framed message. Every later byte belongs to the
  opaque inner TLS stream and must not be interpreted by Relay.

## Derivation

All HMAC operations use HMAC-SHA-256. `||` denotes byte concatenation. Text is
UTF-8, UUIDs are their 16 network-order bytes, integers are big-endian, and all
32-byte wire values use unpadded Base64URL.

The endpoint peers share a route secret that Relay never receives. Stable Native
routes use the device-specific Native bearer Token; ephemeral QR pairing routes
use the QR bootstrap Token. Both derive:

```text
route_root = SHA-256(route_secret UTF-8)
route_id = HMAC(route_root,
  "agentpulse.relay.v1.route\0" || endpoint_authority)
client_auth_key = HMAC(route_root,
  "agentpulse.relay.v1.client-auth\0" || endpoint_authority)
```

Relay initialization generates a one-time Host enrollment Token. The Host keeps
that Token in a private file and Relay stores only `SHA-256(token UTF-8)` as its
Host authentication key.

## Authentication state machine

Relay first sends `Challenge(connection_id, nonce, expires_at_unix_seconds)`.
The UUIDv7 connection ID and random 32-byte nonce are unique per outer
connection; the challenge expires after ten seconds.

Every proof begins with:

```text
prefix(role) =
  "agentpulse.relay.v1.proof\0" ||
  role_u8 ||
  connection_id_uuid_bytes ||
  nonce_32 ||
  expiry_i64
```

The Host sorts 1..16 unique registrations by raw `route_id` text and sends:

```text
host_proof = HMAC(host_auth_key,
  prefix(1) || host_id_uuid_bytes || route_count_u16 ||
  (route_id_32 || client_auth_key_32) repeated)
```

After verification, Relay installs one of at most four concurrent Host
registrations. Their route IDs must be disjoint. Relay sends `HostWaiting`, then
sends a UUIDv7 `Ping` after
15 seconds without a client; Host must return the matching `Pong` within five
seconds. A changed paired-device set closes and refreshes the registration.

A client sends:

```text
client_proof = HMAC(client_auth_key, prefix(2) || route_id_32)
```

Relay atomically claims the matching waiting Host registration and sends each
peer one `TunnelReady`. There are at most four waiting Host registrations, at
most 16 routes per registration, and at most 32 outer connections per Relay
process. Separate registrations allow a stable Native route set and a
short-lived QR pairing route to coexist. Route registrations are memory-only
and disappear on restart.

## Tunnel limits and failures

Each direction has a fixed 64 KiB userspace buffer. An opaque tunnel closes after
60 seconds without bytes in either direction or after ten seconds of an
undrainable full buffer. Native v3 retains its own 1 MiB inner message limit and
authorization rules. Revoking a device is therefore enforced by the Host even
during the bounded Relay route-refresh interval.

Stable errors are `authentication_failed`, `invalid_handshake`,
`invalid_request`, `host_unavailable`, `host_busy`, `resource_limit`, and
`internal`. Authentication errors do not reveal whether the route, key, or proof
was wrong. The Host treats all Relay failures as optional-path health failures;
they never stop the LAN Native listener.

Canonical, non-production test vectors are in [`fixtures/relay-v1`](fixtures/relay-v1).
