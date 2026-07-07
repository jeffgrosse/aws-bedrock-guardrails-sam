# aws-bedrock-guardrails-sam

A minimal, fully-parameterized AWS SAM template that deploys a single
[Amazon Bedrock Guardrail](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html)
with a realistic content policy (violence/hate/sexual/insults/misconduct
filters), a topic policy (a denied topic with worked examples), a sensitive
information policy (PII entities plus a custom regex), and a word policy
(managed profanity list plus custom terms). One file, one resource, no
click-ops - `sam build && sam deploy --guided` and you have a working
guardrail. See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) for the full
design writeup.

## Why this repo exists: the 200-character folded-scalar gotcha

While building the topic policy for a production guardrail, a `Definition`
field written as a nicely-wrapped multi-line YAML **folded scalar** (`>`)
failed guardrail creation with:

```
Resource handler returned message: "1 validation error detected: Value at
'topicPolicyConfig.topicsConfig.1.member.definition' failed to satisfy
constraint: Member must have length less than or equal to 200" ...
```

The source looked like four short lines. The problem: YAML folded scalars
join line breaks with spaces rather than deleting them, and Bedrock's
200-character limit on `Definition` applies to that **folded, single-line
result** - not to how short each wrapped line looks in your editor. The
error message names the API's camelCase field path
(`topicPolicyConfig.topicsConfig.1.member.definition`), gives no hint that
YAML formatting is involved, and doesn't mention folding, whitespace, or
character counts at all.

Full root cause, the wrong pattern, the correct pattern, and a rule of
thumb for any Bedrock Guardrail YAML: **[docs/FOLDED-SCALAR-GOTCHA.md](docs/FOLDED-SCALAR-GOTCHA.md)**.

## Prerequisites

- An AWS account with [Amazon Bedrock enabled](https://docs.aws.amazon.com/bedrock/latest/userguide/getting-started.html)
  in the region you plan to deploy to (Bedrock requires explicit model
  access to be granted per account/region even once the service itself is
  available - Guardrails creation itself doesn't require model access, but
  you'll want it to actually test the guardrail against `InvokeModel`).
- AWS SAM CLI installed (`sam --version`; see
  [AWS's install docs](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html)).
- AWS CLI configured with credentials that can create Bedrock resources.

## Region availability

Amazon Bedrock, including Guardrails, is available in 30+ AWS regions
across the Americas, Europe, and Asia Pacific as of this writing - there is
no Guardrails-specific region restriction narrower than wherever Bedrock
itself is enabled for your account. This template does **not** lock to a
single region (see [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md#region) for
why that's a deliberate difference from this repo's SAM siblings). Check
current availability and which regions have the specific filter types/PII
entities you need at the
[Bedrock endpoints and quotas page](https://docs.aws.amazon.com/general/latest/gr/bedrock.html)
before picking a region.

## Deploy

```bash
sam build
sam deploy --guided
```

`--guided` will prompt for:

| Parameter | Example | Notes |
|---|---|---|
| `GuardrailName` | `example-guardrail` | alphanumeric, `-`, `_` only; max 50 chars |
| `GuardrailDescription` | `Example guardrail...` | max 200 chars |
| `BlockedInputMessage` | `Sorry, I can't help with that request.` | shown to callers when input is blocked |
| `BlockedOutputMessage` | `Sorry, I can't provide that response.` | shown to callers when a model response is blocked |

Answers are saved to `samconfig.toml` (gitignored - see
`samconfig.toml.example` for the format if you'd rather write it by hand and
skip `--guided` on subsequent deploys).

The stack usually finishes in under a minute - a guardrail is lightweight
compared to resources like CloudFront distributions or ACM certificates.

## Verify

```bash
# Confirm the guardrail and its policies deployed as expected
aws bedrock get-guardrail --guardrail-identifier <GuardrailId-from-stack-outputs>

# Test invoke: apply the guardrail directly against sample text
aws bedrock-runtime apply-guardrail \
  --guardrail-identifier <GuardrailId> \
  --guardrail-version <GuardrailVersion-from-stack-outputs> \
  --source INPUT \
  --content '[{"text": {"text": "Should I put my savings into individual stocks?"}}]'
```

The second command should come back with `action: GUARDRAIL_INTERVENED` and
the `FinancialAdvice` topic listed as the trigger, since that phrase matches
the denied-topic example in `template.yaml`. Try it again with unrelated
text (e.g. `"What's a good recipe for banana bread?"`) and expect
`action: NONE`.

`GuardrailId` and `GuardrailVersion` are printed as stack outputs after
`sam deploy` completes, or retrievable anytime with:

```bash
aws cloudformation describe-stacks --stack-name aws-bedrock-guardrails-sam \
  --query "Stacks[0].Outputs"
```

## Cost estimate

Guardrails pricing is per "text unit" (up to 1,000 characters, prorated for
partial units), billed per policy category evaluated:

| Policy | Price per 1,000 text units |
|---|---|
| Content filters | $0.15 |
| Denied topics | $0.15 |
| Sensitive information filters | $0.10 |
| Word filters | Free |

A short test call (a sentence or two, well under 1,000 characters) against
all three billed policies in this template costs a small fraction of a
cent. Realistic testing during development - dozens to low hundreds of
calls - runs to pennies, not dollars. See
[Bedrock pricing](https://aws.amazon.com/bedrock/pricing/) for current
rates and the sensitive-information/contextual-grounding tiers this
template doesn't use.

## Cleanup

```bash
sam delete
```

Removes the guardrail and its version. No other resources are created by
this template, so there's nothing else to empty or detach first.

## Security / repo hygiene

No secrets, ARNs, or account IDs are committed. `samconfig.toml` (which
would hold your real stack name and region choice) is gitignored -
`samconfig.toml.example` shows the format with placeholder values. The
sample policies (denied financial-advice topic, generic PII types, generic
profanity/competitor-name blocks) are intentionally generic reference
content, not tied to any specific product or application.

## Related

- [aws-sam-static-website](https://github.com/jeffgrosse/aws-sam-static-website) -
  same design philosophy (clarity over cleverness, fully parameterized,
  one-command deploy) applied to a static site behind CloudFront.
- [aws-serverless-domain-redirect](https://github.com/jeffgrosse/aws-serverless-domain-redirect) -
  same philosophy applied to a CloudFront Function domain redirect.
