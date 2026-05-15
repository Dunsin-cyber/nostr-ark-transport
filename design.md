# Nostr as client to ASP transport for Ark

## Status

Draft, for discussion. Author: Dunsin Abisuwa.

## Summary

This is a proposal for using Nostr as the transport between Ark wallets and Ark Service Providers (ASPs), sitting alongside or replacing today's gRPC over TLS endpoints. The goal is **connection portability**: make a given ASP reachable from a browser, a PWA, or a censored network, by reducing the ASP's network identity to a Nostr pubkey and exchanging requests and responses as Nostr events on relays both parties share. Neither arkd nor bark runs as a Tor onion service today; bark has a client side SOCKS5 proxy feature (`bark/server-rpc/src/client.rs:49-104`) that a wallet could point at Tor, but the ASP itself is still a TLS endpoint, so this is a client side layer, not an ASP property.

The proposal is **not** about state portability. Ark is a service model. Each ASP holds its own on chain capital, and a VTXO is a claim against that specific ASP's UTXO. Switching transport does not let a wallet move a balance from one ASP to another, and nothing in this document tries to.

Scope is the client to ASP wire surface only. The gRPC channels between an ASP and its own internal components (wallet, signer, indexer) stay as they are. Open design questions in §4 are presented with options rather than picks. The accompanying `nip-draft.md` does pick, so the spec is implementable.

## 1. Background

### 1.1 The current client to ASP transport

Ark has two implementations today, with two **incompatible** wire surfaces. That's worth flagging clearly, but it doesn't block a Nostr transport NIP. Ark is a service model, users pick one ASP and stay with it, and wallet developers that want to support both kinds of ASP ship two adapters, the same way a Lightning wallet that supports both LND and Core Lightning ships two adapters. The Nostr NIP can scope itself to the envelope layer (connection, encryption, correlation, streaming, discovery) and leave method names per implementation.

#### arkd (Go reference)

Public proto package `ark.v1`. Two client facing services:

* `ArkService`, round and offchain coordination. `arkd/api-spec/protobuf/ark/v1/service.proto:8-174`.
* `IndexerService`, historical and live state queries (commitment txs, VTXO trees, scripts). `arkd/api-spec/protobuf/ark/v1/indexer.proto:6-131`.

Operator surfaces (`AdminService`, `WalletService`, `SignerService`, `arkwallet.v1`) are out of scope.

Server construction in `arkd/internal/interface/grpc/service.go:250-516`. HTTP/2 mux that routes REST (via `meshapi/grpc-api-gateway`) and gRPC on the same listener. h2c when TLS is disabled (line 438), HTTP/2 with TLS otherwise (line 449). Optional admin port (lines 455-514). Auth is macaroon based via interceptors in `arkd/internal/interface/grpc/interceptors/`. Default ports are `7070` (public) and `7071` (admin).

Three server streaming RPCs:

* `ArkService.GetEventStream` (`service.proto:99-107`, handler `arkservice.go:231-291`). Round and batch event stream. The first event is always `StreamStartedEvent` carrying a `stream_id`. Topics are adjustable in flight via `UpdateStreamTopics` (`service.proto:112-117`, handler `arkservice.go:293-349`). Heartbeats every `HeartbeatInterval` seconds. Broker in `arkd/internal/interface/grpc/handlers/broker.go`.
* `ArkService.GetTransactionsStream` (`service.proto:155-163`, handler `arkservice.go:430-481`). All commitment, ark, and sweep tx notifications. No history; events only flow from the moment the stream opens.
* `IndexerService.GetSubscription` (`indexer.proto:122-130`, handler `indexer.go:424-485`). Server streaming channel for per script subscriptions, created or extended via `SubscribeForScripts` / `UnsubscribeForScripts`. Auto expires after 1 minute idle.

RPC design is **intent based**. Clients register `Intent` objects (BIP 322 proven), confirm participation, and submit nonces, signatures, and forfeit txs as the round progresses, driven by phase transitions on `GetEventStream`. Offchain spend is a two leg `SubmitTx` then `FinalizeTx` flow (`service.proto:123-139`).

#### bark (Rust)

The repo contains **both** a client (`bark/src/`) and a complete alternative ASP server (`bark/server/src/rpcserver/`). Protos in `bark/server-rpc/protos/` define this implementation's wire surface.

Three client facing services:

* `bark_server.ArkService` (`bark/server-rpc/protos/bark_server.proto:9-79`). Boarding, arkoor offchain payments, lightning send and receive, offboarding, rounds.
* `mailbox_server.MailboxService` (`bark/server-rpc/protos/mailbox_server.proto:9-16`). VTXO delivery and incoming event notifications.
* `intman.IntegrationService` (`bark/server-rpc/protos/intman.proto`). Out of scope, operator facing token management.

Two server streaming RPCs:

* `ArkService.SubscribeRounds` (`bark_server.proto:67`, consumed in `bark/src/round/mod.rs:1579-1623`). Round event stream of `RoundAttempt`, `VtxoProposal`, `RoundFinished`, `RoundFailed`.
* `MailboxService.SubscribeMailbox` (`mailbox_server.proto:12`, consumed in `bark/src/mailbox.rs:106-160`). Checkpointed stream of arkoor, lightning, and recovery messages, resumable via the `checkpoint` field of `MailboxRequest`.

Transport is tonic gRPC, optional TLS, optional SOCKS5, with a 10 minute request timeout and 20 second HTTP/2 keep alive (`bark/server-rpc/src/client.rs:111-149`, `:121-124`). A single `Handshake` RPC at connection time negotiates a protocol version (currently min=1 max=1), then stamps every later call with a `pver` metadata header (`:413-451`). An optional access token rides on the `ark-access-token` header.

RPC design is **round based** with explicit per feature methods: `RequestBoardCosign`, `RegisterBoardVtxo`, `RequestArkoorCosign`, `RequestLightningPay…`, `StartLightningReceive`, `Prepare…` / `FinishOffboard`, `SubmitRoundParticipation` / `RoundParticipationStatus`, `RequestForfeitNonces`, `ForfeitVtxos`. Lightning and offboard get their own multi step flows instead of being expressed as intents.

#### Cross implementation RPC mapping

The two implementations share no wire compatible methods. They are different services that coordinate the same Ark protocol on chain. A wallet built against arkd's protos cannot talk to bark's ASP and vice versa. The full table is in §5; key divergences worth calling out here are:

1. **Method namespaces are disjoint.** Even where both cover the same Ark concept (round participation, forfeit), the methods, request shapes, and ordering of multi step flows differ. No unified Nostr method name mapping is derivable from the two protos alone.
2. **bark has an explicit handshake. arkd does not.** bark negotiates a `protocol_version` once per connection and stamps every subsequent call with a `pver` header.
3. **bark has a separate Mailbox service** (`mailbox_server.proto`). arkd has no counterpart; arkd assumes deliveries happen via the per script subscription stream and on chain observation.
4. **Server streaming endpoints differ in event vocabulary.** arkd's `GetEventStream` and `GetTransactionsStream` versus bark's `SubscribeRounds` and `SubscribeMailbox`. The set of events delivered, topic filtering, and resumption semantics all differ.
5. **Auth tokens** are inline `IndexerIntent` proofs and opaque `auth_token` strings on arkd's indexer surface (`indexer.proto:213-219, 232-238`), versus an out of band `ark-access-token` header plus per method anti DoS `oneof(input_vtxo \| token)` payloads on bark.

So any single NIP for client to ASP transport in Ark can only define **how messages flow over Nostr**, not **what they say**. The rest of this document treats the Nostr layer as a method agnostic envelope and treats the method mapping as a per implementation appendix in §5.

This is the Lightning precedent. The BOLTs specify the wire protocol between Lightning nodes, the exact bytes on the peer to peer link. They do not specify the API a wallet uses to drive its own node. LND, Core Lightning, Eclair, and LDK each expose their own client API; a wallet that integrates with more than one ships more than one adapter. Ark sits one step inward: the wire layer (between ASP and chain) is governed by the Ark protocol on Bitcoin, and the Nostr transport sits in the position of "wallet to its own service", which the Lightning community concluded did not need to be standardised universally. This NIP standardises the *connection* so a wallet can reach the ASP it has chosen from any network. The *methods* stay the implementation's business.

### 1.2 Relevant Nostr primitives

* **NIP 01, events, filters, subscriptions.** Each event is a tagged Schnorr signed envelope: `{id, pubkey, created_at, kind, tags, content, sig}`. Clients open subscriptions via `["REQ", subscription_id, filters…]`, receive matches as `["EVENT", subscription_id, event]`, close via `["CLOSE", subscription_id]`. Filters target `kinds`, `authors`, `#e` / `#p` tags, `since`, `until`, `limit`. Kind ranges per the NIPs README: 1000 to 9999 regular (stored), 10000 to 19999 replaceable (latest per pubkey and kind stored), 20000 to 29999 ephemeral (not stored), 30000 to 39999 addressable (latest per pubkey, kind, and `d` tag stored).
* **NIP 44, versioned encryption.** ECDH then HKDF SHA256, then ChaCha20 with HMAC SHA256, with deterministic padding to powers of 2 (min 32 bytes). Plaintext 1 to 65535 bytes; ciphertext is base64 of `version || nonce || ct || mac`, roughly 132 to 87472 bytes in base64. Audited by Cure53 in December 2023. ChaCha20 is faster than AES GCM in pure JS, which matters for browser and PWA wallets.
* **NIP 59, gift wrap.** Three nested events: an unsigned `rumor`, a kind 13 `seal` (NIP 44 encrypted, signed by the author), then a kind 1059 `gift wrap` (NIP 44 encrypted with a fresh random key, p tagged to the recipient). Hides sender pubkey, recipient pubkey, kind, and content from anyone other than the recipient. Roughly 2x the encryption cost and 4/3 the base64 expansion per layer.
* **NIP 17, private direct messages.** Applies NIP 59 to DMs (kinds 14 and 15 inside the rumor). Randomises `created_at` on the outer wrap by up to 2 days back to obscure timing. Recipients advertise preferred DM relays via kind 10050.
* **NIP 42, relay AUTH.** Relay sends `["AUTH", challenge]`; the client replies with a signed kind 22242 ephemeral event referencing the relay URL and challenge. Used by relays to restrict who can subscribe to or publish certain kinds.
* **NIP 65, relay list metadata.** Kind 10002 (replaceable). Tags `["r", url, "read"|"write"]` advertise where the author publishes and reads. The outbox model expects clients to publish to the author's write relays and subscribe at the queried author's write relays.
* **NIP 04, legacy DM encryption.** Deprecated. AES 256 CBC without authentication and with metadata exposure. Mentioned only because NIP 47 still references it for backward compatibility; this proposal will not.

The closest existing prior art for RPC over Nostr is **NIP 47 (Nostr Wallet Connect)**. Kind 13194 replaceable info event published by the wallet service, ephemeral request events kind 23194 p tagged to the wallet service, ephemeral response events kind 23195 p tagged back to the client with an `e` tag referencing the request. Content is JSON RPC ish (`{"method":…, "params":…}` and `{"result_type":…, "result":…, "error":…}`), NIP 44 encrypted; the `encryption` tag advertises which scheme. A `nostr+walletconnect://` URI carries the service pubkey, relay list, and a per connection secret. **NIP 46 (remote signing)** reuses the pattern with kind 24133. **NIP 90 (Data Vending Machines)** uses kinds 5000 to 6999 with a request and result pattern, with optional encryption of params and outputs. These three together set the conventions this proposal builds on.

## 2. Goals and non goals

### Goals

* Let a wallet hold a Nostr pubkey for an ASP and use any mutually supported relay to exchange RPC traffic.
* Be usable from browser and PWA wallets without bundling a custom networking stack or relying on a transport (such as Tor) that browsers cannot run.
* Preserve, on the Nostr layer, the semantics Ark's current gRPC surface relies on: request and response correlation, ordered server streamed events, idempotency where the underlying RPC is idempotent.
* Let ASPs run the Nostr transport **alongside** their existing gRPC or TLS surface for at least one release cycle.
* Be implementable by both arkd and bark without forcing reconciliation of their wire surfaces. The Nostr layer is method agnostic; each implementation maps its own methods.

### Non goals

* **Cross operator VTXO or state portability.** Ark is a service model, not a federated network. Each ASP holds its own on chain capital; a VTXO is a claim against that specific ASP's UTXO and cannot be redeemed at another ASP. Moving value between operators requires on chain redemption or an atomic swap and is entirely unaffected by this transport proposal. Any reading of "portability" in this document means *connection* portability (a wallet can reach the ASP it has chosen from any network), never balance, state, or VTXO portability across ASPs.
* Reconciling the arkd and bark wire surfaces. Separate problem.
* Replacing the gRPC surface between an ASP's own components (wallet, signer, indexer). Those stay as they are.
* Operator and admin tooling. `AdminService`, `WalletAdminService`, `RoundAdminService`, and so on are not exposed over Nostr here.
* Stronger anonymity than Nostr itself provides. The proposal does not claim to hide from relays that a given pubkey is an Ark wallet talking to an Ark ASP. See §7.
* Anonymising the client's network path. Users who want that can keep routing their connection through Tor (via bark's SOCKS5 feature or an equivalent shim), independent of whether the underlying transport is gRPC or Nostr.
* Defining a fresh encrypted key URI format when an existing one (such as NIP 47's `nostr+walletconnect://`) can be adapted.

## 3. Design overview

The transport reduces to four primitive operations.

**Discovery.** A wallet learns, out of band, an ASP's npub (Nostr pubkey) and an initial relay set. It can refresh the relay set from the ASP's kind 10002 NIP 65 list. The ASP publishes a versioned info event (replaceable, kind discussed in §4.2) that carries enough metadata for the wallet to confirm it can talk to this ASP at all: network, protocol version, supported methods.

**Unary RPC.** The client publishes an ephemeral event (kind picked in §4.2) to the chosen relay(s), with tags:

* `["p", <asp_pubkey>]`, addressee. Enables the ASP side filter `{"kinds":[X], "#p":[asp_pubkey], "since": now}`.
* `["m", <method>]`, method name (`GetInfo`, `RequestArkoorCosign`, etc).
* `["req-id", <client_uuid>]`, correlation identifier from a CSPRNG, opaque.
* `["v", <protocol_version>]`, wire format version of this transport.
* Optionally `["expiration", <unix_ts>]` and `["encryption", "nip44_v2"]`.

Content is `nip44_encrypt(asp_pubkey, client_privkey, JSON({"method":…, "params":…}))`. The ASP side subscribes on its relays, decrypts, runs the call, and publishes a response event addressed back to the client pubkey, with `["p", <client_pubkey>]`, `["m", <method>]`, `["req-id", <same uuid>]`, `["e", <request_event_id>]` (for tooling), and a NIP 44 encrypted `{"result_type":…, "result":…, "error":…}` in `content`.

Idempotency on the request side is the client's responsibility. Relays can deliver an event more than once (multi relay fan out, reconnect, mirror). The `req-id` lets a wallet that submitted the same intent twice deduplicate responses.

**Server streaming RPC.** The ASP publishes events under a single regular kind authored by `asp_pubkey` so relays retain them. Each event carries a `["stream", <name>]` tag (`rounds`, `transactions`, `script-subscription/<id>`, etc) and an ASP issued monotonic `["seq", <int>]` so clients can detect a gap. The client opens a long lived REQ subscription:

```
["REQ", "<sub>", {"kinds":[STREAM_KIND], "authors":[asp_pubkey], "#stream":[name], "since": last_seen_or_now}]
```

On reconnect the client passes `since = max(created_at seen)`. The relay backfills as far as its retention policy allows; the client deduplicates by event id. Streams that need per client gating (`GetSubscription` on arkd, `SubscribeMailbox` on bark, both addressed to a specific subscription or mailbox id) put the id in a tag (`["sub-id", <id>]`) on each event and filter on it. Per client streams encrypt event content to the client pubkey with NIP 44; broadcast streams (`GetTransactionsStream`, public batch events) can stay cleartext.

**Long poll RPC.** A few of bark's methods (`CheckLightningPayment`, `CheckLightningReceive`) accept a `wait` flag and behave as long polls. Over Nostr these stay single request, single response; the ASP just delays publishing the response until the underlying state changes or a timeout fires. No new event machinery. The client should set its own timeout and re issue if it elapses.

**Correlation, ordering, idempotency, replay.**

* *Correlation.* A `req-id` tag on requests, mirrored on responses, plus filter side `#req-id`. NIP 01 supports filters on arbitrary single letter and `#`-prefixed tag keys. The `e` tag pointing at the request event id is useful for tooling but is not strictly required; §4.4 covers the tradeoff.
* *Ordering.* Nostr makes no global ordering promise across relays. Within a single relay, responses arrive in send order; across relays they can interleave. Clients order server stream events by `created_at` with `event.id` as a deterministic tiebreaker. Streams that require strict phase ordering (e.g. `BatchStarted` then `TreeSigningStarted` then `TreeNoncesAggregated` then `BatchFinalization` then `BatchFinalized` on arkd) use the `seq` tag to detect gaps and pull the missing event by id.
* *Idempotency.* Underlying RPC methods inherit idempotency from the existing protos. `RegisterIntent` returns the same `intent_id` for the same intent; `ConfirmRegistration` is naturally idempotent; `SubmitTreeNonces` and `SubmitTreeSignatures` keyed by `(batch_id, pubkey)` overwrite. The Nostr layer adds a duplicate delivery hazard: a client publishing to multiple relays will see the ASP receive and act on the request multiple times if no deduplication happens. Recommendation: ASPs dedupe by `req-id` for a configurable window (say 5 minutes) and replay the cached response. NIP 47 wallet services do this implicitly.
* *Replay.* Relays cache and rebroadcast. A long lived response event sitting on a public relay could be re served to anyone subscribing with `since=0` and `#p=client_pubkey`. NIP 44 encryption protects content but not metadata; if the use case needs no third party correlation, the gift wrap option in §4.1 hides it.

**Identity.** The ASP's network identity collapses to its npub. Compromise of the ASP's private key is equivalent to compromise of its TLS private key today. Rotation is possible (§7.2). Wallets pin the ASP's npub the same way they would pin a TLS certificate today.

## 4. Open design questions

Five questions, each with named options. The `nip-draft.md` picks one of each so the spec is implementable; the picks are documented at the top of that file.

### 4.1 Encryption envelope

**A. NIP 44 direct.** Encrypt request content with NIP 44 between the client pubkey and the ASP pubkey; publish with the `p` tag in cleartext. The relay sees `kinds`, `authors`, and the `p` tagged counterparty, not the content. NIP 47 does this. Size overhead is roughly 4/3 (base64); CPU is one ECDH plus one ChaCha20 pass, browser implementable in a few milliseconds at typical sizes. Replay protection is the client's job via `req-id` plus an optional `expiration` tag.

*Tradeoff.* Reveals to the relay that pubkey X is talking to ASP Y, with timing and frequency. Fine if the ASP's client set is already public knowledge, problematic if not. Best when metadata leakage to the relay is acceptable.

**B. NIP 17 / 59 gift wrap.** Wrap each request and response in a NIP 59 gift wrap: unsigned rumor, sealed (kind 13), wrapped (kind 1059, p tagged with a random outer key). The outer event reveals nothing about the participants or the underlying kind. Adds a second encryption pass (roughly 2x CPU) and another 4/3 size pass.

*Tradeoff.* Hides the client to ASP link from relays. Costs more CPU and bytes, and complicates ASP side filter design (the ASP must subscribe to all kind 1059 events `#p` tagged to it, with no kind or method filter to scope it). Best when client and ASP traffic has to be unlinkable to the relay.

**C. Plaintext on a shared secret channel.** No per message encryption; the ASP and client agree on a per session pubkey via Diffie Hellman during handshake and publish to a relay that authenticates both parties via NIP 42. The relay enforces the channel.

*Tradeoff.* Lowest per message overhead, but the ASP has to operate or contract a NIP 42 relay it controls and trust it not to log content. That trust is partly what this proposal is trying to move away from. Best when an ASP controlled relay is mandatory anyway and metadata exposure to that one relay is acceptable.

### 4.2 Kind allocation

NIP 01 kind ranges: 1000 to 9999 regular, 10000 to 19999 replaceable, 20000 to 29999 ephemeral, 30000 to 39999 addressable.

**A. Single ephemeral kind with method discriminator.** One ephemeral kind (illustrative: `27483`) for requests, one regular kind (illustrative: `13355`) for ASP authored events, method name in an `m` tag. Filter efficiency is fine; `{"kinds":[27483], "#p":[asp], "#m":["RegisterIntent"]}` filters tag side in any compliant relay.

*Tradeoff.* Lowest registry footprint. Adds one tag lookup per message. Best when methods evolve quickly (which Ark's protos do) and we'd rather not request new kinds whenever a method is added.

**B. One kind per RPC method.** Allocate a range, say 25500 to 25599, and let each method claim a kind. Filters target `{"kinds":[25501]}` directly. More idiomatic Nostr (compare NIP 47's kind 23194 / 23195).

*Tradeoff.* Bloats the kind registry as methods come and go. arkd has 27 plus 12, bark has 31 plus 4, and the two diverge, so full coverage pushes 75 kinds. Best when methods are stable and a global "what does kind N mean" registry is preferable to per event discriminators.

**C. Ephemeral for requests, regular for ASP authored streams.** Combines A and B: requests are ephemeral (20000 to 29999, not stored by relays), stream events are regular (1000 to 9999, stored). NIP 47 effectively does this (kind 23194 ephemeral request, kind 13194 replaceable info, plus notification kinds).

*Tradeoff.* Requests don't pollute relay storage; stream events do persist long enough for reconnection backfill. Best when most relays will store ASP stream events for at least minutes, which typical relays do.

A and C are compatible. The kind allocation request to the NIPs repo could read:

* One ephemeral request kind, illustrative `27483`.
* One regular stream kind, illustrative `7483`.
* One replaceable info kind, illustrative `13483`.
* Optionally a regular notification kind for asynchronous server pushed events, illustrative `7484`.

Numbers are placeholders; real allocations come out of the NIPs PR process.

### 4.3 Relay trust model

**A. ASP operated relay only.** The ASP runs its own Nostr relay and clients must use it. Bundled into the ASP deployment.

*Tradeoff.* Maximum control over availability, rate limiting (per pubkey caps), and retention. Mirrors today's single endpoint failure mode: take down the relay and the ASP is offline. Best when the operator wants tight control and is willing to lose relay agility.

**B. Curated set advertised via NIP 65.** The ASP publishes a kind 10002 event listing its read and write relays. The set can be operated by the ASP, by partners, or by the public commons. Clients pick from this list and can fail over.

*Tradeoff.* Mirrors Nostr's outbox model. The ASP doesn't have to run all relays; clients route around individual failures. Some loss of rate limit control. Best when censorship resistance matters and the operator is willing to publish to public relays.

**C. Client chosen any relay with NIP 42 auth.** The wallet picks the relay, possibly subject to constraints the ASP advertises (minimum size, minimum retention). Relays may demand NIP 42 auth of either party.

*Tradeoff.* Maximum client agility; a censored client switches relays without touching ASP infrastructure. Hardest for the ASP to police rate limits (it can't blocklist a relay it doesn't operate). Best when client side censorship resistance dominates and the operator is OK delegating DoS pushback to the relay operators.

### 4.4 Request and response correlation

**A. `e` tag to request event id.** Response carries `["e", <request_event_id>]`. Client filters `{"#e":[id], "authors":[asp_pubkey]}`. NIP 47 does this.

*Tradeoff.* Efficient filter on any compliant relay (`#e` is tag indexed in NIP 01). Couples correlation to relay side event id behaviour (event id is deterministic from content, author, timestamp, and tags, so resubmitting an identical request yields the same id, slightly surprising but harmless). Best when alignment with NIP 47 and NIP 46 matters more than independent client side identifier control.

**B. Dedicated `req-id` tag.** Client generates an opaque UUID and attaches `["req-id", <uuid>]`. Response mirrors it. Client filters `{"#req-id":[uuid], "authors":[asp_pubkey]}`.

*Tradeoff.* Decouples correlation from event id semantics. Lets a wallet resubmit the same request with the same `req-id` to a different relay and still match the response (whereas with `e` tag, the resubmission has a different event id under a different `created_at`). Slightly larger event. Best when wallets need cross relay retry that's clean at the application layer.

**C. Structured JSON envelope (id inside content).** No correlation tag; the encrypted content carries `{"id": <client_uuid>, "method":…, "params":…}`. Response carries `{"id":…, "result":…}`. Client decrypts every response from the ASP and matches on `id`.

*Tradeoff.* Cleanest if Nostr is just transport and the wire format mirrors a hypothetical HTTP based JSON RPC. Loses relay side filter selectivity: the client has to subscribe to all responses from the ASP and decrypt all of them to find its own (or narrow with `#p=client_pubkey`). Best when filter side selectivity isn't important, which is unusual.

### 4.5 Streaming and long lived subscriptions

**A. Repeated polling of a replaceable event.** ASP publishes a replaceable event (kind 10000 to 19999) that always reflects current state of stream X. Clients re fetch periodically.

*Tradeoff.* Trivial to implement. Loses per event identity; the client only sees the latest state, not the transitions. Unusable for arkd round events where every phase transition matters. Best when only the latest snapshot matters.

**B. Ephemeral event stream.** ASP publishes ephemeral events (kind 20000 to 29999, not stored). Clients hold an open REQ.

*Tradeoff.* No relay storage cost; backfill is impossible. Acceptable for `GetTransactionsStream`, which already disclaims history per the proto comment (`service.proto:152-154`). Painful for `GetEventStream`, where missing a `BatchFinalized` event can leave a wallet stuck. Best when the stream has no history requirement.

**C. Long lived REQ on an ASP authored regular kind.** ASP publishes events with a regular kind so relays retain them. Clients subscribe with `{"kinds":[K], "authors":[asp], "since": last_seen}`. On reconnect, `since=max(created_at_seen)` and the relay backfills.

*Tradeoff.* Costs relay storage per stream event. The relay's retention becomes part of the trust model: a relay that drops events after 6 hours bounds how long a client can be offline before it has to resync by other means (gRPC, chain observation). Best when the stream needs reconnect with backfill.

For arkd's `GetEventStream` and bark's `SubscribeRounds`, option C fits. For arkd's `GetTransactionsStream` (no history per proto), option B is enough. For arkd's `GetSubscription` and bark's `SubscribeMailbox` (per client data), option C with per client encryption is the realistic choice.

## 5. Method by method mapping

The cross implementation Nostr mapping. Encryption column reflects the recommended posture; gift wrap is an alternative for everything (§4.1).

### 5.1 arkd

Full proto surface, grouped by service.

| impl | method | direction | request (summary) | response (summary) | streaming | server path:lines |
|---|---|---|---|---|---|---|
| arkd | `ArkService.GetInfo` | C to S | `{}` | `version, signer_pubkey, forfeit_pubkey, forfeit_address, checkpoint_tapscript, network, session_duration, *_exit_delay, utxo_min/max, vtxo_min/max, dust, fees, scheduled_session, deprecated_signers, service_status, digest, max_tx_weight, max_op_return_outputs` | unary | `service.proto:10-14` / `arkservice.go:51-89` |
| arkd | `ArkService.RegisterIntent` | C to S | `Intent {proof, message}` (BIP 322) | `intent_id` | unary | `service.proto:20-25` / `arkservice.go:91-105` |
| arkd | `ArkService.EstimateIntentFee` | C to S | `Intent` | `fee` | unary | `service.proto:30-35` / `arkservice.go:107-121` |
| arkd | `ArkService.DeleteIntent` | C to S | `Intent` | `{}` | unary | `service.proto:41-46` / `arkservice.go:123-136` |
| arkd | `ArkService.ConfirmRegistration` | C to S | `intent_id` | `{}` | unary | `service.proto:50-55` / `arkservice.go:138-151` |
| arkd | `ArkService.SubmitTreeNonces` | C to S | `batch_id, pubkey, tree_nonces map` | `{}` | unary | `service.proto:62-67` / `arkservice.go:153-178` |
| arkd | `ArkService.SubmitTreeSignatures` | C to S | `batch_id, pubkey, tree_signatures map` | `{}` | unary | `service.proto:74-79` / `arkservice.go:180-203` |
| arkd | `ArkService.SubmitSignedForfeitTxs` | C to S | `signed_forfeit_txs[], signed_commitment_tx` | `{}` | unary | `service.proto:84-89` / `arkservice.go:205-229` |
| arkd | `ArkService.GetEventStream` | C to S | `topics[]` | stream of `BatchStartedEvent`, `BatchFinalizationEvent`, `BatchFinalizedEvent`, `BatchFailedEvent`, `TreeSigningStartedEvent`, `TreeNoncesAggregatedEvent`, `TreeTxEvent`, `TreeSignatureEvent`, `TreeNoncesEvent`, `Heartbeat`, `StreamStartedEvent` | **server stream** | `service.proto:99-107` / `arkservice.go:231-291` |
| arkd | `ArkService.UpdateStreamTopics` | C to S | `stream_id, modify\|overwrite` | `topics_added/removed/all` | unary | `service.proto:112-117` / `arkservice.go:293-349` |
| arkd | `ArkService.SubmitTx` | C to S | `signed_ark_tx, checkpoint_txs[]` | `ark_txid, final_ark_tx, signed_checkpoint_txs[]` | unary | `service.proto:123-128` / `arkservice.go:351-374` |
| arkd | `ArkService.FinalizeTx` | C to S | `ark_txid, final_checkpoint_txs[]` | `{}` | unary | `service.proto:134-139` / `arkservice.go:376-394` |
| arkd | `ArkService.GetPendingTx` | C to S | `Intent` | `PendingTx[]` | unary | `service.proto:143-148` / `arkservice.go:396-428` |
| arkd | `ArkService.GetTransactionsStream` | C to S | `{}` | stream of `commitment_tx`, `ark_tx`, `sweep_tx`, `heartbeat` notifications | **server stream** | `service.proto:155-163` / `arkservice.go:430-481` |
| arkd | `ArkService.GetIntent` | C to S | `txid \| Intent` | `Intent, Intent[]` | unary | `service.proto:165-173` / `arkservice.go:692-732` |
| arkd | `IndexerService.GetCommitmentTx` | C to S | `txid` | `started_at, ended_at, batches, totals` | unary | `indexer.proto:9-13` / `indexer.go:81-113` |
| arkd | `IndexerService.GetForfeitTxs` | C to S | `txid, page` | `txids[], page` | unary | `indexer.proto:18-22` / `indexer.go:177-198` |
| arkd | `IndexerService.GetConnectors` | C to S | `txid, page` | `IndexerNode[], page` | unary | `indexer.proto:27-31` / `indexer.go:200-229` |
| arkd | `IndexerService.GetVtxoTree` | C to S | `batch_outpoint, page` | `IndexerNode[], page` | unary | `indexer.proto:36-40` / `indexer.go:115-144` |
| arkd | `IndexerService.GetVtxoTreeLeaves` | C to S | `batch_outpoint, page` | `IndexerOutpoint[], page` | unary | `indexer.proto:45-49` / `indexer.go:146-175` |
| arkd | `IndexerService.GetVtxos` | C to S | `scripts[] \| outpoints[], filters, page, time range` | `IndexerVtxo[], page` | unary | `indexer.proto:54-58` / `indexer.go:231-316` |
| arkd | `IndexerService.GetVtxoChain` | C to S | `outpoint, page, intent \| token` | `chain[], page, auth_token` | unary | `indexer.proto:63-67` / `indexer.go:318-372` |
| arkd | `IndexerService.GetVirtualTxs` | C to S | `txids[], page, intent \| token` | `txs[], page, auth_token` | unary | `indexer.proto:71-75` / `indexer.go:374-404` |
| arkd | `IndexerService.GetAsset` | C to S | `asset_id` | `supply, metadata, control_asset` | unary | `indexer.proto:78-82` / `indexer.go:51-79` |
| arkd | `IndexerService.GetBatchSweepTransactions` | C to S | `batch_outpoint` | `swept_by[]` | unary | `indexer.proto:94-98` / `indexer.go:406-422` |
| arkd | `IndexerService.SubscribeForScripts` | C to S | `scripts[], subscription_id?` | `subscription_id` | unary | `indexer.proto:102-107` / `indexer.go` |
| arkd | `IndexerService.UnsubscribeForScripts` | C to S | `subscription_id, scripts[]` | `{}` | unary | `indexer.proto:110-115` / `indexer.go:487-509` |
| arkd | `IndexerService.GetSubscription` | C to S | `subscription_id` | stream of `IndexerSubscriptionEvent`, `IndexerHeartbeat` | **server stream** | `indexer.proto:122-130` / `indexer.go:424-485` |

Nostr mapping for arkd:

| arkd method | Nostr kind | request tags | response tags | encryption | streaming | error semantics |
|---|---|---|---|---|---|---|
| `GetInfo` | `ARK_REQUEST` | `p`, `m=GetInfo`, `req-id`, `v` | `p`, `m=GetInfo`, `req-id`, `e` | NIP 44 (could be cleartext, see §4.1) | no | absent params, `INVALID_ARGUMENT` |
| `RegisterIntent` | `ARK_REQUEST` | `m=RegisterIntent`, `req-id`, `p`, `v` | as above | NIP 44 (mandatory; intent payload carries BIP 322 proof) | no | `INTENT_INVALID`, `BATCH_CLOSED` |
| `EstimateIntentFee` | `ARK_REQUEST` | `m=EstimateIntentFee` | … | NIP 44 | no | as `RegisterIntent` |
| `DeleteIntent` | `ARK_REQUEST` | `m=DeleteIntent` | … | NIP 44 | no | `INTENT_NOT_FOUND`, `INTENT_LOCKED` |
| `ConfirmRegistration` | `ARK_REQUEST` | `m=ConfirmRegistration` | … | NIP 44 | no | `INTENT_NOT_SELECTED`, `BATCH_CLOSED` |
| `SubmitTreeNonces` | `ARK_REQUEST` | `m=SubmitTreeNonces` | … | NIP 44 | no | `INVALID_PUBKEY`, `INVALID_NONCES`, `BATCH_PHASE_MISMATCH` |
| `SubmitTreeSignatures` | `ARK_REQUEST` | `m=SubmitTreeSignatures` | … | NIP 44 | no | as above |
| `SubmitSignedForfeitTxs` | `ARK_REQUEST` | `m=SubmitSignedForfeitTxs` | … | NIP 44 | no | `INVALID_FORFEIT`, `COMMITMENT_INVALID` |
| `GetEventStream` | open REQ on `ARK_EVENT_STREAM` | n/a | each event carries `stream=batch`, `seq`, `topic` | broadcast cleartext | yes | `StreamStartedEvent` acts as the bind; server emits `BatchFailedEvent` on failure |
| `UpdateStreamTopics` | `ARK_REQUEST` | `m=UpdateStreamTopics` | … | NIP 44 | no | `STREAM_NOT_FOUND` |
| `SubmitTx` / `FinalizeTx` | `ARK_REQUEST` | `m=…` | … | NIP 44 | no | `TX_INVALID`, `CHECKPOINT_INVALID` |
| `GetPendingTx`, `GetIntent` | `ARK_REQUEST` | `m=…` | … | NIP 44 | no | `NOT_FOUND` |
| `GetTransactionsStream` | open REQ on `ARK_TX_STREAM` | n/a | `stream=transactions`, `kind=commitment\|ark\|sweep`, `seq` | broadcast cleartext | yes | server emits no errors; absence equals nothing happening |
| All `IndexerService.Get*` | `ARK_REQUEST` | `m=Indexer.<method>` | … | NIP 44 (BIP 322 carrying calls must be encrypted) | no | `NOT_FOUND`, `AUTH_REQUIRED` |
| `SubscribeForScripts` / `UnsubscribeForScripts` | `ARK_REQUEST` | `m=…` | … | NIP 44 | no | `SUBSCRIPTION_NOT_FOUND` |
| `GetSubscription` | open REQ on `ARK_INDEXER_STREAM` | n/a | `stream=indexer`, `sub-id=<subscription_id>`, `seq` | NIP 44 to client (events contain scripts and VTXOs gated by ownership) | yes | timeout after idle, server closes, client reopens |

### 5.2 bark

Full proto surface, grouped by service.

| impl | method | direction | request (summary) | response (summary) | streaming | server path:lines |
|---|---|---|---|---|---|---|
| bark | `ArkService.Handshake` | C to S | `bark_version?` | `min_protocol_version, max_protocol_version, psa?` | unary | `bark_server.proto:10` / `bark/server-rpc/src/client.rs:432-434` |
| bark | `ArkService.GetArkInfo` | C to S | `Empty` | `network, server_pubkey, round_interval_secs, exit and expiry deltas, max_vtxo_amount, mailbox_pubkey, fees, FeeSchedule, max_vtxo_exit_depth, …` | unary | `bark_server.proto:11` / `bark/server-rpc/src/client.rs:441` |
| bark | `ArkService.GetOffboardFeeRate` | C to S | `Empty` | `sat_vkb` | unary | `bark_server.proto:12` / `bark/src/lib.rs:1210-1212` |
| bark | `ArkService.GetFreshRounds` | C to S | `last_round_txid?` | `txids[]` | unary | `bark_server.proto:17` |
| bark | `ArkService.GetRound` | C to S | `RoundId{txid}` | `funding_tx, signed_vtxos` | unary | `bark_server.proto:18` |
| bark | `ArkService.GetVtxo` | C to S | `vtxo_id` | `vtxo` | unary | `bark_server.proto:19` |
| bark | `ArkService.RequestBoardCosign` | C to S | `amount, utxo, expiry_height, user_pubkey, pub_nonce` | `pub_nonce, partial_sig` | unary | `bark_server.proto:23` / `bark/src/board.rs:226-232` |
| bark | `ArkService.RegisterBoardVtxo` | C to S | `board_vtxo` | `Empty` | unary | `bark_server.proto:24` / `bark/src/board.rs:282-284` |
| bark | `ArkService.RegisterVtxoTransactions` | C to S | `vtxos[]` | `Empty` | unary | `bark_server.proto:27` / `bark/src/lib.rs:1969-1970` |
| bark | `ArkService.RegisterVtxos` | C to S | `vtxos[]` | `Empty` | unary | `bark_server.proto:33` (deprecated alias) |
| bark | `ArkService.RequestArkoorCosign` | C to S | `ArkoorPackageCosignRequest{parts[]}` (each: `input_vtxo_id, outputs, user_pub_nonces, attestation, use_checkpoint, isolated_outputs`) | `parts[]{server_pub_nonces, server_partial_sigs}` | unary | `bark_server.proto:39` / `bark/src/arkoor.rs:133-134` |
| bark | `ArkService.RequestLightningPayHtlcCosign` | C to S | `LightningPayHtlcCosignRequest{parts[]}` | `ArkoorPackageCosignResponse` | unary | `bark_server.proto:43` / `bark/src/lightning/pay.rs:665-666` |
| bark | `ArkService.InitiateLightningPayment` | C to S | `invoice, htlc_vtxo_ids, payment_amount_sat, mailbox_id?` | `Empty` | unary | `bark_server.proto:44` / `bark/src/lightning/pay.rs:738-739` |
| bark | `ArkService.CheckLightningPayment` | C to S | `hash, wait` | `oneof(pending\|success{preimage}\|failed)` | unary (long poll when `wait=true`) | `bark_server.proto:45` / `bark/src/lightning/pay.rs:346-347` |
| bark | `ArkService.RequestLightningPayHtlcRevocation` | C to S | `ArkoorPackageCosignRequest` | `ArkoorPackageCosignResponse` | unary | `bark_server.proto:46` |
| bark | `ArkService.FetchBolt12Invoice` | C to S | `offer, amount_sat?` | `invoice` | unary | `bark_server.proto:48` / `bark/src/lightning/pay.rs:494-495` |
| bark | `ArkService.StartLightningReceive` | C to S | `payment_hash, amount_sat, min_cltv_delta, mailbox_id?, description?` | `bolt11` | unary | `bark_server.proto:52` / `bark/src/lightning/receive.rs:103-104` |
| bark | `ArkService.CheckLightningReceive` | C to S | `hash, wait` | `invoice, amount_sat, status, htlc_vtxos[]` | unary (long poll) | `bark_server.proto:53` / `bark/src/lightning/receive.rs:308-309` |
| bark | `ArkService.PrepareLightningReceiveClaim` | C to S | `payment_hash, user_pubkey, htlc_recv_expiry, oneof(input_vtxo \| token)` | `CheckLightningReceiveResponse, htlc_vtxos[]` | unary | `bark_server.proto:54` / `bark/src/lightning/receive.rs:351-352` |
| bark | `ArkService.ClaimLightningReceive` | C to S | `payment_hash, payment_preimage, ArkoorPackageCosignRequest` | `ArkoorPackageCosignResponse` | unary | `bark_server.proto:55` / `bark/src/lightning/receive.rs:187-188` |
| bark | `ArkService.CancelLightningReceive` | C to S | `payment_hash` | `Empty` | unary | `bark_server.proto:56` / `bark/src/lightning/receive.rs:546-547` |
| bark | `ArkService.PrepareOffboard` | C to S | `OffboardRequest, input_vtxo_ids[], attestation[]` | `offboard_tx, forfeit_cosign_nonces[]` | unary | `bark_server.proto:60` / `bark/src/offboard.rs:182-190` |
| bark | `ArkService.FinishOffboard` | C to S | `offboard_txid, user_nonces[], partial_signatures[]` | `signed_offboard_tx` | unary | `bark_server.proto:61` / `bark/src/offboard.rs:208-214` |
| bark | `ArkService.NextRoundTime` | C to S | `Empty` | `timestamp` | unary | `bark_server.proto:66` / `bark/src/round/mod.rs:1282-1283` |
| bark | `ArkService.SubscribeRounds` | C to S | `Empty` | stream of `RoundEvent{attempt\|vtxo_proposal\|finished\|failed}` | **server stream** | `bark_server.proto:67` / `bark/src/round/mod.rs:1579-1623` |
| bark | `ArkService.LastRoundEvent` | C to S | `Empty` | `RoundEvent` | unary | `bark_server.proto:69` / `bark/src/round/mod.rs:1496-1497` |
| bark | `ArkService.SubmitPayment` | C to S | `input_vtxos[], vtxo_requests[]{SignedVtxoRequest}, unblinded_mailbox_id?` | `unlock_hash` | unary | `bark_server.proto:70` / `bark/src/round/mod.rs:701-703` |
| bark | `ArkService.ProvideVtxoSignatures` | C to S | `pubkey, signatures[]` | `Empty` | unary | `bark_server.proto:71` / `bark/src/round/mod.rs:1139-1141` |
| bark | `ArkService.SubmitRoundParticipation` | C to S | `input_vtxos[], vtxo_requests[], unblinded_mailbox_id?` | `unlock_hash` | unary | `bark_server.proto:74` / `bark/src/round/mod.rs:1366-1367` |
| bark | `ArkService.RoundParticipationStatus` | C to S | `unlock_hash` | `status, round_funding_tx?, output_vtxos[], unlock_preimage?` | unary | `bark_server.proto:75` / `bark/src/round/mod.rs:946-947` |
| bark | `ArkService.RequestLeafVtxoCosign` | C to S | `vtxo_id, public_nonce` | `public_nonce, partial_signature` | unary | `bark_server.proto:76` / `bark/src/round/mod.rs:743-744` |
| bark | `ArkService.RequestForfeitNonces` | C to S | `unlock_hash, vtxo_ids[]` | `public_nonces[]` | unary | `bark_server.proto:77` / `bark/src/round/mod.rs:787-788` |
| bark | `ArkService.ForfeitVtxos` | C to S | `forfeit_bundles[]` | `unlock_preimage` | unary | `bark_server.proto:78` / `bark/src/round/mod.rs:817-818` |
| bark | `MailboxService.SubscribeMailbox` | C to S | `MailboxRequest{unblinded_id, authorization?, checkpoint}` | stream of `MailboxMessage{arkoor \| round_participation_completed \| incoming_lightning_payment \| recovery_vtxo_ids \| lightning_send_finished, checkpoint}` | **server stream** | `mailbox_server.proto:12` / `bark/src/mailbox.rs:106-160` |
| bark | `MailboxService.ReadMailbox` | C to S | `MailboxRequest` | `MailboxMessages{messages[], have_more}` | unary | `mailbox_server.proto:13` / `bark/src/mailbox.rs:182-183` |
| bark | `MailboxService.PostArkoorMessage` | C to S | `blinded_id, vtxos[]` | `Empty` | unary | `mailbox_server.proto:11` / `bark/src/arkoor.rs:221-225` |
| bark | `MailboxService.PostRecoveryVtxoIds` | C to S | `unblinded_id, vtxo_ids[]` | `Empty` | unary | `mailbox_server.proto:15` / `bark/src/mailbox.rs:436-437` |

Nostr mapping for bark:

| bark method | Nostr kind | request tags | response tags | encryption | streaming | error semantics |
|---|---|---|---|---|---|---|
| `Handshake` | `ARK_REQUEST` | `m=Handshake`, `v` | `m=Handshake` | NIP 44 (could be cleartext) | no | `PROTOCOL_VERSION_UNSUPPORTED` |
| `GetArkInfo` / `GetOffboardFeeRate` | `ARK_REQUEST` | `m=…` | … | NIP 44 | no | trivial |
| `GetFreshRounds` / `GetRound` / `GetVtxo` | `ARK_REQUEST` | `m=…` | … | NIP 44 | no | `NOT_FOUND` |
| `RequestBoardCosign` / `RegisterBoardVtxo` | `ARK_REQUEST` | `m=…` | … | NIP 44 | no | `BOARD_INVALID`, `AMOUNT_OUT_OF_RANGE` |
| `RegisterVtxoTransactions` / `RegisterVtxos` (dep.) | `ARK_REQUEST` | `m=…` | … | NIP 44 | no | `VTXO_INVALID` |
| `RequestArkoorCosign` / `RequestLightningPayHtlcCosign` / `RequestLightningPayHtlcRevocation` | `ARK_REQUEST` | `m=…` | … | NIP 44 | no | `COSIGN_REJECTED`, `ATTESTATION_INVALID` |
| `InitiateLightningPayment` / `FetchBolt12Invoice` | `ARK_REQUEST` | `m=…` | … | NIP 44 | no | `INVOICE_INVALID`, `NO_ROUTE` |
| `CheckLightningPayment` / `CheckLightningReceive` | `ARK_REQUEST` | `m=…`, `wait` | … | NIP 44 | long poll on ASP side | `DEADLINE_EXCEEDED` |
| `StartLightningReceive` / `PrepareLightningReceiveClaim` / `ClaimLightningReceive` / `CancelLightningReceive` | `ARK_REQUEST` | `m=…` | … | NIP 44 | no | `HTLC_INVALID`, `ANTI_DOS_REQUIRED` |
| `PrepareOffboard` / `FinishOffboard` | `ARK_REQUEST` | `m=…` | … | NIP 44 | no | `OFFBOARD_INVALID` |
| `NextRoundTime` / `LastRoundEvent` | `ARK_REQUEST` | `m=…` | … | NIP 44 | no | trivial |
| `SubmitPayment` / `ProvideVtxoSignatures` / `SubmitRoundParticipation` / `RoundParticipationStatus` / `RequestLeafVtxoCosign` / `RequestForfeitNonces` / `ForfeitVtxos` | `ARK_REQUEST` | `m=…` | … | NIP 44 | no (status polled; forfeit follows the rounds subscription) | `ROUND_PHASE_MISMATCH`, `INVALID_NONCES`, `INVALID_FORFEIT_BUNDLE` |
| `SubscribeRounds` | open REQ on `ARK_EVENT_STREAM` | n/a | `stream=rounds`, `seq` | broadcast cleartext | yes | no error event; client receives `RoundFailed` |
| `MailboxService.SubscribeMailbox` | open REQ on `ARK_MAILBOX_STREAM` | n/a | `stream=mailbox`, `sub-id=<unblinded_id>`, `seq`, event content includes the `checkpoint` of the underlying mailbox message | NIP 44 to client (mailbox contents are sensitive) | yes | resume via `since` and `checkpoint` |
| `MailboxService.ReadMailbox` | `ARK_REQUEST` | `m=Mailbox.Read` | … | NIP 44 | no, pull | trivial |
| `MailboxService.PostArkoorMessage` / `PostRecoveryVtxoIds` | `ARK_REQUEST` | `m=Mailbox.Post…` | … | NIP 44 (the body contains a blinded id; gift wrap may be preferable for `PostArkoorMessage`, see §4.1 and §7.1) | no | `INVALID_RECIPIENT` |

### 5.3 Cross cutting notes

* **Error envelope.** Both implementations adopt the NIP 47 style `{"error":{"code":…, "message":…}}` shape. Codes follow the existing gRPC status mapping (`UNAUTHENTICATED`, `PERMISSION_DENIED`, `INVALID_ARGUMENT`, etc), so wallets have a single error handling path that only changes the human readable strings.
* **Auth and proof carrying.** Where existing methods take BIP 322 proofs (`RegisterIntent.Intent`, `DeleteIntent.Intent`, `GetVtxoChain.IndexerIntent`), the proof rides inside the encrypted content. Macaroons on arkd and `ark-access-token` on bark do too.
* **Streams that need per client gating** (`arkd.GetSubscription`, `bark.SubscribeMailbox`) cannot be public broadcast events; the subscription or mailbox id would leak to anyone watching the kind. Two viable paths: (a) NIP 44 encrypt each stream event to the client pubkey and tag by `sub-id` (the relay sees the tag but not the content); (b) gift wrap each event so even `sub-id` is hidden. (a) is much cheaper. See §4.1 and §7.1.

## 6. Discovery and backwards compatibility

### 6.1 Identity, info event, relay set

The ASP's network identity is its npub. This replaces the (TLS cert, DNS name) pair the wallet pins today. A wallet pins a single ASP pubkey at onboarding.

The ASP advertises capabilities via a replaceable info event (illustrative kind `13483`, awaiting allocation). Tags include `["network", "bitcoin"]`, `["protocol-version", "1"]`, `["impl", "arkd 0.7.0"]`, `["encryption", "nip44_v2"]`, `["methods", "GetInfo RegisterIntent ..."]`, and optionally `["r", <url>, "read"|"write"]` hints that complement the ASP's NIP 65 list.

The relay set is a NIP 65 kind 10002 event authored by the ASP's pubkey. Wallets fetch it on first contact and on reconnect failure.

The onboarding URI follows the shape of NIP 47's `nostr+walletconnect://`:

```
ark+nostr://<asp_pubkey_hex>?relay=wss%3A%2F%2Frelay.a&relay=wss%3A%2F%2Frelay.b
```

It carries the pubkey and a starter relay list; the wallet fetches the info event and the NIP 65 list to populate the rest. Whether a fresh URI scheme is the right shape, or whether to extend an existing one, is an open question (§9).

### 6.2 Running both transports during transition

An ASP runs the Nostr daemon alongside the existing gRPC server:

* The gRPC server stays up on its current port (arkd: 7070 public, `arkd/internal/interface/grpc/service.go:250-516`; bark: tonic endpoint, `bark/server-rpc/src/client.rs:111-149`).
* A new Nostr daemon subscribes to the inbound kind on the configured relay set, decrypts requests, and dispatches into the same internal handler the gRPC server uses (`arkd/internal/interface/grpc/handlers/` for arkd, `bark/server/src/rpcserver/` for bark). The handlers are thin shims over the core application services in both implementations, so a second adapter that translates Nostr events into the same handler interface is plausible without a deep refactor. I'm extrapolating from the layering visible in the rpcserver structure; I have not traced every call to confirm there is no per transport state.

### 6.3 Migration path

* Phase 1 (no spec changes). Wallet announces support; ASPs advertise the Nostr transport in their existing channel (in arkd's case, the `service_status` map in `GetInfo`, `service.proto:195`).
* Phase 2 (this NIP). Wallets and ASPs implement Nostr transport. Both transports run side by side. Wallets default to gRPC when reachable and fall back to Nostr if not.
* Phase 3 (later, optional). Wallets default to Nostr where the ASP supports it, gRPC as fallback. Depends on Nostr transport latency proving acceptable in practice (§8).

No proto changes required on the gRPC side. Nothing here forces existing wallets to upgrade.

## 7. Security

### 7.1 The relay as adversary

A relay can:

* **Correlate.** Even with NIP 44 encryption, a relay sees a fixed `asp_pubkey` and the set of `client_pubkey` values that talk to it. Repeated request volume per client over time is observable. Gift wrap hides the relationship from a single relay; multi relay fan out makes the relay side picture probabilistic.
* **Withhold.** A relay can refuse to deliver a request or response. If the client's relay set is just one relay and it drops the message, the call fails silently. Mitigation: clients should publish to at least two relays with retry and backoff. ASPs should publish stream events to at least two relays.
* **Replay.** Relays cache and rebroadcast. NIP 44 has no built in replay window. The application layer uses `created_at` plus `req-id` to detect duplicates. ASPs should reject requests with `created_at` older than 60 seconds or more than 5 seconds in the future.
* **Reorder.** Inter relay ordering is undefined; intra relay ordering is preserved. The `seq` tag lets clients detect gaps.

### 7.2 Pubkey rotation

Pubkey rotation is harder than TLS cert rotation because Nostr has no chain of trust primitive. A pragmatic approach:

* The ASP signs a "next pubkey" event with its current pubkey (a regular kind with a `["next-pubkey", <new_npub>, <effective_after_unix>]` tag).
* Wallets that have seen the event before the rotation date pre trust the new pubkey.
* Wallets that come fresh after the rotation see only the new pubkey via discovery channels; if the operator wants to maintain a chain, the new pubkey signs a "rotated from" event referencing the old.

Under specified. See §9.

### 7.3 Rate limiting and DoS

Relays are not built to enforce per ASP per method rate limits. Today an ASP can rate limit a TCP connection or a gRPC stream; over Nostr, the ASP rate limits **what it processes**, but the relay still accepts and stores or forwards events the ASP will eventually drop. A spammer can flood the ASP's relay set with kind X events `p` tagged to the ASP and force the ASP to decrypt and discard, which is work. NIP 42 auth on the publish path mitigates this. On the subscribe path, the ASP can throttle which decrypted requests it acts on by client pubkey, which is useful but not airtight.

### 7.4 Timing attacks

NIP 44 sizes are deterministic (powers of 2 padding from 32 bytes upward). That blunts but doesn't eliminate size based fingerprinting; methods with very different request sizes (e.g. `GetInfo` at roughly 60 bytes plaintext vs `SubmitTreeSignatures` carrying maps of nonces, easily 4 KB or more) fall into different padding buckets. A determined relay observer can probabilistically attribute method calls.

### 7.5 What encryption does and does not hide

NIP 44 hides plaintext content (including method name, params, errors).

NIP 44 does **not** hide: the client and ASP pubkeys (visible as event `pubkey` and `p` tag), the kind, the `m` tag, the `req-id` tag, timestamps, other tags, or the set of relays involved.

NIP 59 gift wrap additionally hides: the client pubkey from the relay (replaced by a random outer pubkey), the kind of the rumor, and the recipient (the outer `p` tag is to the recipient but the rumor's kind and method are opaque).

NIP 59 still leaks: total traffic volume per outer relay, and the fact that someone is sending gift wrapped messages.

### 7.6 First contact ASP authentication

TLS gives the client a server authentication step: the cert proves the server's DNS name (or pinned cert) is the one the wallet expected. Nostr offers no PKI equivalent; an ASP pubkey is just bits.

This proposal punts on PKI. The user gets the npub out of band (QR code, link, page) and trusts whatever the source said. Once the wallet has the npub, it can detect a MITM (the attacker would have to sign with a key the wallet did not pin), but it cannot detect that the **original** npub was malicious. Same trust model as scanning a Lightning node pubkey from a QR code today.

Pragmatic addition: ASPs should publish a NIP 05 verifier file at `https://asp.example.com/.well-known/nostr.json` linking a human readable identifier (`asp@asp.example.com`) to their pubkey. Wallets can show the identifier in the UI; it doesn't eliminate the trust problem but improves the auditability of "is this the npub I think it is".

## 8. Performance and operational considerations

Estimates below. To be benchmarked.

* **Request size.** Most arkd unary requests have plaintext bodies under 1 KB. After NIP 44 v2 padding to 1024 bytes and base64 encoding, on wire size is around 1.4 KB per request event. Outliers: `SubmitTreeNonces` and `SubmitTreeSignatures` carry maps that can run several KB; `RegisterIntent` carries a BIP 322 proof (500 to 1000 bytes). `SubmitTx` carries a signed Ark tx plus checkpoint txs and could easily be 4 KB or more plaintext, 5.5 KB or more on the wire.
* **Stream event size.** `BatchStartedEvent`, `TreeTxEvent`, and similar on arkd are small (I'm extrapolating from the handler code in `arkservice.go:484-641`, not from `types.proto` which I didn't read in detail). Most stream events should be well under 1 KB.
* **Throughput.** A typical Ark round on arkd is bounded by the number of intents per batch; at, say, 100 participants, the round generates roughly O(100 × small event), a few hundred events per few second window. Public relay loads can sustain that. bark's `SubscribeRounds` is lower volume.
* **Latency.** gRPC over local TLS has p50 latency on the order of 1 to 10 ms for unary calls. Nostr over a relay adds the client to relay leg, the relay to ASP leg, and the response symmetric. Realistic public relays add 50 to 200 ms per leg, so a Nostr unary RPC will likely run 100 to 400 ms p50, an order of magnitude higher than gRPC. For round coordination (intent submission, tree signing) that's acceptable. For tight loops (status polling during a wait), it argues for long polling on the ASP side instead of client side polling.
* **ASP side hosting cost.** If the ASP runs its own relay it pays for the relay's hosting (mostly bandwidth and storage for stream events). At the volumes above, that's hundreds of MB per day for an active ASP, which is small. If the ASP uses public relays the monetary cost is zero but trust spreads.
* **Behaviour under relay failure.** With a multi relay client setup, individual relay failure is transparent: the client publishes to N relays and the response arrives from whichever the ASP saw and replied to first. If all the client's chosen relays fail, the client falls back to the ASP's advertised relays (NIP 65). If those fail too, the client is offline. Wallets should surface this state.

## 9. Open questions and risks

* **Whose surface goes first.** arkd and bark have non overlapping methods. This is a wallet developer ergonomics question, not a NIP blocker. The NIP scope is the envelope (connection, encryption, correlation, streaming, discovery); each implementation's methods ride inside as opaque payloads. Wallets that target both kinds of ASP ship two adapters, the way Lightning wallets that target both LND and Core Lightning do. `nip-draft.md` takes the envelope only path; a later revision could elevate one implementation's method set to canonical.
* **Kind allocation.** Four kinds requested. NIP allocations need a PR to `nostr-protocol/nips`; numbers shouldn't be reserved before community review.
* **Backfill window.** How far back into a stream can a client resync after disconnect? Depends on the relay's storage policy. The ASP can advertise a recommendation but cannot enforce it for relays it doesn't run. Wallets that disconnect for longer than the worst case retention need to re derive state by other means (chain observation, gRPC).
* **NIP 42 in practice.** Most public relays don't enforce NIP 42. ASPs that want auth on their published events have to operate or contract a NIP 42 relay. Real operational cost.
* **Browser WebSocket caps.** Browsers cap concurrent WebSocket connections per origin around 255. A wallet talking to a few relays plus the ASP's relays is fine; a wallet that wants to subscribe to many ASPs simultaneously could hit the cap.
* **Discovery URI scheme.** `ark+nostr://` is just a candidate. Coexistence with `bitcoin:`, `lightning:`, `nostr:` URIs and OS deeplink handling needs design.
* **Pubkey rotation chain.** No precedent in the NIPs I read. A new NIP for "rotated pubkey delegation" may be needed.
* **Long poll method semantics on Nostr.** `CheckLightningPayment(wait=true)` is a gRPC unary the server holds open for minutes. Over Nostr it's two events separated by an arbitrary delay; the client must set a timeout and re issue if it elapses. Round trip cost of re issuing is higher than a held connection; this may justify Nostr side support for pull based polling (`wait=false`, retry on a timer).
* **Anti DoS gating.** bark's `PrepareLightningReceiveClaim` requires either an `input_vtxo` or a `token`. Over Nostr the application layer is unchanged, but the relay level DoS surface is bigger than gRPC (no per TCP rate cap). ASPs may want to require NIP 42 plus a relay side allowlist for write traffic.
* **Stream event ordering** under fan in from multiple relays is the hardest semantic issue. The `seq` tag mitigates it but adds ASP side bookkeeping (per stream monotonic counter, persisted across restarts). To be specified in the NIP.
