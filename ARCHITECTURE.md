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

This is intentionally provider-neutr