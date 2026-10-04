---
layout: post
title: "I SIGKILLed vLLM on an NVIDIA L40 under load – it served again after 42 seconds"
date: 2026-10-04 12:00:00 +0100
---

When a vLLM server process dies under load, what do its clients see, and how
long is it gone? On 2 October I killed one five times while it served steady
traffic, and recorded what happened to every request. Traffic kept coming
while it was down, and no failed request was retried. The server ran
Qwen/Qwen2.5-7B-Instruct on a single NVIDIA L40 GPU, inside a Docker container set to restart whenever its process fails.
Each kill was a SIGKILL (signal 9, which a process can neither catch nor clean
up after), sent from the host to the vLLM server process.

This test is a single-machine variant of a larger experiment: one
container, with no Kubernetes and no second replica to take
over. Docker restarted the same container in place, which keeps its files,
vLLM's compile cache among them. Kubernetes, after a crash, restarts a
container "with a clean state", losing files written outside a mounted volume
([Kubernetes documentation](https://kubernetes.io/docs/concepts/storage/volumes/));
I did not measure that case. The main experiment, which removes one of two
replicas behind Kubernetes, needs two GPUs and has not run yet. Requests arrived at 3.12 a second, 65% of the capacity the
[16 September calibration](https://percentes.ai/writing/2026/calibrating-one-l40-configuration/)
measured under the same vLLM, model and GPU settings; each asked for a 256-token answer.

## What the clients saw

A kill caught 111 requests partway through, across the five runs. All 111
failed the same way. The server had already sent HTTP status 200,
and 106 of them had received part of their answer; then the stream ended
before the usage chunk, carrying the token count, that every completed
request received. A client that checks only the status code
would record a success with a cut-off or empty answer. Every one of these streams ended
5 to 10 ms after the estimated moment of the kill.

While the server was down, 684 requests were sent, and each failed to connect
within 2 ms. None waited for the 30-second client timeout. Six more, sent after
the server logged `Application startup complete`, were answered. The outage
ends when a probe, which asks the server for one token every half second, gets
its first answer, 0.1 to 0.4 s after that log line.

Every one of the 8,671 requests sent from the probe's answer to the end of the
recovery phase completed within the latency target (first token within 1 s,
full answer within 14 s). The median time to a full answer
was 6.64 s, against 6.65 s before the kill.

{% include fig-pk-timeline.html %}

## Where the 42 seconds went

From the kill to the probe's first answer took 41.7 to 43.7 s across the five
runs, with a median of 42.2 s. Docker's timestamps on vLLM's log lines divide
that time into stages:

- 0.6 to 0.8 s until Docker restarted the container, almost all of it the old
  process exiting;
- 11.7 to 12.9 s from the container's start to vLLM's first log line, its
  version banner;
- 10.4 to 11.2 s to the engine's first log line, of which the last 10.2 to
  11.0 s produced no log output;
- 7.6 to 8.7 s until the model was loaded, of which loading the weights took
  2.6 to 3.0 s;
- 6.6 to 6.8 s loading the cached compiled model, reserving GPU memory for the
  KV cache and capturing CUDA graphs;
- 2.9 to 3.0 s until the log line `Starting vLLM server`, and 0.4 to 0.8 s more
  until the probe's first answer.

So 22.7 to 24.8 s, more than half of every outage, passed before the engine's
first log line, and the log is silent for most of it. That line is the first from
vLLM's second process, `EngineCore`; seeing what either process did before it
needs a profiler inside the container, which this run did not have. Figure 2 sets each restart beside the cold start from
16 September.

{% include fig-pk-gantt.html %}

## The compile cache

vLLM compiles the model with torch.compile and
keeps the result in a cache directory. Here that directory sat in the
container's own writable layer, and it survived each of Docker's restarts.
This container's first start on 2 October, with no cache yet, spent 17.8 s in
torch.compile; each restart spent 0.2 s. That first start also downloaded the
model weights to the host's disk, so no restart downloaded them. The 16 September cold start
did both, and took 83 s from its version banner to `Starting vLLM server`;
each restart here took 28.1 to 29.5 s over the same span.

## Predictions and results

I wrote three predictions into the experiment's
[specification](https://github.com/percentes/percentes/blob/bce0628868ca6c4a231a5b398c169aaf586ac8a8/SPEC.md) and
[pushed them to GitHub](https://github.com/percentes/percentes/commit/0abffa235291439c3e53ada51459ea22b5bcee87) the evening
before the runs. Each one names the result that would prove it wrong, and the
thresholds come from the 16 September cold start.

- **C1, bring-up.** A restart gets from the version banner to
  `Starting vLLM server` in under 61.3 s: the cold start's 83.0 s less its
  21.7 s download. It held: 28.1 to 29.5 s.
- **C2, failures.** Every request in flight at the kill fails, and no request sent
  during the outage hangs until the timeout. Nothing hung, so the second half held.
  The first half has no verdict, though all 111 failed. The kill time is known
  only to within 12 to 14 ms, since it is read on the host's clock and
  converted to the client's by an offset measured over SSH.
  All 111 ended inside that margin, and the rule written beforehand counts
  such requests for neither side.
- **C3, the compile cache.** Each restart reuses the cache: vLLM reports under
  5.0 s of compilation and under 17.0 s on its `init engine` line, while CUDA graph
  capture, which the cache does not cover, still takes at least 4 s. It held:
  0.18 to 0.20 s, 8.0 to 8.3 s and 5 s.

C1's bound is the cold start without its download, so it checked only that a
restart is no slower than that. C3 tests the cache itself: the first
start, with no cache, would have failed it.

## How it was measured

The load generator is my open-source tool
[Percentes](https://github.com/percentes/percentes), written in Go to measure
how inference servers fail. It schedules every request before the run starts and times each
answer from its scheduled send time, so time the server spends stalled counts
against the server. Each run lasted 17 minutes: one minute of warm-up, five of
steady load, then the kill, ten minutes of recovery and one of cool-down. A dry
kill with no load came first, so every measured run was the container's second
restart or later. The kill is a short
[shell script](https://github.com/percentes/percentes/blob/bce0628868ca6c4a231a5b398c169aaf586ac8a8/internal/orchestrator/ssh.go#L44)
that checks the target process belongs to the vLLM container, reads the host
clock, sends the signal and reads the clock again. The vLLM, model and GPU settings (vLLM
0.29.0, the model revision, the GPU driver) matched the 16 September
calibration, and the client passed its own timing checks on all five runs: its
worst 99th-percentile send delay was 17 microseconds.

---

## Appendix: the data and the commands

**Stages of each restart, in seconds.** The stages run back to back, so each
column sums to that run's outage.

| stage | run 1 | run 2 | run 3 | run 4 | run 5 |
|---|---|---|---|---|---|
| kill to container start | 0.659 | 0.685 | 0.628 | 0.738 | 0.802 |
| container start to version banner | 11.850 | 11.653 | 12.692 | 12.618 | 12.892 |
| banner to engine init | 10.459 | 10.386 | 11.239 | 10.965 | 11.152 |
| engine init to model loaded | 8.676 | 8.731 | 8.696 | 7.612 | 7.697 |
| model loaded to `torch.compile took` line | 0.903 | 0.882 | 0.812 | 0.822 | 0.873 |
| compile end to graph capture end | 5.856 | 5.756 | 5.846 | 5.732 | 5.928 |
| capture end to `Starting vLLM server` | 2.875 | 2.914 | 2.957 | 2.923 | 3.028 |
| `Starting vLLM server` to the probe's answer | 0.407 | 0.682 | 0.825 | 0.780 | 0.801 |
| outage | 41.685 | 41.689 | 43.695 | 42.190 | 43.173 |

The mean outage is 42.5 s, with a 95% confidence interval of 41.4 to 43.6 s
that assumes the five runs are independent and normally distributed; they were
successive restarts of one container.

**Requests the kill touched, per run.**

| run | in flight at the kill (all cut off) | sent during the outage | failed to connect | answered | timed out |
|---|---|---|---|---|---|
| 1 | 15 | 144 | 142 | 2 | 0 |
| 2 | 25 | 144 | 143 | 1 | 0 |
| 3 | 19 | 124 | 123 | 1 | 0 |
| 4 | 29 | 136 | 135 | 1 | 0 |
| 5 | 23 | 142 | 141 | 1 | 0 |

**vLLM's own timings, cold start against restart, in seconds.** The cold
column is the 16 September start; the restart column lists the five runs.

| figure | log line | cold | restarts |
|---|---|---|---|
| weight download | `Time spent downloading weights` | 21.73 | none |
| torch.compile | `torch.compile took` | 17.12 | 0.19, 0.18, 0.20, 0.18, 0.19 |
| graph capture | `Graph capturing finished` | 5 | 5, 5, 5, 5, 5 |
| init engine | `init engine (profile, create kv cache, warmup model) took` | 29.24 | 8.28, 8.17, 8.10, 7.98, 8.33 |

**The predictions as registered.** Quoted from section 7 of the
specification. In them, § marks one of its sections, API is application
programming interface, "fire" is the kill and "replica-ready" is the probe's
first answer. An "indeterminate" request (§1) ended so close to the kill that
the clocks cannot say which came first, and a "censored" one had no result
when the 30 s timeout expired. Dynamo is the torch.compile step the start log
calls its bytecode transform. Line numbers refer to the
[16 September start log](https://percentes.ai/assets/data/calibration-16Sep2026/vllm-startup-16Sep2026.log).

> **C1:** in every valid run, the API-server bring-up (`log_bringup`, the version banner to the server-start line in the restart's log) is under 61.3 s. The cold banner (line 3, 19:22:27) to the server-start line (line 57, 19:23:50) took 83.0 s, of which 21.7 s was the weight download (line 21), which does not recur on a restart; the expected torch.compile cache saving is left out of the bound. Refuted by one valid run at 61.3 s or more, or by either line absent from the restart's log.

> **C2:** in every valid run, every in-flight request that is not indeterminate (§1) ends errored, under any §3 error class, with the class split published, and no request scheduled from the fire to replica-ready ends censored. A refused connection fails at once (class `connect`); a dropped connection runs to the 30 s deadline and ends censored. Refuted by one determinate in-flight request ending completed or censored, or by one censored request scheduled in the outage. Indeterminate requests neither confirm nor refute.

> **C3:** in every valid run, the restart reuses the torch.compile cache: the compilation figure of the init-engine line (`init_engine_compilation_s`) is under 5.0 s, the init-engine figure (`init_engine_s`) is under 17.0 s, and the logged CUDA-graph capture (`graph_capture_s`) is at least 4 s. Cold, torch.compile took 17.12 s (line 40); the cache removes the Dynamo transform (5.82 s, line 36) and the graph compile (7.50 s, line 37), leaving 3.80 s that includes the cache write itself (lines 38 and 39), and the bound adds 1.2 s. Init engine took 29.24 s cold (line 51); less the 13.32 s the cache removes, that is 15.92 s, rounded up to 17.0 s. Capture is not cached; it read 5 s cold (line 48), printed at 1 s resolution. Refuted by one valid run at 5.0 s or more, at 17.0 s or more, or with capture under 4 s, or by an absent init-engine or graph-capture line.

In C3, 15.92 s rounded up is 16 s; the registered bound is 17.0 s.

**The summary sentence.** Filled into the template the specification fixed
before these runs:

> Under a SIGKILL of the only vLLM replica at 65 percent of measured single-replica capacity, 100% of in-flight requests failed and 0% timed out at 30 s; the replica served again 42.190 s after the kill (median of 5 valid runs, range 41.685 to 43.695 s), every request scheduled in the outage ended errored (684) or completed (6), and the outage decomposed as kill to container start (0.738 s), container start to version banner (12.618 s), banner to engine init (10.965 s), engine init to model loaded (7.612 s), model loaded to the `torch.compile took` line (0.822 s), compile end to capture end (5.732 s), capture end to `Starting vLLM server` (2.923 s) and `Starting vLLM server` to first served inference (0.780 s) in run 4, the median run, with container start to version banner dominating; recovery to the pre-fault baseline took 41 s (median; range 41 to 43 s). The methodology and the raw per-run data are at this page, with the harness that reproduces them.

Recovery there is the instrument's own detector: the start of the first
10-second window in which at least 90% of requests met the latency target and
kept doing so for 30 s. It reads 41 s, just before the probe's first answer,
because it dates each window by its start.

**The run.** The instrument is at
[github.com/percentes/percentes](https://github.com/percentes/percentes); this
run was built from commit `bce0628868ca6c4a231a5b398c169aaf586ac8a8`, recorded
in every run report, and the configuration file
`configs/phase1-process-kill.yaml` at that commit has the SHA-256 hash
`9ebb9245d33d3f89fe0a21bdb8cfd432a3718f31fcf1eac7b6fb2e64776750b5`, also
recorded in each report. Each prompt was a unique prefix followed by the word
`tok` 512 times; the client did not record the model's input-token count per
request, and every completed request returned 256 tokens. The first run's load
started at 16:09:31 UTC and the last run's was scheduled to end at 17:35:08
UTC on 2 October 2026. The campaign command, reconstructed from the run
procedure and the settings recorded in `campaign.json`:

```
./percentes-campaign --config phase1-process-kill.yaml --target http://10.0.0.249:8000 --metrics http://10.0.0.249:8000/metrics --probe-direct http://10.0.0.249:8000 --ssh-target ubuntu@10.0.0.249 --ssh-identity $HOME/.ssh/pk_gpu --container vllm --out results/pk-20261002T1609Z
```

The dry kill ran the same command with `--dry-kill --out results/dry-kill` in
place of the last flag. The files:

- [campaign.json](https://percentes.ai/assets/data/process-kill-02Oct2026/campaign.json) and [campaign.txt](https://percentes.ai/assets/data/process-kill-02Oct2026/campaign.txt), the campaign report.
- [run-1.json](https://percentes.ai/assets/data/process-kill-02Oct2026/run-1.json), [run-2.json](https://percentes.ai/assets/data/process-kill-02Oct2026/run-2.json), [run-3.json](https://percentes.ai/assets/data/process-kill-02Oct2026/run-3.json), [run-4.json](https://percentes.ai/assets/data/process-kill-02Oct2026/run-4.json) and [run-5.json](https://percentes.ai/assets/data/process-kill-02Oct2026/run-5.json), every request's times, outcome and error class, and the kill record; [run-1.txt](https://percentes.ai/assets/data/process-kill-02Oct2026/run-1.txt), [run-2.txt](https://percentes.ai/assets/data/process-kill-02Oct2026/run-2.txt), [run-3.txt](https://percentes.ai/assets/data/process-kill-02Oct2026/run-3.txt), [run-4.txt](https://percentes.ai/assets/data/process-kill-02Oct2026/run-4.txt) and [run-5.txt](https://percentes.ai/assets/data/process-kill-02Oct2026/run-5.txt), the same reports in text.
- [run-1-server.log](https://percentes.ai/assets/data/process-kill-02Oct2026/run-1-server.log), [run-2-server.log](https://percentes.ai/assets/data/process-kill-02Oct2026/run-2-server.log), [run-3-server.log](https://percentes.ai/assets/data/process-kill-02Oct2026/run-3-server.log), [run-4-server.log](https://percentes.ai/assets/data/process-kill-02Oct2026/run-4-server.log) and [run-5-server.log](https://percentes.ai/assets/data/process-kill-02Oct2026/run-5-server.log), the container's log from 5 s before each kill, with Docker's timestamps.
- [run-1-fingerprint-before.txt](https://percentes.ai/assets/data/process-kill-02Oct2026/run-1-fingerprint-before.txt), [run-1-fingerprint-after.txt](https://percentes.ai/assets/data/process-kill-02Oct2026/run-1-fingerprint-after.txt), [run-2-fingerprint-before.txt](https://percentes.ai/assets/data/process-kill-02Oct2026/run-2-fingerprint-before.txt), [run-2-fingerprint-after.txt](https://percentes.ai/assets/data/process-kill-02Oct2026/run-2-fingerprint-after.txt), [run-3-fingerprint-before.txt](https://percentes.ai/assets/data/process-kill-02Oct2026/run-3-fingerprint-before.txt), [run-3-fingerprint-after.txt](https://percentes.ai/assets/data/process-kill-02Oct2026/run-3-fingerprint-after.txt), [run-4-fingerprint-before.txt](https://percentes.ai/assets/data/process-kill-02Oct2026/run-4-fingerprint-before.txt), [run-4-fingerprint-after.txt](https://percentes.ai/assets/data/process-kill-02Oct2026/run-4-fingerprint-after.txt), [run-5-fingerprint-before.txt](https://percentes.ai/assets/data/process-kill-02Oct2026/run-5-fingerprint-before.txt) and [run-5-fingerprint-after.txt](https://percentes.ai/assets/data/process-kill-02Oct2026/run-5-fingerprint-after.txt), the GPU and container read-backs before and after each run.
- [dry-kill.json](https://percentes.ai/assets/data/process-kill-02Oct2026/dry-kill.json) and [dry-kill-server.log](https://percentes.ai/assets/data/process-kill-02Oct2026/dry-kill-server.log), the dry kill.
- [restart-test.txt](https://percentes.ai/assets/data/process-kill-02Oct2026/restart-test.txt), [docker-version.txt](https://percentes.ai/assets/data/process-kill-02Oct2026/docker-version.txt), [gpu-fingerprint.txt](https://percentes.ai/assets/data/process-kill-02Oct2026/gpu-fingerprint.txt), [vllm-pins.txt](https://percentes.ai/assets/data/process-kill-02Oct2026/vllm-pins.txt), [vllm-final-state.txt](https://percentes.ai/assets/data/process-kill-02Oct2026/vllm-final-state.txt) and [vllm-full.log](https://percentes.ai/assets/data/process-kill-02Oct2026/vllm-full.log): the Docker restart test, the Docker version, the GPU's details at setup, the server's settings read back, the container's final state and its whole log from its first start.
