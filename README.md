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
 :caveats  [[:and [:<= :req/bytes 1000000] [:prefix? :req/path "ds/raw/"]]]
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
                          :caveats [[:<= :req/bytes 1000000]]
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

Caveats (`dango.caveat`) are keyword-headed vectors —
`[:and [:<= :req/bytes 10] [:prefix? :req/path "ds/"]]` — never lists, and a
closed, total, **three-valued** predicate
language: a fact the request does not carry, or operands of different types,
make a comparison *unknown*, and `not` / `and` / `or` keep unknown unknown
(Kleene). Only a final `true` holds. Two-valued logic here would be fail-open:
`[:not [:= :req/kind :data/write]]` would hold for a request with no kind.

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

**The caveat evaluator in Kotoba: `src/dango/caveat_eval.kotoba`** (2026-09-24).
Same three-valued semantics over a typed tree (`:dango/term`); the `.cljk`
stays the oracle. Native parity:

```
AMU=../amu KBB_ENGINE=../org-babashka-nbb/cli.js kbb --backend sci test/native_parity.cljk            # exit 0: 13/13 agree
AMU=../amu KBB_ENGINE=../org-babashka-nbb/cli.js kbb --backend sci test/native_parity.cljk --control  # exit 0: two-valued not disagrees on exactly 7 8 9 10
```

Exit 1 = a disagreement (named), exit 2 = could not answer (never a pass).
Measured on aarch64-macos with amu `bb73470a`; x86_64 not measured.

What the native build could not do, measured the same day — these shape the
next step, they are not language ceilings:

- **Exports carry no ADT.** Native export boundaries accept i64 / string /
  scalar records only, so a native host cannot hand the evaluator a tree.
  The native entry therefore has to take the wire bytes and parse them
  inside — which needs the `kotoba.value.v1` decoder as a guest module.
- **No ADT-returning imports across modules** ("project import result type
  has no closed stub value"): the parity cases are spliced into the module
  instead of requiring it.
- **One case per instance.** All 13 cases in one `main` is refused at compile
  time ("native artifact oracle evaluation rejected", reason not reported);
  each case alone passes. Trees are also bounded by osaho's ADT node budget
  (64), well under the wire form's 256-node bound.
- **`kotoba.lang.text` cannot be linked natively** (it carries a
  `[:list :string]` function), so `prefix-of?` is written with string
  builtins.
- **`wasm32-browser` fails with an internal compiler error** on the same
  program (kotoba-lang/amu#1073), so Workers still run the `.cljk`.

**Not yet in Kotoba:** parsing the wire form into the tree, and the chain
verifier (CID, links, signatures — hash and signature verification are to
arrive as declared capability imports, not in-guest crypto). Until those
land, the `.cljk` is the oracle, not the Q9 migration, and no consumer cuts
over on it.

Also not built: sealing (dropping the proof after a final signature), the
revocation list's home, per-surface fact vocabularies, Authn minting, and the
one-time edge cutover (adr-2609241800 stays in force until then).
