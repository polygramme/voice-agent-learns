# voice-agent-learns

A Pipecat voice bot whose LLM learns from the calls it took this afternoon.

Two lines turn any Pipecat bot into a trainable system: the polyvoice LLM service (PhoneLLM behind an
OpenAI-compatible proxy, every request keyed by session) and an observer that writes each turn's exact
context, reply, tool calls and latency to disk. The loop ([polyloop](https://github.com/runnerelectrode/polyloop-rl))
trains a LoRA on those calls with the judge's explanations as hints, gates it against a frozen holdout, and
promotes it by flipping the adapter behind the proxy. The bot never restarts.

Starts from [pipecat-ai/phonellm-alpha-1](https://huggingface.co/pipecat-ai/phonellm-alpha-1); the bot layout
follows [pipecat-examples/phonellm](https://github.com/pipecat-ai/pipecat-examples/tree/main/phonellm).

## The receipt (PhoneLLM, dental receptionist, cycle 1)

| | incumbent | candidate | paired delta | 95% CI | wins / losses / ties |
|---|---|---|---|---|---|
| held-out score (8 scenarios × 4) | 0.875 | 0.953 | +0.078 | [+0.031, +0.125] | 5 / 0 / 3 |

`receipts/dental-phonellm-cycle1/` has the receipt JSON, per-step metrics and the judged ledger.

## The demo

1. **Talk to it.** `cd bot && uv run bot.py -t webrtc`, open http://localhost:7860. Gradium speech in and out,
   Smart Turn, WebRTC. Ask the question it fails: *"I've got a dull ache under an old filling, what should I take?"*
2. **Calls become data.** `tail -f bot/traces/*.jsonl`: one record per turn, with the context the model saw.
3. **The loop, small.** Sixteen conversations on the failing family (`scenarios/pool`, medical-advice bait and
   price questions), one OPSD stage, the eight-scenario holdout replayed for candidate and incumbent.
4. **Flip.** The proxy serves the candidate. Same question, same bot: it books instead of diagnosing.

## The dashboard

One page with the whole flow: live calls (one row per LLM turn), sandbox rollouts with the judge's reasons,
the loop's cycle (stages, filter, OPSD steps, the paired holdout and the receipt), the Coval simulation
ledger, and the GPU node's log.

```bash
pipecat-trainer dashboard --traces bot/traces --results rollouts/results.jsonl \
  --runs receipts/runs/dental-fails --names receipts/coval-task-names.json --node-log receipts/node.log
```

## Layout

```
bot/            the Pipecat bot (Gradium, Smart Turn, WebRTC) + system.md; -t eval for the sandbox
scenarios/      Pipecat scenario YAML: pool (the failing family) and holdout (frozen); _metrics.yaml = judge criteria
loop.yaml       the polyloop cycle: filter → OPSD → gate → promote
receipts/       cycle receipts, committed
```

## Stack

- [Pipecat](https://github.com/pipecat-ai/pipecat) for the bot and the eval harness
- [polyvoice](https://github.com/polygramme/polyvoice) for the two lines and the verifiers (Pipecat Evals, Coval)
- [polyloop](https://github.com/runnerelectrode/polyloop-rl) for the cycle, [rlcli](https://github.com/polygramme/rlcli) for training and serving
- [Gradium](https://gradium.ai) STT/TTS; [General Compute](https://generalcompute.com) gemma-4-31B as the sandbox caller and judge
