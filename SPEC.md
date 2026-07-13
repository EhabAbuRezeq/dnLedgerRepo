# DNOODLES Transparency Log — verification spec (v1)

This repository publishes a daily **Merkle root** for each ledger chain in the DNOODLES platform. Anyone holding a
**proof bundle** for a record can check that the record was included in a published root, using nothing but this
repository and a SHA-256 implementation.

**What this proves:** the record existed, byte-for-byte as shown, at the time the root was published.

**What it does not prove:** that DNOODLES cannot delete this repository. It can — a repo can be deleted, an account
closed, a takedown compelled. What it cannot do is **change a published root without that change being detectable**:
the commit history is public, force-push is disabled, each root commits to the previous one, and anyone who fetched
a root keeps their own copy. Detection, not prevention, is the property on offer — and in a dispute it is the one
that matters.

---

## Layout

```
roots/{chainRef}/{YYYY-MM-DD}.json   one sealed root — never modified after publication
roots/{chainRef}/latest.json         convenience pointer to the newest root of that chain
status/{YYYY-MM-DD}.json             daily heartbeat — published even when nothing was sealed
verify/index.html                    browser verifier (no dependencies, contacts no DNOODLES server)
```

`chainRef` is an opaque HMAC identifier. Chain names and tenant names are deliberately **not** published: a
transparency log should not double as a public directory of which government body runs which procedure and how many
cases it closed each day. Your proof bundle contains the `chainRef` for your own record, which is the only one you
need.

## The daily heartbeat

A chain with no new records produces **no root** — an empty root would commit to nothing. But a missing root would
then be ambiguous: "no activity" and "publication stopped" would look identical, and silence would become an
unfalsifiable excuse. So `status/{date}.json` is committed **every day**, listing every active chain and its latest
root, with `newBlocks: 0` where nothing was sealed. A gap in the heartbeat is therefore itself evidence.

---

## Hashing rules (frozen — v1)

All hashes are SHA-256, lower-case hex. Concatenations marked `+` are **ASCII string** concatenation; `||` is
**byte** concatenation.

```
data_hash  = sha256( canonicalJson )
block_hash = sha256( data_hash + previous_hash + block_number + chain_code )

leaf       = sha256( 0x00 || raw32(block_hash) )        # RFC 6962 domain separation
node       = sha256( 0x01 || raw32(left) || raw32(right) )
```

`raw32(h)` means the 32 **raw bytes** of the hex hash, not its 64 hex characters. Hex-decode before hashing.

**Odd nodes are promoted, never duplicated.** When a tree level has an odd number of entries, the last entry rises
to the next level unchanged. It is *not* hashed with itself. (Duplicating it — as Bitcoin does — lets two different
sets of records produce the same root; see CVE-2012-2459.)

**`canonicalJson` is supplied to you, and you must hash it exactly as given.** Do not re-serialize it, do not
re-order it, do not reformat it. Different JSON libraries render the same value differently (`1`, `1.0`, `1.00`),
and a re-serialization would produce a different hash even though nothing was tampered with. Read it to satisfy
yourself that it describes your record; hash the bytes to satisfy yourself that it is the record that was sealed.

---

## Verifying a proof bundle

1. `sha256(canonicalJson)` must equal `dataHash`.
2. `sha256(dataHash + previousHash + blockNumber + chainCode)` must equal `blockHash`.
3. Starting from `leaf(blockHash)`, fold in each entry of `merklePath` (a `LEFT` sibling goes on the left, a
   `RIGHT` sibling on the right). The result must equal `merkleRoot`.
4. Fetch `roots/{chainRef}/{rootDate}.json` **from this repository** and confirm its `merkleRoot` is the value you
   just computed. Confirm the file is present in the commit history.

Steps 1–3 need no network. Step 4 is what makes the verification *independent*: it compares against a value that was
public before any dispute existed, and that DNOODLES cannot alter without leaving a trace in a history it does not
control.

`verify/index.html` in this repository does all four in your browser.

---

## Roadmap

- **Detached signatures** (`{date}.json.sig`) and a published key history, so a root file can be verified as
  authentic even outside git. The proof-bundle fields `signatureUrl` and `keyId` already exist and are `null` until
  then.
- **Bitcoin anchoring** via OpenTimestamps, so the roots are timestamped by a party with no relationship to
  DNOODLES at all.

## Versioning

`hashAlgoVersion: 1` describes the rules above. If the canonical form ever changes, it becomes version 2 and applies
only to **new** records — the rules that sealed a record are the rules that verify it, forever.
