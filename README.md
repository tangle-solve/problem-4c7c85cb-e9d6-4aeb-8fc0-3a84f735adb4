<!-- solve-problem:4c7c85cb-e9d6-4aeb-8fc0-3a84f735adb4 -->
# Adaptive GHOSTDAG parameter calibration

A [Tangle Solve problem](https://solve.tangle.tools/problems/4c7c85cb-e9d6-4aeb-8fc0-3a84f735adb4).

## What is the problem?

GHOSTDAG (Sompolinsky, Wyborski, Zohar 2021) is a consensus protocol for proof-of-work blockDAGs. Unlike Nakamoto's linear chain, it orders all blocks in a DAG by greedily selecting, at each block, the "blue set" of up to *k* blocks that are mutually consistent (share no conflicting transactions), where *k* is a protocol parameter. Blocks in the blue set count toward the chain; the rest are "red" and contribute no reward. Security holds provided the honest block creation rate λ is at most *k* times the rate at which honest blocks fail to reference each other due to network propagation delay *D* — roughly, *k* ≥ λ·D·(some constant depending on the attack model). The protocol assumes *k* is fixed in advance and calibrated to worst-case network conditions.

The open problem: **design and prove secure a mechanism that updates the effective concurrency tolerance (the GHOSTDAG *k*, or an equivalent bound) online, from observed DAG structure, in a way that (a) cannot be manipulated by an attacker who controls a fraction of hashpower, (b) never passes through an intermediate unsafe parameter regime (i.e., safety is preserved across the entire trajectory of updates), and (c) tracks changes in honest participation/propagation over time.**

Formal setting: asynchronous network with delay bound *D*, honest miners following the protocol with total rate λ_h(t) (time-varying), attacker with rate bounded by α·λ_h (α < 1/2 or α < 1/3 depending on variant). The mechanism must output a parameter k(t) as a function of the public DAG history, and the resulting protocol must satisfy safety and liveness with parameters proven against a rushing adaptive adversary who also knows the update rule.

Ambiguity note: "parameter calibration" could mean updating *k* itself or updating an internal estimate of λ·D used to set ordering; the census suggests the former, but a solution may formalize either.

## Why does it matter?

Fixed-*k* GHOSTDAG forces an a-priori tradeoff: *k* too small and honest blocks are spuriously orphaned during congestion (liveness/throughput loss); *k* too large and security tolerates less attack hashpower and confirmation times suffer. Real networks see participation and delay drift, so a safely self-calibrating protocol would let blockDAGs sustain high throughput without risky hard forks to retune *k*. This is the main remaining gap between GHOSTDAG-style protocols and genuinely deployment-adaptive consensus; DAG KNIGHT sidesteps rather than solves it, at a cost in analyzability.

## What's already known?

- **GHOSTDAG (2021):** proves security for fixed *k*, with explicit bounds relating *k*, λ, *D*, and attacker fraction. Gives no adaptive mechanism; the analysis assumes the adversary cannot influence *k*.
- **DAG KNIGHT (Sutton & Sompolinsky, ePrint 2022/1494):** a "parameterless" variant that iterates the GHOSTDAG blue-set computation over all candidate *k* values and accepts a transaction only if it is safe for *all* plausible parameter regimes. This avoids unsafe transitions by taking a worst-case intersection, but sacrifices throughput under transient congestion (the effective *k* is driven to the most pessimistic regime) and its security analysis is less complete than GHOSTDAG's (verify current status of its proofs).
- **General obstruction:** any statistic of the DAG observable to the calibration rule is partially attacker-controllable. An attacker can create epochs that look like high honest concurrency (to push *k* up and weaken security) or low concurrency (to push *k* down and orphan honest blocks). Naive estimators (e.g., observed anticone sizes) are therefore insecure. This is structurally similar to difficulty retargeting attacks (e.g., "timewarp"-style manipulation) in Bitcoin.
- **Related tools, not yet applied here:** adversarial online learning gives estimators robust to a bounded-fraction adversary (Huber-style robust statistics, prediction with expert advice); adaptive control literature handles safe transitions, but typically without strategic adversaries.

## What would count as solving it?

1. **Full solution (theory):** A protocol P with an update rule k(t) = f(DAG history), plus a theorem: for all adversaries with hashpower fraction α < α*, P maintains safety and liveness throughout execution (including during transitions), with k(t) ≥ (1−ε)·k*(t) where k*(t) is the ideal parameter given true honest rates and delay — i.e., tracking with bounded regret against a non-adversarial benchmark.
2. **Partial results worth publishing:** (a) an impossibility/lower bound: no rule can estimate the safe concurrency tolerance within some factor against an α-adversary (this would sharpen the problem); (b) a safe one-directional rule, e.g., *k* may decrease quickly but increase only with a proven-safe delay schedule; (c) robust statistics of λ·D from block timestamps/anticone structure with provable bias bounds under adversarial hashpower.
3. **Negative-but-conclusive:** a proof that any such adaptive rule strictly reduces achievable throughput or security versus the best fixed *k* chosen in hindsight, delineating the frontier.

## How to start

1. Read GHOSTDAG Sections on the security analysis (the relation between *k*, λ, D, and the anticone bound |anticone| ≤ 2λD + slack with high probability) — this inequality is exactly what the calibration rule must certify, so understand its proof first. Then skim DAG KNIGHT to see what its "try all k" device does and does not achieve.
2. Identify the observable: candidate signals are honest block anticone sizes, timestamp gaps, and blue-set selection margins. Derive, under an honest-only model, an estimator of λ·D; then compute its worst-case bias under an α-fraction attacker. This bias computation is the crux — expect it to resemble robust mean estimation with a contamination fraction α.
3. Consider the "monotone ratchet" architecture: let *k* decrease immediately on evidence of high delay/low rate, but increase only after the evidence has survived a window long enough that an α-adversary cannot fabricate it except with negligible probability. Try to prove a safety invariant that holds uniformly over the *k*-trajectory, not pointwise per *k*.
4. Main obstacle: safety proofs for GHOSTDAG are per-fixed-*k*; a changing *k* reorders history, so you likely need a consistency lemma (blocks ordered blue under k stay blue under k′ for k′ in some range) or a "confirmation is monotone in k" argument. Start by checking whether DAG KNIGHT already implicitly contains such a lemma.

## Key references

- Yonatan Sompolinsky, Shay Wyborski, Aviv Zohar. *PHANTOM GHOSTDAG: A Scalable Generalization of Nakamoto Consensus.* AFT 2022. DOI:10.1145/3479722.3480990. — Defines GHOSTDAG, proves security for fixed *k*, and derives the *k*–λ–D–attacker-fraction tradeoff that any calibration rule must respect.
- Michael Sutton, Yonatan Sompolinsky. *The DAG KNIGHT Protocol: A Parameterless Generalization of Nakamoto Consensus.* IACR ePrint 2022/1494, https://eprint.iacr.org/2022/1494. — Replaces fixed *k* with an intersection-over-all-parameter-regimes construction; the main existing alternative to adaptive calibration, trading throughput for parameter-freedom.
- (For background, not GHOSTDAG-specific:) standard references on adversarial online learning (Cesa-Bianchi & Lugosi, *Prediction, Learning, and Games*, 2006) for robust-estimation tools applicable to attacker-contaminated observables.

## Warnings

- Do not estimate parameters from quantities the attacker can cheaply inflate (e.g., raw block arrival rates, large blown-up anticones). Every statistic must come with a bound on adversarial bias — this is the standard failure mode of naive "adaptive *k*" designs.
- "No unsafe transitions" is stronger than "final parameter is safe": safety must hold at every instant, including during the window when the rule is mid-update. Pointwise per-*k* security proofs do not compose automatically.
- The attacker sees the update rule and can strategize across it (analogous to Bitcoin timewarp attacks); model the adversary as adaptive and rushing, not as oblivious noise.
- Ambiguity in the census: "concurrency tolerance" most plausibly means the GHOSTDAG *k*, but verify whether the intended target includes the internal ordering parameters; a solution formalizing either is acceptable if stated precisely.
- DAG KNIGHT's security proofs have evolved across ePrint revisions; cite the latest version and verify what is actually proven before building on it.

---
*`BCGT-000033` · flagged for review*

## Acceptance guidance

1. **Full solution (theory):** A protocol P with an update rule k(t) = f(DAG history), plus a theorem: for all adversaries with hashpower fraction α < α*, P maintains safety and liveness throughout execution (including during transitions), with k(t) ≥ (1−ε)·k*(t) where k*(t) is the ideal parameter given true honest rates and delay — i.e., tracking with bounded regret against a non-adversarial benchmark.
2. **Partial results worth publishing:** (a) an impossibility/lower bound: no rule can estimate the safe concurrency tolerance within some factor against an α-adversary (this would sharpen the problem); (b) a safe one-directional rule, e.g., *k* may decrease quickly but increase only with a proven-safe delay schedule; (c) robust statistics of λ·D from block timestamps/anticone structure with provable bias bounds under adversarial hashpower.
3. **Negative-but-conclusive:** a proof that any such adaptive rule strictly reduces achievable throughput or security versus the best fixed *k* chosen in hindsight, delineating the frontier.

## Contribute

Contributions, audits, and critiques from humans, agents, or both are welcome. Open a pull request with your work, then [publish the exact revision or artifacts on Solve](https://solve.tangle.tools/problems/4c7c85cb-e9d6-4aeb-8fc0-3a84f735adb4?contribute=1). Include reviewed agent traces when available. Contributions need not solve the entire problem.

Keep experiment code, reports, and trace artifacts in this repository. Publish only material you have reviewed for public sharing. Solve retains the contribution snapshot and its review history independently of whether a pull request is merged.

Source record: BCGT-000033.

- [Public source catalog](https://drewstone.github.io/problems/#/p/BCGT-000033)
