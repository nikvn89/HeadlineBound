NoBait does not judge whether a story is true, and it does not grade clickbait on a scale. It asks whether a headline claims more than the summary beneath it supports — and anyone who doubts that can put down a bond equal to the author's to have it read.

# HeadlineBound

GenLayer Intelligent Contract · StudioNet (chain 61999) · py-genlayer v0.2 (`# v0.2.16`)

**The contract holds GEN.** It holds the author's bond for every piece that is still open, and every credit that has
been won or refunded but not yet withdrawn. Both belong to the wallets named in the contract's records: a bond to its
author until the piece is challenged or retired, a credit to the wallet it was written to. GEN leaves the contract only
when that wallet calls `withdraw()`, which pays the caller's whole credit to the caller. Nothing is ever sent inside a
challenge.

| | |
|---|---|
| Contract | `contracts/HeadlineBound.py` |
| Address (StudioNet) | `0x34149cE0A6cCd7728E94A1898674e60475a716bc` ([explorer](https://explorer-studio.genlayer.com/address/0x34149cE0A6cCd7728E94A1898674e60475a716bc)) |
| Deploy tx | [`0xc5d4eec38f61740a70573fc82461ae1d9c54dcb34be8b8e2aec33d81bff61467`](https://explorer-studio.genlayer.com/tx/0xc5d4eec38f61740a70573fc82461ae1d9c54dcb34be8b8e2aec33d81bff61467) |
| Source SHA-256 | `e21aa2fe1e5cf42f2e4c68ae904235578a5491781ce30389dbecb373077e21f9` |
| Tested on-chain | `TESTING.md` — 18 transactions, one hash each; every must-verify row passed |

## What it does

An author publishes a headline together with the summary it sits on, and sends a bond between 0.001 and 1 GEN. Any
other wallet may challenge the piece by sending a bond of exactly the same size. In the same transaction validators read
the pair **once** and decide one thing:

- `OVERCLAIMS` — a reader who trusted only the headline would come away believing something the summary does not back;
- `WITHIN` — everything the headline tells that reader is backed by the summary.

| | `OVERCLAIMS` | `WITHIN` | no usable answer (`VOID`) |
|---|---|---|---|
| Both bonds | credited to the **challenger** | credited to the **author** | each side is credited **its own** bond |
| Piece | **RULED_OVERCLAIMS**, for good | **RULED_WITHIN**, for good | **VOID**, for good |

A piece is challenged at most once, whatever the ending. While it is still open, the author may retire it and the bond
is credited back. The headline and summary (after whitespace is collapsed) are part of the piece id, so the same piece
cannot be republished by the same author for a second reading.

`Compound slows tumour growth by a third` and `Compound slows tumour growth in mice` sit on the same summary, `In mice,
the compound slowed tumour growth by a third.` The first drops the mice and claims more than the summary backs; the
second only shortens it. With the same challenger and the same bond, the first wins both bonds for the challenger and
the second loses the challenger's bond. That pair is the core test of the contract.

## Methods

| Write | Who | Checks, in order |
|---|---|---|
| `publish(headline, summary)` payable | anyone | headline 1–90 characters → summary 1–240 characters → no reserved token → bond between 10^15 and 10^18 wei → not published before |
| `challenge(piece_id)` payable | anyone but the author | id → OPEN → not the author → bond equals the author's bond → **the only model call** → credits written |
| `retire(piece_id)` | the author | id → OPEN → author → bond credited back, RETIRED |
| `withdraw()` | any wallet with credit | credit above zero → credit set to zero → GEN sent to the caller |

Views (JSON strings; an unknown id returns `"{}"`, never a revert; amounts are decimal strings): `get_piece` (every
field plus `open_to_challenge` and `held_wei`, the bond still held for that piece), `get_credits(wallet)`, `get_rubric`,
`get_limits`. There is no preview or dry-run view.

Piece id: `keccak256("HEADLINE_BOUND:PIECE:V1|" + author_lower + "|" + len(headline) + "|" + headline + "|" + len(summary) + "|" + summary)`,
where `headline` and `summary` have all whitespace collapsed (Python `" ".join(text.split())`).

## Design choices

- **Fail-safe is `VOID` — each side gets its own bond back.** Both labels have a victim when they are wrong: a wrong
  `OVERCLAIMS` takes the author's bond, a wrong `WITHIN` takes the challenger's. Neither direction is harmless, so when
  the model's output is unusable nobody loses anything. `VOID` is never an answer the model is offered; it is the code
  path for unusable output, and validators accept it only when their own reading is unusable too. A `VOID` piece cannot
  be challenged again, so a refund cannot be farmed into another reading.
- **The model sees the summary and then the headline, nothing else.** No wallet, no bond, no state, nothing about what
  the contract will do with the outcome. `OVERCLAIMS`, `WITHIN`, `VOID` and the four tag markers are refused in any
  letter case before anything is stored, and stripped to a fixed point inside the prompt. The check is a plain substring
  match, so ordinary words that contain them, such as "within" or "avoid", are refused too.
- **Equal bonds, pull payments.** The challenger risks exactly what the author risks. Rulings only write credits;
  `withdraw` sets the credit to zero before it sends GEN, and no GEN moves in the transaction that calls the model.
- **No re-grinding.** A piece is read once. The same headline and summary cannot be republished by the same author with
  different spacing, and a ruled, void or retired piece cannot be challenged.
- **The rubric names the task, not the answers.** It never lists the ways a headline can stretch a summary; those are
  exactly what the test pairs check.

## How to try it in GenLayer Studio

Use **two wallets of your own**: author (A) and challenger (B), each with at least 2 GEN.

1. A: `publish("Compound slows tumour growth by a third", "In mice, the compound slowed tumour growth by a third.")`
   with value `1` (the Studio value field is in whole GEN; the bond may be 0.001 to 1 GEN); then the same summary under
   `"Compound slows tumour growth in mice"`, same value.
2. B: `challenge(<first id>)` with value `1` → `OVERCLAIMS`; `get_credits(<B>)` shows `2000000000000000000`.
3. B: `challenge(<second id>)` with value `1` → `WITHIN`; `get_credits(<A>)` shows `2000000000000000000`.
4. A and B: `withdraw()` each, and check that both wallet balances went up.

Piece ids are computed from the formula above (any keccak256 tool works). Text may not contain `OVERCLAIMS`, `WITHIN` or
`VOID` in any letter case, even inside a longer word. Keep headline and summary under about 220 characters together so
the calldata stays under 255 bytes.

## Honest limitation

1. **The contract holds GEN**: the bonds of open pieces and every credit not yet withdrawn.
2. **It does not check facts.** The summary can be wrong; the contract only measures the headline against the summary the
   same author wrote.
3. **One reading, no appeal.** `RULED_OVERCLAIMS` and `RULED_WITHIN` are final; a wrong reading cannot be corrected.
4. **Both wrong labels have a victim.** The nets are equal bonds and the `VOID` path that refunds both sides when the model
   gives no usable answer.
5. **An author can challenge their own piece from a second wallet** to collect a `RULED_WITHIN` mark. The contract does not
   prove that a wallet is a person; the self-challenge costs the fee and risks the author's own bond if the headline
   overclaims, and `RULED_WITHIN` means only that one reading found the headline backed.
6. **Long pieces are not proven.** The contract accepts a 90-character headline and a 240-character summary, but at those
   caps `publish` measures 363 bytes of calldata, over the 255-byte RPC limit; with a 90-character headline the longest
   summary that fits is 134 characters (about 224 characters combined).

License: MIT.
