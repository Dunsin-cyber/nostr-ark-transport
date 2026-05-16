# Sequence: wallet learns an ASP's identity for the first time

A user adds an ASP to their wallet. The wallet receives an `ark+nostr://` URI (scanned, pasted, or deep linked), uses it to fetch the ASP's kind 13483 info event and kind 10002 NIP 65 relay list, validates network and protocol compatibility, and stores the resulting `(asp_pubkey, relay_set, capabilities)` tuple for later calls.

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant W as Wallet
    participant R1 as Initial relay (from URI)
    participant R2 as Second relay (from NIP-65)
    participant A as ASP

    U->>W: scan or paste<br/>"ark+nostr://asp_pubkey?relay=wss://r1"

    W->>W: parse URI -><br/>(asp_pubkey, [wss://r1])

    W->>R1: ["REQ", "info-sub",<br/>{kinds:[13483], authors:[asp_pubkey], limit:1}]
    R1->>W: ["EVENT", "info-sub", info_event]
    R1-->>W: ["EOSE", "info-sub"]
    W->>R1: ["CLOSE", "info-sub"]
    W->>W: parse tags:<br/>network=bitcoin, protocol-version=1,<br/>methods=[...], encryption=nip44_v2

    W->>R1: ["REQ", "rl-sub",<br/>{kinds:[10002], authors:[asp_pubkey], limit:1}]
    R1->>W: ["EVENT", "rl-sub", relay_list_event]
    R1-->>W: ["EOSE", "rl-sub"]
    W->>R1: ["CLOSE", "rl-sub"]
    W->>W: parse "r" tags -> [wss://r1, wss://r2, wss://r3]

    Note over W: Wallet now has full ASP record. Persists.

    Note over W: First real call: GetInfo (or GetArkInfo).<br/>Wallet publishes to multiple relays for redundancy.

    W->>R1: ["EVENT", req_event for GetInfo]
    W->>R2: ["EVENT", req_event for GetInfo]
    R1->>A: ["EVENT", req_event]
    %% ASP dedupes by event id
    R2->>A: ["EVENT", req_event]
    A->>R1: ["EVENT", resp_event]
    R1->>W: ["EVENT", resp_event]
    W->>W: ASP confirmed reachable,<br/>onboarding complete
```

Trust posture: the wallet trusts whatever channel delivered the URI. The kind 13483 info event is signed by the ASP's pubkey, so the wallet can confirm the relay didn't fabricate it. The wallet cannot, from Nostr alone, confirm that the npub belongs to the real world ASP the user thinks it does. A NIP 05 verifier file (`/.well-known/nostr.json` on a known domain) can carry an additional human readable bind, but that is out of scope for the core transport.
