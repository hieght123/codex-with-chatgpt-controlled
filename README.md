# Codex with ChatGPT Controlled Profile

This is a publishable, environment-neutral Skill profile for a read-only collaboration loop:

1. Web ChatGPT plans or reviews.
2. Local Codex makes changes and runs verification.
3. Web ChatGPT reviews the resulting diff and test evidence.

The profile deliberately separates review authority from execution authority. Web ChatGPT receives only read-only workspace capabilities. Local Codex remains the only actor that can change files, run commands, or create commits.

## What this profile hardens

- Uses the Codex in-app browser only.
- Avoids coordinate-based clicks and unstable send-button selection.
- Separates prompt fill, Enter submission, and reply collection into independent operations.
- Checks for an in-progress generation and unsent user draft before writing to the composer.
- Uses bounded retries and a circuit breaker instead of unbounded recovery loops.
- Treats a successful local process start as insufficient; the workflow must verify a usable end-to-end connector before declaring readiness.

## Before publishing

- Set the generic runtime configuration required by your Codex installation.
- Add upstream attribution and the applicable license notice after confirming the source license.
- Do not add real workspace names, absolute paths, connector URLs, credentials, logs, screenshots, or local configuration backups.

## Contents

- `skill/SKILL.md`: the environment-neutral controlled C2C profile.
- `.gitignore`: prevents common local secrets and generated state from entering version control.
- `NOTICE.md`: release and attribution checklist.

