# Responsible Use

This repository describes an application-side output admission pattern for systems you own or are explicitly authorized to modify.

Its purpose is to separate **model generation** from **host commitment**: a locally rejected candidate should not automatically become transcript, memory, retry input, or delivered output.

## Expected practices

- Keep provider safeguards, access controls, and platform rules intact.
- Use output guards to enforce your own product's admission rules, not to bypass third-party moderation or authorization.
- Distinguish genuine user-grounded emergency / safety needs from unwanted or out-of-context escalation.
- In high-stakes domains, use appropriate expert review and domain-specific policies rather than relying on a small regex or toy detector.
- Keep rejected raw text out of durable logs unless there is a clear, consented, security-reviewed reason to retain it.
- Treat hashes as correlation metadata, not as guaranteed anonymization.
- Use synthetic fixtures in public repositories, issues, screenshots, and CI logs.
- Bound retries and provider calls so a detector failure cannot create an uncontrolled regeneration loop.
- Make provider context-reset capability explicit and fail closed when a clean retry cannot be proven.
- Preserve an audit trail of categorical outcomes without exposing private conversation content.

## Uses this project is not intended to support

- Circumventing provider safety systems, access controls, moderation, or authentication
- Suppressing genuine, user-established emergency needs merely to make an assistant sound less cautious
- Hiding policy violations or unsafe outputs from operators while continuing to deliver them elsewhere
- Collecting, retaining, or publishing private conversation data without authorization
- Manipulating users through undisclosed or deceptive filtering
- Unauthorized modification of third-party bots, accounts, or infrastructure

The repository name is tongue-in-cheek. The engineering pattern is about **host-owned commit boundaries**, not disabling safety.

This document describes the maintainer's intended and responsible use of the project. It does not replace or modify the repository license.
