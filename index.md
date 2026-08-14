---
layout: default
---

# Reliability measurement for LLM inference

When your application depends on a hosted model API, that provider's
reliability under load is a production dependency. Percentes measures it
the hard way: open-loop load with coordinated-omission-correct timing,
every scheduled request accounted for as **completed, errored, or
censored**, Kaplan–Meier completion curves for the requests that never
finish, and a client that must prove *it* wasn't the bottleneck before
any number counts.

The instrument is open source and the methodology is pre-registered —
every published number cites the exact commit and configuration that
produced it, so you can re-run it rather than trust it.

- **[Writing](/writing/)** — measurement notes and teardowns, published as they happen
- **[Methodology](/methodology/)** — what the numbers mean and what would invalidate them
- **[The instrument](https://github.com/percentes/percentes)** — Go, Apache-2.0

<p class="meta">Percentes <span class="pron">(per-SEN-teez)</span>, from
<em>percentile</em>: the tail is where reliability lives. The measure,
not the metric.</p>
