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

## Use

```clojure
(require '[dango.chain :as chain])

(def token                                   ; Authn mints the root
  (chain/root {:issuer-seed issuer-seed
               :grant {:caps #{[:data/read "kotoba://storage/<did>/ds/*"]}
                       :before 1800000000}
               :caveats [] :next-seed bearer-seed}))

(def narrowed                                ; the bearer narrows it, offline
  (chain/attenuate token {:grant {:caps #{[:data/read "kotoba://storage/<did>/ds/raw/*"]}
                                  :holder "did:key:z6MkAgent" :before 1750000000}
                          :caveats ['(<= :req/bytes 1000000)]
                          :next-seed agent-seed}))

(chain/verify narrowed {:roots #{issuer-public-key} :revoked #{}
                        :request {:req/kind :data/read :req/resource "kotoba://storage/<did>/ds/raw/a"
                                  :req/now 1700000000 :req/holder "did:key:z6MkAgent"
                                  :req/bytes 4096}})
;; => {:dango/allowed? true :dango/reason :granted :dango/cids [...] :dango/effective {...}}
```

`chain/encode-token` / `decode-token` move a token as one canonical value.

## Deciding at an edge — in this order, first failure wins

| stage | refused as |
|---|---|
| every block decodes canonically and is well-formed (caveats included) | `:dango/malformed-block` `:dango/malformed-caveat` `:dango/unknown-operator` `:dango/unknown-fact` `:dango/caveat-too-large` |
| 1–16 blocks | `:dango/empty-chain` `:dango/chain-too-long` |
| every `:prev` links the previous block's CID | `:dango/broken-link` |
| every signature verifies under the parent's `:next-key`, the root under a trusted root | `:dango/bad-signature` |
| a holder, once set, is never changed | `:dango/holder-readdressed` |
| no block CID is revoked | `:dango/revoked` |
| the proof matches the last `:next-key` (a truncated chain fails here) | `:dango/bad-proof` |
| the request names a kind, a resource and a time | `:dango/malformed-request` |
| `authority.chain/authorize` over the meet of all blocks | `:dango/not-covered` `:dango/expired` `:dango/wrong-holder` |
| every caveat of every block holds | `:dango/caveat-false` |

A block's CID is computed from the bytes received, never from a re-encoding,
so there is no "CID mismatch": tampering is a `:bad-signature`.

Caveats (`dango.caveat`) are a closed, total, **three-valued** predicate
language: a fact the request does not carry, or operands of different types,
make a comparison *unknown*, and `not` / `and` / `or` keep unknown unknown
(Kleene). Only a final `true` holds. Two-valued logic here would be fail-open:
`(not (= :req/kind :data/write))` would hold for a request with no kind.

Capabilities become authority scopes with the kind as the first segment —
`[:data/read "kotoba://storage/<did>/ds"]` → `["data.read" "kotoba" "storage"
"<did>" "ds"]` — so kinds never cover each other. `:before` / `:req/now` are
epoch seconds, compared as 12-digit zero-padded strings (string order =
numeric order), because `authority.grant` compares instants as strings.

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

**Portable reference implementation (2026-09-24).** `.cljk`, runs under kbb /
nbb / ClojureScript (Workers) from the same source. Built: mint, attenuate,
verify (all ten stages above), the caveat evaluator, token codec.

```
kbb --backend sci --classpath "src:resources:test:<deps src dirs>" run-tests.cljk
```

where `<deps src dirs>` are the `src` of authority, io-ipld (+ `resources`),
io-multiformats, org-ietf-cbor, org-nist-sha2, org-ietf-ed25519,
org-ietf-x25519, text, coll, edn, dev-protobuf and test from the west
checkouts. 55 assertions; each safety check was removed once in a copy and
the suite went red at the test named for it (proof, three-valued `not`,
revocation, holder rule, caveats).

**Not yet:** the `.kotoba` build that ADR-2609242100 §9 asks for (amu native
first, `wasm32-browser` for Workers). Measured 2026-09-24 with
`amu check --jvm-free` (amu `bb73470a`) on `dango.caveat`, fixing each refusal
in a copy to reach the next: explicit `(:export …)` required → `kotoba.lang.text`
resolves from `kotoba-lang/lang/compat`, not `text/src` → no `volatile!`
(rewritten as a fold here) → map keys must be keywords / integers / strings, so
the operator table cannot be keyed by symbols → `some` is the option constructor
→ an untyped map accumulator is refused (`record-get without a type
descriptor`). What is left is a real port: typed records, and a caveat form as a
typed recursive value (`:document` is the candidate) — which also decides
whether the wire form stays a quoted list `(and …)` or becomes keyword-headed
`[:and …]`. That choice is open. Until the port lands this is the oracle, not
the Q9 migration, and no consumer cuts over on it.

Also not built: sealing (dropping the proof after a final signature), the
revocation list's home, per-surface fact vocabularies, Authn minting, and the
one-time edge cutover (adr-2609241800 stays in force until then).
