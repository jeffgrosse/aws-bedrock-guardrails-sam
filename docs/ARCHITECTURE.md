# Architecture

## Overview

```
                     ┌──────────────────────────────┐
  Your app / Lambda  │  AWS::Bedrock::Guardrail       │
  calling             │  ┌────────────────────────┐  │
  bedrock:InvokeModel │  │ ContentPolicyConfig      │  │  violence, hate, sexual,
  or                  │  │ (filters)                 │  │  insults, misconduct
  bedrock:ApplyGuardrail  ├────────────────────────┤  │
        │             │  │ TopicPolicyConfig         │  │  denied topics + examples
        │             │  ├────────────────────────┤  │
        │             │  │ SensitiveInformationPolicy│  │  PII entities + regex
        │             │  ├────────────────────────┤  │
        │             │  │ WordPolicyConfig          │  │  managed lists + custom
        │             │  └────────────────────────┘  │
        │             └───────────────┬──────────────┘
        │                             │ GuardrailId
        │                             ▼
        │             ┌──────────────────────────────┐
        └────────────▶│  AWS::Bedrock::GuardrailVersion│  numbered, immutable
                       │  (Version "1")                │  snapshot of the DRAFT
                       └──────────────────────────────┘
```

A guardrail is a single resource holding up to five independent policies -
content, topic, sensitive information, word, and contextual grounding. This
template configures the first four; contextual grounding
(`ContextualGroundingPolicyConfig`) is deliberately out of scope here since
it scores model output against reference source/RAG context supplied at
`ApplyGuardrail`/`InvokeModel` time, which doesn't fit a minimal,
no-external-dependencies reference template. Each configured policy is
optional and additive - you can ship just a content policy, just a topic
policy, or all four together, as this template does. Policies
are evaluated together on every `ApplyGuardrail` or guarded `InvokeModel`
call; if any policy blocks, the whole call is blocked.

## Why a separate GuardrailVersion resource

`AWS::Bedrock::Guardrail` alone only gives you a mutable `DRAFT` - editing
the guardrail's properties and redeploying changes `DRAFT` in place, which
means anything pinned to `DRAFT` picks up in-flight edits without warning.
`AWS::Bedrock::GuardrailVersion` snapshots the current `DRAFT` into an
immutable, numbered version (`"1"`, `"2"`, ...). Production callers should
reference a specific version, not `DRAFT`, so a template change doesn't
silently alter guardrail behavior for a caller that hasn't redeployed.

This template creates exactly one version (`"1"`) so the guardrail is
immediately usable end-to-end. If you evolve the policies later, add a
second `GuardrailVersion` resource (with a distinct logical ID) rather than
relying on `DRAFT` for anything beyond testing.

Both `ExampleGuardrail` and `ExampleGuardrailVersion` carry
`DeletionPolicy: Retain` and `UpdateReplacePolicy: Retain` for the same
reason: a caller pinned to a numbered version has no way to know that
version disappeared until their next `ApplyGuardrail`/`InvokeModel` call
starts failing. Retain means `sam delete` (or a replacement-triggering
parameter change) removes the CloudFormation stack but leaves the guardrail
and version in place in the account - cleanup becomes a deliberate,
separate step (see the README's Cleanup section) rather than a side effect
of tearing down the stack.

## Policy design in this template

- **Content policy** (`ContentPolicyConfig`): category-based filters -
  `VIOLENCE`, `HATE`, `SEXUAL`, `INSULTS`, `MISCONDUCT` - each with an
  independent input/output strength (`NONE`, `LOW`, `MEDIUM`, `HIGH`), plus
  `PROMPT_ATTACK`, which only meaningfully supports `InputStrength` since it
  classifies the user's prompt for injection/jailbreak attempts and has no
  output-side equivalent (`OutputStrength` is set to `NONE`).
  `INSULTS` is set to `MEDIUM` here as a realistic default (insults are
  common in legitimate frustrated-customer input; blocking too
  aggressively creates false positives), while the rest default to `HIGH`.
- **Topic policy** (`TopicPolicyConfig`): free-text `Definition` plus up to
  5 `Examples` per topic, used to classify intent beyond fixed categories.
  This template denies `FinancialAdvice` as a generic, plausible example -
  swap in your own domain's off-limits topics. See
  [FOLDED-SCALAR-GOTCHA.md](FOLDED-SCALAR-GOTCHA.md) before editing
  `Definition` across multiple lines.
- **Sensitive information policy** (`SensitiveInformationPolicyConfig`):
  splits into `PiiEntitiesConfig` (Bedrock's built-in probabilistic
  detectors - `EMAIL`, `PHONE`, `US_SOCIAL_SECURITY_NUMBER`,
  `CREDIT_DEBIT_CARD_NUMBER` here) and `RegexesConfig` (custom patterns for
  identifiers Bedrock has no built-in entity for, like an internal ticket
  ID format). PII actions here mix `ANONYMIZE` (mask and continue) and
  `BLOCK` (refuse outright) to show both are available per entity.
- **Word policy** (`WordPolicyConfig`): the managed `PROFANITY` list is
  free and covers common cases; `WordsConfig` adds custom exact-match
  terms (placeholder competitor/codename strings here - replace with your
  own).

## Parameterization

- `GuardrailName` is constrained (`AllowedPattern`, `MaxLength: 50`) to
  match Bedrock's own validation for the `Name` field, so a bad value fails
  at `sam deploy` parameter validation rather than at guardrail creation.
- `GuardrailDescription`, `BlockedInputMessage`, `BlockedOutputMessage` are
  free-form strings with `MaxLength` set to Bedrock's documented caps (200
  and 500 respectively) for the same reason.
- Topic/content/PII/word policy contents are not parameterized - they're
  the reusable *pattern*, not deploy-time configuration. Fork the template
  and edit the policy blocks directly for your own use case rather than
  threading every filter type and topic through `Parameters`, which would
  turn a one-file reference template into a config-language exercise.

## Region

Amazon Bedrock (and Guardrails specifically) is available in 30+ AWS
regions across the Americas, Europe, and Asia Pacific - there is no
Guardrails-specific region restriction beyond wherever Bedrock itself is
enabled for your account. Unlike
[aws-sam-static-website](https://github.com/jeffgrosse/aws-sam-static-website)
(hard-locked to `us-east-1` because CloudFront requires ACM certs from that
region specifically), this template has no cross-service constraint forcing
a single region, so it deliberately does **not** add a `Rules` block to
enforce one. Deploy in whichever Bedrock-enabled region makes sense for
your account - see the README for how to check.

## Cost model

Guardrails pricing is per "text unit" (up to 1,000 characters each,
prorated for partial units) and charged per policy category evaluated, not
per guardrail or per request flatly - see the README's cost estimate for
current per-unit pricing. Word filters are free; content filters, denied
topics, and sensitive information filters are billed separately, so a
request evaluated against all three policies in this template is charged
for all three, not once.
