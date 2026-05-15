# Sequence: long lived server streamed subscription

Round event stream: arkd's `ArkService.GetEventStream` or bark's `ArkService.SubscribeRounds`. The client opens a long lived REQ subscription against the ASP's authored stream kind, filtered by `stream` and optionally `since` for backfill. The ASP publishes phase events as they happen. The client may disconnect and reconnect using `since = max(created_at_seen)` to resume without loss, bounded by the relay's retention policy.

```mermaid
sequenceDiagram
    autonumber
    participant W as Wallet
    participant R as Relay
    participant A as ASP

    W->>R: ["REQ", "rounds-sub",<br/>{kinds:[7483], authors:[asp_pubkey],<br/>"#stream":["rounds"], since: t0}]
    R-->>W: ["EOSE", "rounds-sub"]  %% no backfill needed at t0

    Note over A: ASP enters new round phase

    A->>A: evt1 = sign({kind:7483, pubkey:asp_pubkey,<br/>tags:[stream:rounds, t:BatchStarted, seq:42],<br/>content:"{...round info...}"})
    A->>R: ["EVENT", evt1]
    R->>W: ["EVENT", "rounds-sub", evt1]
    W->>W: persist (id, seq=42), notify caller

    A->>A: evt2 (t=TreeSigningStarted, seq=43)
    A->>R: ["EVENT", evt2]
    R->>W: ["EVENT", "rounds-sub", evt2]

    Note over W,R: Wallet's network drops for ~30s

    A->>A: evt3 (t=TreeNoncesAggregated, seq=44)
    A->>R: ["EVENT", evt3]
    Note over R: relay retains evt3 (regular kind)

    W->>R: reconnect: ["REQ", "rounds-sub",<br/>{kinds:[7483], authors:[asp_pubkey],<br/>"#stream":["rounds"], since: evt2.created_at}]
    R->>W: ["EVENT", "rounds-sub", evt3]
    R-->>W: ["EOSE", "rounds-sub"]
    W->>W: detect seq=44 follows seq=43, no gap

    A->>A: evt4 (t=BatchFinalized, seq=45)
    A->>R: ["EVENT", evt4]
    R->>W: ["EVENT", "rounds-sub", evt4]

    Note over W: round complete; wallet keeps subscription open for next round
```

For per client streams (mailbox, indexer subscription) the shape is the same with two changes. Each event additionally carries `["p", client_pubkey]` and `["sub-id", <id>]` tags, and the filter includes `"#p":[client_pubkey], "#sub-id":[id]`. Event content is NIP 44 encrypted to the client.
