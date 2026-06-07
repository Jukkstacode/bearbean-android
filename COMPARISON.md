# BearBean Android — Model Comparison Report

Same harness, same spec, same scope, **two different models**. Fill this in at the end, pulling
numbers from each arm's `EXPERIMENT_LOG.md` and the git history.

## Setup & limitations (read first — this is what makes the result credible)

- **Held constant:** the **harness** (Android Studio's Gemini assistant — same Chat/Agent mode
  for both, via "use a remote model"), the spec (`AGENTS.md`), the MVP scope (chat screen +
  facts browser), and the target devices.
- **The variable:** the **model** only —
  - **Gemini** `<version>` (native in Android Studio), vs.
  - **Claude** `<version>` (remote model via `https://api.anthropic.com` + Anthropic API key).
- **Same assistant mode for both:** [Agent / Chat — must match]. Note any Gemini-only feature
  Claude-as-remote lacked (per Android Studio's "some specialized features may not function"
  caveat); that's the one residual harness difference to disclose.
- **Build method:** two **sequential** builds. **`<Claude / Gemini>` was built first.** The
  second build benefits from a learning effect (I already knew the solution) — this is the
  primary limitation of an n=1 study. State which arm got that tailwind.

## Metrics

| Metric | Claude Code | Gemini-in-AS | Notes |
|---|---|---|---|
| Time to working MVP | | | |
| Prompts / turns | | | |
| Build failures encountered | | | |
| …fixed by assistant unaided | | | |
| …fixed manually by me | | | |
| Hallucinated / wrong APIs | | | |
| Ignored-the-spec incidents | | | |
| AGENTS.md architecture adherence (1–5) | | | |
| Idiomatic Kotlin / Compose (1–5) | | | |
| Quality of explanations (1–5) | | | |
| Caught *my* mistakes (1–5) | | | |
| Error recovery (1–5) | | | |

## Code divergence

From an identical scaffold baseline:

```
git diff --stat build/claude..build/gemini
```

- Files where the two diverged most:
- Notable architectural differences:
- Which implementation hews closer to `AGENTS.md`:

## Narrative

### Where Claude Code was stronger

### Where Gemini-in-Android-Studio was stronger

### Surprises

### Verdict
_Given the learning-effect caveat, the defensible conclusion is…_
