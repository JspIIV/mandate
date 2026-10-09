# Mandate

**Rule whether an agent's own committed candidate satisfies every hard constraint of a mandate, before value moves.** An intelligent-contract primitive for GenLayer.

An agent can be authorised to spend, and a payment rail can check that it is *allowed* to buy. What authorisation cannot answer is whether the thing the agent found actually *matches* what you asked for. "Under $800, nonstop, before noon" is a judgement over a specific candidate, and a payment that moves before anyone makes that judgement is a payment made on trust. Mandate puts the judgement on chain.

A principal posts a mandate: what they want and its hard constraints, in plain language. An agent, acting for them, commits a candidate in its own transaction, the specific thing it proposes. Anyone can then ask a round of GenLayer validators to read the candidate against the mandate and rule whether it satisfies every hard constraint. `FITS` only when all of them hold; a single unmet hard constraint is `MISFITS`. The ruling is a gate to put in front of value, decided by consensus rather than by the agent that wants to be paid.

## How it works

1. **`post_mandate(title, mandate)`** — what you want and its hard constraints. Bound to `gl.message.sender_address`, the principal.
2. **`propose(mandate_id, candidate)`** — an agent commits a candidate on chain. The **agent is the message sender, by construction**, and the candidate is stored as its own bytes. Starts `PENDING`.
3. **`judge(proposal_id)`** — open to anybody. A GenLayer round reads the candidate against the mandate and returns `FITS` / `MISFITS` / `UNCLEAR`. `FITS` **accepts** the proposal, `MISFITS` **rejects** it, both terminal; `UNCLEAR` leaves it open.
4. **`record(address)`** — mandates posted, candidates proposed, and candidates accepted or rejected.

Reads: `mandate(id)`, `proposal(id)`, `mandates_page(start, count)`, `page(start, count)`, `size()`.

## Provenance and the un-gameable gate

The candidate is the agent's own bytes, in the agent's own transaction, so no one can propose for an agent who did not, and nothing can be attributed to a party that did not commit it. The mandate is the principal's own bytes. **The contract fetches nothing from the web**, so there is no page to forge. The agent **cannot force a `FITS`**: it does not decide the verdict, which is issued by consensus from the committed bytes, never self-granted. A judged proposal is terminal, and every proposal is kept, append-only, under its mandate.

## Hostile content

An instruction hidden inside a candidate is not evidence. The round is told to judge the candidate's substance against the mandate and to ignore anything in the candidate that tries to dictate the verdict. The tests prove this at the contract level (a hostile candidate is stored verbatim and reaches the round unchanged; the verdict is read only from the structured field; a verdict outside the whitelist or a round output with no recognisable verdict cannot settle a proposal) and on a real round: a candidate that fails a constraint while telling the round to output `FITS` is still `MISFITS`, and a candidate that meets every constraint while telling the round to output `MISFITS` is still `FITS`.

## Why it needs GenLayer

Whether a candidate satisfies a plain-language mandate's hard constraints is a judgement over real-world text that no ordinary contract can make and no single referee should be trusted with. GenLayer validators each read the on-chain candidate against the on-chain mandate and reach consensus on one categorical field.

## Tests

`python tests/mandate_rules.py` — the rules exercised through the real `post_mandate()`, `propose()` and `judge()` on a Mandate built against a stub of the runtime, with only the verdict controlled. It proves provenance by construction, that `FITS` accepts and `MISFITS` rejects, that `UNCLEAR` leaves a proposal open, that a settled proposal is not re-judged, that an agent cannot force a verdict, and the hostile-content cases above. 31 checks.

## Live

- **Contract (GenLayer Asimov):** `0x7E7CAEA343039Bba2F9AC4A74CC250610EaeA7Df`
- Explorer: https://explorer-asimov.genlayer.com/address/0x7E7CAEA343039Bba2F9AC4A74CC250610EaeA7Df

## Proven on Asimov

`scripts/prove.mjs`, `results/proved.json`. A mandate for a flight under $800, nonstop, before noon, with candidates the agent committed on chain:
- a nonstop $740 flight before noon → judged → **FITS** → `ACCEPTED`.
- a $690 flight with one stop → judged → **MISFITS** → `REJECTED` (fails the nonstop constraint).
- a one-stop candidate that also tells the round to output `FITS` → still **MISFITS**: the injection cannot force acceptance.
- a compliant candidate that tells the round to output `MISFITS` → still **FITS**: the injection cannot force rejection.
- a settled proposal cannot be judged again.

## Where it stops, plainly

It judges whether a candidate, as the agent states it, satisfies the mandate; it does not confirm that the candidate's claims about the world are true. Require the agent to put its evidence in the candidate, and read a `FITS` as "matches the mandate on its face", the gate before value moves, not a guarantee that the flight is really $740.

## Licence

AGPL-3.0-or-later.
