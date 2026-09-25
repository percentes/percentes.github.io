---
layout: post
title: "Calibrating one L40 configuration"
date: 2026-09-25 12:00:00 +0100
---

The experiment Percentes runs takes one of two serving replicas away under
sustained load and measures what the survivor does. Each replica is vLLM, an
engine that loads a model and serves requests against it. The published
specification (SPEC) fixes the per-replica arrival rate lambda_r at "0.65 of
the measured single-replica capacity lambda_max" in section 1 (§1), and §10
measures lambda_max by walking a ladder of arrival rates against recorded pins
and a fixed load shape. On 16 September 2026 the procedure ran on one NVIDIA
L40 card serving one replica in a container started with Docker on the host,
reached at its private address, with no Kubernetes in the path.

The procedure ran twice, and on both runs (ramps) the highest passing rung was
4.8 requests a second. At 5.2, the first rung above it, 182 and 134 requests
missed the 14 s end-to-end deadline while the sampled waiting queue stayed at
or near zero and every first token arrived on time. At twice lambda_r, the
whole load a surviving replica inherits when its partner is lost, 41 of 1900
requests met both the 1000 ms first-token deadline and the 14 s end-to-end
deadline.

## What was pinned

The pins are the hardware and software settings SPEC §6 requires to be fixed
and published, and lambda_max belongs to all of them.

| Pin | Value | Source |
|---|---|---|
| GPU (graphics processing unit) | NVIDIA L40, driver 570.195.03, persistence enabled, power limit 300.00 W (watts) of a 300.00 W maximum | host read-back |
| CUDA (NVIDIA's GPU computing platform) inside the container | 12.9, with cuDNN (CUDA Deep Neural Network library) 9.20.0 and NCCL (NVIDIA Collective Communication Library) 2.29.7 | recorded pin, read back inside the container at bring-up |
| Serving stack | vLLM 0.29.0, image `vllm/vllm-openai@sha256:7ef5a35d1ef8ce2cf9d671dd91eec6e367c5849262e0362b4d3d4a26be0d87d2` | server read-back and recorded pin |
| Model | Qwen/Qwen2.5-7B-Instruct at revision `a09a35458c702b33eeacc393d103063234e8bc28`, bfloat16 (a 16-bit floating-point format), no quantization | server read-back |
| Sampling | the model's own generation defaults, applied by the server: repetition_penalty 1.05, temperature 0.7, top_k 20, top_p 0.8 | server startup log |
| Context | max_model_len 1024, the context length for prompt and output together | server read-back |
| Scheduler | max_num_seqs 256 (at most this many sequences per iteration), max_num_batched_tokens 2048 (at most this many tokens per iteration), chunked prefill on (a prompt's processing may be split across iterations) | server read-back |
| CUDA graphs, the §6 pin for CUDA-graph enablement | on, mode `FULL_AND_PIECEWISE`, compilation mode `VLLM_COMPILE`, optimization level 2 | server read-back |
| KV (key/value) cache, the engine's store of attention state for requests in flight | 17,179,869,184 bytes, 299,584 tokens, 18,724 blocks of 16 | server read-back |
| Prefix caching | off, read from `vllm:cache_config_info` as `enable_prefix_caching="False"`, as SPEC §6 requires | server read-back |
| KV-cache maximum concurrency | 292.56x for 1,024-token requests | server read-back |
| Kubernetes | none: this run had no Kubernetes, so the four settings SPEC §6 pins for it (version, container network interface (CNI), dataplane mode, kube-proxy mode) are recorded as none | configuration |
| Load shape | Poisson arrivals (independent, exponentially distributed gaps between requests); each prompt repeated `tok ` text after a unique per-request prefix, input-length parameter 512, token count as seen by the model not recorded; max_tokens 256 with ignore_eos, so the server does not stop at the model's end-of-sequence (EOS) token | configuration |
| Client | separate non-spot machine (dedicated, as SPEC §6 requires) of the cloud host's n1-cpu-medium type, zone CANADA-1, subnet 10.0.0.0/24 (a block of 256 addresses sharing their first 24 bits), posting to `/v1/chat/completions`, 30 s HTTP (the request protocol) timeout, zero retries | configuration and bring-up log |
| Path | TCP (the transport under HTTP) connect 0.474 to 2.254 ms over five probes | client measurement |

The GPU row was captured before the run. At bring-up the clocks were at the
driver default and unlocked, and the recorded policy string ends "not settable
in guest". The run did not record the GPU's clocks, temperature or power while
it ran, the host's CPU (central processing unit) model and core count, or the
server's log for the run window; the published server log covers startup only.

## The procedure

SPEC §10 starts at 2 requests per second (rps) and doubles the rate until a
step fails (the coarse ramp), then climbs from the last passing rate in steps
of 10 percent of it until a step fails again (the fine ramp); lambda_max is
the highest rate that passed. Each step runs 30 s to settle, which is
discarded, then 120 s measured, and it passes "iff, over its measured portion:
goodput is at least 99 percent, the mean of the vLLM waiting-queue gauge is at
most 1.0, and the §2 client-validity gate is clean." `iff` means "if and only
if". The queue mean counts only if at least 90 percent of the expected samples
landed inside the measured portion; otherwise the step is not judged and the
procedure stops. "The procedure runs twice; if the two lambda_max values differ
by more than 10 percent of the larger value, a third ramp decides by median;
when they agree, lambda_max is the lower of the two."

Goodput is the SPEC §3 quantity: the fraction of a window's scheduled requests
that complete within the service level objective (SLO), the pass condition a
single request has to meet. Time to first token (TTFT) is measured from
intended dispatch to the client's first observed nonempty content event, so it
includes any delay between the scheduled send and the actual one, and SPEC §4
pins the objective:

> A request meets SLO iff TTFT at most 1000 ms, end-to-end at most 14 s
> (1000 ms plus 256 tokens at a 20 tokens-per-second floor, rounded up to
> 14 s), and completion without error.

Goodput's denominator is every request scheduled in the window, so a request
still open when the pinned 30 s client timeout expired, with no terminal
event, counts against the fraction; such a request is called censored.

The client-validity gate checks the client's own timing and health against
fixed limits: send skew (how late each send left, against its schedule) at
most 5 ms at the 99th percentile (p99) and 50 ms at worst; zero
scheduled-but-never-dispatched requests; client CPU at most 70 percent over any
5 s window; and, because the client is written in the Go language,
garbage-collection (GC) pause p99 under 1 ms. Every step of both ramps, and the
single-replica run at twice lambda_r described below, cleared it. The worst
send-skew p99 anywhere was 16 microseconds against the 5 ms limit, the worst
5 s CPU window 6.2 percent against 70, and the worst GC pause p99 0.786 ms
against 1 ms. The waiting-queue gauge was sampled once a second, and every
step landed 120 of 120 expected samples with zero scrape errors.

## The ladder, twice

In the tables the rows are in run order, the `gate` column is the
client-validity gate, and the `queue` column is the mean of
`vllm:num_requests_waiting` over the measured portion.

**Ramp 1**

| rate_rps | seed | scheduled | completed | errored | censored | goodput | queue | gate | verdict |
|---|---|---|---|---|---|---|---|---|---|
| 2 | 42 | 259 | 259 | 0 | 0 | 1.0000 | 0.000 | pass | pass |
| 4 | 43 | 488 | 488 | 0 | 0 | 1.0000 | 0.000 | pass | pass |
| 8 | 44 | 884 | 164 | 0 | 720 | 0.0283 | 0.633 | pass | FAIL: goodput 0.0283 below 0.99 |
| 4.4 | 45 | 528 | 528 | 0 | 0 | 1.0000 | 0.000 | pass | pass |
| 4.8 | 46 | 519 | 519 | 0 | 0 | 1.0000 | 0.000 | pass | pass |
| 5.2 | 47 | 622 | 622 | 0 | 0 | 0.7074 | 0.008 | pass | FAIL: goodput 0.7074 below 0.99 |

**Ramp 2**

| rate_rps | seed | scheduled | completed | errored | censored | goodput | queue | gate | verdict |
|---|---|---|---|---|---|---|---|---|---|
| 2 | 1042 | 275 | 275 | 0 | 0 | 1.0000 | 0.000 | pass | pass |
| 4 | 1043 | 499 | 499 | 0 | 0 | 1.0000 | 0.000 | pass | pass |
| 8 | 1044 | 1008 | 148 | 0 | 860 | 0.0000 | 5.967 | pass | FAIL: goodput 0.0000 below 0.99; waiting-queue mean 5.967 above 1.0 |
| 4.4 | 1045 | 562 | 562 | 0 | 0 | 1.0000 | 0.000 | pass | pass |
| 4.8 | 1046 | 556 | 556 | 0 | 0 | 1.0000 | 0.000 | pass | pass |
| 5.2 | 1047 | 618 | 618 | 0 | 0 | 0.7832 | 0.000 | pass | FAIL: goodput 0.7832 below 0.99 |

Both ramps stopped at 5.2 and selected 4.8, so lambda_max is 4.8. The fine
ramp's steps are 10 percent of the last coarse pass, 4 rps, so each step is
0.4 rps and nothing between 4.8 and 5.2 was tested.

The `rate_rps` column is the parameter of the Poisson schedule, so the number
of arrivals in a 120 s window depends on the seed as well as the label: the 4.8
step drew 519 requests on ramp 1, 4.325 a second, and 556 on ramp 2, 4.633 a
second, while ramp 2's 4.4 step drew 562, 4.683 a second. Counted by actual
arrivals, the fastest passing window carried 4.683 rps and the slowest failing
window 5.150 rps.

The two 4.8 rps steps both passed with goodput 1.0000 and a queue mean of
0.000, at different latencies. Ramp 1's median end-to-end time was 9.937 s,
its 95th percentile (p95) 12.362 s and its slowest of 519 requests 13.034 s.
Ramp 2's median was 11.575 s, and its slowest of 556 finished at 13.990 s,
under 10 ms inside the 14 s bound.

## Where 5.2 failed

On both ramps the 5.2 step failed on end-to-end time alone. On ramp 1, all 622
requests scheduled in the measured portion completed without error before
their individual client deadlines. Goodput was 0.7074: 440 of the 622 met the
SLO and 182 were late. Over the 622 completions TTFT had a median of 209 ms
and a maximum of 797 ms, under the 1000 ms bound for every request. End-to-end
time had a median of 11.608 s, close to ramp 2's 11.575 s at its passing 4.8
step, a p95 of 17.449 s and a slowest request at 18.065 s against the 14 s
bound. From 4.8 to 5.2 on this ramp the median end-to-end time rose by
1.671 s while the median TTFT rose by 31 ms, and with no first token later
than 797 ms, every late request spent more than 13.2 s after its first token.
On ramp 2 the 5.2 step had goodput 0.7832, 484 of 618 on time and 134 late, a
TTFT maximum of 626 ms and an end-to-end p95 of 16.466 s.

The waiting gauge read zero at 119 of its 120 once-a-second samples on ramp 1
and at all 120 on ramp 2; vLLM documents it as "Number of requests waiting to
be processed." A request already in the running batch is not waiting, so those
zeros say nothing about how loaded the batch was. Inter-chunk latency, the gap
between successive nonempty content events of a stream, pooled across the
completed requests in the window, rose: on ramp 1 its median went from 27.4 ms at
4.8 rps to 30.1 ms at 5.2, and its p95 from 106 ms to 126 ms; on ramp 2, from
28.7 ms to 30.2 ms and from 115 ms to 124 ms. SPEC §3 keeps the name
inter-chunk latency until the client's content-event count has been checked
against the server-reported token count, because a content event is not
inherently one token. The ladder windows recorded 241 to 244 gaps per
completed request against a configured 256-token output budget, and the
published trace holds no server-reported token counts, so that check has not
been made.

The trace does not show where that time was spent: in the server, on the
network, in the client's own reading of the stream, or inside events that
carried more than one token. On the two 5.2
windows the client dispatched on time and sat nearly idle, with a maximum send
skew of 18 and 50 microseconds and a worst 5 s CPU window of 1.10 and 1.05
percent. These measurements do not bound receive-side delay: SPEC §2 states
that the client-validity gate "cannot detect" delay from kernel buffering, from the
server coalescing several tokens into one event, or from the scheduling of the
client's read loop. SPEC §2 defines two checks for the client's receive side,
and neither was run in this calibration.

## One replica under the survivor load

Before any characterization, meaning a published measurement of the
two-replica experiment, SPEC §5 requires "a one-off single-replica no-fault
calibration run per (model, config) at the identical offered load (2 lambda_r,
the post-fault survivor load of §10), prompt set, and timeout policy, over the
§1 warm-up and baseline durations". From the ladder's result:

```
lambda_max   = 4.8 rps
lambda_r     = 0.65 x 4.8  = 3.12 rps
offered rate = 2 x 3.12    = 6.24 rps
```

SPEC §10 states: "The post-fault survivor load of 2 times lambda_r operates
the survivor at 130 percent of measured single-replica capacity."

At 6.24 rps, 60 s warm-up and 300 s measured:

| rate_rps | seed | scheduled | completed | errored | censored | goodput | censored_rate | queue |
|---|---|---|---|---|---|---|---|---|
| 6.24 | 2042 | 1900 | 519 | 0 | 1381 | 0.0216 | 0.7268 | 0.037 |

The mean sampled waiting gauge was 0.037, from seven nonzero samples of 300
with a maximum of three. The timeout censored 1381 requests, and SPEC §5
treats the timeout as shedding excess work, so this run describes the offered
load under the pinned 30 s timeout policy.

The censored share is 0.7268, above the 5 percent at which SPEC §3's
conditional-percentile rule requires the completion-incidence curve beside any
percentile taken over the completed subset. The curve is the cumulative
fraction of all 1900 scheduled requests that had completed by a given time:

| By | Completed | Share of the 1900 scheduled |
|---|---|---|
| 10 s | 13 | 0.0068 |
| 14 s, the end-to-end SLO bound | 41 | 0.0216 |
| 20 s | 174 | 0.0916 |
| 25 s | 341 | 0.1795 |
| 30 s, the client timeout | 519 | 0.2732 |

The 519 completed requests all met the 1000 ms first-token bound, with TTFT
median 383 ms and maximum 809 ms, and their end-to-end median was 23.134 s.
All three figures describe the completed subset alone.

## What the number covers

The value 4.8 rps is the highest ladder label that passed on this container on
16 September. Near it the 14 s end-to-end bound decided every verdict, and the
queue limit never decided one on its own. The value holds only while the §6
pins hold, and the 14 s bound itself rests on one of them, max_tokens 256:
SPEC §10 permits recalibration "only if a §6 pin changes".

The Kubernetes row of the table is a §6 pin too, and it records none. SPEC §10
measures lambda_max on one replica inside the Phase 1 environment, the
project's real-GPU Kubernetes cluster, with the experiment's exact pins,
before any characterization run. The profile in the repository holds this
run's values, lambda_max 4.8, lambda_r 3.12 and an offered rate of 6.24 rps,
as the standalone container's capacity, and the rate the experiment runs at
comes from the same ladder run again inside the cluster.

---

## Appendix: the trace and the commands

The instrument is at
[github.com/percentes/percentes](https://github.com/percentes/percentes), and
this run was built from commit
`86e96ceb642b5449ec15830fc816d38ede1bad95`, recorded on the first line of the
trace. The run started at 19:42:05 UTC (Coordinated Universal Time) on 16
September 2026 and finished at 20:19:50. The trace carries the configuration
the run was launched with under `config` in `calibration.json`; its
`load.rate_rps` of 20, the value the profile carried on the day, and
`target.replicas` of 2 were not used: each step sets its own rate and seed and
drives one replica. Slowest completions are read from the completion-incidence
record, and the percentile figures from the histogram summaries. The Phase 1
profile, `configs/phase1.yaml`, was added to the repository after the pinned
commit. The command, run on the client
machine, with `ulimit -n 65536` raising the file-descriptor limit first:

```
ulimit -n 65536 && ./percentes-calibrate --config phase1.yaml --target http://10.0.0.122:8000 --metrics http://10.0.0.122:8000/metrics --gauge vllm:num_requests_waiting --out calibration-$(date -u +%Y%m%dT%H%MZ)
```

The server was started from the pinned image by digest:

```
sudo docker run -d --name vllm --gpus all --ipc=host -p 8000:8000 \
  -v "$HOME/.cache/huggingface:/root/.cache/huggingface" \
  --env VLLM_SERVER_DEV_MODE=1 \
  vllm/vllm-openai@sha256:7ef5a35d1ef8ce2cf9d671dd91eec6e367c5849262e0362b4d3d4a26be0d87d2 \
  Qwen/Qwen2.5-7B-Instruct \
  --revision a09a35458c702b33eeacc393d103063234e8bc28 \
  --served-model-name Qwen/Qwen2.5-7B-Instruct \
  --host 0.0.0.0 --port 8000 \
  --max-model-len 1024 \
  --max-num-seqs 256 \
  --kv-cache-memory-bytes 16G \
  --no-enable-prefix-caching \
  --optimization-level 2 \
  --cudagraph-metrics
```

Gaps longer than a second. Every ladder window from 4.4 rps up, both 8 rps
windows included, had at least one inter-chunk gap longer than a second,
between 1.102 s and 6.234 s; no window at 4 rps or below had one, the longest
there being 930 ms. The second-long gaps lay above the pooled 99.9th percentile,
which went from 249 to 258 ms at 4 rps to 263 to 321 ms at 4.4 to 5.2 rps, and
they also occurred in windows that passed. The fine steps, 4.4 to
5.2 rps, all ran after the 8 rps step on the same server launch, once the
client had drained that step and the 30 s settle had passed, and an aborted
first attempt had run against that server for four minutes before the trace
began, so the trace cannot separate a rate effect from an after-effect of the
8 rps step.

SPEC §10 requires that "The full trace (every step's rate, goodput, and
queue-gauge series) is published with the report." These are that trace and the
read-backs behind the pins table:

- [calibration.txt](https://percentes.ai/assets/data/calibration-16Sep2026/calibration.txt), the human-readable form of the tables above.
- [calibration.json](https://percentes.ai/assets/data/calibration-16Sep2026/calibration.json), which contains each step's queue-gauge series and client-validity values, with latency percentile summaries and completion-incidence data.
- [gpu-pins-16Sep2026.txt](https://percentes.ai/assets/data/calibration-16Sep2026/gpu-pins-16Sep2026.txt), the host read-backs, with the host's public address redacted.
- [vllm-pins-16Sep2026.txt](https://percentes.ai/assets/data/calibration-16Sep2026/vllm-pins-16Sep2026.txt), the server read-backs.
- [vllm-startup-16Sep2026.log](https://percentes.ai/assets/data/calibration-16Sep2026/vllm-startup-16Sep2026.log), the server's startup log, which records the sampling defaults it applied.
- [client-rtt-16Sep2026.txt](https://percentes.ai/assets/data/calibration-16Sep2026/client-rtt-16Sep2026.txt), the path probes.
