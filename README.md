# Nostr as Ark transport, design package

This directory holds a design proposal for using Nostr as the transport between Ark wallets and Ark Service Providers (ASPs), as an alternative to today's gRPC over TLS endpoints. The scope is **connection portability** only, which means making any given ASP reachable from a browser, a PWA in a censored network. It is not state portability. Ark is a service model, every ASP holds its own on chain capital, and a VTXO is a claim against that specific ASP's UTXO. None of that changes by switching the transport.

Contents:

* `design.md`, the technical design document. RPC surface in both arkd (Go) and bark (Rust), five open design questions presented with options, a method by method Nostr mapping, security and performance notes.
* `nip-draft.md`, a draft NIP modelled on NIP 47 (Nostr Wallet Connect). It picks one option for each open question so a developer can implement directly from the spec. The picks sit in a comment block at the top.
* `diagrams/`, four Mermaid diagrams covering topology, a single request and response, a streaming subscription, and first contact discovery.

The analysis is a snapshot. It was captured against `arkd` at commit `f8aefab4` (May 11 2026), `bark` at `3de34bba` (May 13 2026), and the NIPs repo at `05d3f19` (April 28 2026). The protos, method names, and streaming event vocabulary are the parts most likely to drift, so the tables in `design.md` §1.1 and §4 should be re verified against the current state of those repos before anything goes out for review.
