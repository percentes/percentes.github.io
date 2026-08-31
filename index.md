---
layout: default
---

# Reliability measurement for LLM inference

Served LLM inference fails under load in ways dashboards are worst at
counting: the request that never returns, the stream that dies
mid-answer, and the 200 with nothing in it.

Percentes is built to measure exactly those. Load is dispatched on a
schedule fixed before the run, and latency is measured from each
request's intended send time rather than its actual one, so a stalling
backend cannot slow the load generator into hiding its own worst moments
(the fix for *coordinated omission*). Every scheduled request is
accounted for as **completed**, **errored**, or **censored**: still
running when the pinned timeout expired, so known only to have taken *at
least* that long. Completion-incidence curves (Aalen–Johansen: errors
are competing terminal events, only timeouts are censored) are computed
over every scheduled request, so the ones that never finish still count.
And the client must pass four pinned self-checks before any number
counts.

Current status, plainly: the instrument runs against a mock serving stack,
with its acceptance-criteria re-run pending against the revised
specification; no provider or real-GPU measurements are published yet.

Percentes is built and run by Varun Mahadkar. The instrument is open
source, and the methodology was pre-registered before any measurement
data was collected. A number is published only with the commit and
configuration that produced it.

- **[Writing](/writing/)**: measurement notes and teardowns
- **[Methodology](/methodology/)**: what the numbers mean and what would invalidate them
- **[The instrument](https://github.com/percentes/percentes)**: Go, Apache-2.0
