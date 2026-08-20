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
would have completed at the same rate as the survivors, so it can only
overestimate completion, and the overestimate is greater than zero
whenever any completion occurs after an error.
When a window has no errors the curve is exactly one minus the
Kaplan–Meier survival curve, the standard estimator for data where some
observations are cut off before the outcome is known.

A quantile the curve never crosses inside the timeout is refused, and the
refusal names which of two cases holds. The decision rests on the
window's *ceiling*: the final completion incidence plus the share of
requests still outstanding at the timeout, which is the highest the curve
could ever reach. Where the ceiling reaches the quantile, the refusal
reports the quantile as *greater than* the timeout and prints the
ceiling. Where the ceiling falls short of the quantile, the refusal
reports it as *unattainable* and prints the final completion incidence
with the ceiling. In a window with errors the curve can plateau below
one.

## The baseline stops before the fault

The replica-loss study holds steady load on two serving replicas, takes
one of them out at a moment fixed in advance, and watches what the
survivor does. The last 30 seconds of the baseline, one full client
timeout before the fault fires, are a guard window. Load through it is
unchanged, but a
request dispatched there can still be unresolved when the fault lands,
and its outcome would be caused by the fault while counted in the
baseline. The guard window is reported on its own and feeds no
baseline-derived number, so the pre-fault figures come from the roughly
270 seconds before it.

## Two recovery baselines

Recovery is reported against two baselines, kept apart because they
answer different questions: the degraded plateau the surviving replica
settles into (the single-replica equilibrium), and the two-replica
performance from before the fault. The plateau is a measured operating
point inside one run, held there by the offered load and by the 30-second
timeout shedding work, so it is not a steady state in the queueing-theory
sense. Its window also ends at the same run's recovery to the pre-fault
baseline, so the plateau estimate is not independent of that detection.
Where a run has no estimable plateau, the report marks the equilibrium
baseline *not estimable* with the reason, and the time to equilibrium is
reported N/A. The run still appears in the published table. Under the
black-hole variant, which cuts the node running one replica off the
network for a pinned 120 seconds, the two-replica baseline is reached
when the partition expires and the same pod comes back; that time cannot
be shorter than the partition itself, so it is labelled partition heal
and carries no claim about recovery from node loss.

## The client must pass its own gates

Every run carries run-failing self-checks, with the thresholds pinned in
the specification: send-skew against the intended schedule at p99 ≤ 5 ms
and max ≤ 50 ms, zero scheduled-but-never-dispatched requests, host CPU
at most 70% sustained over any 5-second window, and Go garbage-collection
pauses at p99 under 1 ms during the measurement windows. If any of the
four fails, the harness fails the run and publishes nothing from it. The
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
