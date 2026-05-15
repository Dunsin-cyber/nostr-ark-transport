# Sequence: single request and response RPC

`GetInfo` on arkd or `GetArkInfo` on bark, the simplest in scope unary RPC. The client publishes one ephemeral request event tagged to the ASP's pubkey. The ASP filters for matching events on its relay set, decrypts, processes, and publishes an ephemeral response event tagged back to the client. Correlation is by `e` tag pointing at the request event id.

```mermaid
sequenceDiagram
    autonumber
    participant W as Wallet (client_pubkey)
    participant R as Relay (wss)
    participant A as ASP (asp_pubkey)

    Note over W,A: Preconditions: wallet has asp_pubkey and a working relay URL.

    W->>W: enc_content = nip44_encrypt(asp_pubkey, client_priv,<br/>{"method":"GetInfo","params":{}})
    W->>W: req_event = sign({kind:27483, pubkey:client_pubkey,<br/>tags:[p:asp_pubkey, m:GetInfo, v:1, encryption:nip44_v2],<br/>content:enc_content})

    W->>R: ["EVENT", req_event]
    R-->>W: ["OK", req_event.id, true, ""]

    Note over A: ASP holds an open REQ on this relay:<br/>filter = {kinds:[27483], "#p":[asp_pubkey], since:now}

    R->>A: ["EVENT", "<sub>", req_event]
    A->>A: decrypt content, dispatch to internal handler,<br/>compute result

    A->>A: enc_resp = nip44_encrypt(client_pubkey, asp_priv,<br/>{"result_type":"GetInfo","result":{...},"error":null})
    A->>A: resp_event = sign({kind:27484, pubkey:asp_pubkey,<br/>tags:[p:client_pubkey, e:req_event.id, m:GetInfo, encryption:nip44_v2],<br/>content:enc_resp})

    A->>R: ["EVENT", resp_event]
    R-->>A: ["OK", resp_event.id, true, ""]

    Note over W: Wallet's REQ filter:<br/>{kinds:[27484], "#e":[req_event.id], authors:[asp_pubkey]}

    R->>W: ["EVENT", "<sub>", resp_event]
    W->>W: decrypt content, route to caller by e-tag match
```

In a multi relay setup the wallet publishes the request to N relays in parallel. The response arrives from whichever relay the ASP saw first. The wallet deduplicates by event id.
