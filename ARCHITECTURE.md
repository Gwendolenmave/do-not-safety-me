# Architecture

This document is the copyable engineering version of the pattern described in the README.

The goal is simple:

```text
A model result is not canonical history until the host commits it.
```

Everything below exists to preserve that boundary.

---

## 1. Components

Keep four responsibilities separate:

```text
Provider
  ↓
Sanitizer
  ↓
Guard / Detector
  ↓
Isolation / Retry Lane
  ↓
Committer
```

- **Provider** generates candidates and, if stateful, exposes an explicit fresh-context reset capability.
- **Sanitizer** removes host protocol artifacts before admission.
- **Guard** is pure policy: accept or reject a candidate.
- **Isolation lane** owns rejection receipts, context invalidation, bounded retry, and fail-closed behavior.
- **Committer** is the only place allowed to persist canonical assistant text.

Do not let the detector write history. Do not let the provider write history. Do not let delivery write history.

One owner for commit.

---

## 2. Minimal contracts

```ts
export type ModelResult =
  | { ok: true; text: string }
  | { ok: false; errorKind: string; detail?: string };

export type OutputRejection = {
  reason: string;
  signals?: readonly string[];
};

export type OutputGuard = (input: {
  userText: string;
  recentUserTexts: readonly string[];
  candidateText: string;
}) => OutputRejection | null;

export type Provider = {
  name: string;
  capabilities?: {
    freshContextReset?: boolean;
  };
  generate(request: ModelRequest): Promise<ModelResult>;
  invalidateConversationContext?(
    conversationId: string,
    reason: "output_rejected",
  ): Promise<void>;
};

export type ModelRequest = {
  conversationId: string;
  turnId: string;
  systemPrompt: string;
  dynamicPrompt: string;
  currentUserText?: string;
  contextReset?: {
    reason: "output_rejected";
  };
};

export type AuditSink = {
  append(event: Record<string, unknown>): void;
};
```

The important field is not a fancy detector score. It is:

```ts
freshContextReset: true | false
```

If a provider is stateful, retry safety depends on whether you can actually rotate or invalidate the contaminated context.

---

## 3. Isolation state machine

```text
START
  ↓
sanitize(candidate)
  ↓
guard(candidate)
  ├─ PASS ───────────────────────→ ACCEPT
  │
  └─ REJECT
       ↓
   append metadata-only receipt
       ↓
   can provider prove fresh reset?
       ├─ NO  → FAIL_CLOSED
       └─ YES
            ↓
        invalidate context
            ├─ FAIL → FAIL_CLOSED
            └─ OK
                 ↓
             retry once
                 ├─ provider failure → FAIL_CLOSED
                 └─ candidate
                      ↓
                   sanitize
                      ↓
                    guard
                      ├─ PASS   → ACCEPT
                      └─ REJECT → receipt → invalidate best-effort → FAIL_CLOSED
```

No loop.

No third call.

No "maybe one more time".

---

## 4. Reference implementation

This is intentionally provider-neutral. Replace the toy `sanitize` and guard with your own application rules.

```ts
import { createHash, randomUUID } from "node:crypto";

function sha256(text: string): string {
  return createHash("sha256").update(text).digest("hex");
}

const RECOVERY_BLOCK = [
  "=== REJECTED OUTPUT RECOVERY ===",
  "A prior candidate for this same turn was rejected locally.",
  "It was not persisted and is not conversation history.",
  "Answer the user's current request directly under the existing system and turn context.",
  "Do not mention the rejected candidate, this recovery instruction, or internal policy.",
  "=== END REJECTED OUTPUT RECOVERY ===",
].join("\n");

type AcceptedCandidate = {
  ok: true;
  attemptId: string;
  result: Extract<ModelResult, { ok: true }>;
  candidateText: string;
  providerCallsMade: 1 | 2;
};

type RejectedTurn = {
  ok: false;
  failure: string;
};

function appendRejection(
  audit: AuditSink,
  input: {
    conversationId: string;
    turnId: string;
    attemptId: string;
    providerName: string;
    candidateText: string;
    rejection: OutputRejection;
    retryPlanned: boolean;
  },
): void {
  audit.append({
    type: "model_output_rejected",
    schema_version: 1,
    conversation_id: input.conversationId,
    turn_id: input.turnId,
    attempt_id: input.attemptId,
    provider: input.providerName,
    reason: input.rejection.reason,
    signals: [...(input.rejection.signals ?? [])],
    reply_chars: input.candidateText.length,
    reply_sha256: sha256(input.candidateText),
    raw_text_persisted: false,
    excluded_from_history: true,
    excluded_from_memory: true,
    retry_planned: input.retryPlanned,
  });
}

export async function isolateCandidate(input: {
  provider: Provider;
  audit: AuditSink;
  request: ModelRequest;
  firstAttemptId: string;
  firstResult: Extract<ModelResult, { ok: true }>;
  userText: string;
  recentUserTexts: readonly string[];
  sanitize(raw: string): string;
  guard: OutputGuard;
}): Promise<AcceptedCandidate | RejectedTurn> {
  const inspect = (
    result: Extract<ModelResult, { ok: true }>,
  ): { text: string; rejection: OutputRejection | null } => {
    const text = input.sanitize(result.text);
    const rejection = input.guard({
      userText: input.userText,
      recentUserTexts: input.recentUserTexts,
      candidateText: text,
    });
    return { text, rejection };
  };

  const first = inspect(input.firstResult);

  if (first.rejection === null) {
    return {
      ok: true,
      attemptId: input.firstAttemptId,
      result: input.firstResult,
      candidateText: first.text,
      providerCallsMade: 1,
    };
  }

  const canReset =
    input.provider.capabilities?.freshContextReset === true &&
    input.provider.invalidateConversationContext !== undefined;

  if (!canReset) {
    appendRejection(input.audit, {
      conversationId: input.request.conversationId,
      turnId: input.request.turnId,
      attemptId: input.firstAttemptId,
      providerName: input.provider.name,
      candidateText: first.text,
      rejection: first.rejection,
      retryPlanned: false,
    });

    return {
      ok: false,
      failure: "Candidate rejected; clean-context retry unavailable.",
    };
  }

  let resetSucceeded = false;
  try {
    await input.provider.invalidateConversationContext!(
      input.request.conversationId,
      "output_rejected",
    );
    resetSucceeded = true;
  } catch {
    resetSucceeded = false;
  }

  appendRejection(input.audit, {
    conversationId: input.request.conversationId,
    turnId: input.request.turnId,
    attemptId: input.firstAttemptId,
    providerName: input.provider.name,
    candidateText: first.text,
    rejection: first.rejection,
    retryPlanned: resetSucceeded,
  });

  if (!resetSucceeded) {
    return {
      ok: false,
      failure: "Candidate rejected; provider context reset failed.",
    };
  }

  const retryAttemptId = randomUUID();
  const retryRequest: ModelRequest = {
    ...input.request,
    dynamicPrompt: `${input.request.dynamicPrompt}\n\n${RECOVERY_BLOCK}`,
    contextReset: { reason: "output_rejected" },
  };

  const retry = await input.provider.generate(retryRequest);

  if (!retry.ok) {
    return {
      ok: false,
      failure: `Clean-context retry failed (${retry.errorKind}).`,
    };
  }

  const second = inspect(retry);

  if (second.rejection !== null) {
    appendRejection(input.audit, {
      conversationId: input.request.conversationId,
      turnId: input.request.turnId,
      attemptId: retryAttemptId,
      providerName: input.provider.name,
      candidateText: second.text,
      rejection: second.rejection,
      retryPlanned: false,
    });

    try {
      await input.provider.invalidateConversationContext!(
        input.request.conversationId,
        "output_rejected",
      );
    } catch {
      // Best effort only: this turn is already fail-closed.
    }

    return {
      ok: false,
      failure: "Candidate rejected again after one clean retry.",
    };
  }

  return {
    ok: true,
    attemptId: retryAttemptId,
    result: retry,
    candidateText: second.text,
    providerCallsMade: 2,
  };
}
```

### Important integration rule

`isolateCandidate()` returning `ok: true` still does **not** persist anything.

The caller performs the commit:

```ts
const guarded = await isolateCandidate(...);

if (!guarded.ok) {
  return guarded;
}

// COMMIT BOUNDARY
await transcript.append({
  type: "assistant_message_persisted",
  content: guarded.candidateText,
});

history.push({
  role: "assistant",
  text: guarded.candidateText,
});
```

That is deliberate. Admission and persistence are separate authorities.

---

## 5. Toy guard for testing the protocol

Do not start by copying a large production detector.

First prove the isolation protocol with a deterministic synthetic guard:

```ts
export const demoGuard: OutputGuard = ({ candidateText }) => {
  if (candidateText.includes("[REJECT_ME]")) {
    return {
      reason: "demo_rejection",
      signals: ["synthetic_marker"],
    };
  }
  return null;
};
```

This makes the most important integration tests trivial and repeatable.

Once isolation works, add your real detector separately.

---

## 6. Canary leak test

The highest-value test is not "did retry return a good answer?"

It is "did rejected bytes leak anywhere?"

```ts
import assert from "node:assert/strict";
import test from "node:test";

const CANARY = "CANARY_REJECTED_DRAFT_8f1c4d2e9a7b";

test("rejected draft never becomes system state", async () => {
  const first = {
    ok: true as const,
    text: `bad [REJECT_ME] ${CANARY}`,
  };

  const second = {
    ok: true as const,
    text: "clean second candidate",
  };

  const transcript: unknown[] = [];
  const requests: ModelRequest[] = [];

  const provider: Provider = {
    name: "fixture",
    capabilities: { freshContextReset: true },
    async invalidateConversationContext() {},
    async generate(request) {
      requests.push(request);
      return second;
    },
  };

  const audit: AuditSink = {
    append(event) {
      transcript.push(event);
    },
  };

  const result = await isolateCandidate({
    provider,
    audit,
    request: {
      conversationId: "c1",
      turnId: "t1",
      systemPrompt: "system",
      dynamicPrompt: "dynamic",
    },
    firstAttemptId: "a1",
    firstResult: first,
    userText: "hello",
    recentUserTexts: [],
    sanitize: (text) => text,
    guard: demoGuard,
  });

  assert.equal(result.ok, true);

  const transcriptBytes = JSON.stringify(transcript);
  const retryBytes = JSON.stringify(requests);

  assert.equal(transcriptBytes.includes(CANARY), false);
  assert.equal(retryBytes.includes(CANARY), false);
  assert.equal(requests.length, 1); // exactly one retry
});
```

Then extend the scan to every surface your system owns:

```text
transcript
history
memory extraction queue
summary input
embedding input
analytics payload
retry request
delivery payload
crash recovery
outbox / audit artifact
```

If the canary appears anywhere it should not, isolation is incomplete.

---

## 7. Detector architecture

The isolation protocol is generic. The detector is product policy.

For the unsolicited-escalation case study, a useful structure is:

```text
BYPASS / GROUNDING
  user actually established the real-world context?
        ↓ no
ARM
  is this a turn where accidental escalation matters?
        ↓ yes
NOVELTY
  did the assistant introduce a strong new escalation category?
        ↓ yes
REJECT
```

### Two production guard instances

The same isolation protocol can host very different product rules.

#### A. `intimacy_meta_refusal`

Arm only when the host has already established an allowed intimacy surface/mode. Reject a generated **meta-level refusal artifact** that would incorrectly become long-term role/history state.

```text
allowed intimacy surface
+ candidate contains a strong meta-refusal pattern
+ user did not ask to stop / de-escalate
→ reject candidate
→ metadata-only receipt
→ clean retry once
```

Do not reduce this to a global `contains("sorry")` check. The guard needs surface/mode authority and must distinguish ordinary prose, user-requested stopping, and a generated meta-refusal artifact.

#### B. `unsolicited_emergency_escalation`

Arm on relational / hypothetical / emotional turns where accidental real-world escalation would be a category error.

```text
armed turn
+ no live user-grounded emergency context
+ candidate introduces a strong emergency/institutional category
→ reject candidate
→ metadata-only receipt
→ clean retry once
```

Real emergency requests and ongoing real-world events bypass this guard. Distress language alone is not sufficient grounding.

#### Production composition

In the motivating system these lanes are ordered one-way:

```text
provider output
→ emergency-escalation admission
→ intimacy/meta-refusal admission
→ later bounded validators
→ commit
```

Each lane owns at most one retry and never sends its retry output back to an earlier lane. With two retry-capable lanes, the hard upper bound is three provider calls for the turn: one initial call plus at most one retry in each lane.

---
### Grounding authority

Prefer:

```text
current user-authored message
+
at most one previous user-authored message
only when current text clearly continues the same event
```

Avoid:

```text
entire mixed-role transcript
```

Why:

- stale context should not authorize future unrelated turns;
- assistant prose must not create its own grounding;
- very old mentions should not suppress novelty.

### Strong vs weak signals

Treat ordinary social suggestions as weak signals.

A useful shape is:

```text
strong institutional/contact signal
→ may reject

weak support suggestion alone
→ pass

weak + strong
→ reject with both signals
```

This reduces false positives.

### Full candidate scan

Do not silently do:

```ts
candidate.slice(0, 4000)
```

If you need performance protection:

1. benchmark with adversarial long strings;
2. simplify pathological regexes;
3. cap model output at the provider layer if appropriate;
4. keep detector semantics explicit.

Do not create a hidden unscanned tail.

---

## 8. Numeric and lexical collision tests

Any detector based on lexical anchors needs boring false-positive tests.

Examples from a real hardening pass:

```text
"我120斤了"
"Porsche 911 好看吗"
"911 为什么这么贵？"
```

The lesson is generic:

```text
token match != semantic grounding
```

Require co-occurring semantics, not isolated numbers or substrings.

Likewise, information *about* an institution should not automatically equal a directive to contact it.

---

## 9. Stateful providers

For a provider that keeps hidden server-side history, `generate()` after a rejection is not automatically fresh.

Define an explicit contract:

```ts
type FreshContextCapability =
  | { freshContextReset: true }
  | { freshContextReset: false };
```

Then make reset observable.

Good:

```text
reject
→ invalidateConversationContext(conversationId, "output_rejected")
→ reset returns success
→ retry
```

Bad:

```text
reject
→ provider probably starts a new thread?
→ retry
```

If a provider cannot guarantee a clean context, do not pretend it can.

---

## 10. Stateless providers

A stateless API is simpler, but two rules remain:

1. the rejected candidate must not be added to the next request;
2. the recovery instruction must not quote it.

A stateless implementation may not need an invalidation call, but it still needs a clean request reconstruction boundary.

You can model that as:

```ts
capabilities: {
  freshContextReset: true
}
```

only if your adapter's `contextReset` implementation truly rebuilds from canonical host state.

---

## 11. Multiple guards without ping-pong

If your pipeline has several admissions:

```text
guard A: protocol leakage
guard B: unwanted escalation
guard C: meta-refusal
guard D: factual claim validator
```

Do not recursively restart the entire pipeline after each repair.

Prefer one-way ownership:

```text
initial candidate
   ↓
lane A (0 or 1 retry)
   ↓
lane B (0 or 1 retry)
   ↓
lane C (0 or 1 retry)
   ↓
bounded deterministic validators
   ↓
commit
```

Each lane receives the candidate produced by the previous lane and sends only downstream.

This gives you a hard upper bound on calls.

For two retry-capable lanes:

```text
initial call + max 1 retry in A + max 1 retry in B
```

No ping-pong.

---

## 12. Audit receipts

Persist rejection metadata, not rejected prose.

Recommended fields:

```ts
type RejectionReceipt = {
  type: "model_output_rejected";
  schema_version: 1;
  conversation_id: string;
  turn_id: string;
  attempt_id: string;
  provider: string;
  reason: string;
  signals: string[];
  reply_chars: number;
  reply_sha256: string;
  raw_text_persisted: false;
  excluded_from_history: true;
  excluded_from_memory: true;
  retry_planned: boolean;
};
```

Optional:

```text
served model
reasoning effort
provider call index
context reset result
```

Avoid:

```text
raw rejected body
full prompt
private transcript
memory packet body
credentials
```

A hash is useful for correlation, but do not treat it as guaranteed anonymization.

---

## 13. Failure matrix

| Case | Retry? | Persist candidate? | Expected result |
|---|---:|---:|---|
| first candidate passes | no | yes | normal commit |
| first rejects, reset succeeds, second passes | once | second only | commit second |
| first rejects, reset unsupported | no | no | fail closed |
| first rejects, reset throws | no | no | fail closed |
| first rejects, retry provider fails | no more | no | fail closed |
| first rejects, second rejects | no third call | no | fail closed |
| rejected raw text appears in retry request | invalid implementation | no | test failure |
| rejected raw text appears in transcript | invalid implementation | no | test failure |

---

## 14. Hardening checklist

### Persistence

```text
[ ] no assistant_message_persisted before all admissions pass
[ ] no history.push before all admissions pass
[ ] no memory/summary/embedding enqueue before all admissions pass
```

### Rejection

```text
[ ] raw rejected text not persisted
[ ] category + chars + hash sufficient for audit
[ ] retry_planned reflects actual reset success
```

### Provider state

```text
[ ] stateful provider exposes explicit reset capability
[ ] reset failure blocks retry
[ ] stateless retry reconstructs from canonical host state
```

### Retry

```text
[ ] exactly one retry per lane
[ ] recovery block contains no rejected prose
[ ] recovery block contains no private detector details unless needed
[ ] second rejection has no third call
```

### Detector

```text
[ ] full sanitized candidate scanned
[ ] grounding comes from user-authored evidence
[ ] stale history cannot grant unlimited grounding
[ ] keyword / number collisions tested
[ ] weak-only signals tested
[ ] real grounded cases tested to avoid false rejection
```

### Composition

```text
[ ] lanes flow one direction
[ ] provider call indexes remain auditable
[ ] later lane retries never jump back to earlier lane
```

---

## 15. Where to place the seam

The best place is usually:

```text
provider result
→ canonical sanitation
→ admission lanes
→ commit boundary
```

Not:

```text
Telegram / UI renderer
```

because by UI time the text may already be in persistence.

Not:

```text
memory extractor
```

because history may already be contaminated.

Not:

```text
system prompt only
```

because prompts reduce probability; admission enforces a host-owned boundary.

---

## 16. Deployment order

A conservative rollout:

```text
Phase 0  detector-only shadow logging, no rejection
Phase 1  synthetic canary isolation tests
Phase 2  rejection enabled, retry disabled
Phase 3  fresh-context capability verified
Phase 4  one bounded clean retry enabled
Phase 5  multi-guard call-budget tests
Phase 6  production canary / observability review
```

This lets you separate false-positive tuning from retry correctness.

---

## 17. Copyable coding-agent prompt

```text
Implement a pre-persistence LLM output admission lane in the existing chat/agent system.

Goal:
Model output is only a candidate until it passes host-side admission. A rejected
candidate must not become transcript, history, memory, summary, embedding,
delivery, retry input, or durable raw logs.

First inspect the real call chain and find the single canonical assistant
persistence boundary. Do not create a second parallel transcript path.

Required design:

1. Separate pure detector policy from isolation/retry mechanics.
2. Run sanitation before admission.
3. Run admission before assistant persistence/history/memory/delivery.
4. On rejection, record metadata only:
   - reason/category
   - optional categorical signals
   - char count
   - sha256
   - raw_text_persisted=false
   - excluded_from_history=true
   - excluded_from_memory=true
   - retry_planned=<truthful value>
5. Never include rejected raw text in the repair prompt.
6. For stateful providers, retry only after a confirmed fresh-context reset.
7. If reset is unsupported or fails, fail closed without retry.
8. Allow at most one clean retry per lane.
9. If the second candidate rejects, do not make a third call.
10. If multiple guards exist, compose them as one-way lanes; no retry ping-pong.
11. Keep normal clean-candidate behavior unchanged.

Testing:
- clean candidate passes with zero extra calls
- first reject + reset + second pass
- first reject + second reject, no third call
- reset unavailable, no retry
- reset throws, no retry
- canary rejected bytes absent from transcript/history/memory/retry/delivery
- long-tail detector case beyond any former scan window
- stale grounding does not authorize unrelated turns
- harmless lexical/numeric collisions
- genuine grounded case still passes
- weak-only signal does not hard reject
- multi-lane call budget stays bounded

Use only synthetic fixtures in public tests.
Do not copy private prompts or private conversation text into source, tests,
issues, logs, or PR descriptions.

Deliver:
- exact changed files
- persistence boundary before/after
- provider reset contract
- retry state machine
- receipt schema
- focused test results
- full test/typecheck/build results
- rollback path
```

---

## 18. The principle to keep

The implementation details can change.

The invariant should not:

```text
A rejected model draft never becomes a fact merely because the model generated it.
```

Generation proposes.

The host commits.
