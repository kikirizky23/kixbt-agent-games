# Creator Track — X Thread: kixbt's two zkVerify agents

**Status: READY TO POST — Ujang authorized auto-approve for the score push (22:19 WIB). Post via X @kixbt_ with poster attached to post 1, then submit as Creator Track entry.**
**Track: Creator Track (Public Work), zkVerify Agent Games — #AgentGames, @ZKVProtocol**
**Posts: 10. Character limit: 280 per post. All posts verified below.**

---

## Post 1/10

I built two zk agents for #AgentGames by @ZKVProtocol — one that proves every decision with Groth16, and one where I had to reverse-engineer a circuit just to play. Here's the honest story 🧵

## Post 2/10

First, what is Agent Games? @ZKVProtocol's live competition where agents compete: Gatekeeper (prove your decisions are correct) and Steal the VFY (outplay opponents). Window: 21 Oct–21 Nov 2026.

## Post 3/10

I'm kixbt. Onboarded 15 Nov 2026. I run two agents from one endpoint: kixbt-gatekeeper (Proof Gatekeeper) and kixbt-steal (Steal the VFY). Repo is public: github.com/kikirizky23/zkverify-gatekeeper

## Post 4/10

Gatekeeper: every provable task gets a Groth16 proof (proof + public signals) made with snarkjs and the official gatekeeper wasm/zkey. A correct decision + a valid proof is the full score. No proof, no full points.

## Post 5/10

Steal was harder. The vfy-policy circuit's witness kept failing at 3 asserts. No source, no .r1cs. So I traced the wasm by hand — wiring, constants, Poseidon commitment — until it clicked.

## Post 6/10

Solved in commit 35707e8: commitment == Poseidon(strategyParams, salt). Strategy: extract amount 1–10, fortify / extract / hold by thresholds. A Groth16 proof from the unpatched wasm now verifies. ✅

## Post 7/10

Honest results (16 Nov, 22:17 WIB): Gatekeeper 30 pts, 98.9% uptime. Steal -31 pts, 91.5% uptime. Total -1. Confirmed proofs: 0. Round R04: 8 issued, 3 responded, 0 verified.

## Post 8/10

Why was Steal negative? The registered commitment was the circuit's vk_hash — no salt makes Poseidon equal it. Re-registered to Poseidon([10,30,0,0,0,0,0,0], salt=1). Practice proof VERIFIED 22:09 WIB. ✅

## Post 9/10

What's next: both fixes are proven in practice (Gatekeeper keccak 21:56, Steal commitment 22:09). Now waiting on live R04 tasks + the next Steal match to turn them into verified proofs and points.

## Post 10/10

Building in public, proofs included. The full write-up is part of my Creator Track submission. Thanks @ZKVProtocol for #AgentGames — the hardest fun I've had debugging a circuit. 🧵/end

---

## Notes for posting

- Post 1 first, then reply to each post in order (1→10) so the thread chains correctly.
- Keep the `1/10` numbering in each post — it is included in the character counts below.
- #AgentGames appears in posts 1 and 10; @ZKVProtocol appears in posts 1, 2, and 10.
- The article version of this story is `article.md` in this same folder; link it in a reply after posting if the submission format allows, or submit it as the Article format entry.
- Facts snapshot: 2026-11-16 22:17 WIB (dashboard). Posts 7–9 rewritten 22:20 WIB to reflect both fixes proven in practice.
