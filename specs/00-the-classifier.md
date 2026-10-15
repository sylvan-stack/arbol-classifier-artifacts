---
role: authored
spec: CLS-0
status: implemented
---
# Classifier · 00 The classifier

`arbol-classifier` answers closed questions about a text. An [Arbol Trunk](../../arbol/GLOSSARY.md#arbol-trunk) starts it with the text and a list of questions; it asks Jev, the decision model of TypeSafe AI, and gives for each question the answer and how sure that answer is. In its first version it is a wrapper around Jev and nothing more ([CLS-D1](#decisions)). This specification states its commands, what it takes and gives, what it sends to Jev, where its key comes from, its limits and its errors. It matters because a Trunk needs small judgements, such as which of a few kinds a message is, without choosing a model and without reaching back to Arbol, and because the key, the exact model version and the limits belong in one place and not in every Trunk.

It follows the conventions of the [Driver Contract](../../arbol/driver-contract/README.md#conventions). The architecture names the component and its place: [ARC-0](../../arbol/architecture/00-components-and-boundaries.md), "Small model calls" (ARC-D56). The owner decided its points on 13 October 2026 and accepted the specification the same day. It was implemented on 14 October 2026, when the first Build passed all three checks.

## Scope

Covered: what the classifier is, its three commands, the request and the result, what is sent to Jev, the model version, the key, the limits, the errors, the settings, the checks and when a Build counts as working, what Jev's documentation says.

Not covered: which questions a Trunk asks and what it does with a low confidence, which belong to the Trunk's specification; the small text calls a Trunk makes through a Driver (ARC-0); Mycel's own use of Jev ([MYC-14](../../mycel/specs/14-drift-detection.md)); any model other than Jev.

## Behaviour

### What it is

| | |
|---|---|
| A program of its own | `arbol-classifier`, a component with numbered Builds that belong to no Root ([BLD-0](../../arbol/builds/00-builds-and-versions.md)). A Trunk picks the newest working Build that implements the version of this contract it speaks (ARC-0) |
| One call, one answer | It starts, sends one request to Jev, prints one result and ends |
| Nothing kept | It has no state directory, no cache and no record of what it was asked. The text it judges is never written to disk |
| Not a Driver | It has no Subscriptions, no sign-in, no tools and no conversation. Of the Driver Contract only the rules of the command line hold, as named below |
| Nothing to its left | It reaches Jev and nothing else. The programs it may start are the command that prints its key and, in `add-key`, the system's `security` command that stores it |

### The commands

```sh
arbol-classifier classify < request.json
arbol-classifier classify --timeout 5 < request.json
arbol-classifier info
arbol-classifier -v 3 info
arbol-classifier add-key
```

| Command | Takes | Gives |
|---|---|---|
| `classify` | A request, as one JSON document on standard input | The result, as one JSON document on standard output |
| `info` | Nothing | Its name, its package version and build number, the version of this contract, its settings file, the Jev model it uses, the address it calls, and whether a key was found. Never the key |
| `add-key` | The Jev key, typed at the terminal when the command asks for it. It is not shown as it is typed ([CLS-D7](#decisions)) | It stores the key in the classifier's Keychain item, replacing a key stored there before, and says where it stored it and whether it replaced one. Never the key |

The rules of the [Driver Contract's command line](../../arbol/driver-contract/00-overview-and-boundary.md) hold:

| Rule | Detail |
|---|---|
| Build | `-v <build number>` as the first argument runs that Build. Without it the newest working one runs (BLD-0) |
| Options | `--timeout <seconds>` sets the time limit of one call. `--settings <file>` names another settings file, for tests |
| Secrets | The key is never given to the program, neither as an argument nor on standard input nor in a file it is handed. The one exception is `add-key`, which asks the person at the terminal for it |
| Output | One JSON document on standard output. A caller ignores fields it does not know |
| Errors | A call that is refused or fails prints `{ "error": … }` with the error object of [DRV-4](../../arbol/driver-contract/04-errors-and-limits.md) |
| Exit status | 0: done as asked. 1: refused or failed |
| Standard error | Diagnostics for a person, without the key and without the text that was judged |

The version of this contract is a whole number, and the first one is 1. It rises when a command, an option, a field or an error code is removed or changes its meaning, as with the Driver Contract (DRV-0).

### The request and the result

They are Jev's own ([CLS-D1](#decisions)). `arbol-classifier` adds the model and the key to the request and takes nothing away from the reply.

A request holds the *state*, which is the text or the data to judge, and one or more *questions*. Each question has a name the caller chooses, and its answer comes back under the same name.

```json
{
  "state": "Can you also update the README while you are at it?",
  "questions": {
    "kind": {
      "type": "choice",
      "instructions": "What is this message?",
      "criteria": {
        "request": "It asks for work to be done",
        "question": "It asks for information only",
        "other": null
      }
    }
  }
}
```

```json
{
  "model": "jev-1.13.0",
  "answers": {
    "kind": {
      "type": "choice",
      "choice": "request",
      "confidence": 0.94,
      "probabilities": { "request": 0.96, "question": 0.03, "other": 0.01 }
    }
  },
  "usage": { "input_tokens": 212, "output_tokens": 30 }
}
```

Jev knows three kinds of question, and all three are passed through:

| `type` | Asks | `criteria` | The answer holds |
|---|---|---|---|
| `choice` | Which one of the named options fits | Required: each option with its description, or `null` where the name says enough. Up to 255 options | `choice`, the option chosen; `probabilities`, one for every option, which add up to 1; `confidence`, from 0 to 1 |
| `noul` | Yes or no | Optional: what `true` and what `false` means | `noul`, a number from 0 to 1, where 1 means yes |
| `score` | Where on an ordered scale | Required: 2 to 10 descriptions of the levels, the lowest first | `score`, `legend`, `probabilities` and `confidence` |

`instructions` is the question itself. It and each description may be a text, or data as a JSON object or array, and so may the state.

| | What `arbol-classifier` checks |
|---|---|
| Before the call | That the request is one JSON object with a `state` and at least one question, that every question has a `type` and `instructions`, that the request names no `model`, and that it is within the size limit |
| After the call | That the reply is JSON, that it names the model that was asked for, and that it holds an answer for every question asked and for no other |

It interprets no question and does not judge whether one is well put. What Jev refuses, it reports as Jev said it. It does not recompute a confidence, and what a low one means is the caller's business (ARC-0).

### What is sent to Jev

One request: `POST https://api.typesafe.ai/v1/systemone`, with the key as a bearer token and a JSON body that is the caller's request with the field `model` added.

Nothing else is sent: no identifier of the chat, of the Root or of the machine. The state and the questions leave the Mac for the service of TypeSafe AI, and they go nowhere else.

### The model version

The classifier asks for one exact version of Jev, by default `jev-1.13.0`. The settings can name another exact version, written as `jev-` and three numbers, such as `jev-1.14.0`. A name that moves, such as `jev-latest` or `jev-preview`, is refused as a setting: the same question must not be answered by another model from one day to the next without anyone having decided so.

The reply names the model that answered. When that is not the version asked for, the call fails, and the result of a call that succeeds names the version.

### The key

| | |
|---|---|
| Where it is | A generic password in the login Keychain: the service `com.sylvan-stack.arbol.classifier`, the account `jev` ([CLS-D2](#decisions)) |
| Who stores it | The Leather Bag, with `arbol-classifier add-key` ([CLS-D7](#decisions)) or on the Secrets page of Seqoya Lab, where every secret of Arbol 2 is managed (ARC-D62 in ARC-0). The classifier writes the item only in `add-key`, and never removes it |
| How it is read | At each call, with the system's `security` command, as a Driver reads a key (DRV-0). The settings can name a command that prints the key instead |
| Where it never is | In the settings file, an argument, the environment, the output or a diagnostic |

Mycel keeps a Jev key of its own, in the item `mycel` / `jev-api-key` ([MYC-5](../../mycel/specs/05-settings.md)), because Mycel has to work without Arbol. The same key may be stored in both items.

### Limits

The numbers are values to start with, not measured ones ([CLS-D4](#decisions)).

| Limit | Default | When it is reached |
|---|---|---|
| Time for one call, from its start, with reading the key and every attempt in it | 10 seconds. `--timeout` takes 1 to 60 | The call ends with `limit_reached` |
| Size of the request as it is sent | 64,000 bytes | The request is refused with `invalid_input` before anything is sent. Nothing is cut |
| Attempts | 3 | A further attempt is made only after the statuses 429 and 529, another status from 500 on, or a connection that failed. It waits half a second, then one second, and never goes past the time limit: when the wait would go past it, no further attempt is made, and the call ends with what the last attempt gave |

Jev has limits of its own on its side. What it refuses for them is passed on as `invalid_input`.

### Errors

A failure is the error object of DRV-4 with one of these codes:

| Code | When | Trying again can help |
|---|---|---|
| `invalid_input` | The request is not as described above, names a model, or is too large; or Jev refused it as invalid (status 422) or as too large (status 413), in which case the message says what Jev said; or `add-key` found no terminal to ask at, or was given nothing that can be a key | No |
| `not_signed_in` | No Jev API key was found: the Keychain item does not exist, is empty, cannot be read or holds nothing that can be a key, or the command the settings name for the key failed or printed nothing that can be a key. A key is one line of visible characters without spaces. The message says so plainly and says where the key was looked for ([CLS-D6](#decisions)) | No. The remedy says how to store the key |
| `credential_rejected` | Jev refused the key (status 401) | No |
| `rate_limited` | Jev still answered 429 after the attempts | Yes, after waiting |
| `vendor_unavailable` | Jev could not be reached, or still answered 529 or another status from 500 on after the attempts | Yes |
| `vendor_error` | Jev refused for another reason, or its reply cannot be accepted: it is not JSON, names another model, or lacks an answer | No |
| `limit_reached` | The time limit of the call was reached | No |
| `internal` | A defect in the classifier, or, in `add-key`, a Keychain that did not store the key | No |

A call without a key ends like this, before anything is sent to Jev:

```json
{
  "error": {
    "code": "not_signed_in",
    "message": "No Jev API key was found. arbol-classifier looked in the login Keychain for the item with the service com.sylvan-stack.arbol.classifier and the account jev.",
    "retryable": false,
    "remedy": "Store the key with: arbol-classifier add-key"
  }
}
```

A caller treats them all alike in one respect: it goes on without the answer, because a small model call never fails a Chat Exchange (ARC-0).

### Settings

The classifier's file is `~/.arbol/arbol-classifier.json`. It is read at the start of every call, it may be missing, and the classifier never writes it. A faulty file refuses the call with `invalid_input`, naming the file and the fault, as with a Driver (DRV-0). A key the classifier does not know, such as a mistyped `timoutSeconds`, is a fault too ([CLS-D5](#decisions)).

`endpoint` may be Jev's own address or an address on this Mac, which is what the checks use: `localhost` or a loopback address such as `127.0.0.1` or `::1`, over `http` or `https`. Any other address is refused, so that a changed file cannot send the key somewhere else.

### Checks, and when a Build counts as working

| Check | What runs | Costs |
|---|---|---|
| unit | The parts, without network and without the Keychain | Nothing |
| contract | The built program against a stand-in for Jev on this Mac: each kind of question, each failure of the table above, the time limit, the size limit, the attempts, and that neither the key nor the judged text appears in any output | Nothing |
| live | One real question to Jev with the real key | A fraction of a cent |

A Build counts as working only when all three have passed ([CLS-D3](#decisions)), as a Driver's Build needs one real message (BLD-0). The first real question to Jev settled the one point that Jev's documentation left unclear, named below.

### What Jev's documentation says

Read on 13 October 2026. This is a dated reading of the vendor's pages and not a tested result; only the contradiction below was tested, by the first real question to Jev on 14 October 2026.

- **The call.** One address, `POST https://api.typesafe.ai/v1/systemone`, with a bearer token. The body has `state`, `model` and `questions`; the reply has `model`, `answers` and `usage`. The kinds of question and their answers are as in the table above. [API reference](https://docs.typesafe.ai/api), [Choice](https://docs.typesafe.ai/primitives/choice)
- **Failures.** 401 for a missing or wrong key, 422 for a request that fails validation, with the field named, 429 for too many requests and 529 for an overloaded service. For the last two the vendor advises trying again with growing waits.
- **Not stated on the pages read.** The form of an error's body, headers about rate limits, and a limit on the size of a request.
- **One contradiction.** The API reference names the field `model`. The example on the Choice page shows `selectedModels` with a list instead. This specification follows the API reference. The first real question proved it right: Jev took `model`, and its reply named `jev-1.13.0` written the same way.
- **Price and size**, from the reading of 8 October 2026 in MYC-14: 0.042 US dollars for a million tokens of input, output free; 64,000 tokens for a request, and 32,000 for the state with its longest question.

Observed in version 1: there was no classifier and no use of Jev. A small model call, such as the one for a chat's title, was a Completion made through an Intelligence Provider, and it could neither delay nor fail the Exchange it belonged to.

Historical evidence (frozen version 1; ordinary locators, not Source Refs):

- `arbol_deprecated_v1:product-spec/03-turn-execution.md#Lifecycle of one Exchange`

## Requirements

| ID | `arbol-classifier` must… | Origin | Verified by |
|---|---|---|---|
| CLS-0.1 | be one program that handles one request for each invocation and keeps nothing between two of them: no state directory, no cache and no record of the text it judged | New | contract: `classify.writes-nothing`, `info.writes-nothing`, passed on Build 2 (SYL-157) |
| CLS-0.2 | take the request as one JSON document on standard input, and print one JSON document on standard output, the result or `{ "error": … }` with the error object of DRV-4, ending with status 0 or 1 | New | contract: `classify.choice`, `classify.malformed`, `classify.other-model`, passed on Build 2 (SYL-157) |
| CLS-0.3 | accept a request in Jev's own form, a `state` and named `questions` of the types `choice`, `noul` and `score`, and refuse one that names a `model` | New | contract: `classify.choice`, `classify.noul`, `classify.score`, `classify.names-a-model`, passed on Build 2 (SYL-157) |
| CLS-0.4 | check of a request only that it is well formed and within the size limit, and interpret no question | New | unit: `Requests.test_only_the_listed_checks`, `Requests.test_no_question_is_interpreted`, `Classify.test_interprets_no_question`; contract: `classify.passes-questions-through`, `classify.malformed`; passed on Build 2 (SYL-157) |
| CLS-0.5 | send to Jev the request with the model added and the key as a bearer token, and nothing that names the chat, the Root or the machine | New | contract: `classify.what-is-sent`, `classify.choice`, `classify.noul`, `classify.score`, passed on Build 2 (SYL-157) |
| CLS-0.6 | ask for the exact Jev version of its settings, by default `jev-1.13.0`, refuse a name that moves as a setting, and fail when the reply names another model | New | contract: `classify.pinned-model`, `classify.other-model`, `settings.moving-model`, passed on Build 2 (SYL-157) |
| CLS-0.7 | give the reply as Jev gave it, after checking that every question asked has an answer and no other has | New | contract: `classify.choice`, `classify.noul`, `classify.score`, `classify.missing-answer`, passed on Build 2 (SYL-157) |
| CLS-0.8 | read the key at each call from its Keychain item or from the command its settings name, write the item only in `add-key` and never remove it, and let it appear in no argument, environment, settings file, output or diagnostic; without a key, refuse the call with `not_signed_in` and a message that says plainly that no Jev API key was found and where it was looked for | New | contract: `key.keychain`, `key.not-found`, `key.only-in-the-bearer-token`, `add-key.at-a-terminal`; unit: `Keychain.test_the_key_is_read_from_the_item_at_each_call`, `StoreKey.test_the_key_goes_to_security_on_standard_input_never_as_an_argument`, `StoreKey.test_the_item_is_never_removed`; passed on Build 3 (SYL-161) |
| CLS-0.9 | end within the time limit of the call, by default 10 seconds, with every attempt in it | New | contract: `limits.time-limit`, `limits.no-wait-past-the-limit`; unit: `Transport.test_an_attempt_ends_with_the_time_limit_of_the_call`, `Attempts.test_no_wait_goes_past_the_time_limit`, `Attempts.test_a_slow_key_reaches_the_time_limit`, `Invocation.test_standard_input_is_read_until_it_ends_or_the_time_limit`; passed on Build 4 (SYL-162) |
| CLS-0.10 | refuse a request larger than the size limit before anything is sent, and cut nothing | New | unit: `SizeLimit.test_the_limit_is_measured_on_the_body_as_it_is_sent`, `SizeLimit.test_nothing_is_cut`, `Classify.test_a_refused_request_reads_no_key_and_sends_nothing`; contract: `classify.size-limit`; passed on Build 2 (SYL-157) |
| CLS-0.11 | make at most three attempts, a further one only after the statuses 429 and 529, another status from 500 on or a failed connection | New | contract: `attempts.until-an-answer`, `attempts.waits`, `errors.rate-limited`, `errors.vendor-unavailable`, `limits.no-wait-past-the-limit`; unit: `Errors.test_statuses_and_failed_connections_that_are_tried_again`, `Errors.test_a_further_attempt_only_where_it_can_help`, `Attempts.test_the_waits_between_attempts`; passed on Build 4 (SYL-162) |
| CLS-0.12 | report every failure with one of the eight codes of this specification | New | contract: `classify.malformed` (`invalid_input`), `errors.invalid-at-jev` (`invalid_input`), `key.not-found` (`not_signed_in`), `errors.credential-rejected`, `errors.rate-limited`, `errors.vendor-unavailable`, `errors.vendor-error`, `limits.time-limit` (`limit_reached`), `add-key.keychain-refuses` (`internal`); unit: `Errors.test_statuses_that_no_further_attempt_can_help`, `Errors.test_the_message_says_what_jev_said_without_the_judged_text_or_the_key`, `Errors.test_the_eight_codes`; passed on Build 4 (SYL-162) |
| CLS-0.13 | call only Jev's own address or an address on this Mac, and refuse any other `endpoint` | New | unit: `Addresses.test_only_jev_or_this_mac`, `Transport.test_an_endpoint_on_this_mac_that_leads_elsewhere_is_refused_before_anything_is_sent`; contract: `classify.endpoint-elsewhere`; passed on Build 2 (SYL-157) |
| CLS-0.14 | print with `info` its name, package version, build number, contract version, settings file, model and address, and whether a key was found, never the key | New | contract: `info.identity`, `info.defaults`, `info.settings-file`, `info.settings-option`, `info.accepted-settings`, passed on Build 1 (SYL-152); whether a key was found: `key.keychain`, `key.not-found`, passed on Build 3 (SYL-161) |
| CLS-0.15 | count as a working Build only when its unit checks, its contract checks against a stand-in and one real question to Jev have passed | New | live: `live`, passed on Build 6, the first Build recorded as working (SYL-164); unit: `Live.test_one_choice_question_through_the_builds_own_program`, `Live.test_a_stand_in_for_jev_cannot_pass`, `Judge.test_a_reply_from_another_model_fails`, `WithoutAKey.test_a_build_made_while_no_key_is_stored_fails_the_live_check` |
| CLS-0.16 | offer `add-key`, which asks for the Jev key at the terminal without showing it, stores it in its Keychain item, replacing a key stored there before, and prints where it stored it and whether it replaced one, never the key | New | contract: `add-key.at-a-terminal`, `add-key.no-terminal`, `add-key.not-a-key`, `add-key.keychain-refuses`, `cli.add-key-arguments`; unit: `Terminal.test_the_key_is_read_with_echo_off_and_the_terminal_is_set_back`, `NoTerminal.test_without_a_terminal_add_key_refuses`; passed on Build 3 (SYL-161) |

## Interface

```ts
// arbol-classifier classify [--timeout <seconds>] [--settings <file>]     a ClassifyRequest on standard input
interface ClassifyRequest {
  state: Content;                          // the text or the data to judge
  questions: Record<string, Question>;     // by a name the caller chooses
}
type Content = string | object | unknown[];
type Question =
  | { type: "choice"; instructions: Content; criteria: Record<string, Content | null> }
  | { type: "noul";   instructions: Content; criteria?: { true?: Content; false?: Content } }
  | { type: "score";  instructions: Content; criteria: Content[] };          // 2 to 10 levels, the lowest first

interface ClassifyResult {
  model: string;                           // the exact Jev version that answered
  answers: Record<string, Answer>;         // by the names of the questions
  usage: { input_tokens: number; output_tokens: number };
}
type Answer =
  | { type: "choice"; choice: string; probabilities: Record<string, number>; confidence: number }
  | { type: "noul";   noul: number }
  | { type: "score";  score: number; legend: Record<string, Content>; probabilities: Record<string, number>; confidence: number };

// arbol-classifier info [--settings <file>]
interface ClassifierInfo {
  name: "arbol-classifier";
  version: string;                         // the package version
  build: string;                           // the build number of the Build that runs (BLD-0)
  contract: number;                        // the version of this contract: 1
  settingsFile: string;
  model: string;
  endpoint: string;
  keyFound: boolean;
}

// arbol-classifier add-key                asks for the key at the terminal, without showing it
interface KeyStored {
  keychain: { service: string; account: string };   // "com.sylvan-stack.arbol.classifier" and "jev"
  replaced: boolean;                       // the item existed before, and this key replaced what it held
}

interface ClassifierSettings {             // ~/.arbol/arbol-classifier.json
  model?: string;                          // an exact Jev version; default "jev-1.13.0"
  timeoutSeconds?: number;                 // default 10; from 1 to 60, as --timeout
  maxRequestBytes?: number;                // default 64000; a positive whole number
  endpoint?: string;                       // default "https://api.typesafe.ai/v1/systemone"; otherwise only an address on this Mac
  secret?: SecretSource;                   // where the key comes from (DRV-0); default: the Keychain item
}
```

## Known problems in version 1

There was no classifier, so there is nothing of its own to avoid. One rule of version 1 is kept: a small model call can neither delay nor fail the Exchange it serves. Here that is the time limit of a call, together with the caller going on without an answer.

## Decisions

| ID | Question | Proposed default | Outcome |
|---|---|---|---|
| CLS-D1 | Does the first contract take Jev's own request and reply, or a form of the classifier's own? | Jev's own. The request and the reply keep Jev's field names, and all three kinds of question are passed through. A form of its own would need a translation that has to follow every change of Jev | Decided 13 October 2026 by the owner: "In first version it's just Jev wrapper." That the field names are kept is the technical consequence, drawn the same day |
| CLS-D2 | Where is the Jev key kept? | In a Keychain item of the classifier's own, the service `com.sylvan-stack.arbol.classifier` with the account `jev`. Mycel keeps its own item for the same key, because Mycel works without Arbol. The alternative: both read one item | Decided 13 October 2026 by the owner: "API key is in keychain." The key is kept in the Keychain, in the item named here. Neither this item nor Mycel's existed that day. The owner believed the key was in the Keychain already, did not remember where, and decided that a new key is made when it is not found. All secrets are managed on the Secrets page of Seqoya Lab (ARC-D62) |
| CLS-D3 | Does a Build count as working only after one real question to Jev? | Yes, as a Driver's Build needs one real message. It costs a fraction of a cent, and it is the only proof that the key, the address and the model version work. The alternative: only checks against a stand-in, and a real question on demand | Decided 13 October 2026 by the owner: the proposed default |
| CLS-D4 | Which limits does it start with? | 10 seconds for a call, three attempts and 64,000 bytes for a request. They are first values and not measured ones | Decided 13 October 2026 by the owner: the proposed default |
| CLS-D5 | What does the classifier do with a key in its settings file that it does not know, such as a mistyped `timoutSeconds`? | Refuse the file with `invalid_input` and name the key, as the owner decided for Mycel (MYC-D23). The alternative: ignore the key and name it in a notice, as a Driver did (DRV-0) | Decided 14 October 2026 by the owner: such a key is ignored neither by the classifier nor by a Driver. The file is refused with `invalid_input`, and the message names the key. DRV-0 was changed the same day to match |
| CLS-D6 | What does a call say when no Jev API key is found? | `not_signed_in`, with a remedy that names the Keychain item | Decided 14 October 2026 by the owner: a call without a Jev API key ends with a clear error message that says so. The message says plainly that no Jev API key was found and where it was looked for, and the remedy says how to store the key (Errors) |
| CLS-D7 | How does the Leather Bag store the Jev key without the Secrets page of Seqoya Lab? | By hand with the system's `security` command; the classifier only reads its item (ARC-D62) | Decided 14 October 2026 by the owner: with a command of the classifier, `arbol-classifier add-key`. That it asks for the key at the terminal and does not show it, and that it replaces a key stored before, are the technical consequences, drawn the same day: a key written as an argument stays in the shell's history and can be seen by other programs while the command runs. The Secrets page stays the place where all secrets are managed |
