---
layout: default
title: Methodology
permalink: /methodology/
---

# Methodology

The full specification lives with the instrument:
[SPEC.md](https://github.com/percentes/percentes/blob/main/SPEC.md) is
the pre-registered, authoritative document (v0.2), and
[ARCHITECTURE.md](https://github.com/percentes/percentes/blob/main/docs/ARCHITECTURE.md)
maps every package to the clause it serves. What follows is the short
version: the commitments every published number is bound by.

## Open-loop load

Requests dispatch on a schedule fixed before the run. A load generator
that waits for responses before sending more slows down exactly when the
system under test stalls, and the worst moments vanish from the data
(*coordinated omission*). Percentes never waits: latency is measured
from each request's **intended** send time, so client-side queueing and
provider-side stalls are both in the number.

## Three outcomes, nothing dropped

Every scheduled request terminates as exactly one of **completed**,
**errored**, or **censored** (no terminal event by the pinned timeout;
the observation is a lower bound, not a duration). Only completions enter
latency histograms, and they are labelled *conditional on completion*.
Errors are excluded from the latency histograms and reported as a failure
rate over all scheduled requests. The completion curve is the
**Aalen–Johansen cumulative incidence of completion**, computed over all
scheduled requests: the estimated probability that a scheduled request
has completed by time t. A request cut off at the timeout is *censored*:
its outcome is still unknown, so it stays at risk until its cutoff,
carrying a lower bound on its duration. An errored request is different. It can
never complete, so it enters as a *competing terminal event* that
permanently consumes probability mass. Treating errors as censored is
the standard competing-risks mistake: it assumes the errored requests
would have completed at the same rate as the survivors, which biases the
completion estimate upward.
When a window has no errors the curve is exactly one minus the
Kaplan–Meier survival curve, the standard estimator for data where some
observations are cut off before the outcome is known. A quantile the curve never crosses inside
the timeout is reported as *beyond the horizon*: quantiles are claimed
only inside the timeout, the curve carries every observed step, and the
fraction still outstanding is stated alongside it. In a window with
errors the curve can plateau below one.

## The client must pass its own gates

Every run carries run-failing self-checks, with the thresholds pinned in
the specification: send-skew against the intended schedule at p99 ≤ 5 ms
and max ≤ 50 ms, zero scheduled-but-never-dispatched requests, host CPU
at most 70% sustained over any 5-second window, and Go garbage-collection
pauses at p99 under 1 ms during the measurement windows. If any check
fails, the harness fails the run and nothing from a failed run is
published. The
four checks bound the send path. The same refusal applies to the
statistics: tail confidence intervals are computed from order statistics
only where the sample budget supports them, and refused otherwise. A
refused interval prints as `p99-CI omitted (sample budget insufficient, §7)`,
never as a blank, and never as a bare point estimate.

## Pinned parameters

Every gate, tolerance, and detector parameter carries a number pinned
in the specification and enforced at configuration load: a config that
weakens one refuses to load. Published pieces cite the exact instrument
commit and full configuration, so the procedure can be repeated with the
pinned environment; for the replica-loss study that is a two-GPU
Kubernetes cluster and the pinned serving stack. Measurement against
hosted provider endpoints gets its own pre-registered protocol before
any such data is collected. Against a live third-party endpoint a repeat
is a fresh measurement: the method and the refusal rules are what carry
over.
