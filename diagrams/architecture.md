# Architecture

Topology: client wallets, the relay mesh, and one Ark Service Provider, with pubkey identities labelled. The ASP's network identity is its npub. Wallets and the ASP talk only through Nostr relays; neither side connects to the other directly.

```mermaid
flowchart LR
    subgraph clients["Client wallets"]
        W1["Wallet A<br/>npub: client_a"]
        W2["Wallet B<br/>npub: client_b"]
        W3["Browser wallet<br/>npub: client_c"]
    end

    subgraph relays["Nostr relay mesh"]
        R1["relay-a.example<br/>wss"]
        R2["relay-b.example<br/>wss"]
        R3["relay-asp.example<br/>wss · NIP-42"]
    end

    subgraph asp["Ark Service Provider"]
        N["Nostr daemon<br/>npub: asp_pubkey"]
        G["Internal gRPC<br/>(unchanged)"]
        S["Application core<br/>(rounds, signer, indexer)"]
        N --> G
        G --> S
    end

    W1 -- "kind 27483 req<br/>kind 27484 resp" --> R1
    W2 -- "kind 27483 req<br/>kind 27484 resp" --> R2
    W3 -- "kind 27483 req<br/>kind 27484 resp" --> R1

    W1 -. "REQ kind 7483<br/>streams: rounds, mailbox" .-> R2
    W2 -. "REQ kind 7483" .-> R3
    W3 -. "REQ kind 7483" .-> R1

    R1 <--> N
    R2 <--> N
    R3 <--> N

    classDef wallet fill:#eef,stroke:#225
    classDef relay fill:#efe,stroke:#252
    classDef asp fill:#fee,stroke:#522
    class W1,W2,W3 wallet
    class R1,R2,R3 relay
    class N,G,S asp
```

Solid arrows are request and response (ephemeral kinds 27483 and 27484). Dashed arrows are long lived stream subscriptions (regular kind 7483). The ASP operated relay shown here is optional; in the recommended deployment the ASP advertises a relay set via NIP 65 and clients pick from it.

The diagram shows wallets reaching **one** ASP. Each ASP is a discrete liquidity island holding its own on chain capital. A wallet does not roam between ASPs the way a Lightning node routes through peers, and nothing in this transport tries to enable that. A wallet that wants to use multiple ASPs maintains a separate connection record (npub plus relay set) per ASP and treats each balance independently.
