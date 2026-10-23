# DeepSeek-V4.1-Flash on 2× DGX Spark (GB10) with an x86 RTX 3090 offload host: progress report, October 2026

A local deployment of DeepSeek-V4.1-Flash-1017 (EXL3, about 2.75 bits per weight) that runs the target model across two NVIDIA DGX Spark / GB10 nodes and moves two of its parts to an ordinary x86 server: the speculative drafter now runs on a pair of NVLinked RTX 3090s, and the Engram n-gram tables are served from the server's RAM over RDMA. This page records what is in production, what was measured, what did not work and why, and what comes next. Every number is labelled.

**Stack overview:** [GitHub](https://github.com/ProprietaryLegal/DeepSeek-V4.1-Flash-2xDGX-Spark-x86-offload) · [Hugging Face](https://huggingface.co/ProprietaryLegal/DeepSeek-V4.1-Flash-2xDGX-Spark-x86-offload) · **Collection:** [All entries](https://huggingface.co/collections/ProprietaryLegal/deepseek-v41-flash-on-2x-dgx-spark-x86-offload-6adab604a517b426ded3916e)

> Published by [PLI Labs](https://proprietarylegal.ai/labs.html), the open research arm of [Proprietary Legal Intelligence](https://proprietarylegal.ai). PLI builds AI systems around a legal organization's own work product, combining retrieval, staged workflows, validation, and attorney review. Visit [proprietarylegal.ai](https://proprietarylegal.ai) to explore our work and discuss what your organization needs.

## Status

2026-10-23 — production: the two-card NVLink drafter with asynchronous rounds, remote Engram, and KV page packing at a 327,680-token per-request limit. The 524,288-token profile serves correctly but is held for a prefill slowdown. 1M-token context was not tested. Drafter training (self-distillation) is built; its training data is being generated and no speed gain from it has been measured yet.

## How to read the numbers

- **MEASURED**: a saved observation on this hardware, with its date and conditions.
- **ESTIMATED**: arithmetic or a model built on measurements.
- **UPSTREAM CLAIM**: a figure published by someone else that we have not reproduced.

Benchmark conditions unless stated: temperature 1, top_p 1, thinking on at high effort, cold prompts on an idle server, 1,500 generated tokens per request. "Decode tok/s" is generated tokens per second after the first token, per request; reasoning and answer tokens count alike. A "round" is one target verification step: the target checks the drafted tokens and emits one token plus every accepted draft. Throughput per request is R = (1 + E[A]) / C, where E[A] is accepted draft tokens per round (the extra target token is not counted in A) and C is the full round time.

## Hardware and topology

| part | what it is | role |
|---|---|---|
| spark-a, spark-b | two NVIDIA DGX Spark (GB10 Grace-Blackwell, 128 GB unified memory each) | the target model, tensor parallel over the two nodes' own direct 200 Gb RoCE link |
| x86 host | EPYC server, 512 GB RAM, four RTX 3090 24 GB, one ConnectX-5 Ex | Engram tables in RAM; the drafter on two NVLinked 3090s (325 W cap each); the two other 3090s for training and tests |
| links | one direct 40 Gb/s RDMA link from the x86 host to each Spark | the DAC cables are 40GBASE-CR4, so the 100 Gb card runs at 40 Gb/s. MEASURED: 39.2 Gb/s per link, 1.38 µs average 64-byte RDMA send |

The served target is the public [DeepSeek-V4.1-Flash-1017 EXL3 2.75 bpw legal-weighted build](https://huggingface.co/ProprietaryLegal/DeepSeek-V4.1-Flash-1017-EXL3-2.75bpw-LegalWeighted-DGX-Spark). It is a re-quantization of DeepSeek's own checkpoint with a legal-leaning bit allocation, not a weight fine-tune. Engine: vLLM in the upstream DGX Spark image (HeNryous patch stack), EXL3 MoE kernels, sparse MLA with an indexer, FP8 KV, at most 4 requests at once.

## Architecture

| component | where it runs | production role |
|---|---|---|
| target model, 40 layers | both GB10 nodes, tensor parallel 2 | verification, sampling and every output token |
| KV cache | both GB10 nodes | holds the memory freed by moving the drafter off the Sparks |
| Engram tables (layers 1 and 14; DeepSeek V4's hashed n-gram embedding memory) | x86 host RAM, about 189 GiB resident | each Spark reads rows over its own RDMA link; the local NVMe reader stays as an exact fallback |
| DSpark speculative drafter (proposes up to 5 tokens per round) | two NVLinked RTX 3090s, tensor parallel 2 | drafts only; the Sparks verify every token |
| async proxy | a separate torch-free CPU process on the engine host | runs each remote drafting round off the engine's thread, with host flags in shared memory, a 200 ms watchdog and bounded respawn |
| KV page packing | both GB10 nodes | upstream page-packing patches plus memory switches: more KV tokens in the same memory |

**Outputs are exact.** Drafts are checked by the target with the standard speculative-sampling acceptance rule, so the text the system produces has the target model's own distribution whatever the drafter proposes. A better drafter makes the model faster. It never changes what the model writes. MEASURED: with temperature 0 and thinking off, six fixed prompts gave identical outputs across every configuration tested.

## Measured results

### Production today (MEASURED 2026-10-23, after the final promotion; one request at a time)

| prompt length | time to first token | decode tok/s | round time, p50 | repeats |
|---|---|---|---|---|
| 124 tokens | 0.36 s | 49.7–53.1 | 37.2–37.4 ms | 3 |
| 32,000 tokens | 14.5 s | 58.7–66.2 | 39.8–40.4 ms | 2 |
| 150,000 tokens | 68.6–68.7 s | 53.0–54.9 | 39.5–39.7 ms | 2 |

Prompt reading is about 2,200 tokens per second up to 150K (32K in 14.5 s; 150K in 68.6 s). MEASURED 2026-10-22 on the expanded-KV configuration: 256K in 118.7 s and 300K in 140.3 s, about 2,140 tokens per second.

### Several requests at once (MEASURED)

| test | date | result |
|---|---|---|
| 3 × 32K at once, production | 2026-10-23 | each stream 30.0–38.3 tok/s; about 105 tok/s combined (sum of per-request decode rates); first tokens at 42.8–52.2 s because the prompts queue for prefill |
| 4 × 32K at once, two-card drafter + async, before packing | 2026-10-23 | about 119 tok/s combined while all four decode (31.0 / 31.6 / 32.7 / 23.7); 6,000 tokens in 116.0 s = 51.7 tok/s including prefill |
| 4 × 300K at once, expanded KV | 2026-10-22 | all four completed; first tokens at 620–640 s (1.2M tokens prefilled); 16.9–23.4 tok/s per stream. The original configuration could not fit this |

Combined throughput rises with the number of streams while each stream slows. The engine runs at most 4 requests at once.

### Context capacity (MEASURED, engine log)

| configuration | KV pool, tokens | per-request limit | full-length requests the pool can hold |
|---|---|---|---|
| original: drafter and Engram on the Sparks | 989,460 | 327,680 | about 3.0 |
| drafter moved to the 3090 host, freed memory spent on KV | 1,604,926 | 327,680 | 4.90 |
| plus KV page packing (production) | 2,502,600 | 327,680 | 7.64 |
| packing at the 524,288 limit (held) | 2,820,756 | 524,288 | 5.38 |

The pool is shared by every request, and the 4-request limit applies separately. A larger pool does not mean more full-length requests at once than the engine admits.

### 500K prompts (MEASURED 2026-10-23, 524,288 profile)

- Needle retrieval at 450K and 500K: 4 of 4 runs found all 6 needles. Prefix-cache reuse was exact at 250K and 500K.
- One 500K request: first token after 322 s (about 1,550 prompt tokens per second), then 48.2 tok/s.
- Two 500K requests at once: first tokens after 648 and 655 s, then 29.9 and 34.5 tok/s. Free memory on one Spark fell to about 3.1 GiB, so two concurrent ~500K prompts are the riskiest load.
- The drafter drafted every round at 500K.
- **Held:** on this profile every cold prompt reads about 35% slower than at 327,680. 150K took 93.4 s against 68.9 s, and 32K took 19.8 s against 14.6 s: a flat +0.163 ms per prompt token. Decode is unchanged. The cause is under investigation, so production stays at 327,680.

### What each step bought (MEASURED against the original configuration unless stated)

| step | effect |
|---|---|
| speculative drafting itself (original setup) | +31% (short) to +57% (32K) decode against speculation off (41–42 tok/s) |
| Engram tables in x86 RAM over RDMA | first token −5.4% at 32K and −5.7% at 150K; decode unchanged within noise; 839,040 looked-up rows per rank checked byte-identical; 0 fallbacks over the run |
| drafter moved to one RTX 3090, synchronous rounds | freed 3–5 GiB per Spark; but +7–9% per round at one stream, so not used on its own |
| freed memory spent on KV | KV +62% (989,460 → 1,604,926); 4 × 300K at once now fits |
| asynchronous rounds through a separate process | −1.0% to −4.0% per round against synchronous, first-token time at parity; a killed or frozen proxy is replaced in about 1 s and requests complete |
| drafter split over the NVLink pair, with async | −3.9% to −6.3% per round against synchronous one-card, which brings per-round time back to the original (37.25 / 40.41 / 39.67 ms at short / 32K / 150K against 37.2 / 40.2 / 39.2) |
| KV page packing at 327,680 | KV 1,604,926 → 2,502,600; round time −0.8% (short) and −1.2% (32K) against the configuration before; about 1 GiB more free memory per Spark |

## What did not work, and why

- **Synchronous remote drafting.** The remote drafter is faster than the local one (3.1 vs about 3.6 ms), yet rounds got 7–9% slower. Waiting synchronously serialized CPU work that the local path had overlapped. Asynchronous rounds fixed it.
- **Async through a thread inside the engine.** Decode improved, but time to first token rose 4–8% and some rounds hit the watchdog every run. Two suspected causes (the interpreter switch interval, eager CUDA module loading) were tested and ruled out. Moving the round into a separate process removed the cost.
- **A faster drafter does not make serving much faster.** The drafter is 2–4 ms of a 38–40 ms round, and async hides most of it. The two-card split cut the drafter's own time by about a third, but the round by only about 1–2.5%; faster native kernels by about 0.6% (ESTIMATED). The same arithmetic rules out a second, parallel drafter on the other two 3090s: the engine runs at most 4 requests, and one two-card drafter handles 4 in about 3.7 ms. Most of each round is the target model on the GB10s.
- **Official full-precision drafter weights.** Acceptance was the same: 1.817 against 1.818 accepted drafts per round. The limit is the distance between drafter and target at temperature 1, not precision. A strict numerical gate for this "source-faithful" native drafter also proved unpassable by any GPU implementation, including DeepSeek's own kernel. The native drafter stays opt-in, and a live A/B on the NVLink pair will judge it on accepted tokens and throughput.
- **A smarter prefill-ordering scheduler.** It looked like it would cut the first wait for one of four 300K requests from 620 s to about 156 s. Two independent checks against the real scheduler found that the whole gain came from holding other prefills back, which could stall them indefinitely. A reorder-only version is safe but gains nothing. Shelved.
- **Remote Engram lookahead (upstream).** It hides slow local-disk reads. Our remote lookup already takes about 3 ms, and the lookahead's own overhead is about 2.4 ms. Dropped.
- **A per-step agreement gate for a fast drafter trainer.** Any non-identical arithmetic is amplified by the drafter's 4-bit quantize-dequantize steps, so near-bitwise agreement is unattainable. It was replaced by a matched outcome pilot (below).
- **The 524,288 profile.** It works but is held for the 35% prefill slowdown described above.
- **The start-up warm-up request** produced 1 token instead of 32, because it was allowed to stop early. A bounded warm-up fix is written and tested on CPU, and is not yet applied.

## In progress

- **Drafter training (self-distillation).** The model's own outputs on 538 public prompts (court opinions, public filings and general text; no client material) train a low-rank adapter on the drafter, so it predicts the target's legal writing better. Generation of about 1M tokens is running. Next, a replay pass captures the target's hidden states and top-20 probabilities.
  - Then a matched pilot: reference trainer, fast trainer, and fast trainer with a different data order, on equal blocks, updates and tokens. It is judged on accepted drafts per round on held-out development families under a rule fixed before any data: a useful gain of at least 0.05 accepted per round with a 90% lower bound above zero, and the fast trainer within −0.03 of the reference.
  - Twelve new public test prompts are frozen for the final candidate only.
  - Today's drafter accepts about 1.82 drafts per round (MEASURED, 521 rounds). ESTIMATED payoff: +3% to +24% decode, central about +10%. An exact target-greedy oracle analysis on 529 rounds from 4 requests scored 3.151 against a 1.831 lower bound for the current drafter. That is a ceiling for an idealized drafter on a small sample, not a forecast.
  - Training cannot change outputs, only speed.
- **Native drafter live A/B** on the NVLink pair, judged by outcomes.
- **Target profiling on GB10.** Measuring where the 38–40 ms round goes, plus a cheap screen of draft depth.
- **Diagnosing the 524,288 prefill slowdown.** Per-chunk prefill cost against prefix length at 150K–500K.
- **Exact probabilistic proposals** (research, off by default). Drafts sampled from a proposal distribution with exact rejection sampling, as an alternative to greedy drafts.

## Next steps

1. Finish drafter-training data, replay and the matched pilot. Promote an adapted drafter only on measured acceptance and throughput gains, with its own identity and label (adapted, not source-faithful).
2. Run the native drafter A/B, then the profiling window and the depth screen.
3. Find and fix the 524,288 prefill slowdown before raising the production limit to about 500K.
4. Test selected upstream speed patches one at a time on GB10. Then test larger concurrency and clock settings only where measurements justify them.

## Upstream claims (not reproduced here)

- UPSTREAM CLAIM: an intermediate prefill replay skip gives +4.9% to +6.5% long-prompt prefill at 478K–949K.
- UPSTREAM CLAIM: background-prefill scheduling raises foreground decode next to one or two 60K background prefills from 31.3 / 22.5 to 57.4 / 56.5 tok/s.

## Component repositories

| component | GitHub | Hugging Face |
|---|---|---|
| Full stack overview | [source](https://github.com/ProprietaryLegal/DeepSeek-V4.1-Flash-2xDGX-Spark-x86-offload) | [entry](https://huggingface.co/ProprietaryLegal/DeepSeek-V4.1-Flash-2xDGX-Spark-x86-offload) |
| Async host-flag handoff | [source](https://github.com/ProprietaryLegal/dsv41-async-hostflag-handoff) | [entry](https://huggingface.co/ProprietaryLegal/dsv41-async-hostflag-handoff) |
| Remote RTX 3090 drafter | [source](https://github.com/ProprietaryLegal/dsv41-remote-dspark-drafter-3090) | [entry](https://huggingface.co/ProprietaryLegal/dsv41-remote-dspark-drafter-3090) |
| NVLink two-card drafter | [source](https://github.com/ProprietaryLegal/dsv41-dspark-drafter-tp2-nvlink) | [entry](https://huggingface.co/ProprietaryLegal/dsv41-dspark-drafter-tp2-nvlink) |
| Remote Engram over RDMA | [source](https://github.com/ProprietaryLegal/dsv41-remote-engram-rdma) | [entry](https://huggingface.co/ProprietaryLegal/dsv41-remote-engram-rdma) |
| RDMA transport | [source](https://github.com/ProprietaryLegal/ofl-rdma-transport) | [entry](https://huggingface.co/ProprietaryLegal/ofl-rdma-transport) |
| Native 1017 drafter kernels | [source](https://github.com/ProprietaryLegal/dsv41-native-1017-drafter-sm86) | [entry](https://huggingface.co/ProprietaryLegal/dsv41-native-1017-drafter-sm86) |
| KV page packing | [source](https://github.com/ProprietaryLegal/dsv41-kv-page-packing-1m) | [entry](https://huggingface.co/ProprietaryLegal/dsv41-kv-page-packing-1m) |
| Long-context tools | [source](https://github.com/ProprietaryLegal/dsv41-longctx-bench-tools) | [entry](https://huggingface.co/ProprietaryLegal/dsv41-longctx-bench-tools) |
| Exact acceptance ceiling | [source](https://github.com/ProprietaryLegal/dsv41-exact-acceptance-ceiling) | [entry](https://huggingface.co/ProprietaryLegal/dsv41-exact-acceptance-ceiling) |
| Findings | [source](https://github.com/ProprietaryLegal/dsv41-spark-offload-findings) | [entry](https://huggingface.co/ProprietaryLegal/dsv41-spark-offload-findings) |

Upstream forks: [Spark image and offload integration](https://github.com/ProprietaryLegal/deepseek-v41-1017-dgx-spark) · [two-node launcher](https://github.com/ProprietaryLegal/dgx-spark-deepseek-v41).

Some component pages were written before the promotions listed under Status above. Where they differ, this page is the newer record.

## Credits and licences

This page is Apache-2.0 documentation and contains no code, weights, prompts, captured tensors or generated text. Credits: DeepSeek for the [target and drafter](https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash-1017); HeNryous for the Spark image, patches and EXL3 build; sudoingX for the two-node recipe; turboderp for [exllamav3](https://github.com/turboderp-org/exllamav3); [vLLM](https://github.com/vllm-project/vllm) for the serving engine. Upstream software and model weights keep their own licences.

---

## About PLI Labs

“Your proprietary work product is your proprietary intelligence.” PLI Labs is the open research arm of Proprietary Legal Intelligence. We publish legal model prunes, serving recipes for owned hardware, and open-source tools so readers can inspect our methods and test our claims before hiring us.

Founded by a practicing trial lawyer, PLI builds AI systems around a legal organization's own work product: retrieval, staged workflows, validation, and attorney review. The approach grew from systems running on live matters in a litigation practice. Our consulting work supports deployment through approved frontier APIs, private cloud, or open-weight models on hardware you own. Explore the research or contact us about your organization's work.

[Website](https://proprietarylegal.ai) · [PLI Labs](https://proprietarylegal.ai/labs.html) · [Research](https://proprietarylegal.ai/research/index.html) · [GitHub](https://github.com/ProprietaryLegal) · [Hugging Face](https://huggingface.co/ProprietaryLegal) · [Contact](https://proprietarylegal.ai/contact.html)

Proprietary Legal Intelligence LLC is a technology consultancy, not a law firm, and does not provide legal services or legal advice.
