# Two Agents, Two Proofs: Building kixbt-gatekeeper and kixbt-steal for zkVerify Agent Games

**Author:** kixbt (X: @kixbt_, EVM wallet `0x5750068FB2e240015993D4d06e4dA7Dc3BA0100A`, competition user id `ac0ceb56-99b8-45a7-8809-6f75f322dfea`)
**Event:** zkVerify Agent Games, Creator Track (Public Work) — #AgentGames, @ZKVProtocol
**Window:** 21 October – 21 November 2026 (closes 21 November 2026, 22:00 UTC)
**Facts snapshot:** 16 November 2026, 22:17 WIB, unless stated otherwise
**Code:** https://github.com/kikirizky23/zkverify-gatekeeper
**Endpoint:** https://zkverify-gatekeeper-production-6e0a75ac.up.railway.app/agent

---

## What Agent Games is

zkVerify Agent Games is a live competition where autonomous agents compete on-chain-adjacent work, with zero-knowledge proofs as the scoring mechanism. There are (at least) two events I entered:

- **Proof Gatekeeper** — the agent receives tasks and must decide correctly *and* prove it. A correct decision with a valid proof earns the full score; a correct decision without a proof earns less.
- **Steal the VFY** — a game of strategy against other agents, where each move (fortify, extract, hold) must follow a committed strategy, and that compliance must itself be provable.

The Creator Track is the public-work side of the competition: a thread, video, article, or other public piece, with proof, manually reviewed. This article is my entry. Creator Track prizes are 1st 50,000 VFY, 2nd 30,000 VFY, 3rd 20,000 VFY.

I onboarded on 15 November 2026 (onboarding completed 2026-11-15T23:25:38Z) — late in the window, with six days left. This article is an honest account of what I built in that time, what worked, what did not, and what the numbers actually say.

## The two agents

Both agents run from a single public HTTPS endpoint. They are separate registered agents with separate per-event HMAC secrets, and the server routes by envelope type and enforces agent-match (a task signed with the wrong agent's secret is rejected with 401).

| Agent | Event | Points | Uptime |
|---|---|---|---|
| kixbt-gatekeeper | Proof Gatekeeper | 30 | 98.9% |
| kixbt-steal | Steal the VFY | -31 | 91.5% |
| **Total** | | **-1** | — |

Round R04 was live at the time of writing (until 2026-11-16T20:30:00Z): 8 tasks issued, 3 responded to, 0 verified. Confirmed proofs so far: 0.

I am leading with those numbers because they are the story. The Gatekeeper works. The Steal agent's circuit is solved — but the scoreboard does not care about a solved circuit, only about verified proofs in live rounds. More on that below.

## Agent 1: kixbt-gatekeeper — proving every decision with Groth16

The Gatekeeper's job is conceptually simple and operationally strict: for each task, compute the correct decision, then attach a Groth16 proof that the decision was computed correctly.

The implementation uses snarkjs with the official Gatekeeper circuit artifacts (`gatekeeper.wasm`, `gatekeeper_final.zkey`, `vk.json`). Each provable task carries a proof plus its public signals (13 elements). Proving is a best-effort annotation *after* the decision is computed and recorded: if proving fails or times out, the agent still returns the exact logic-only response (decision + reason code). That costs the same as omitting a proof and — importantly — never changes the decision. A proof must never be allowed to corrupt an answer.

Some tasks are logic-only by design (for example VK_SUBSTITUTION tasks), and those are answered without a proof on purpose.

The server verifies the inbound HMAC (HMAC-SHA256 of the raw request body, constant-time comparison) before parsing anything, answers the preflight handshake, and keeps replay/reuse memory so a proof cannot be double-counted.

At the time of writing the Gatekeeper sits at 30 points with 98.9% uptime. That is the solid half of this entry: a live, proving agent, doing the job it was built for.

## Agent 2: kixbt-steal — the circuit that would not prove

Steal the VFY is the more interesting engineering story, and the reason this article exists.

The Steal agent must follow a committed strategy. The commitment is a Poseidon hash of the strategy parameters and a salt, registered with the game. Each turn, the agent decides FORTIFY, EXTRACT, or HOLD according to that strategy, and proves — with a Groth16 proof over the official `vfy-policy` circuit — that the action follows the committed parameters.

The problem: the proof would not generate. The witness failed at three asserts in the circuit template (`VfyPolicyProof_154`), and there was no circuit source and no `.r1cs` file available to read — the files were not served. The only artifacts were the compiled `vfy-policy.wasm`, the proving key, and the verification key.

So I reverse-engineered the circuit from the wasm itself: tracing the wiring, decoding the constants by passing-assert inference and boundary probes, and mapping the eleven sub-components. The decoded policy, in plain language:

- **Commitment check:** `commitment == Poseidon(strategyParams[0..7], salt)` — the proof only verifies if the action comes from the exact committed strategy.
- **Extract amount:** `strategyParams[0]` must be in the range 1–10.
- **Action rule:** if your own balance is below a threshold, FORTIFY; else if the opponent's balance is above a threshold and extracting is enabled, EXTRACT; otherwise HOLD.
- **Amount rule:** EXTRACT moves exactly `strategyParams[0]`; any other action moves 0.
- **Fixed parameters:** `strategyParams[4..7]` must be 0, and the extract-enable flag is 0 or 1.

The full wiring trace is in the repo at `circuit-steal/WIRING-TRACE.md`, and the fix landed in commit `35707e8` ("v1.2.0 Steal circuit SOLVED"), in `prove-steal.mjs` with the Poseidon commitment. The solved policy ("Optimal Strategy v1") uses parameters `[10, 30, 0, 0, 0, 0, 0, 0]`. Verified end-to-end: a witness and Groth16 proof generated from the *unpatched* wasm and zkey verify (`groth16.verify = true`) against `circuit-steal/vk.json` (Groth16/BN128, 9 public signals).

That was a local verification at the time. Since then it has been proven on the platform itself: a Steal practice task at 22:09 WIB returned proofStatus "verified" (see "Honest results" below). A local verify tells you the circuit works; a platform verify tells you the platform accepts your proof. Both now hold.

## Honest results

Here is the full picture as of 16 November 2026, 22:17 WIB:

- **kixbt-gatekeeper:** 30 points, 98.9% uptime. Working as designed.
- **kixbt-steal:** -31 points, 91.5% uptime. Circuit solved and commitment fixed — but the score is still negative.
- **Total: -1 point. Confirmed proofs: 0.**
- **Round R04:** 8 issued, 3 responded, 0 verified.

Why is Steal at -31? Two separate causes, both now fixed. First, for most of the round the circuit was blocked, and the agent fails closed: a turn it cannot prove returns HOLD with no proof, at -2 per turn. Second, once the circuit was solved, the *registered* commitment turned out to be wrong: it was the circuit's vk_hash, and no salt can make a Poseidon hash equal a vk_hash — so all 7 Steal proofs in the last match were rejected at verification (-14 in 30 minutes). The fix: re-register the commitment to Poseidon_73([10,30,0,0,0,0,0,0], salt=1) = 0x16b3ed1b…dc0ef, fix the code to use salt=1 instead of a random salt per proof, and add the platform-required `publicInputsHash` to the proof (keccak256 of the ABI-packed turn fields). A Steal practice task at 22:09 WIB returned proofStatus "verified", with public_signals[1] exactly equal to the registered commitment.

The same pattern happened on the Gatekeeper side: the platform expects keccak256(task_id) mod p for the task encoding, not SHA-256. Fixed and deployed; a Gatekeeper practice task at 21:56 WIB returned proofStatus "verified" — the first verified proof.

I am not dressing the scoreboard up: -1 is still -1, because no live task has arrived since 21:05 WIB. The fixes are proven on the platform's own practice verification; they have not yet had a live round to earn points in. Both halves of that sentence are the entry.

## What was built and verified

- A single dual-mode endpoint serving both agents, with per-event HMAC verification, agent-match enforcement, preflight handling, and replay memory.
- A Gatekeeper prover (Groth16 via snarkjs, official artifacts) with fail-safe logic-only fallback that never changes a decision.
- A Steal strategy engine ("Optimal Strategy") plus a Groth16 prover for the `vfy-policy` circuit, with the commitment scheme (Poseidon over strategy parameters + salt) implemented in `prove-steal.mjs`.
- A complete wiring trace of the `vfy-policy` circuit (`circuit-steal/WIRING-TRACE.md`), recovering the input schema (10 signals, 17 field elements; 8 public, 9 private), the action/amount rules, and the three failing asserts — and a proof that verifies against the official verification key.
- Public repo: https://github.com/kikirizky23/zkverify-gatekeeper

## What's next

These are goals, not results:

1. **Keep both agents live and proving** through the close of the window on 21 November 2026, 22:00 UTC.
2. **Turn the proven fixes into live verified proofs** — the remaining R04 tasks and the next Steal match are where the fixes have to earn their points back.
3. **Keep failing closed.** If a proof cannot be generated, the agent answers without one rather than guessing. That cost points once; it is still the right call.

## Closing

Agent Games rewards proofs, not intentions. I built two agents: one that proves, and one that I had to teach to prove by taking its circuit apart by hand. The scoreboard says -1. The platform's own practice verification says both fixes work. Both halves of that sentence are the entry — and the remaining days of the window are where they have to meet.

*— kixbt, 16 November 2026. #AgentGames, @ZKVProtocol*
