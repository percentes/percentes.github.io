---
layout: default
title: Methodology
permalink: /methodology/
---

# Methodology

The full specification lives with the instrument —
[SPEC.md](https://github.com/percentes/percentes/blob/main/SPEC.md) is
the pre-registered, authoritative document, and
[ARCHITECTURE.md](https://github.com/percentes/percentes/blob/main/docs/ARCHITECTURE.md)
maps every package to the clause it serves. What follows is the short
version: the commitments every published number is bound by.

## Open loop, or the stall hides itself

Requests dispatch on a schedule fixed before the run. A load generator
that waits for responses before sending more slows down exactly when the
system under test stalls — and the worst moments vanish from the data
(*coordinated omission*). Percentes never waits: latency is measured
from each request's **intended** send time, so client-side queueing and
provider-side stalls are both in the number.

## Three outcomes, nothing dropped

Every scheduled request terminates as exactly one of **completed**,
**errored**, or **censored** (no terminal event by the pinned timeout —
we know only that it took *at least* that long). Only completions enter
latency histograms, and they are labelled *conditional on completion*.
Failure rates are first-class results. Censored requests enter
**Kaplan–Meier completion curves** computed over all scheduled requests
— and a quantile the curve never crosses inside the timeout is reported
as "beyond the horizon", never extrapolated.

## The client must prove its own innocence

Every run carries run-failing self-checks: send-skew against the
intended schedule (p99 ≤ 5 ms), client CPU, and GC-pause bounds. If the
measuring machine cannot demonstrate it was not the bottleneck, the run
is declared invalid and no number from it is published. Statistics
behave the same way: tail confidence intervals are computed from order
statistics only where the sample budget supports them, and are refused
otherwise — a refused interval is published as exactly that.

## Pre-registered, pinned, reproducible

Every gate, tolerance, and detector parameter carries a number pinned
in the specification and enforced at configuration load — a config that
weakens one refuses to load. Published pieces cite the exact instrument
commit and full configuration, so any engineer can re-run the
measurement rather than take it on trust.
