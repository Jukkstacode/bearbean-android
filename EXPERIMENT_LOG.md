# Experiment Log — <ARM: Claude Code | Gemini-in-Android-Studio>

> One copy of this file lives on each build branch (`build/claude`, `build/gemini`). Fill in
> the arm name above and log as you go. Keep entries to bullets — speed matters more than prose.

**Arm (the variable — model):** <Gemini `<version>` | Claude `<version>`>
**Harness (held constant):** Android Studio Gemini assistant, [Agent | Chat] mode — same for
  both arms; Claude is plugged in via "use a remote model" (`https://api.anthropic.com`).
**Build order:** this arm was built [FIRST | SECOND].
  _Learning-effect note: the SECOND-built arm benefits from prior knowledge regardless of the
  model. Build the model you expect to be weaker FIRST so the bias works against your favorite,
  not for it. Disclose the order in COMPARISON.md._
**Spec:** AGENTS.md — identical across both arms. Do not copy code from the other arm.
**Residual harness note:** record any Gemini-only feature unavailable to Claude-as-remote.

## Running tally (update as you go)

| Count | Value |
|---|---|
| Active build time (sum of session durations) | |
| Total prompts / turns | |
| Build failures encountered | |
| …resolved by the assistant unaided | |
| …I had to fix manually | |
| Hallucinated / wrong API calls | |
| Times it ignored AGENTS.md (wrong base URL, short timeout, etc.) | |

## Milestone timestamps (also tag them in git: `<arm>-<milestone>`)

| Milestone | Time reached | Git tag |
|---|---|---|
| Project scaffolded / builds | | |
| Networking layer + /health works | | |
| Chat screen works against live backend | | |
| Facts browser works (list + filter + delete) | | |
| MVP complete | | |

## Session entries

### Session N — YYYY-MM-DD (HH:MM–HH:MM)
- **Goal:**
- **Asked:**
- **Assistant did:**
- **Stumbles / friction:**
- **Manual fixes I made:**
- **Notes / quotes worth keeping:**
