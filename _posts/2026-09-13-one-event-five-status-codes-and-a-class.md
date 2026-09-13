---
layout: post
title: "One event, five status codes and a class"
date: 2026-09-13 18:15:00 +0100
---

When an inference provider runs out of serving capacity, requests fail
while the caller is inside its published limits. Seven providers
document that single event under five HTTP status codes, 429, 498, 500,
503 and 529. Where capacity has been bought in advance it also appears
as a 5XX class, the 500 to 599 server-error range. At four of the
seven, the code that carries capacity exhaustion is also the code for a
different event, the caller breaching its own quota or rate limit.
DeepInfra and Google's Gemini Enterprise Agent Platform, formerly
Vertex AI, put both events, capacity exhaustion and the caller's
breach, on 429, and the Gemini API does the same on its Flex inference
tier. Fireworks AI uses 429 for the caller's breach
on serverless and for capacity exhaustion on dedicated deployments, and
OpenAI uses 429 for the caller's breach and, on Flex processing, for
capacity exhaustion.

Every quotation below was checked against the linked page on
13 September 2026.

| Provider | Capacity exhaustion | Caller's quota or rate breach |
|---|---|---|
| Amazon Bedrock | 503; 529 | 429 |
| OpenAI | 503; 429 on Flex processing | 429 |
| Anthropic | 529 | 429 |
| Groq | 498 on the flex tier; 503 for maintenance or overload | 429 |
| DeepInfra | 429 | 429 |
| Fireworks AI | 503 on serverless; 429 on dedicated | 429 on serverless; on dedicated its pages disagree (below) |
| Google | 503 on the Gemini API, with 429 as well on its Flex inference tier; 429 and 500 on the Gemini Enterprise Agent Platform (formerly Vertex AI); 5XX under Provisioned Throughput while usage is below the purchased amount | 429 |

## Provider by provider

**Amazon Bedrock** keeps capacity and quota on separate codes of one
troubleshooting page, and gives capacity two of them. A
[503](https://docs.aws.amazon.com/bedrock/latest/userguide/troubleshooting-api-error-codes.html#ts-service-unavailable)
is "The service is temporarily unable to handle the request.", raised
when the service is "experiencing high demand or temporary capacity
constraints", and the entry adds: "This is not related to your
account-level quotas or rate limits (which return 429
ThrottlingException)." A
[529](https://docs.aws.amazon.com/bedrock/latest/userguide/troubleshooting-api-error-codes.html#ts-overloaded-error),
which the page labels overloaded_error, is "The model is temporarily
unable to process the request because of high demand or insufficient
serving capacity." A
quota breach is a
[429](https://docs.aws.amazon.com/bedrock/latest/userguide/troubleshooting-api-error-codes.html#ts-throttling-exception):
"The request was denied due to exceeding the account quotas for Amazon
Bedrock."

**OpenAI** lists both in one
[error-code table](https://developers.openai.com/api/docs/guides/error-codes#api-errors):
"503 - Model temporarily overloaded", with the cause "The requested
model is temporarily overloaded.", and "429 - Rate limit reached for
requests". The same table has another 429, "429 - Slow down", with the
cause "Your request rate increased too quickly.", which the page says
"can occur even when your traffic is within its requests-per-minute and
tokens-per-minute limits". On
[Flex processing](https://developers.openai.com/api/docs/guides/flex-processing#resource-unavailable-errors),
OpenAI's lower-cost tier with slower responses, the capacity refusal is
a 429 as well: "Flex processing may sometimes lack sufficient resources
to handle your requests, resulting in a 429 Resource Unavailable error
code."

**Anthropic** answers capacity exhaustion with 529, as Amazon Bedrock
does, listed on its
[errors page](https://platform.claude.com/docs/en/api/errors#http-errors):
"529 - overloaded_error: The API is temporarily overloaded." The 429
beside it is the caller's, "Your organization has hit a rate limit", an
entry that also covers spend caps. Under the 529 entry the page adds a
third cause of a 429: "if your organization has a sharp increase in
usage, you might see 429 errors because of acceleration limits on the
API."

**Groq** documents three relevant codes across its
[client](https://console.groq.com/docs/errors#client-error-codes) and
[server](https://console.groq.com/docs/errors#server-error-codes) error
lists. The 429 is the caller's: "Too many requests were sent in a given
timeframe. Implement request throttling and respect rate limits." A 503
covers maintenance or overload: "The server is not ready to handle the request, often
due to maintenance or overload." Groq
[describes](https://console.groq.com/docs/flex-processing) its flex
tier as "a service tier optimized for high-throughput workloads that
prioritizes fast inference and can handle occasional request failures".
Capacity on that tier has a custom code, 498, on the
[client list](https://console.groq.com/docs/errors#client-error-codes):
"This is a custom status code we use and will
return in the event that the flex tier is at capacity and the request
won't be processed."

**DeepInfra** stays on 429 for both capacity exhaustion and the
caller's breach, and its
[rate-limits page](https://docs.deepinfra.com/account/rate-limits#rate-limit-errors)
reads:
"You may occasionally receive 429 errors when a model becomes very
busy, even if you’re under the limit. Auto-scaling will kick in
shortly."

**Fireworks AI** maps capacity exhaustion differently on each
deployment type.
The top of its
[serverless rate-limits page](https://docs.fireworks.ai/serverless/rate-limits)
names both codes, "When using Serverless, you may experience 429 Too
Many Requests or 503 Service Overloaded.", and ties the 429 to the
caller: "To avoid 429s, you need to stay below our adaptive rate
limits." The page's
[FAQ](https://docs.fireworks.ai/serverless/rate-limits#am-i-guaranteed-successful-responses-up-to-my-rate-limit)
puts capacity exhaustion on the 503:
"Staying within your rate limits does not guarantee that every request
succeeds. When a deployment is busy, your traffic can still be load
shed, and those responses are 503 Service Overloaded." On
[dedicated and on-demand deployments](https://docs.fireworks.ai/guides/inference-error-codes#dedicated-and-on-demand-deployments),
which the table groups as dedicated,
"there are no account-level rate limits", so a 429 there reports the
deployment's own capacity, described there as "a capacity signal, not
quota enforcement". The
[account quotas page](https://docs.fireworks.ai/guides/quotas_usage/account-quotas#account-wide-request-limits)
says otherwise about account-level limits: "The 6,000 RPM cap applies
account-wide" (requests per minute) and "is not a separate
serverless-only limit", volume above it "is rejected (for example HTTP
429)", and
[on those deployments](https://docs.fireworks.ai/guides/quotas_usage/account-quotas#on-demand-deployment-quotas)
"Requests still count toward account-wide request limits".

**Google** gives different answers on its two APIs. The
[Gemini API](https://ai.google.dev/gemini-api/docs/api-errors#api-error-codes),
the version reached with an API key at ai.google.dev, uses 503: "The
service is temporarily overloaded or down." Its
[troubleshooting page](https://ai.google.dev/gemini-api/docs/troubleshooting#retry-strategy)
lists 429 and 503 together as retryable, "such as a 429
RESOURCE_EXHAUSTED or 503 UNAVAILABLE". On the Gemini API's
[Flex inference](https://ai.google.dev/gemini-api/docs/flex-inference#error-codes)
tier, the lower-priority option, the page lists both codes under "When
Flex capacity is unavailable or the system is congested". The two
entries read "503 Service Unavailable: The system
is currently at capacity." and "429 Too Many Requests: Rate limits or
resource exhaustion." On the Gemini Enterprise Agent Platform side, the
[API errors reference](https://docs.cloud.google.com/gemini-enterprise-agent-platform/reference/models/api-errors#api-errors)
lists "API quota over the limit." and "Server overload due to shared
server capacity." among the causes under 429. It gives overload a
second code, 500: "Server error due to overload or dependency failure."
The platform's
[429 page](https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/deploy/error-code-429)
names acceleration limits too: "You may encounter 429 errors because
of acceleration limits if your project has a sharp increase in usage."
On
the pay-as-you-go quota framework the 429 message is "Resource
exhausted, please try again later.", returned when "the
number of your requests exceeds the capacity allocated to process
requests". A capacity shortfall stays on the same code: "If you don't
have a Provisioned Throughput subscription and resources aren't
available to your application, then an error code 429 is returned."
Buying Provisioned Throughput, Google's reserved-capacity subscription,
changes the code for the shortfall while usage stays under the
purchased amount: "errors that might otherwise be 429 are returned as
5XX and count toward the SLA error rate", the SLA being the
service-level agreement, while on the subscription's Single Zone
variant the same errors are "treated as 5XX but don't count". Above the
purchased amount, "the additional requests are processed on-demand as
pay-as-you-go".

## Comparing error rates across providers

An error count grouped by status code and compared across these
providers puts different events under the same label. At Amazon
Bedrock, Anthropic and Groq a 429 count holds only the caller's own
behaviour: quota or rate breaches, and at Anthropic a traffic ramp.
Elsewhere it also holds capacity refusals, at DeepInfra, on the Gemini
Enterprise Agent Platform, on Fireworks AI's dedicated deployments, on
OpenAI's Flex processing and on the Gemini API's Flex inference, and
on the Gemini Enterprise Agent Platform what it holds changes with what
the account has bought. An earlier essay,
[The failure your dashboard can't see](https://percentes.ai/writing/2026/the-failure-your-dashboard-cannot-see/),
treats what this does to published reliability numbers.

## Claims

1. Seven providers document capacity exhaustion under five HTTP status
   codes, 429, 498, 500, 503 and 529, and under a 5XX class where capacity
   has been bought in advance.
2. At four of the seven, DeepInfra, Google, Fireworks AI and OpenAI, the
   code that carries capacity exhaustion is also the code for the caller's
   own quota or rate breach.
3. Amazon's documentation contradicts itself on the status that accompanies
   ThrottlingException: 429 on three of its pages, 400 on a fourth.

Amazon's documentation disagrees with itself. The User Guide's
[troubleshooting page](https://docs.aws.amazon.com/bedrock/latest/userguide/troubleshooting-api-error-codes.html#ts-throttling-exception)
pairs Amazon's ThrottlingException with 429. The API Reference's
[CommonErrors page](https://docs.aws.amazon.com/bedrock/latest/APIReference/CommonErrors.html#CommonErrors-ThrottlingException)
pairs the same exception with 400 and describes it as "Your request
rate is too high. The AWS SDKs automatically retry requests that
receive this exception." That page calls itself a generic list, "Not
all services return all error types listed here.", and continues, "For
errors specific to an API action for this service, see the topic for
that API action." The pages for two such actions,
[InvokeModel](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_runtime_InvokeModel.html)
and
[Converse](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_runtime_Converse.html),
both pair ThrottlingException with 429.

Fireworks AI's pages disagree on dedicated deployments. The
[error-codes page](https://docs.fireworks.ai/guides/inference-error-codes#dedicated-and-on-demand-deployments)
says there are no account-level rate limits there; the
[account quotas page](https://docs.fireworks.ai/guides/quotas_usage/account-quotas#account-wide-request-limits)
says the request cap is account-wide and that requests on those
deployments count toward it, in the words quoted above. The table's
capacity column follows the error-codes page.

Every quotation here and the page it came from are in a
[CSV beside this essay](https://percentes.ai/assets/data/status-codes-sources.csv).
