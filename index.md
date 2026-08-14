---
layout: default
---

# Reliability measurement for LLM inference

Hosted model APIs stall and time out under load, and the request that
never returns is the one your own metrics are worst at counting.

Percentes is built to measure exactly those. Load is dispatched on a
schedule fixed before the run, and latency is measured from each
request's intended send time rather than its actual one, so a stalling
provider cannot slow the load generator into hiding its own worst moments
(the *coordinated omission* correction). Every scheduled request is
accounted for as **completed**, **errored**, or **censored** — still
running when the pinned timeout expired, so known only to have taken *at
least* that long. Kaplan–Meier completion curves are computed over every
scheduled request, so the ones that never finish still count. And the
client must prove it wasn't the bottleneck before any number counts.

Percentes is built and run by Varun Mahadkar. The instrument is open
source, and I pre-registered the methodology before running anything:
every published number cites the exact instrument commit and the full
configuration that produced it, so the run can be repeated.

- **[Writing](/writing/)** — measurement notes and teardowns
- **[Methodology](/methodology/)** — what the numbers mean and what would invalidate them
- **[The instrument](https://github.com/percentes/percentes)** — Go, Apache-2.0
