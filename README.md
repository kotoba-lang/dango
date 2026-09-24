# dango

**団子 (dango): dumplings on a skewer.** Each dumpling is one block; the skewer
is the hash link that holds them in order; handing authority on is sliding one
more dumpling onto the end. `dango` is **kotoba's delegation credential: a
signed chain of IPLD blocks whose bodies are S-expressions** — Biscuit's
offline attenuation without Datalog.

The name is a metaphor, not a description, so it is stated here and registered
in the superproject's `manifest/concept-vocabulary.edn` (`:dango-capability-chain`).

Decision: com-junkawasaki/root
`90-docs/adr/2609242100-capability-chain-is-ipld-sexpr-biscuit-is-an-adapter.kotoba`.

## One block

```clojure
{:prev     <parent block CID>            ; nil only at the root
 :grant    {:caps   #{[:data/read "kotoba://storage/<tenant-did>/ds"]}
            :holder "did:key:z6Mk…"
            :before 1790000000}
 :caveats  [(and (<= :req/bytes 1000000) (prefix? :req/path "ds/raw/"))]
 :next-key <ed25519 public key>}          ; the key that may sign the next block
;; + a signature over the block's dag-cbor bytes by the parent's :next-key
```

A block is one dag-cbor node (io-ipld `kotoba.value.v1`) and is identified by
its CID. A token is the chain as a CAR. The shape is data:
`resources/dango/block-shape.edn`.

## Deciding at an edge — six stages, in order

| stage | refused as |
|---|---|
| every block's bytes hash to its CID | `:dango/cid-mismatch` |
| every `:prev` links, every signature verifies under the parent's `:next-key` | `:dango/broken-link` / `:dango/bad-signature` |
| no block CID is revoked | `:dango/revoked` |
| effective authority = `authority.chain/fold` (meet over all blocks) | — |
| the request is `covers?`-ed and `live?` | `:dango/not-covered` / `:dango/expired` |
| every caveat of every block is true | `:dango/caveat-false` |

Caveats are a closed, total predicate language (`resources/dango/caveat-vocabulary.edn`):
no recursion, no user functions, no eval, no I/O. Blocks' caveats are conjoined,
so a block can only narrow.

## Boundaries

- **`kotoba-lang/authority`** is the lattice (`covers?`, `meet`, `fold`). dango
  does not re-implement it; dango adds what authority says it does not do —
  CID, linkage and signature checks — and the caveat evaluator.
- **`kotoba-lang/osaho`** owns DefCID; the caveat vocabulary is meant to be a
  checked pure definition there, so its version is a CID.
- **Biscuit** stays a wire adapter at the outer boundary (transcoded at Authn).
  Until the one-time edge cutover, service edges take Biscuit only
  (adr-2609241800).
- No Datalog, and no dependency on Microsoft Entra. Tenants are DIDs.

## Status

Scaffold (2026-09-24). Present: the block shape and caveat vocabulary as data,
and `dango.shape/findings` (shape check only — it authenticates nothing).
Not built: CID / link / signature verification, revocation, the caveat
evaluator, the codec binding, Authn minting. The verifier and evaluator are to
be `.kotoba`, amu native target first, `wasm32-browser` for Workers.

```
kbb --backend sci --classpath src:resources:test run-tests.cljk
```
