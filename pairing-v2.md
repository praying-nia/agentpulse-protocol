# Direct discovery v2 / 直连二维码发现包 v2

This is a discovery-bundle extension, not a new pairing WebSocket protocol.
Existing Relay deployments still emit the exact v1 bundle. New Android clients
accept both. A v2 bundle uses:

```text
agentpulse://pair/v2/<base64url-without-padding(JSON)>
```

Its exact fields are `pairing_version` (integer 2), `pairing_id`, `host_id`,
`host_name`, `server_name`, `address`, `port`, `leaf_sha256`, `bootstrap_token`,
`route` (exactly `direct`), and `expires_at_unix_seconds`. Identity, fingerprint,
secret and expiry semantics are inherited from [Pairing v1](pairing-v1.md).
`relay_endpoint` is absent and rejected, including null. URI and bundle versions
must agree; unknown fields/routes/versions and expired bundles are rejected.

`address` is a unicast IP literal or ASCII DNS name without brackets, scheme,
port, scope ID, path or query; `port` is 1..65535. These identify the **external
pairing destination**, independently of the local listener. Configuration CLI
IPv6 authorities use `[address]:port`; the decoded address itself has no brackets.
The client resolves all DNS addresses and connects directly, without a Relay
socket factory or automatic Relay fallback.

TLS still authenticates the stable `server_name`, with the exact QR leaf
fingerprint. The destination IP/DNS name does not replace TLS identity or require
a public CA certificate. The underlying WebSocket keeps `/agentpulse/pair/v1`,
subprotocol `agentpulse.pair.v1`, and `pairing_version: 1` message envelopes.
Native/domain protocol versions are unchanged.

After terminal approval, `pairing_succeeded.native_address/native_port` contain
the configured **external Native destination**, not the temporary pairing port
or local bind address. Direct DNS addresses may be up to 253 bytes. Android
stores this destination with the issued CA and device token, selects `DIRECT`,
and reconnects using that destination without mDNS replacement. Unknown or
changed Host identity still fails. Existing LAN/RELAY profiles keep their routes.

The Host uses an explicitly configured concrete unicast bind IP and two distinct
fixed TCP ports (defaults: Native 49320, pairing 49321). Port mapping may use
different external ports. Pairing opens its port only for one session, retains
the two-minute deadline/five-attempt limit/terminal approval, and closes on
success, rejection, expiry or process exit. Direct terminal approval is bounded
by the remaining session lifetime; late approval cannot issue credentials.
No port auto-discovery, automatic firewall changes or reachability claims are made.

Saved direct settings take effect at the next Host startup. `pair` reads the
running Host's effective settings; direct takes precedence over Relay for new
pairing. Removing direct settings and restarting restores legacy selection.
Existing Relay connectors remain enabled when configured. Old Android clients
must upgrade to scan direct v2; their legacy Relay pairing remains supported.

Canonical sample: [direct bundle](fixtures/pairing-v2/pairing_bundle.json).
