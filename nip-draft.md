<!--
Picks taken from the design document's open questions (design.md §4),
documented here so the spec is implementable. Open to reconsideration:

  * Encryption envelope (§4.1):  NIP 44 direct.
        Reason: NIP 47 precedent; gift wrap CPU cost is not justified for
        traffic where the (client, ASP) relationship is already public facing.
  * Kind allocation (§4.2):      Single ephemeral request kind, single
        regular stream kind, replaceable info kind. Method discriminator on
        the `m` tag.
        Reason: keeps the kind registry footprint to four numbers no matter
        how many methods Ark grows.
  * Relay trust (§4.3):          Client chosen any relay with the ASP's
        NIP 65 list as the default. NIP 42 optional.
        Reason: matches Nostr's outbox model; doesn't tie availability to
        ASP operated relays.
  * Correlation (§4.4):          `e` tag pointing at the request event id,
        aligned with NIP 47. A `req-id` tag is OPTIONAL for cross relay
        retry (see below).
        Reason: aligns with NIP 47 / NIP 46 and uses NIP 01's first class
        `#e` filter.
  * Streaming (§4.5):            Long lived REQ on an ASP authored regular
        kind, with `since` checkpointing.
        Reason: only option that supports reconnect with backfill, which
        Ark's round event stream requires.

Kind numbers in this draft are placeholders. Real allocations come out of
the NIPs PR / kinds.md process.
-->

NIP-XX
======

Ark Service Provider transport over Nostr
-----------------------------------------

`draft` `optional`

## Abstract

This NIP defines a transport between Ark wallets and Ark Service Providers (ASPs) over Nostr. Client requests and ASP responses are exchanged as Nostr events on relays both parties share. The ASP's network identity is its Nostr pubkey. The transport is method agnostic: it standardises envelope, encryption, correlation, and streaming, while leaving method names and request/response shapes to each Ark implementation.

## Motivation

Ark ASPs today are reachable over gRPC at a TLS endpoint. Public TLS endpoints have a DNS name and an IP address, both censorable and both uniquely identifying the ASP. A Nostr based transport restores browser native operation (plain WebSockets), lets clients switch relays without ASP cooperation, and reduces the ASP's network identity to a pubkey. Routing the client's connection through Tor remains possible at a lower layer (bark already supports a client side SOCKS5 proxy that can be pointed at Tor) and is orthogonal to the choice of application transport.

Ark is a service model, not a federated network: every ASP holds its own on chain capital, and a VTXO at one ASP is not redeemable at another. Moving value between ASPs requires on chain redemption or atomic swap and is unchanged by the transport in use. This NIP targets **connection portability**, making a given ASP reachable from any wallet on any network. It does not attempt **state portability**, which is not meaningful in Ark's architecture.

This NIP describes only the **transport**: the envelope of requests, responses, streams, and notifications. The set of method names and request/response shapes is defined by each Ark implementation, the same way Lightning's BOLTs leave each node implementation (LND, Core Lightning, Eclair, LDK) free to expose its own client API. An informative appendix lists representative methods.

## Specification

### Terms

* **Client.** A Nostr enabled application acting as an Ark wallet.
* **ASP.** A Nostr enabled service that signs and accepts Ark protocol artifacts on behalf of clients (rounds, intents, forfeit transactions, etc).
* **ASP key.** A Nostr secp256k1 keypair, the public half of which is the ASP's network identity.
* **Client key.** A Nostr secp256k1 keypair held by the client. SHOULD be distinct from the user's social identity; SHOULD be unique per ASP connection.
* **Relay.** A NIP 01 compliant relay reachable over WebSocket.

### Event kinds

Four event kinds:

| Kind | Range | Use | Author |
|---|---|---|---|
| `27483` | ephemeral | Ark client request | client |
| `27484` | ephemeral | Ark response (to a request) | ASP |
| `7483`  | regular   | Ark server streamed event (rounds, mailbox, indexer, etc) | ASP |
| `13483` | replaceable | Ark service description ("info") | ASP |

Numbers above are illustrative placeholders. Allocation is requested via the NIPs repository's kind allocation process.

### Encryption

Request and response content MUST be encrypted with NIP 44 v2 between the client public key and the ASP public key. The encryption tag MUST be present and set to `nip44_v2`:

```
["encryption", "nip44_v2"]
```

Stream events (kind `7483`) that carry per client data MUST be encrypted to that client's public key with NIP 44 v2. Stream events that are intrinsically broadcast (publicly observable, such as "a round was finalized") MAY be sent in plaintext. The ASP indicates per stream whether events are encrypted by setting the `encryption` tag on each event.

The replaceable info event (kind `13483`) MUST be plaintext.

### Info event (kind `13483`)

Replaceable. Published by the ASP. There SHOULD be exactly one info event per ASP key, kept current.

```jsonc
{
  "kind": 13483,
  "pubkey": "<asp_pubkey_hex>",
  "created_at": <unix>,
  "tags": [
    ["network", "bitcoin"],                                  // bitcoin | testnet | signet | regtest
    ["protocol-version", "1"],                               // wire format version of this transport
    ["impl", "arkd 0.7.0"],                                  // freeform implementation and version
    ["encryption", "nip44_v2"],                              // space separated supported schemes
    ["methods", "GetInfo RegisterIntent ConfirmRegistration ..."], // space separated
    ["streams", "rounds transactions indexer mailbox"],      // space separated logical stream names
    ["r", "wss://relay-a.example", "read"],                  // optional NIP 65 style hints
    ["r", "wss://relay-b.example", "write"]
  ],
  "content": "",
  "sig": "..."
}
```

Clients SHOULD also consult the ASP's NIP 65 kind 10002 relay list event for the authoritative relay set.

### Request event (kind `27483`)

Published by the client. Ephemeral (relays SHOULD NOT store).

```jsonc
{
  "kind": 27483,
  "pubkey": "<client_pubkey_hex>",
  "created_at": <unix>,
  "tags": [
    ["p", "<asp_pubkey_hex>"],
    ["m", "<method>"],                  // implementation defined method name
    ["v", "1"],                         // protocol version (from info event)
    ["encryption", "nip44_v2"],
    // optional:
    ["req-id", "<opaque_client_uuid>"], // for cross relay retry deduplication
    ["expiration", "<unix>"]            // ASP MUST ignore if past
  ],
  "content": nip44_encrypt({
    "method": "<method>",
    "params": { /* method specific JSON */ }
  }),
  "sig": "..."
}
```

The ASP filters incoming requests with:

```jsonc
["REQ", "<sub>", { "kinds": [27483], "#p": ["<asp_pubkey_hex>"], "since": <now> }]
```

### Response event (kind `27484`)

Published by the ASP. Ephemeral.

```jsonc
{
  "kind": 27484,
  "pubkey": "<asp_pubkey_hex>",
  "created_at": <unix>,
  "tags": [
    ["p", "<client_pubkey_hex>"],
    ["e", "<request_event_id>"],         // correlation
    ["m", "<method>"],                   // mirrors the request method
    ["encryption", "nip44_v2"],
    // optional, mirrored when present on the request:
    ["req-id", "<opaque_client_uuid>"]
  ],
  "content": nip44_encrypt({
    "result_type": "<method>",
    "result": { /* method specific JSON, or null on error */ },
    "error": null
    // or, on error:
    // "error": { "code": "<CODE>", "message": "<human readable>" }
  }),
  "sig": "..."
}
```

The client filters incoming responses with:

```jsonc
["REQ", "<sub>", { "kinds": [27484], "#e": ["<request_event_id>"], "authors": ["<asp_pubkey_hex>"] }]
```

If the client uses a `req-id` tag, the filter MAY use `#req-id` instead of `#e` to enable cross relay retry.

### Error codes

The `error.code` field on responses MUST use the values below where applicable. Implementations MAY define additional codes; clients SHOULD treat unknown codes as `INTERNAL`.

* `INVALID_ARGUMENT`, request payload malformed or semantically invalid.
* `UNAUTHENTICATED`, request lacks valid authentication (BIP 322, macaroon, token, etc).
* `PERMISSION_DENIED`, authenticated but not authorised for this operation.
* `NOT_FOUND`, referenced entity does not exist.
* `FAILED_PRECONDITION`, request would violate protocol state (batch already closed, wrong round phase, etc).
* `ALREADY_EXISTS`, duplicate of an existing entity.
* `RESOURCE_EXHAUSTED`, rate limited.
* `UNIMPLEMENTED`, method not supported by this ASP.
* `UNAVAILABLE`, transient inability to service the request.
* `DEADLINE_EXCEEDED`, long poll timed out.
* `INTERNAL`, anything else.

### Server streamed events (kind `7483`)

Published by the ASP. Regular kind so relays retain them and clients can reconnect with backfill.

```jsonc
{
  "kind": 7483,
  "pubkey": "<asp_pubkey_hex>",
  "created_at": <unix>,
  "tags": [
    ["stream", "<stream_name>"],        // e.g. "rounds" | "transactions" | "mailbox" | "indexer"
    ["seq", "<int>"],                   // monotonic per stream sequence number, ASP issued
    ["t", "<event_type>"],              // freeform, e.g. "BatchStarted" | "RoundFinished" | "Heartbeat"
    // when the stream is per client (mailbox, indexer subscription):
    ["p", "<client_pubkey_hex>"],
    ["sub-id", "<subscription_or_mailbox_id>"],
    ["encryption", "nip44_v2"]
    // when the stream is broadcast:
    // (omit p / encryption)
  ],
  "content": (
     nip44_encrypt({ /* event payload */ })   // for per client streams
     OR
     "<plain-json>"                            // for broadcast streams
  ),
  "sig": "..."
}
```

Clients subscribe with:

```jsonc
["REQ", "<sub>", {
    "kinds": [7483],
    "authors": ["<asp_pubkey_hex>"],
    "#stream": ["<stream_name>"],
    "since": <last_seen_created_at>
    // and for per client streams, additionally:
    // "#p": ["<client_pubkey_hex>"],
    // "#sub-id": ["<id>"]
}]
```

On reconnect a client passes `since = max(created_at)` of events it has processed. The relay backfills as far as its retention policy allows. The client SHOULD deduplicate by event id and SHOULD detect sequence gaps via the `seq` tag.

Heartbeats are RECOMMENDED at an ASP chosen interval so clients can tell "stream alive, no events" apart from "connection silently dropped".

### Notifications (out of band events)

The ASP MAY publish point to point notifications (e.g. "your pending tx confirmed") as ephemeral events with kind `27484` or as stream events on kind `7483`. The choice is up to the ASP; clients listen on both.

### URI

```
ark+nostr://<asp_pubkey_hex>?relay=<wss_url>[&relay=<wss_url>...]
```

The wallet decodes the URI, fetches the kind `13483` info event from the listed relays, optionally fetches the ASP's kind 10002 NIP 65 list, and stores the (pubkey, relay set, info) tuple.

## Rationale

* **Why NIP 44 (not gift wrap).** Per message size and CPU costs are lower, which matters for browser wallets and for round coordination where dozens of messages may be exchanged in a few seconds. The relay still learns that pubkey X talks to ASP Y, which is comparable to the metadata leakage of an HTTPS connection from a browser. Implementations that want to hide the relationship can layer NIP 59 gift wrap on top in a future revision.
* **Why a single request kind with a method discriminator (not one kind per method).** Ark's two existing implementations have roughly 30 client facing methods each, and a unified surface is not yet defined. Kind allocation is a slow process; one kind plus an `m` tag absorbs method churn without further NIPs.
* **Why `e` tag for correlation.** Aligns with NIP 47 and NIP 46 and uses the relay's built in `#e` filter. The optional `req-id` tag is for clients that need application level deduplication across relays.
* **Why a regular kind for streams.** Ark's round event stream and mailbox stream both need backfill on reconnect. Ephemeral kinds cannot provide that. A regular kind with `since` based replay is the simplest mechanism in NIP 01 that does.

## Backwards Compatibility

This NIP defines a new transport that runs in parallel to the existing gRPC over TLS endpoint. ASPs that adopt it SHOULD continue serving the gRPC transport for at least one release cycle. Wallets discover Nostr support via the kind `13483` info event; absence of the event means the ASP does not implement this NIP and the wallet falls back to gRPC.

No changes are required to Ark's existing protobuf interfaces. The Nostr layer is an envelope: each implementation maps its existing methods onto the request, response, and stream events defined here.

## Reference Implementation

Reference implementation is planned in the Ark Go reference (`arkd`) and the Rust implementation (`bark`). At the time of writing this NIP, no reference implementation exists. Implementers should coordinate via the Ark protocol working group.

## Appendix A. Representative method list

The `m` tag values used by Ark implementations are implementation defined and not part of this NIP. The following is informative.

| ASP method | Notes |
|---|---|
| `GetInfo` (arkd) / `GetArkInfo` (bark) | Cheap; clients call on connect. |
| `RegisterIntent`, `ConfirmRegistration` (arkd) | BIP 322 in payload. |
| `SubmitTreeNonces`, `SubmitTreeSignatures` (arkd) | Sent during round signing phases. |
| `SubmitTx`, `FinalizeTx` (arkd) | Offchain VTXO spend. |
| `RequestBoardCosign`, `RegisterBoardVtxo` (bark) | Onboarding. |
| `RequestArkoorCosign` (bark) | Offchain payment. |
| `SubmitRoundParticipation`, `RoundParticipationStatus` (bark) | Non interactive round path. |
| `Mailbox.Subscribe`, `Mailbox.Read`, `Mailbox.PostArkoor` (bark) | Mailbox. |

## Appendix B. Stream catalogue

| Stream name | Authored by | Encryption | Use |
|---|---|---|---|
| `rounds` | ASP | none (broadcast) | Round and batch phase events. |
| `transactions` | ASP | none (broadcast) | Commitment, ark, sweep tx notifications. |
| `indexer/<sub-id>` | ASP | NIP 44 to subscribing client | Per script subscription events. |
| `mailbox/<sub-id>` | ASP | NIP 44 to mailbox owner | Mailbox messages. |
