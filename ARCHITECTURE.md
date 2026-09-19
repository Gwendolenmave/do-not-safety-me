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
guard D