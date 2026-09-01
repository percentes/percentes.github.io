---
layout: post
title: "The failure your dashboard can't see"
date: 2026-08-31 23:35:00 +0100
---

The first time I pointed a load generator at a hosted inference API, I
learned something about my own measurement before I learned anything
about the provider. The generator
was a throwaway script, deliberately naive; the instrument this piece is
about comes later. The run was on 26 August 2026, against a large
provider's free tier.

The provider goes unnamed on purpose. The measurement protocol this
project publishes under has preconditions that a run must meet before a
named provider result may appear, and a throwaway script meets none of
them. The numbers that follow stay because they are evidence about my
own client, not a measurement of the provider: every refusal in them was
self-inflicted.

The script sent 300 requests through six workers, each worker waiting for
its reply before sending the next, with no retries. That waiting is what
makes a client closed-loop, and it matters later. The run was over in 3.0
seconds: 28 requests completed, and 272 came back `429 Too Many
Requests`.

A refusal does not say which budget ran out. So the moment the run
finished, I sent one more request, a single tiny one, to read the
account's rate-limit headers. Call it the header check. It succeeded, and
the headers came back:

```
x-ratelimit-limit-requests: 7000
x-ratelimit-limit-tokens: 6000
x-ratelimit-remaining-requests: 6971
x-ratelimit-remaining-tokens: 53
x-ratelimit-reset-requests: 5m57.942s
x-ratelimit-reset-tokens: 59.47s
```

Two budgets are visible there, and the run barely touched the first.
The daily allowance stood at 6,971 of 7,000 requests remaining: 29 had
been counted, which is the 28 completions plus the header check itself,
and the 272 refusals never reached it. That budget refills continuously, one
request's worth at a time, and `x-ratelimit-reset-requests`
reports the time to replace what was spent, which makes the number
checkable: at 7,000 a day, one request is replaced every 12.3429 seconds,
and 29 of them take 357.94 seconds, matching the printed 5m57.942s.

The second budget was nearly gone. Only 53 of the minute's 6,000 tokens
remained, too few for another real request.

This tier also has a third limit, and it appears in no header: a
per-minute request ceiling. Its value is an inference from an earlier
session, on 22 August, when I sent single requests one at a time until
one was refused. The 31st failed, which fits a ceiling of 30; that
refusal, too, named no meter. The published limits table does not cover this model, and
exact limits are set per organization, so probing was the only way to
learn the ceiling. What that meter counts is also unpublished: nothing says
whether a refused attempt spends it. If only accepted requests count, the
28 completions plus the header check left this run one short of the
ceiling, and the per-minute limit refused nothing at all; if attempts
count, 300 landed on it in three seconds.

So the daily budget was barely touched, the minute's token budget ended
within one real request of empty, and whether the third limit was even in
play turns on counting rules the provider does not publish. Which limit
refused me, nothing in any response says, and the arithmetic cannot
settle it either.
The one 429 I have captured from this account, saved from the 22 August
probing session, carries the three request-budget fields and no token
field at all. The run itself recorded no response headers, so I cannot
say whether its refusals looked the same. Everything I know about my own
run, I know from requests that succeeded.

The rest of this piece argues for two measurement disciplines: count
every scheduled request, and send on a schedule fixed before the run,
with latency measured from each request's scheduled time, which is Gil
Tene's correction for coordinated omission, the measurement error a
waiting client commits by not sending while the server stalls.
Neither would have rescued this run, because the defect was not in the
measuring. My
tooling was ready to record my own quota breach as the provider's error
rate, and the fix for that was knowing my own account's limits before
sending a single request.

The failure here is attribution. A request did not succeed, and the response
would not name the limit that stopped it. Knowing my own limits fixes my
instance of the problem. It does not fix the general case: at more than
one provider a genuine capacity refusal arrives as the same status code
as a quota breach that was your own doing.

That is the small version. The large version is worse: the request returns
success, so there is no event to attribute at all.

## Three ways a request can fail, and only one of them looks like failure

When you call an inference API under real load, a request can end in more
states than your dashboard probably counts. One of them is fine: the
request **completes**, and you get your tokens. The failures are the
other three.

1. It can **error**: a 500, a connection reset, or a malformed stream,
   loud and counted everywhere.

2. It can be **censored**, in the statistical sense: your client gave up
   at its timeout and you genuinely do not know what happened. You know
   only that it took *at least* as long as your timeout.

3. And it can be **silently dropped**: HTTP status 200, a well-formed
   stream, a `[DONE]` (the marker that ends the response stream), and no
   usable content. Or content truncated far below the budget you asked
   for, with nothing in `finish_reason`, the field that names why
   generation stopped, to explain why. The status line says success, and
   your metrics agree, while your user is looking at an empty box.

I have not caught one. My own run of 300 requests was checked for empty
200s and found none, and of the drop's two shapes, the empty response
and the unexplained truncation, the instrument below can score only the
first. So take the evidence from OpenRouter's own documentation. OpenRouter
operates
[automatic billing protection](https://openrouter.ai/docs/guides/features/zero-completion-insurance)
for one half of this, applied to every account automatically and without
configuration: you are not charged when a response has "zero completion
tokens AND a blank/null finish reason". Their
[error documentation](https://openrouter.ai/docs/api_reference/errors-and-debugging#when-no-content-is-generated)
says "Occasionally, the model may not generate any content", and lists two
causes: "The model is warming up from a cold start" and "The system is
scaling up to handle more requests". A separate section of the same page,
on mid-stream errors, gives the mechanism that keeps a broken response
looking successful: once streaming has started, "The HTTP status remains 200
OK since headers were already sent".

What that covers is the empty case with a stop reason that explains
nothing, which is half the definition above. It does not reach output that arrives,
falls short of the budget and ends cleanly. OpenRouter's nearest category to
that is a mid-stream error after partial output, which is a visible failure
and counts against uptime. One of the two causes they name, a system
scaling up to handle more requests, is a response to load. For the
truncation half, I have no external evidence at all.

Three different things can leave you a success status with empty or
truncated output, and they must be separated before any of them is
counted. Output cut at the `max_tokens` you set is a legitimate cut:
your budget did that, and `finish_reason` says so. A policy refusal
with a `finish_reason` that names it is a product decision. The defect is the third case: empty or truncated
output under load with nothing in the stop reason to explain it.

Censored requests and silent drops are where measurement quietly
breaks. Censored requests fall out
of latency statistics by construction: a request that never returned has no
latency to contribute, so unless someone deliberately imputes a value, it is
absent from the percentile. And since censored requests are, by construction,
the slowest ones, deleting them removes only the worst observations and
can only flatter the percentile: as more requests time out, the reported
99th percentile (p99) describes a shrinking, faster subset, and it can hold steady or even
improve while the provider degrades. Silent drops are worse still: they arrive looking like
successes on both axes, so an uptime figure records a success, and a
truncated one also posts a fast, clean completion to a latency leaderboard. Either way, nothing counts
them as failures.

## Who actually measures this today

Several people already count failures.

**OpenRouter publishes a genuinely failure-inclusive uptime metric.**
[Their documentation](https://openrouter.ai/docs/guides/community/for-providers#10-uptime-monitoring-and-traffic-routing)
defines it for providers as "successful requests ÷ total requests
(excluding user errors)", and the list of what counts against you is
unusually specific. It takes in 401s, 402s,
404s and all server errors, and then two that neither of the other
measurements in this piece counts: "Mid-stream errors", and "Successful
requests with error finish reasons". The second of those treats a 200 as a
failure when its own stop reason says something went wrong.

Which is where the silent drop reappears. OpenRouter's billing protection
covers two cases: a response with "zero completion tokens AND a blank/null
finish reason", and a response with "an error finish reason". Only the
second of those appears on OpenRouter's list of errors that count against
uptime. A blank stop reason is not an error
stop reason, so the case they detect well enough not to bill is the case
their reliability figure does not count.

429s sit on [the other
list](https://openrouter.ai/docs/guides/community/for-providers#10-uptime-monitoring-and-traffic-routing),
annotated "Rate limiting (429) - tracked separately".

Excluding them is defensible. In the opening run the 429s were the caller's:
300 requests in 3.0 seconds into a tier that meters by the minute.
Many 429s will be like that. Whether most are, I have not counted. Counting those
against the provider would have measured me.

The difficulty is that one status code covers two unrelated events. A 429
because you exceeded your quota is your fault. A 429 because the provider
ran out of capacity while you were inside your limits is theirs.

Providers agree the second case is real and do not agree on how to report
it. Amazon Bedrock gives capacity its own code and says so: a 503 means
"high demand or temporary capacity constraints" and "is not related to your
account-level quotas or rate limits (which return 429 ThrottlingException)".
OpenAI sends its own saturation to 503, "The engine is currently overloaded,
please try again later". Anthropic sends it to 529. Groq uses a custom 498
for flex-tier capacity. DeepInfra keeps it on 429: "You may occasionally
receive 429 errors when a model becomes very busy, even if you’re under the
limit." Fireworks documents both codes on serverless, 429 for its adaptive
rate limits and 503 when "your traffic can still be load shed", while on
dedicated deployments a 429 means "your deployment’s processing capacity is
saturated", which Fireworks calls "a capacity signal, not quota
enforcement". Google splits by surface. The Gemini developer API returns
503, "The service is temporarily overloaded or down." Vertex, whose 429
page now sits under Gemini Enterprise Agent Platform, puts both meanings on
the one code: a pay-as-you-go quota breach answers "Resource exhausted,
please try again later.", and "If you don't have a Provisioned Throughput
subscription and resources aren't available to your application, then an
error code 429 is returned." Buy standard Provisioned Throughput and the shortfall
changes code: "errors that might otherwise be 429 are returned as 5XX and
count toward the SLA error rate". Vertex and Bedrock were read on
26 August 2026, the remaining pages on 24 August 2026.

So one event carries four different status codes across seven providers,
plus a 5XX class where capacity is bought. At DeepInfra and Vertex the
code is shared with the customer's own quota breach on the same endpoint;
at Fireworks the two meanings sit on the one code across its deployment
types. A 429
rate is not a comparable number until you know which provider produced it
and which of its meanings applied.

OpenRouter's guidance to the providers on its marketplace names the
incentive. They are told to "Return early 429s if under
load, rather than queueing requests", and told why: "any queueing on your
end will show up in your throughput metrics". Elsewhere, more plainly, they are
told to return rate-limit errors "so we can retry with another provider and
your metrics stay healthy".

429s are listed under "Errors that DON’T affect uptime". Recall the
fraction: successful requests over total requests. A 429 left in the
total with no success to match would drag the figure down, so for a
refusal to leave uptime untouched, it has to sit outside both halves.

Shedding is not costless elsewhere, though. Consistent rate limiting "can reduce
the volume of successful requests available for evaluation", which counts
against a provider in the tiers OpenRouter uses to route its tool-calling
traffic.

So where a provider does attribute a refusal on the wire, the aggregate
throws that away. OpenRouter excludes a 429 from uptime whether the provider
issued it because you overshot your quota or because it had run out of
capacity.

**[vLLM's benchmark](https://github.com/vllm-project/vllm/blob/main/vllm/benchmarks/serve.py)**,
from the open-source inference server of the same name, **is honest in a
different way.** Its percentiles are
computed over successful completions only, and the exclusion is enforced by
a guard in the code. It prints the successful and failed
request counts at the head of the same results block, and warns outright
when every request failed. The exclusion is disclosed. That is the minimum bar, and it clears it.

Its success test is where the boundary shows up again, drawn looser. On
the completions path a request counts as successful if any `choices`
event arrived, and the code's own comment notes the text may be empty, so
a stream of empty-text events passes and only a stream with no `choices`
event at all fails. On the chat-completions path even that test is
absent: any 200 stream that ends without an exception is marked
successful. Either way, a
stream that delivers a fifth of the requested budget, say 200 tokens of
a requested 1,000, and stops is a success with a healthy latency: a stream that ends
early finishes fast. OpenRouter declines to bill the
empty case and still bills the truncated one. None of these systems
catches the truncated case, and only some of them catch the empty one.

A newer class publishes per-provider reliability directly. LLM-Stats, an
independent evaluations lab funded by Y Combinator in Summer 2025, carries a
Reliability column on its [provider
rankings](https://llm-stats.com/leaderboards/provider-rankings): OpenAI 92.5%, Google 97.0%,
against a seven-day window when I looked on 26 August 2026, having read
91.5% and 96.1% the day before. The page does
not say what counts as a success. Whatever it counts, one percentage
cannot see five things: whether the endpoint holds under load, whether the
client's own queueing corrupted the latencies, what happened to requests
that timed out, whether a 200 carried usable content, and whose fault any
failure was.

One independent monitor already counts what the others discard.
[llmstatus.io](https://llmstatus.io/methodology) classifies every failed probe by type, `empty_response` among
them. It also counts rate limiting in the error rate, on stated
grounds: "If a provider is consistently returning 429, that is a real
service-availability issue from the customer perspective." What reaches its
provider pages is uptime and a p95 latency measured "across successful
probes only", so the classification shapes the numbers without appearing in
them, and a detector running every sixty seconds says nothing about
behaviour under load, all read on 26 August 2026.

**Artificial Analysis is the reference leaderboard, and it does not clear
the disclosure bar that the vLLM benchmark cleared above.** Their published
[performance methodology](https://artificialanalysis.ai/methodology/performance-benchmarking)
(v2.2.0, dated 2 March 2026, checked August 2026)
defines time-to-first-token (TTFT) as the time until the first token of the
response arrives, says nothing about requests where no token ever arrives, and
aggregates every metric as a median over a trailing window. Search the
rendered page for "error", "failure", "timeout", "retry", or "dropped" and
you will find none of them. No provider has a published error rate or
timeout rate there. The standard measurement is a single prompt at a
time, concurrency one, run eight times a day, with one
ten-parallel-request test daily.

Their published methodology does not say what happens to a request that
never returns, and their published numbers contain no failure metric of
any kind, so the published numbers cannot show the difference between a
provider that times out and drops and one that does neither. The TTFT
definition exists only for a request that produced a token, so the metric
is conditioned on survival by its own wording, and how failed requests
affect what is published is not stated.

One piece of this picture changed on 4 August 2026. Artificial Analysis launched an
[Endpoint Accuracy Index](https://artificialanalysis.ai/articles/endpoint-accuracy-index):
published per endpoint as dated snapshots, scoring each
provider's serving of a model against a self-hosted reference, and its
[methodology](https://artificialanalysis.ai/methodology/endpoint-accuracy-index)
lists "Context window limits, input truncation, empty responses" under "Why
some endpoints might fail" for one of its three evaluations. The Index is a public measurement
that touches this failure family, and evidence that the axis matters. It does not publish a
failure rate. Every scoring line on the methodology page is a mean accuracy,
and errored tasks appear only as a run-selection tie-break, never as a
published rate, so as written, an empty response lands as a non-match
inside the mean. They do
attach a free-text note to an endpoint showing "certain characteristics or
shortcomings that affected the result", which is commentary, with no
classified rate behind it; and there is no load axis or timeout semantics, and no
statement of what happens under stress. None of the following appears as
a whole word or phrase in the body of the Endpoint Accuracy Index
methodology page or its launch article, read on 23 and 24 August 2026:
concurrency, load, stress, throughput, queue, rate limit, timeout,
deadline, time limit. Its own methodology says results are "point-in-time snapshots
rather than live monitoring". The Index covers real ground, and it leaves
this: failure-classified rates under load, from a validity-gated client.
That is the gap this project aims at.

## Measured, but not published

Artificial Analysis also documents a
**[System Load Test](https://artificialanalysis.ai/methodology/system-load-test)**: a stepped
concurrency ladder from 1 to 64 and beyond, three minutes per step, running
until throughput plateaus. Among its four metrics is *response rate*, defined there as "The proportion
of queries sent during the benchmarking phase that received responses (at
least 1 output token)".

Somebody there has already written the harness
that pushes real concurrency and counts what came back. And they publish the
[results](https://artificialanalysis.ai/hardware-inference-stack/datacenter),
behind the AA-SLT Benchmarks toggle on the datacenter page: peak system
output throughput, peak output speed per query, cost per million input
and output tokens, and end-to-end latency against concurrency, by
accelerator and serving stack. Response rate is the metric that does not
appear in either of the page's two views, and neither do the words
failure, failed or error rate. Checked on the rendered page, 31 August
2026.

The pattern repeats in a second harness.
[AA-AgentPerf](https://artificialanalysis.ai/methodology/agentperf) runs simulated agents
that "maintain continuous in-flight requests", and reports time to first
token, output speed, throughput, the maximum concurrent agents each
service-level tier sustains, and power draw per accelerator. Not one of its
metrics counts a request that failed, read on 26 August 2026.

And "response rate" is a coarser instrument than it sounds: a response
needs only one output token to count, so a stream truncated after a
dozen tokens of the requested 1,000 counts the same as a complete
answer. It does not tell you who to blame.

The gap is that **none of the
measurements above publishes failure under load continuously, per
provider, with the failures classified**: whose fault was the 429, was
the timeout a censored observation or a clean error, and was that 200
actually a response. That is the public record as at August 2026, read
logged out.

## What I am going to do about it

I am building a measurement instrument that reports failures with the
same standing as latency. Today, against a mock OpenAI-compatible server
on a local Kubernetes cluster, it classifies every scheduled request as
completed, errored, or censored. As of this writing, the client records
any non-200 status, 429 included, as a generic error; a separately
labelled throttling class comes with the hosted protocol before any
provider number is published. The empty case it does catch: a stream
that reaches `[DONE]` with no content event is errored, under its own
error label. What it cannot catch is the truncated case, output that
arrives, falls short of the budget, and stops with nothing explaining why.
Telling that apart from a short answer that ended naturally needs a
classifier, and no such detector is specified or pinned, because
publishing a rate from an uncertified classifier would be fabrication.
Until one exists there is no truncation rate to publish. The instrument
sends on a fixed schedule, open-loop where the throwaway script in the
opening run was closed-loop, so a provider that slows down does not also slow the rate at
which it is measured, and its latencies are measured from each request's
scheduled send time, with the send-skew gate below bounding how far
actual sends may drift from that schedule. Its completion curves keep
every scheduled request in the denominator, so a timeout is recorded.
Neither discipline is new: wrk2, vegeta and k6 can all drive a constant
open-loop rate and report errors beside latency. What none of them, and
none of the measurements surveyed above, provide is the stream-level
failure classification and a published, gated, per-provider rate. And before it publishes a single
number, it must prove that the measuring client itself was not the
bottleneck: send-skew (the gap between a request's scheduled and actual
send time), undispatched requests, CPU, and garbage-collection pauses
all gated, with the run declared invalid otherwise. The thresholds are
pinned in the methodology, down to whether each gate binds on the single
worst request or on a percentile of them. Those gates cover the send path; receive-side delay
from kernel buffering, event coalescing, and the client runtime's own
scheduling of the read loop sits outside them, and the spec says so.

A 2026 preprint (Chandrasekar and
Kramberger,
*["Identifying and Mitigating Systemic Measurement Bias in
Production LLM Inference Benchmarks"](https://arxiv.org/abs/2605.24217)*)
documented the opposite failure
mode to the one above: widely used single-process, asyncio-driven
benchmarking clients bottleneck on Python's Global Interpreter Lock at
high concurrency and *inflate* the time-to-first-token they report,
making servers look worse than they are.

The paper describes "an industry-wide
anti-pattern" in which practitioners "misapply model server
micro-benchmarking utilities" for production-level evaluation, and vLLM
Bench is the first of the three it lists. It also measured it, in queries
per second (QPS): "At the
1,000 QPS threshold, vLLM Bench and Inference X saturated at 146 and 443
QPS, respectively." Pushed to 5,000 queries per second, both saw
"timeouts and excessive error rates, necessitating their exclusion from
the final results".

The instrument, Percentes, is open source, and the
[methodology](https://percentes.ai/methodology/) was public before the
instrument collected anything from a provider. A published run is
re-runnable by design: a repeat against a live endpoint measures a
different moment, so what carries over is the method and the rules for
refusing to publish an invalid run. So far the
instrument has been built and run against a mock server on a local
cluster, its acceptance criteria passing against the revised
specification on 1 September 2026;
no characterization of a hosted endpoint or a real GPU is published.
The first pre-registered publication is a replica-loss study of
self-hosted vLLM; provider measurements follow under a separate hosted
protocol, registered before the first measurement run against a provider.

The claim that carries this piece is the gap: none of the
measurements above publishes failure under load continuously, per
provider, with the failures classified. Beneath it sit three claims a
reader can check directly:

1. Artificial Analysis's [performance
   methodology](https://artificialanalysis.ai/methodology/performance-benchmarking)
   v2.2.0 (that page carries its own version history) contains no error,
   failure, or timeout metric, and its TTFT definition is silent on requests
   that never return a token.
2. OpenRouter [excludes 429s from
   uptime](https://openrouter.ai/docs/guides/community/for-providers#10-uptime-monitoring-and-traffic-routing), which is right for the ones the
   caller earned. Where capacity exhaustion arrives as a 429, which is how
   OpenRouter's own guidance tells providers to respond under load, the
   exclusion also throws away the refusals that were not the caller's
   fault. Nothing in the published figure separates the two.
3. The free tier I tested meters three ways: 7,000 requests a day, 6,000
   tokens a minute, and a per-minute request ceiling that probing puts at
   30. Only the first two appear in any response header, and the one 429
   I captured carries no token field. Immediately
   after my burst, 6,971 of 7,000 daily requests were unspent and 53 of
   6,000 tokens remained for that minute; the third meter is invisible,
   and whether refused attempts spend it is unpublished, so if only
   accepted requests count, this run's 29 never reached it. The daily
   quota had barely moved, the token budget was nearly empty, and nothing
   in a refusal says which limit applied. The first two limits are my own
   account's, read from response headers on 26 August 2026. The third
   I probed: single requests until one was refused, the 31st, which fits
   30 a minute though no refusal named the meter.
   The provider publishes a limits table that does not cover the model I
   ran, and states that exact limits are set per organization, so reading
   it would not have told me the ceiling either. My account is not
   checkable from outside, so check your own:
   [`cmd/naivesweep`](https://github.com/percentes/percentes/tree/main/cmd/naivesweep)
   takes an
   endpoint, a key and a model id, and any OpenAI-compatible provider will
   do.

If any of this is wrong, tell me.
