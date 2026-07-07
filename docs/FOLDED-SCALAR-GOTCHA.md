# The 200-character folded-scalar gotcha

This is the reason this repo exists. Hit while building a topic policy for a
production Bedrock Guardrail: a `Definition` field that read as reasonably
short in the YAML source failed guardrail creation with an error that gives
no hint that YAML formatting is the cause.

## Exact error text (grep-able)

```
Resource handler returned message: "1 validation error detected: Value at
'topicPolicyConfig.topicsConfig.1.member.definition' failed to satisfy
constraint: Member must have length less than or equal to 200"
(Service: Bedrock, Status Code: 400, Request ID: ...) (HandlerErrorCode: InvalidRequest)
```

The property path uses the **API's camelCase field names**
(`topicPolicyConfig.topicsConfig.1.member.definition`), not the PascalCase
names you wrote in the template (`TopicPolicyConfig.TopicsConfig[0].Definition`).
That mismatch alone makes the error harder to map back to your YAML - and
nothing in the message mentions YAML, folding, or whitespace.

## Root cause

[`GuardrailTopicConfig.definition`](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_GuardrailTopicConfig.html)
has a hard **200-character maximum**. That limit is enforced against the
string Bedrock actually receives over the API - which, for a CloudFormation
template, is whatever your YAML parser produces *after* resolving scalar
style, not the number of characters you typed in the template file.

YAML's **folded scalar** (`>` or `>-`) is designed to let you hand-wrap long
prose across multiple source lines while producing a single line of output.
It does this by:

1. Joining each line break with a single space (not deleting it).
2. Stripping leading indentation from each line.

So a `Definition` written as a folded scalar across 4 lines of ~60 characters
each doesn't yield ~240 characters, and it doesn't yield 4 separate lines -
it yields one ~242-character string with the original line breaks silently
replaced by spaces. The *source* looks short because it's wrapped for
readability in an editor. The *value* CloudFormation sends to Bedrock is the
fully-joined string, and that's what gets checked against the 200-character
constraint.

The AWS docs for `GuardrailTopicConfig` state the 200-character limit as a
plain fact about the API field. They say nothing about YAML scalar styles,
because from the API's point of view there's no such thing - by the time a
request reaches Bedrock, it's just a string. The YAML-folding interaction is
entirely a template-authoring problem, invisible from the API side and from
the CloudFormation console's error surface.

## Wrong pattern (what failed)

```yaml
TopicPolicyConfig:
  TopicsConfig:
    - Name: OffTopicRequests
      Type: DENY
      Definition: >
        Any question unrelated to the roster data shown in this demo, the
        services described on this site, or the demo itself - including
        general knowledge questions, coding help, or requests to role-play
        as something else or ignore prior instructions.
```

This reads as four short, reasonably-wrapped lines in an editor. Folded
(what Bedrock actually receives):

```
Any question unrelated to the roster data shown in this demo, the services described on this site, or the demo itself - including general knowledge questions, coding help, or requests to role-play as something else or ignore prior instructions.
```

244 characters. Over the 200-character cap by 44, and nothing about the
source made that obvious - the line-wrapping that makes the YAML pleasant to
read is exactly what hides the true length from a glance.

## Correct pattern

Two ways to avoid it, in order of preference for this specific field:

**1. Write it as a single physical line, quoted.** This is the fix actually
shipped:

```yaml
Definition: "Any question not about the demo's roster data, the site's services, or the demo itself. Includes general knowledge, coding, current events, and personal advice."
```

There's no scalar style to reason about - the string in the file is the
string Bedrock sees. It's harder to hand-wrap in a diff, but for a
200-character-capped field that's a small price, and most editors soft-wrap
long lines for display without touching the file's actual line breaks.

**2. If you need multi-line source for readability on a field that isn't
length-capped this tightly, use a literal block scalar (`|` or `|-`) instead
of folded (`>` or `>-`).** Literal scalars preserve line breaks as `\n`
rather than collapsing them to spaces, so the resulting value's length is
visible and predictable from the source - each line contributes its own
length plus one newline character, with no silent space-insertion at line
boundaries:

```yaml
Description: |-
  Line one of the description.
  Line two of the description.
```

This doesn't help a field like `Definition` where the *rendered* string
(newlines and all) still needs to fit under a character cap and a
guardrail's classification prompt reads better as flowing prose than as
literal line breaks - but it's the right tool any time you want predictable
length from multi-line YAML source in general.

## Rule of thumb

**Before writing any Bedrock Guardrail string property (`Definition`,
`Description`, `Examples[]`, `BlockedInputMessaging`, etc.) as a multi-line
YAML folded scalar, count the length of the *folded* output, not the source
text.** If a field has a documented character cap, write it as a single
quoted line instead - it costs you line-wrapping in the editor, but it
guarantees the string you can see is the string that gets validated. Reserve
folded/literal scalars for fields with no tight length constraint (or
generous ones, like the 500-character `BlockedInputMessaging`/
`BlockedOutputsMessaging`).

## Interview one-liner

> Hit a 200-character YAML folded-scalar constraint on a Bedrock Guardrail
> topic definition; the error message pointed at the property, not the YAML
> formatting - root cause was folded-scalar whitespace being counted
> post-fold against a limit the docs describe pre-fold.
