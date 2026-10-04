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

Current status, plainly: the instrument is certified against a mock
serving stack. Two measurements on real hardware are published under
Writing: the 16 September 2026 single-replica calibration, and a
2 October 2026 campaign that killed that replica five times under load
and timed Docker's restart of it. No provider measurements are published.

Percentes is built and run by Varun Mahadkar. The instrument is open
source, and the methodology was pre-registered before any measurement
data was collected. A number is published only with the commit and
configuration that produced it.

- **[Writing](/writing/)**: measurement notes and teardowns, so far [The failure your dashboard can't see](/writing/2026/the-failure-your-dashboard-cannot-see/), [One event, five status codes and a class](/writing/2026/one-event-five-status-codes-and-a-class/), [What one vLLM replica on an L40 can carry and how it fails](/writing/2026/calibrating-one-l40-configuration/) and [I SIGKILLed vLLM on an NVIDIA L40 under load; it served again after 42 seconds](/writing/2026/i-sigkilled-vllm-on-an-l40-under-load-it-served-again-after-42-seconds/)
- **[Methodology](/methodology/)**: what the numbers mean and what would invalidate them
- **[The instrument](https://github.com/percentes/percentes)**: Go, Apache-2.0
