# Nostr Wallet Auth (NWA)

**Status:** draft optional

## Abstract

Nostr Wallet Auth (NWA) defines an app-initiated authorization flow for creating a NIP-47 / Nostr Wallet Connect (NWC) connection without manually copying an NWC URI between apps.

NWA supports two setup modes:

- **Client-created secret mode:** the requesting app generates the NWC client secret locally before opening the wallet. The app shares only the corresponding client public key, requested capabilities, relays, and callback information with the wallet. The wallet records the approved policy for that client public key and returns the wallet service public key and selected relay information. The requesting app then assembles the final `nostr+walletconnect://` URI locally using its own secret.
- **Wallet-created secret mode:** the wallet creates the NWC client secret after user approval and returns a complete `nostr+walletconnect://` URI to the requesting app. Because the callback carries a spend-capable private key, this mode is limited to same-device handoffs with a callback destination whose ownership is verified by the operating system or another wallet-trusted mechanism.

This specification does not change NIP-47 request or response events. It standardizes the setup handoff between an app and a wallet.

This flow is inspired by OAuth redirect UX, but it is not OAuth. It does not define OAuth clients, access tokens, refresh tokens, authorization codes, token endpoints, or OpenID Connect behavior.

### Client-Created Secret Flow

```mermaid
sequenceDiagram
    participant A as "Alby Go"
    participant R as "Rebel Wallet App"
    participant NS as "Notification Server"
    participant Relay as "Nostr Relay"
    participant NSE as "Rebel NSE"

    Note over A: Creates NWC client keypair
    A->>R: Open NWA request with Alby Go client pubkey
    R->>R: User approves permissions, budget, relays
    R->>R: Store Alby Go pubkey + Rebel wallet-service keypair
    R->>NS: Register push target + watched pubkeys/relays
    R-->>A: Return Rebel wallet-service pubkey + relay

    Note over A: Alby Go keeps its private key
    A->>Relay: Publish encrypted NWC request to Rebel pubkey

    NS->>Relay: Watches for NWC request events
    Relay-->>NS: Event for Rebel wallet-service pubkey
    NS->>NSE: Send APNs push with event id + relay

    NSE->>Relay: Fetch NWC request event
    Relay-->>NSE: Encrypted NWC request
    NSE->>NSE: Decrypt using Rebel private key + Alby Go pubkey
    NSE->>NSE: Check permissions/budget
    NSE->>NSE: Run wallet action
    NSE->>Relay: Publish encrypted NWC response to Alby Go pubkey

    A->>Relay: Wait for response
    Relay-->>A: Encrypted NWC response
    A->>A: Decrypt using Alby Go private key + Rebel pubkey
```

### Wallet-Created Secret Flow

> Security warning: wallet-created secret mode returns secret-bearing NWC material through the app handoff. It MUST only be used for same-device flows with a verified callback destination, such as an iOS Universal Link, Android App Link, or another platform channel cryptographically bound to the requesting app. A custom URI scheme alone is not verified because another installed app can register the same scheme and intercept the NWC secret. Web, server, cross-device, and unverified custom-scheme flows MUST use client-created secret mode because a returned NWC URI can also be exposed through browser history, server logs, analytics, referrers, crash reports, screenshots, or intermediate redirect handlers.

```mermaid
sequenceDiagram
    participant A as "Alby Go"
    participant R as "Rebel Wallet App"
    participant NS as "Notification Server"
    participant Relay as "Nostr Relay"
    participant NSE as "Rebel NSE"

    A->>R: Open NWA request without client pubkey
    R->>R: User approves permissions, budget, relays
    R->>R: Create NWC client secret + wallet-service keypair
    R->>R: Store generated client pubkey + Rebel wallet-service keypair
    R->>NS: Register push target + watched pubkeys/relays
    R-->>A: Return full nostr+walletconnect URI

    Note over A: Alby Go stores secret from returned NWC URI
    A->>Relay: Publish encrypted NWC request to Rebel pubkey

    NS->>Relay: Watches for NWC request events
    Relay-->>NS: Event for Rebel wallet-service pubkey
    NS->>NSE: Send APNs push with event id + relay

    NSE->>Relay: Fetch NWC request event
    Relay-->>NSE: Encrypted NWC request
    NSE->>NSE: Decrypt using Rebel private key + generated client pubkey
    NSE->>NSE: Check permissions/budget
    NSE->>NSE: Run wallet action
    NSE->>Relay: Publish encrypted NWC response to generated client pubkey

    A->>Relay: Wait for response
    Relay-->>A: Encrypted NWC response
    A->>A: Decrypt using returned NWC secret + Rebel pubkey
```

## Motivation

Today, mobile NWC setup often requires:

1. Open wallet app.
2. Create an NWC string.
3. Copy it.
4. Switch apps.
5. Paste it into the requesting app.

That works, but it is clunky on mobile and pushes secret material through the clipboard. It also gives the wallet little structured context about the requesting app, requested methods, suggested budget, and preferred relays.

NWA improves this flow:

1. Requesting app opens a wallet auth URI.
2. Wallet app opens directly to an NWC approval screen.
3. User reviews and approves wallet-side policy.
4. Wallet returns either connection metadata or a complete NWC URI.
5. Requesting app stores the normal NWC connection.

## Prior Art And Traction

NWA is not a brand-new idea. It appeared in earlier Mutiny Wallet and ZapplePay work and is still present in Alby's current SDK documentation as "Nostr Wallet Auth" for mobile-first or self-custodial wallets.

NIP-47 also includes a simpler deep-link appendix using `nostrnwc://connect`, where the wallet returns a complete NWC pairing code. That flow improves mobile handoff but can still return a URI containing client secret material.

Newer NWC "1-click connection" work includes the client-created-secret model: the requesting app generates the client secret, shares only the client public key with the wallet, and constructs the final NWC URI locally after authorization. This draft keeps that model for web, server, cross-device, and unverified callback environments. It also defines wallet-created secret mode as a first-class profile for verified same-device mobile handoffs, where returning a complete NWC URI matches existing NIP-47 onboarding behavior and avoids manual copy and paste.

## Design Summary

Client-created secret mode:

```text
Requesting app
-> generates NWC client secret and client public key
-> opens NWA URI with client public key and requested policy
-> wallet app displays connection approval screen
-> user chooses permissions, budget, interval, and relays
-> wallet authorizes the client public key
-> wallet publishes normal NIP-47 info metadata
-> wallet redirects to return URI with wallet public key and relay metadata
-> requesting app constructs normal nostr+walletconnect URI locally
```

Wallet-created secret mode:

```text
Requesting app
-> opens NWA URI without a client public key
-> wallet app displays connection approval screen
-> user chooses permissions, budget, interval, and relays
-> wallet creates NWC client secret and authorizes generated client public key
-> wallet publishes normal NIP-47 info metadata
-> wallet redirects to return URI with complete nostr+walletconnect URI
-> requesting app stores returned nostr+walletconnect URI
```

The resulting connection is still a normal NWC connection:

```text
nostr+walletconnect://<wallet_service_pubkey>?relay=<relay>&secret=<client_secret>
```

The wallet auth request is only a setup handoff.

## Goals

This specification aims to:

- Avoid manual copy/paste for NWC setup.
- Avoid NWC client secret handoff through the clipboard and limit callback handoff to verified same-device app channels.
- Permit complete NWC URI returns when needed for compatibility.
- Let requesting apps open a wallet-specific or generic wallet auth flow.
- Let wallets show a native approval screen before authorizing NWC access.
- Let wallets keep permission, budget, and policy decisions wallet-side.
- Work on iOS, Android, and web, including an explicit relay-mediated completion mode for cross-device authorization.
- Preserve standard NIP-47 connection URI semantics.
- Stay compatible with existing NWA and NIP-47 one-click connection ideas where practical.

## Non-goals

This specification does not define:

- A replacement for NIP-47.
- OAuth or OpenID Connect compatibility.
- Server-side token exchange.
- A wallet discovery registry.
- A mandatory permission model for all wallets.
- A required UI.
- A guarantee that every platform can show a generic wallet chooser.

## Terms

### Requesting app

The app that wants wallet access. Examples include a point-of-sale app, bill-pay app, subscription app, web app, or another wallet-adjacent app.

### Wallet app

The app that authorizes the NWC connection for the user.

### Wallet service public key

The NIP-47 wallet service public key that receives encrypted NWC requests.

### Client secret

The NWC client secret key used to sign NWC request events and derive the client public key. In client-created secret mode it is generated and retained by the requesting app. In wallet-created secret mode it is generated by the wallet and transferred to the requesting app inside the returned NWC URI.

### Client public key

The public key corresponding to the client secret. This is shared with the wallet during NWA setup and authorized by the wallet.

### Wallet auth request

The URI opened by the requesting app to start NWC authorization in a wallet app.

### Return URI

The callback URI supplied by the requesting app. The wallet app opens this URI after approval, cancellation, or error.

### NWC URI

The normal NIP-47 connection URI constructed by the requesting app in client-created secret mode or returned by the wallet in wallet-created secret mode.

## URI Forms

Implementations SHOULD support at least one of these forms.

### Generic NWA Scheme

The generic scheme is intended for platforms that can route a URI to one of several capable wallets.

```text
nostr+walletauth://connect
```

Example:

```text
nostr+walletauth://connect?version=1&name=Alby%20Go&app_id=com.alby.go&pubkey=687dd8ece211539364549b1f32c63eceec1e0661009ba65cf8ff2e73ba000746&return_to=com.alby.go%3A%2F%2Fnwa%2Fcallback&state=4f7c8b9a21d64099a7352b97c927c341&relay=wss%3A%2F%2Frelay.getalby.com&relay=wss%3A%2F%2Frelay2.getalby.com&request_methods=pay_invoice%20get_balance%20make_invoice&max_amount=500000000&budget_renewal=monthly
```

### Wallet-Specific NWA Scheme

A wallet-specific scheme is recommended when the requesting app wants to open a specific installed wallet.

```text
nostr+walletauth+rebelwallet://connect
```

Example:

```text
nostr+walletauth+rebelwallet://connect?version=1&name=Alby%20Go&app_id=com.alby.go&pubkey=687dd8ece211539364549b1f32c63eceec1e0661009ba65cf8ff2e73ba000746&return_to=com.alby.go%3A%2F%2Fnwa%2Fcallback&state=4f7c8b9a21d64099a7352b97c927c341&relay=wss%3A%2F%2Frelay.getalby.com&request_methods=pay_invoice%20get_balance
```

Wallets MAY also support private app schemes, such as:

```text
rebelwallet://nwa/connect
```

This document standardizes the query parameters and response behavior, not every wallet's private scheme name.

### HTTPS / Universal Link

Wallets MAY expose an HTTPS entry point for web, iOS Universal Links, Android App Links, and install fallback pages.

```text
https://wallet.example/nwa/connect
```

Example:

```text
https://wallet.example/nwa/connect?version=1&name=Alby%20Go&app_id=https%3A%2F%2Falby-go.example&pubkey=687dd8ece211539364549b1f32c63eceec1e0661009ba65cf8ff2e73ba000746&return_to=https%3A%2F%2Falby-go.example%2Fnwa%2Fcallback&state=4f7c8b9a21d64099a7352b97c927c341&relay=wss%3A%2F%2Frelay.getalby.com&request_methods=pay_invoice%20get_balance
```

When returning non-secret connection metadata to an HTTPS callback, wallets SHOULD place result parameters in the URI fragment, not the query string. When returning a complete NWC URI in wallet-created secret mode, wallets MUST place result parameters in the URI fragment.

## Request Parameters

The wallet auth request uses query parameters.

Parameter names and values MUST be UTF-8 percent-encoded. Repeated parameters are allowed where explicitly defined, such as `relay`.

| Parameter | Requirement | Description |
| --- | --- | --- |
| `version` | REQUIRED | Protocol version. MUST be `1`. |
| `secret_mode` | OPTIONAL | `client` or `wallet`. Defaults to `client` when `pubkey` is present, otherwise `wallet`. |
| `pubkey` | CONDITIONALLY REQUIRED | Client public key generated by the requesting app. Required for client-created secret mode. Omitted for wallet-created secret mode. |
| `response_mode` | OPTIONAL | `callback` or `relay`. Defaults to `callback` when `return_to` is present. |
| `return_to` | CONDITIONALLY REQUIRED | Callback URI opened after approval, cancellation, or error. Required for `response_mode=callback`; omitted for `response_mode=relay`. |
| `state` | REQUIRED | Opaque nonce generated by the requesting app. Returned unchanged. |
| `name` | RECOMMENDED | Human-readable requesting app name. |
| `app_id` | RECOMMENDED | Platform app id, bundle id, package name, or web origin. |
| `icon` | OPTIONAL | HTTPS URL for the requesting app icon. |
| `relay` | RECOMMENDED | Preferred relay URL. MAY appear more than once. |
| `request_methods` | OPTIONAL | Space-separated requested NWC methods. |
| `notification_types` | OPTIONAL | Space-separated requested NIP-47 notification types. |
| `max_amount` | OPTIONAL | Suggested maximum spend amount in millisatoshis per budget period. |
| `budget_renewal` | OPTIONAL | Suggested budget renewal period. |
| `expires_at` | OPTIONAL | Unix timestamp after which the connection should be rejected or expire. |
| `isolated` | OPTIONAL | Boolean hint requesting an isolated wallet service key for this connection. |
| `metadata` | OPTIONAL | URL-encoded JSON object for display-only metadata. |

Wallets MAY ignore optional parameters.

Wallets MUST NOT treat requested methods, budgets, notification types, or metadata as user consent. The user must approve the final wallet-side policy in the wallet app.

Single-valued parameters MUST NOT appear more than once. Wallets MUST reject duplicate single-valued parameters rather than choosing the first or last value. Only parameters explicitly documented as repeatable, such as `relay`, may appear more than once.

Wallets MUST apply implementation-defined but documented limits to the total URI length, number of relays, metadata length, display-name length, and callback length before presenting approval UI.

### `version`

```text
version=1
```

Wallets MUST reject unsupported versions.

### `secret_mode`

The `secret_mode` parameter declares which side creates the NWC client secret.

Allowed values:

- `client`: the requesting app creates the NWC client secret and includes `pubkey`.
- `wallet`: the wallet creates the NWC client secret after user approval and returns a complete NWC URI.

If `secret_mode` is omitted and `pubkey` is present, wallets SHOULD treat the request as `secret_mode=client`.

If `secret_mode` is omitted and `pubkey` is absent, wallets MAY treat the request as `secret_mode=wallet`.

Requesting apps SHOULD use `secret_mode=client` for web, server, cross-device, and unverified callback environments. Native same-device apps MAY use `secret_mode=wallet` when the return channel satisfies the verified app-only callback requirements in this specification.

### `response_mode`

The `response_mode` parameter declares how the wallet returns an authorization result.

Allowed values:

- `callback`: the wallet opens `return_to` on the same device.
- `relay`: the wallet publishes a signed, connection-specific NIP-47 info event to a bootstrap relay. This mode is only valid with `secret_mode=client`.

If `response_mode` is omitted and `return_to` is present, wallets SHOULD treat the request as `response_mode=callback`.

If `response_mode=relay`, `pubkey`, `state`, and at least one `relay` are REQUIRED and `return_to` MAY be omitted. Wallets MUST reject `response_mode=relay` with `secret_mode=wallet` because a relay response must never contain the client secret.

### `pubkey`

The `pubkey` value is the NWC client public key controlled by the requesting app.

```text
pubkey=687dd8ece211539364549b1f32c63eceec1e0661009ba65cf8ff2e73ba000746
```

In client-created secret mode, the requesting app MUST retain the corresponding client secret locally and MUST NOT include it in the NWA request URI.

Wallets SHOULD authorize exactly this client public key for the approved policy.

In wallet-created secret mode, `pubkey` is omitted. The wallet creates a client secret after approval, derives the generated client public key, stores that generated client public key in the approved policy, and returns a complete NWC URI containing the generated client secret.

### `return_to`

The `return_to` value is the callback URI that the wallet opens after the user approves, cancels, or an error occurs.

Examples:

```text
com.alby.go://nwa/callback
https://alby-go.example/nwa/callback
```

The requesting app MUST use a return URI that it controls.

The `return_to` URI SHOULD NOT include a fragment. Wallets use the fragment to return result parameters. If `return_to` already contains a fragment, wallets SHOULD reject the request with `invalid_request`.

For wallet-created secret mode, the wallet MUST verify that the callback destination is bound to the requesting app before approval. Acceptable bindings include:

- an iOS Universal Link validated by the operating system
- an Android App Link validated through Digital Asset Links
- an operating-system callback capability that is bound to the invoking app and cannot be claimed by another app
- a callback origin or application identifier that the wallet provider has explicitly registered and verified

A custom URI scheme by itself is not a verified callback destination. Wallets MUST NOT return `nwc_uri`, `value`, or any other secret-bearing result to an unverified custom scheme.

A valid association file alone does not make secret delivery safe. The wallet MUST also invoke the callback using a platform operation that succeeds only when the verified app handles the URL. The wallet MUST NOT use an operation that may fall back to a web browser.

If the platform reports that no associated app accepted the callback, the wallet MUST treat delivery as failed, MUST NOT retry through a browser or custom URI scheme, and MUST revoke the newly created NWC connection and any related wake registration.

In client-created secret mode, a wallet MAY return non-secret connection metadata to an unverified custom scheme, but it SHOULD display the callback destination and clearly label the requesting identity as unverified.

For an HTTPS callback, the wallet MUST require an exact origin match between `app_id` and `return_to` when `app_id` is an origin. Path matching and any registered callback allowlist are wallet-provider policy.

Wallet-created secret requests MUST include a verifiable `app_id` or arrive with a platform callback capability that identifies the invoking app. A self-asserted native package or bundle identifier is not sufficient by itself.

### `state`

The requesting app MUST generate a fresh, unpredictable `state` value containing at least 128 bits of entropy for each wallet auth request.

Wallets MUST return the exact `state` value in the callback.

Requesting apps MUST verify that the returned `state` matches an outstanding wallet auth request before accepting the returned connection metadata.

### `name`

The wallet SHOULD display `name` to the user.

Example:

```text
name=Alby%20Go
```

### `app_id`

The wallet MAY display or log `app_id`. On native platforms this is commonly a bundle id or package name. On web it SHOULD be the web origin.

Examples:

```text
app_id=com.alby.go
app_id=https%3A%2F%2Falby-go.example
```

The wallet SHOULD NOT trust `app_id` as proof of identity unless it is verified by the platform or by an HTTPS origin relationship.

### `icon`

The wallet MAY display an icon from an HTTPS URL.

Wallets MUST treat the icon URL as untrusted display data. Wallets SHOULD avoid fetching arbitrary non-HTTPS icon URLs.

### `relay`

The request MAY contain one or more relay parameters.

```text
relay=wss%3A%2F%2Frelay.getalby.com
relay=wss%3A%2F%2Frelay2.getalby.com
```

Wallets MAY:

- accept all requested relays
- choose one requested relay
- add their own default relays
- return wallet-selected relays in the callback
- let the user edit relays
- reject unsupported relays

The requesting app MUST use the relay or relays returned by the wallet when constructing the final NWC URI. If the wallet returns no relay, the requesting app MAY use the requested relay set.

Relay URLs are untrusted network destinations. Wallets and wallet-operated wake providers MUST validate relay URLs before connecting. Hosted wake providers MUST:

- require `wss`
- reject URLs containing user information or fragments
- limit the number of relays and permitted destination ports
- reject loopback, private, link-local, multicast, and otherwise non-public destinations after DNS resolution unless the operator explicitly configured that destination
- apply the same destination checks after redirects and subsequent DNS resolutions

Wallet providers MAY use a relay allowlist instead of accepting arbitrary requested relays. A local wallet MAY permit a user-confirmed private relay for direct device use, but it MUST NOT cause a shared hosted wake provider to connect to that destination unless the wake-provider operator explicitly allowed it.

### `request_methods`

The `request_methods` parameter is a space-separated list of requested NWC methods.

Example:

```text
request_methods=pay_invoice%20get_balance%20make_invoice%20list_transactions
```

Wallets SHOULD map requested methods to wallet-side permission controls when possible.

Wallets MAY add mandatory safe methods such as `get_info`.

Wallets MAY show presets such as:

- Full Access
- Read Only
- Custom

### `notification_types`

The `notification_types` parameter is a space-separated list of requested NIP-47 notification types.

Example:

```text
notification_types=payment_received%20payment_sent
```

Wallets MAY ignore unsupported notification types.

### `max_amount`

The requesting app MAY suggest a maximum spend amount in millisatoshis.

```text
max_amount=500000000
```

Implementations MUST interpret `max_amount` as millisatoshis, matching NIP-47 amount fields.

The wallet SHOULD display the budget to the user and MAY let the user edit it.

### `budget_renewal`

The requesting app MAY suggest a budget renewal interval.

Recommended values:

- `never`
- `hourly`
- `daily`
- `weekly`
- `monthly`
- `yearly`

Example:

```text
budget_renewal=monthly
```

Wallets MAY support additional intervals, but SHOULD ignore unknown values from the request.

### `expires_at`

The requesting app MAY include an expiration timestamp.

```text
expires_at=1783555200
```

Wallets SHOULD reject expired requests before showing approval UI.

If the wallet stores an expiration policy for the connection, it SHOULD enforce that policy against future NWC requests.

### `isolated`

The requesting app MAY set `isolated=true` to ask the wallet to use a unique wallet service public key for this connection.

Wallets MAY ignore this hint. Unique wallet service public keys can reduce metadata linkage across connections, but they may increase wallet implementation complexity.

### `metadata`

The `metadata` parameter MAY contain URL-encoded JSON for display-only context.

Example decoded value:

```json
{
  "purpose": "Monthly rent autopay",
  "account_name": "Apartment 4B"
}
```

Wallets MUST treat metadata as untrusted display data. It MUST NOT grant permissions or override wallet policy.

## Wallet Behavior

After receiving a wallet auth request, the wallet SHOULD:

1. Parse the URI.
2. Verify `version`.
3. Determine `secret_mode`.
4. For client-created secret mode, verify `pubkey` is present and well formed.
5. For wallet-created secret mode, verify `pubkey` is absent or ignore it.
6. Verify `response_mode` and its conditionally required parameters.
7. Verify `state` is present.
8. Reject the request if `expires_at` is expired.
9. Show a wallet-native approval screen.
10. Display the requesting app name and requested capabilities.
11. Let the user choose wallet-side permissions, budget, interval, and relays.
12. In client-created secret mode, authorize the requested client public key only after explicit user approval.
13. In wallet-created secret mode, create the NWC client secret and authorize the generated client public key only after explicit user approval.
14. Publish or update the normal NIP-47 info event for the selected wallet service public key.
15. Return the result through the selected response mode.

Wallets SHOULD NOT authorize a requested or generated client public key before the user approves.

Wallets SHOULD let the user cancel.

Wallets MAY remember trusted requesting apps, but SHOULD still show the final connection policy before creating a spend-capable NWC authorization.

## Return URI

For `response_mode=callback`, the wallet returns to the requesting app by opening the `return_to` URI with result parameters.

For non-secret custom-scheme callbacks, wallets MAY use query parameters or fragments. For HTTPS callbacks containing only non-secret metadata, wallets SHOULD use fragments so callback data is not sent to web servers in request paths or query strings. For wallet-created secret mode, the wallet MUST place `nwc_uri` or `value` in the fragment and MUST encode the complete NWC URI as one parameter value.

Percent-encoding is not required to have one canonical byte representation. Requesting apps MUST decode the parameter value exactly once and validate the resulting NWC URI, but MUST NOT require it to match the output of a particular platform's URL encoder. For example, JavaScript `encodeURIComponent` and Apple `URLComponents` can produce different valid encodings for the same parameter value.

Return parameter names and values MUST use the same encoding rules as request parameters.

### Approved: Client-Created Secret

```text
com.alby.go://nwa/callback#state=4f7c8b9a21d64099a7352b97c927c341&status=approved&wallet_pubkey=abcdef...&relay=wss%3A%2F%2Frelay.getalby.com&relay=wss%3A%2F%2Frelay2.getalby.com&lud16=user%40wallet.example
```

HTTPS example:

```text
https://alby-go.example/nwa/callback#state=4f7c8b9a21d64099a7352b97c927c341&status=approved&wallet_pubkey=abcdef...&relay=wss%3A%2F%2Frelay.getalby.com
```

The requesting app verifies `state`, reads the returned wallet service public key and relays, and constructs the final NWC URI locally:

```text
nostr+walletconnect://<wallet_pubkey>?relay=wss%3A%2F%2Frelay.getalby.com&secret=<local_client_secret>&lud16=user%40wallet.example
```

### Approved: Wallet-Created Secret

```text
https://alby-go.example/nwa/callback#state=4f7c8b9a21d64099a7352b97c927c341&status=approved&nwc_uri=<url-encoded-nwc-uri>
```

For compatibility with existing NIP-47 deep-link behavior, wallets MAY return the same value as `value` instead of, or in addition to, `nwc_uri`.

If both `nwc_uri` and `value` are present, they MUST contain the same decoded NWC URI.

```text
https://alby-go.example/nwa/callback#state=4f7c8b9a21d64099a7352b97c927c341&status=approved&value=<url-encoded-nwc-uri>
```

The decoded NWC URI is a normal NIP-47 connection URI:

```text
nostr+walletconnect://<wallet_pubkey>?relay=wss%3A%2F%2Frelay.getalby.com&secret=<wallet_generated_client_secret>&lud16=user%40wallet.example
```

The requesting app verifies `state`, decodes the returned NWC URI, stores it securely, and uses normal NIP-47.

### Cancelled

```text
com.alby.go://nwa/callback#state=4f7c8b9a21d64099a7352b97c927c341&status=cancelled
```

### Error

```text
com.alby.go://nwa/callback#state=4f7c8b9a21d64099a7352b97c927c341&status=error&error=unsupported_methods
```

Suggested error values:

- `invalid_request`
- `unsupported_version`
- `expired_request`
- `unsupported_methods`
- `unsupported_relay`
- `user_cancelled`
- `internal_error`

## Return Parameters

| Parameter | Requirement | Description |
| --- | --- | --- |
| `state` | REQUIRED | The original request `state`. |
| `status` | REQUIRED | `approved`, `cancelled`, or `error`. |
| `wallet_pubkey` | REQUIRED when approved in client-created secret mode | NIP-47 wallet service public key. |
| `relay` | RECOMMENDED when approved in client-created secret mode | Wallet-selected relay URL. MAY appear more than once. |
| `nwc_uri` | REQUIRED when approved in wallet-created secret mode | URL-encoded normal NWC URI containing the wallet-generated client secret. |
| `value` | OPTIONAL | Compatibility alias for `nwc_uri`, matching existing NIP-47 deep-link behavior. |
| `lud16` | OPTIONAL | Lightning address associated with the wallet/user. |
| `error` | RECOMMENDED when error | Machine-readable error code. |
| `error_description` | OPTIONAL | Human-readable error text. |

Requesting apps MUST reject callbacks with missing or mismatched `state`.

In client-created secret mode, requesting apps MUST reject `status=approved` callbacks without `wallet_pubkey`.

In wallet-created secret mode, requesting apps MUST reject `status=approved` callbacks without `nwc_uri` or `value`.

## Relay-Mediated Return

`response_mode=relay` is intended for cross-device authorization, such as a desktop web app displaying an NWA QR code that is approved by a wallet on a phone.

Example request:

```text
nostr+walletauth://connect?version=1&secret_mode=client&response_mode=relay&pubkey=687dd8ece211539364549b1f32c63eceec1e0661009ba65cf8ff2e73ba000746&state=4f7c8b9a21d64099a7352b97c927c341&relay=wss%3A%2F%2Frelay.getalby.com&request_methods=pay_invoice%20get_balance
```

The requesting app generates and retains the client secret, opens or displays an NWA request containing the corresponding `pubkey`, and subscribes to the requested bootstrap relays for a `kind:13194` event with both:

- a `p` tag containing the requesting app's client public key
- an `nwa` tag containing version `1` and the exact request `state`

After approval, the wallet MUST publish a signed, connection-specific NIP-47 info event to at least one requested bootstrap relay. The event:

- MUST be signed by the wallet service key authorized for this connection
- MUST use a wallet service key unique to this connection so the replaceable info event cannot overwrite another connection's metadata
- MUST include exactly one matching `p` tag
- MUST include exactly one matching `nwa` tag
- MUST include an `expiration` tag no more than 10 minutes in the future
- MUST list the methods and notification types approved for this connection using normal NIP-47 info-event fields
- MUST include one `relay` tag for each wallet-selected NWC relay
- MUST NOT contain the NWC client secret, an NWC URI, a push token, or decrypted wallet data

Example:

```json
{
  "kind": 13194,
  "pubkey": "<wallet_service_pubkey>",
  "content": "get_info get_balance pay_invoice",
  "tags": [
    ["p", "<client_pubkey>"],
    ["nwa", "1", "<state>"],
    ["relay", "wss://relay.getalby.com"],
    ["encryption", "nip44_v2"],
    ["expiration", "<unix_timestamp>"]
  ]
}
```

The requesting app MUST verify the event signature, `kind`, `p` tag, `nwa` version and state, expiration, and relay URLs before accepting it. It then uses the event pubkey as `wallet_pubkey`, combines the returned relay set with its locally retained client secret, and constructs the normal NWC URI.

The requesting app MUST stop waiting after its local authorization timeout, MUST ignore duplicate or unknown states, and SHOULD close the bootstrap subscription after accepting one valid response. The bootstrap relay is only an authorization return channel; the wallet-selected NWC relays are used for subsequent NIP-47 traffic.

## Wallet-Created Same-Device Profile

Wallet-created secret mode returns a complete `nostr+walletconnect://` URI through a callback parameter such as `nwc_uri` or `value`.

Wallet-created secret mode is suitable for native same-device app handoffs when the callback destination is verified and the wallet can require app-only delivery. It matches existing NIP-47 connection onboarding because the requesting app receives and stores a complete NWC URI.

Native mobile implementations MAY prefer wallet-created secret mode for this profile. Client-created secret mode remains preferred for web, server, cross-device, and unverified callback environments because the client secret never crosses the authorization return channel.

Wallet-created secret mode MUST NOT be used for web, server, cross-device, or unverified custom-scheme flows because the returned NWC URI contains secret key material and may be intercepted or exposed through browser history, server logs, analytics, referrers, crash reports, or intermediate redirect handlers.

Wallet-created secret mode MUST NOT return a complete NWC URI through an unverified custom URI scheme. Mobile implementations without a verified callback channel MUST use client-created secret mode.

Requesting apps MAY accept `value` or `nwc_uri` from trusted wallets only when the result arrived through a verified callback destination.

## Platform Guidance

### iOS

iOS does not provide a reliable generic chooser for arbitrary custom URI schemes. Requesting apps SHOULD use one of:

- a wallet-specific NWA scheme such as `nostr+walletauth+rebelwallet://connect`
- a wallet private scheme such as `rebelwallet://nwa/connect`
- a wallet Universal Link such as `https://wallet.example/nwa/connect`
- an in-app list of supported wallet apps

Wallet apps SHOULD register their custom scheme and, when possible, a Universal Link for install fallback and web compatibility.

Requesting apps SHOULD use a Universal Link `return_to` that they control. Custom-scheme callbacks MAY be used for client-created secret mode, but MUST NOT carry `nwc_uri`, `value`, or other secret material.

For wallet-created secret mode, the requesting app MUST associate the exact callback path with its application through an `apple-app-site-association` file and the Associated Domains entitlement. The association file MUST be served over HTTPS without redirects and MUST identify the intended application.

When returning a wallet-created secret, an iOS wallet MUST open the callback with the universal-link-only option. If the operation reports that the URL was not opened by an associated application, the wallet MUST fail the handoff, MUST NOT retry with a normal URL open, and MUST revoke the new NWC connection and related wake registration.

### Android

Android can route generic intent filters more naturally than iOS. Wallets MAY register an intent filter for:

```text
nostr+walletauth://connect
```

Requesting apps MAY either:

- open the generic URI and let Android resolve capable wallets
- target a specific wallet package with an explicit intent
- open a wallet-specific URI
- open an HTTPS App Link

Android requesting apps SHOULD still verify `state` on return.

Wallet-created secret mode MUST use a verified Android App Link or another callback capability explicitly bound to the invoking package. An implicit custom-scheme intent MUST NOT carry the NWC URI.

For an Android App Link, the requesting app MUST publish a Digital Asset Links statement for the exact application package and production signing-certificate fingerprint, and its callback intent filter MUST enable link verification.

When returning a wallet-created secret, an Android wallet MUST restrict the callback intent to the verified requesting package. On platforms that support them, it MUST also require a non-browser, unambiguous activity result. If the verified package cannot handle the callback, the wallet MUST fail the handoff, MUST NOT launch a generic browser intent, and MUST revoke the new NWC connection and related wake registration.

### Web

Web flows SHOULD use HTTPS URLs for both request and return and MUST use client-created secret mode.

Wallets returning connection metadata to a web app SHOULD place result parameters in the fragment.

Web apps SHOULD remove the fragment from browser history after consuming it.

Same-device web flows MAY use `response_mode=callback` with client-created secret mode. Cross-device web flows MUST use `response_mode=relay` or another explicitly standardized backchannel; opening an HTTPS callback on the wallet's device does not return authorization to a browser running on another device.

Example:

```text
https://wallet.example/nwa/connect?version=1&pubkey=687dd8ece211539364549b1f32c63eceec1e0661009ba65cf8ff2e73ba000746&return_to=https%3A%2F%2Fapp.example%2Fnwa%2Fcallback&state=4f7c8b9a21d64099a7352b97c927c341
```

Return:

```text
https://app.example/nwa/callback#state=4f7c8b9a21d64099a7352b97c927c341&status=approved&wallet_pubkey=abcdef...&relay=wss%3A%2F%2Frelay.getalby.com
```

## Security Considerations

### Client secret handling

In client-created secret mode, the requesting app MUST generate and retain the NWC client secret locally.

In client-created secret mode, the client secret MUST NOT be included in the NWA request URI.

Wallets SHOULD NOT ask the requesting app to send a client-created secret.

In wallet-created secret mode, the wallet creates the client secret and returns it inside a complete NWC URI. Wallets MUST return that URI through a verified, app-only callback destination, MUST place it in a fragment rather than a query string, and MUST NOT log it.

Wallet-created secret mode MUST be limited to verified same-device app flows. Web, server, cross-device, and unverified custom-scheme flows MUST use client-created secret mode.

### Callback data

In client-created secret mode, the approved callback does not need to contain client secret material. It SHOULD contain only the wallet service public key, relay metadata, and optional display metadata.

In wallet-created secret mode, the approved callback contains a complete NWC URI. Requesting apps MUST treat the returned `nwc_uri` or `value` as secret material.

Wallets and requesting apps SHOULD avoid logging full request and return URIs.

### State validation

The `state` parameter protects against confused-deputy and stale callback handling.

Requesting apps MUST:

- generate a fresh unpredictable `state`
- store it with the pending request and any local client secret
- verify it on return
- reject duplicate or unknown states

`state` correlates a response with a request. It does not authenticate the requesting app, prove ownership of `return_to`, or prevent another app from claiming an unverified custom scheme.

### User approval

Wallets MUST NOT interpret a wallet auth request as consent.

The user must approve the NWC connection in the wallet app.

Spend-capable methods such as `pay_invoice` SHOULD be clearly shown with budget and interval controls.

### Requesting app identity

`name`, `app_id`, and `icon` are self-asserted unless verified by the platform or by an HTTPS origin relationship.

Wallets SHOULD display these values as helpful context, not as cryptographic proof.

### Callback allowlists

Wallets MUST validate `return_to` before opening it and MUST reject malformed, dangerous, or unsupported schemes. Wallet-created secret mode MUST use a callback destination whose ownership is verified by the operating system or by a wallet-provider allowlist. Client-created secret mode MAY use an unverified custom scheme because the response contains no client secret, but the wallet SHOULD warn that the requesting identity is unverified.

For wallet-created secret mode, a wallet-provider allowlist identifies which application is expected to own a callback but does not replace fail-closed platform routing. The wallet MUST require the operating system to open the verified application and MUST reject browser fallback.

As defense in depth, an HTTPS callback path intended only for app handoff SHOULD serve an inert response without scripts, redirects, analytics, or third-party resources and SHOULD use a restrictive Content Security Policy and `Cache-Control: no-store`. This fallback does not make browser delivery acceptable.

Requesting apps SHOULD use stable return URI schemes or origins.

### Metadata

Metadata is untrusted display data. Wallets SHOULD sanitize metadata before display and MUST NOT grant permissions based only on metadata.

## Privacy Considerations

The wallet auth request may reveal the requesting app name, app id, icon URL, requested relays, requested methods, notification types, and suggested budget to the wallet.

The wallet service public key may reveal metadata if reused across many connections. Wallets SHOULD consider unique wallet service keys per connection when privacy matters.

The returned NWC URI gives the requesting app whatever wallet access the user approved. Requesting apps SHOULD store it securely and SHOULD provide users a way to disconnect.

Wallets SHOULD let users review and revoke NWC connections later.

## Compatibility With NWC Wake

NWA only creates or authorizes the NWC connection. If the wallet supports NWC Wake, the wallet can register the approved client public key, wallet service public key, selected relay set, and push token with its notification server after approval.

The requesting app does not need to know whether the wallet uses an always-on service, a foreground websocket, or NWC Wake behind the scenes. It should use the resulting NWC URI normally.

## Example: Client-Created Secret

### Requesting App Setup

The requesting app:

1. Generates a new NWC client secret.
2. Derives the client public key.
3. Stores the pending `state` with the client secret.
4. Opens the wallet auth URI.

### Request

```text
nostr+walletauth+rebelwallet://connect?version=1&name=Alby%20Go&app_id=com.alby.go&pubkey=687dd8ece211539364549b1f32c63eceec1e0661009ba65cf8ff2e73ba000746&return_to=com.alby.go%3A%2F%2Fnwa%2Fcallback&state=4f7c8b9a21d64099a7352b97c927c341&relay=wss%3A%2F%2Frelay.getalby.com&relay=wss%3A%2F%2Frelay2.getalby.com&request_methods=pay_invoice%20get_balance%20make_invoice&max_amount=500000000&budget_renewal=monthly
```

### Wallet Approval

The wallet displays:

- App: Alby Go
- Methods: `pay_invoice`, `get_balance`, `make_invoice`
- Relays: `wss://relay.getalby.com`, `wss://relay2.getalby.com`
- Suggested budget: 500,000 sats (`500000000` msats)
- Suggested renewal: monthly

The user may edit the policy before approving.

### Return

```text
com.alby.go://nwa/callback#state=4f7c8b9a21d64099a7352b97c927c341&status=approved&wallet_pubkey=abcdef...&relay=wss%3A%2F%2Frelay.getalby.com&relay=wss%3A%2F%2Frelay2.getalby.com&lud16=user%40wallet.example
```

The requesting app verifies `state`, looks up the pending local client secret, and constructs:

```text
nostr+walletconnect://abcdef...?relay=wss%3A%2F%2Frelay.getalby.com&relay=wss%3A%2F%2Frelay2.getalby.com&secret=<local_client_secret>&lud16=user%40wallet.example
```

The requesting app then uses normal NIP-47.

## Example: Wallet-Created Secret

### Requesting App Setup

The requesting app:

1. Generates a fresh `state`.
2. Selects a verified same-device callback destination.
3. Opens the wallet auth URI with `secret_mode=wallet`.
4. Waits for an app-only callback containing a complete NWC URI.

### Request

```text
nostr+walletauth+rebelwallet://connect?version=1&secret_mode=wallet&response_mode=callback&name=Alby%20Go&app_id=https%3A%2F%2Falby-go.example&return_to=https%3A%2F%2Falby-go.example%2Fnwa%2Fcallback&state=8d2a91f43bc941778a4b9985274c0a54&relay=wss%3A%2F%2Frelay.getalby.com&relay=wss%3A%2F%2Frelay2.getalby.com&request_methods=pay_invoice%20get_balance%20make_invoice&max_amount=500000000&budget_renewal=monthly
```

### Wallet Approval

The wallet displays the same policy details. After approval, the wallet creates the NWC client secret, derives the generated client public key, stores that public key with the approved policy, and returns a complete NWC URI.

### Return

```text
https://alby-go.example/nwa/callback#state=8d2a91f43bc941778a4b9985274c0a54&status=approved&nwc_uri=nostr%2Bwalletconnect%3A%2F%2Fabcdef...%3Frelay%3Dwss%253A%252F%252Frelay.getalby.com%26relay%3Dwss%253A%252F%252Frelay2.getalby.com%26secret%3D012345...
```

The requesting app verifies `state`, decodes `nwc_uri`, stores the NWC URI securely, and uses normal NIP-47.

The wallet MUST request app-only opening of this callback. If the associated requesting app is not installed, the association is invalid, or the callback cannot be delivered without a browser, the wallet fails the handoff and revokes the connection instead of opening the HTTPS URL in a browser.

## App-Only Callback Conformance

Wallet-created secret implementations MUST test both successful delivery and fail-closed behavior on supported platforms.

A conforming implementation verifies that:

1. With the requesting app installed and its domain association valid, approval opens that app directly and delivers the fragment.
2. With the requesting app uninstalled, approval does not open a browser and the wallet revokes the new connection.
3. With an invalid or stale association, approval does not open a browser and the wallet revokes the new connection.
4. With the requesting app terminated, a valid callback launches it and resumes the pending request.
5. The HTTPS server never receives the NWC URI because secret-bearing result parameters exist only in the fragment.
6. Wallet and requesting-app logs, analytics, navigation state, and crash reports do not contain the callback fragment or decoded NWC URI.

## References

- [NIP-47: Nostr Wallet Connect](https://github.com/nostr-protocol/nips/blob/master/47.md)
- [NIP-47 deep-link appendix](https://github.com/nostr-protocol/nips/blob/master/47.md#deep-links): `nostrnwc://connect`
- [NWC one-click connection / client-created-secret flow](https://docs.nwc.dev/bitcoin-apps-and-websites/connecting-to-the-wallet/1-click-wallet-connections)
- [Alby SDK NWAClient documentation](https://guides.getalby.com/developer-guide/developer-guide/nostr-wallet-connect-api/building-lightning-apps/nwc-js-sdk#nostr-wallet-auth)
- [NIP-47 client-created secret pull request](https://github.com/nostr-protocol/nips/pull/1818)
- Earlier Mutiny Wallet and ZapplePay NWA implementations
- [Apple: Open Universal Links Only](https://developer.apple.com/documentation/uikit/uiapplication/openexternalurloptionskey/universallinksonly)
- [Apple: Supporting Associated Domains](https://developer.apple.com/documentation/xcode/supporting-associated-domains)
- [Android: Verify App Links](https://developer.android.com/training/app-links/verify-applinks)
- [Android: Intent App-Only Activity Flags](https://developer.android.com/reference/android/content/Intent#FLAG_ACTIVITY_REQUIRE_NON_BROWSER)

## Open Questions

- Should `nostr+walletauth://connect` remain the preferred generic scheme, or should wallets standardize only on HTTPS/App Links?
- Should wallet-specific schemes use `nostr+walletauth+<wallet>://connect` or private wallet schemes?
- Should `request_methods` be space-separated, comma-separated, or repeated `method` parameters?
- Should a wallet return relays only in the callback, or also publish a connection-specific discovery event?
- Should an approved callback include a detached wallet signature over `state`, `pubkey`, `wallet_pubkey`, and `relay`?
- Should wallets advertise supported NWA entry points in NIP-47 wallet info events?
