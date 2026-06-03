# Nostr as Ark transport, design package

This directory holds a design proposal for using Nostr as the transport between Ark wallets and Ark Service Providers (ASPs), as an alternative to today's gRPC over TLS endpoints. The scope is **connection portability** only: making any given ASP reachable from a browser, a PWA, or a censored network. It is not state portability. Ark is a service model, every ASP holds its own on chain capital, and a VTXO is a claim against that specific ASP's UTXO. None of that changes by switching the transport.

## The problem

Ark today reaches wallets over gRPC on a TLS endpoint, addressed by a DNS name. That is the same shape as any ordinary web service, and it inherits the same weakness: the endpoint is censorable at the network edge before any Bitcoin logic is ever reached.

Concretely, an ASP is a `(DNS name, TLS certificate, IP)` tuple. Each of those is a chokepoint a network operator or a state can squeeze:

* **DNS is blockable.** A resolver can refuse to answer for the ASP's domain, or hand back a poisoned address. This is the cheapest and most common form of censorship, and it happens upstream of the ASP entirely.
* **IP and SNI are blockable.** Even with the address hardcoded, deep packet inspection reads the server name out of the TLS handshake (SNI is sent in the clear) and drops the connection on a match.
* **The endpoint is a single pinned identity.** A wallet holds a long lived gRPC stream to one host. Block that host and the wallet is simply offline; there is no second door.

This is not hypothetical to me. I am writing this from Nigeria, where X was blocked for seven months, and most platforms became reachable only through VPNs that were themselves blocked intermittently. The lived reality of censorship is not "the service is down," it is a moving target: one transport works today, the workaround works next week, and the workaround for the workaround works the week after. Censorship resistant payments matter most exactly to the people sitting behind that kind of network, and the place the gap bites hardest is the browser and PWA wallet, the lowest friction way for a new user to touch Ark at all.

So the question this package answers is narrow: **can we reduce an ASP's network identity to something that has no DNS name and no fixed TLS endpoint to block, and still reach it from a browser?**

## Other approaches I considered

**Domain fronting / CDN tricks.** Hide the real ASP host behind a large CDN so that blocking it means blocking the whole CDN. This worked for a while in the censorship circumvention world and then largely fell away, as the major CDNs (Google, Amazon, Cloudflare) disabled domain fronting under pressure from states. It is a tactic with an expiry date, not a protocol property, and it puts a third party intermediary in the request path.

**Tor onion services.** This is the obvious one, and it genuinely solves the censorship problem for the threat model it covers. An onion service has no DNS name and no IP to block; reachability is the .onion address itself. The gap is twofold. First, **neither arkd nor bark runs as an onion service today**: bark has a client side SOCKS5 proxy hook a wallet could point at Tor, but the ASP itself is still a TLS endpoint, so this is a client side bolt on, not an ASP property. Second, and decisively for the case I care about, **browsers cannot run Tor.** A web wallet or a PWA cannot open a SOCKS5 connection to the Tor network from inside a page. So Tor helps native desktop and mobile clients and does nothing for the browser/PWA user in a censored network, which is the user I keep coming back to. Tor and this proposal are also not mutually exclusive: a user who wants network path anonymity can still route the underlying connection through Tor independent of whether the wire is gRPC or Nostr (see `design.md` §2 non goals).

**A bespoke relay/mixnet of Ark's own.** Build a custom message relay layer specific to Ark. This is just reinventing Nostr with a smaller network, no existing relay operators, no browser client libraries, and all the bootstrapping cost that implies.

## Why Nostr is the right fit

Nostr is, at its core, a network of dumb relays that accept and forward small signed events, with mature browser client libraries and an existing population of public relays. That maps onto the transport problem almost exactly:

* **The ASP's identity collapses to a pubkey, not a hostname.** There is no DNS name to poison and no SNI to match. A wallet pins the ASP's npub the way it pins a TLS certificate today, and reaches it through *any* relay both parties share.
* **It is reachable from a browser with no custom networking stack.** Nostr relays speak WebSocket, which every browser and PWA already has. No SOCKS5, no bundled transport, no native code. This is the property Tor cannot give the browser case.
* **Censorship routes around the relay, not the ASP.** If one relay is blocked, the wallet switches to another without the ASP changing anything. The single pinned endpoint failure mode goes away. (How strong this is in practice depends on the relay set; the honest bounds are in `design.md` §7.1 and §8.)
* **The crypto is already browser shaped.** NIP 44 encryption is ECDH + ChaCha20, audited by Cure53, and faster than AES GCM in pure JS, which is exactly the budget a web wallet has.
* **There is direct prior art.** NIP 47 (Nostr Wallet Connect), NIP 46 (remote signing), and NIP 90 (Data Vending Machines) already do request/response RPC over Nostr. This proposal is modelled on NIP 47 rather than invented from scratch.
* **It is method agnostic, so it does not force the two Ark implementations to reconcile.** The Nostr layer standardises only how messages flow (connection, encryption, correlation, streaming, discovery). arkd and bark each keep their own method names, the same way LND and Core Lightning each expose their own client API and a multi wallet ships two adapters.

Side benefit beyond censorship: decoupling wallets from a fixed gRPC endpoint also helps the ASP scale horizontally. Routing traffic through relays removes the connection level pinning that long lived gRPC streams create, so an ASP can fail over or scale out across datacenters without forcing every wallet to reconnect.

## What this is not

The proposal is **connection portability**, full stop. It does **not** move balances or state between ASPs. Ark is a service model, not a federated network: a VTXO is a claim against one ASP's UTXO and is redeemable only there. Any use of "portability" in these documents means *a wallet can reach the ASP it already chose, from any network*, never moving value across operators. It also does not reconcile the arkd and bark wire surfaces, does not touch the gRPC between an ASP's own internal components, and does not claim stronger anonymity than Nostr itself provides (a relay can still see that pubkey X talks to ASP Y). The full goals and non goals are in `design.md` §2.

## Contents

* `design.md`, the technical design document. RPC surface in both arkd (Go) and bark (Rust), five open design questions presented with options, a method by method Nostr mapping, security and performance notes.
* `nip-draft.md`, a draft NIP modelled on NIP 47 (Nostr Wallet Connect). It picks one option for each open question so a developer can implement directly from the spec. The picks sit in a comment block at the top.
* `diagrams/`, four Mermaid diagrams covering topology, a single request and response, a streaming subscription, and first contact discovery.

## Snapshot and drift warning

The analysis is a snapshot. It was captured against `arkd` at commit `f8aefab4` (May 11 2026), `bark` at `3de34bba` (May 13 2026), and the NIPs repo at `05d3f19` (April 28 2026). The protos, method names, and streaming event vocabulary are the parts most likely to drift, so the tables in `design.md` §1.1 and §4 should be re verified against the current state of those repos before anything goes out for review.

## License

Released under the [MIT License](LICENSE). Free to use, modify, and distribute; this is an open design and is meant to be built on.
