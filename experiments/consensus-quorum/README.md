# Experiment: raising the quorum bar in the Consensus scenario

**Scenario:** Consensus (`scenarios/consensus.yaml`) — 1 leader, 19 followers, leader proposes
a value each round; followers independently accept/reject; a round *commits* once accepts /
total ≥ `quorum`, otherwise the leader retries with a new value, up to `rounds` attempts.

**Change:** `task.config.quorum` `0.667` → `0.9` (single line, see diff on this branch).

## Hypothesis

Consensus is basically an org's decision-governance question in miniature: how much buy-in do
you actually require before something counts as decided. Each follower votes accept about 70%
of the time on its own (`packages/nest-core/nest_core/scenarios_builtin/consensus.py:91`), so
the default 0.667 quorum sits right around the group's natural agreement rate — close enough
that I expected it to usually clear, maybe with one retry. My bet going in on 0.9: it would
push the bar past what a 70%-average population can deliver, so the group would either burn a
lot more rounds getting there, or not close a decision within the 5-round budget at all.

## Evidence

Ran both configurations with `seed: 42` (deterministic), same otherwise. Round-by-round
outcomes, read directly from message payloads in the trace (`traces/baseline.jsonl` and
`traces/quorum_0.9.jsonl`) since the dashboard metric didn't capture this — see Investigation:

| Round | quorum=0.667 (baseline) | quorum=0.9 |
|---|---|---|
| 1 | 12/19 → aborted | 12/19 → aborted |
| 2 | 11/19 → aborted | 11/19 → aborted |
| 3 | **13/19 → committed** | 13/19 → aborted |
| 4 | — | 10/19 → aborted |
| 5 | — | 15/19 → aborted |

- Baseline: consensus reached on round 3 of 5 (68.4% agreement cleared the 66.7% bar).
- quorum=0.9: **never committed** — best round (round 5) hit 15/19 (78.9%), three votes short
  of the 18/19 (94.7%) needed to clear the 90% bar — the group exhausted all 5 rounds with no
  decision.
- `message_count` roughly doubled (342 → 570) since the leader kept retrying.

## Investigation — the surprise

`nest report`'s headline `success_rate` metric was **1.0000 in both runs**, which on its own
would suggest the change had no effect. I first assumed this was something in how I'd set up
the run, so I checked the source rather than the docs. It isn't a configuration issue on my
end — it's a known, already-acknowledged limitation in the framework itself. In
`packages/nest-core/nest_core/metrics.py:54-66`, the function backing `success_rate` is
literally named `_delivery_rate` internally, with a comment from the maintainers:

> "formerly called `success_rate` — kept as alias for compat... a 100% delivery rate does NOT
> mean the protocol succeeded; it only means every message was delivered, even if every request
> was rejected."

They already fixed this once, for one scenario: `deal_rate` exists specifically to give
Marketplace a real success signal, separate from delivery. That fix was never extended to
Consensus, Voting, Auction, Supply chain, or Reputation — they still only expose the
misleadingly-named `success_rate`, which under the hood is delivery rate. I confirmed the
actual task outcome the only way currently available: reading the raw
`result:<round>:<committed|aborted>:<accepts>/<total>` payloads directly out of the trace, and
cross-checking `packages/nest-core/nest_core/scenarios_builtin/consensus.py:63`
(`reached = accepts / total >= self._quorum`) to confirm how the real pass/fail decision is made.

This is the same trap I've watched play out in real decision reviews: an activity metric
("messages went out," "the meeting happened") standing in for an outcome metric ("did we
actually decide anything"). Nobody's lying, the dashboard just isn't tracking the thing you
think it's tracking. It's a good reminder to check what a metric is actually measuring before
you trust it, especially the closer you get to something with real consequences riding on it.

**Validator output**, for a second, independent line of evidence: this scenario ships three
consensus validators (`validate_consensus_agreement`, `validate_consensus_validity`,
`validate_consensus_no_conflict` in `packages/nest-core/nest_core/validators.py`). Running them
against both traces:

```
baseline (quorum=0.667): consensus_agreement passed — checked 1 committed rounds
                          consensus_validity  passed — all 1 committed values were proposed
                          consensus_no_conflict passed — checked 1 rounds
quorum=0.9:               consensus_agreement passed — checked 0 committed rounds
                          consensus_validity  passed — all 0 committed values were proposed
                          consensus_no_conflict passed — checked 0 rounds
```

One thing worth flagging for the maintainers: `validate_consensus_agreement` hardcodes its
check against a fixed 2/3 threshold (`validators.py:606`), not the scenario's actual configured
`quorum`. It didn't bite us here since our modified run committed zero rounds, but if a
scenario used, say, `quorum: 0.5` and committed a round at 55% agreement, this validator would
report it as a *violation* even though the scenario correctly followed its own configured rule.
Not something I'm fixing as part of this exercise, just flagging it as a real gap I found while
reading the validator source.

## Tools and help used

I used Claude Code, but not as a one-off chatbot — I run my own multi-agent orchestration
system on top of it (workflow skills plus specialized review subagents I've built for real
engineering work). I ran this exercise through my `/spec` workflow to structure the exploration
into checkpoints instead of one long unstructured session, and dispatched one narrow-scope
"scout" subagent to read all six built-in scenario configs and report back what each one
simulates before I committed to picking one. From there I implemented directly — editing the
config, running the CLI, reading source, committing — rather than routing it through my `/build`
pipeline, which would spin up separate implementer and test-writer subagents plus a formal
review gate. A single config line and a short README didn't need that machinery, and reaching
for it would have been the overbuilding this exercise explicitly warns against. I deliberately
also skipped the heavier review agents that system has for production code (security review,
cross-model critique, architecture review) for the same reason. Every command (`nest
doctor/run/inspect/report`, the validator run, the source reads) executed live in my terminal; I
reviewed each result before it went in this writeup, and the `success_rate` / `deal_rate`
finding above came from me flagging the flat number as suspicious and asking to check the
source, not from an AI narrating a plausible-sounding explanation.

## What would you build next for NANDA Town?

Given `deal_rate` already exists as precedent for "give this scenario a real outcome metric
instead of relying on delivery rate," I'd extend the same pattern to Consensus: something like
`quorum_reached_rate` or `rounds_to_commit`, computed the same way `deal_rate` is for
Marketplace, so `nest report`'s headline number actually reflects whether the task succeeded.
Then I'd put a cost on top of it: every extra round costs something, so raising the quorum bar
isn't free, the same way demanding more sign-off in a real org slows things down even when
nobody involved is being unreasonable. That turns Consensus from a pass/fail toy into something
that models the actual tradeoff this experiment was testing for.
