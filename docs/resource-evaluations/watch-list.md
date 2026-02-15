# Watch List

Resources monitored but not yet integrated into the guide. Event-driven re-evaluation (not time-based).

## Lifecycle

```
New resource → /eval-resource
  Score >= 3 but not ready → Active Watch
  Score < 3               → Dropped (or not listed)

Trigger reached → re-evaluation → Integrate (Graduated) / Drop (Dropped)
```

## Active Watch

| Resource | Type | Added | Why Watching | Re-eval Trigger |
|----------|------|-------|--------------|-----------------|
| [ICM](https://github.com/rtk-ai/icm) | MCP | 2026-02-12 | Pre-v1 (1 star, 11 commits) | First release + >20 stars |
| [System Prompts](https://github.com/x1xhlol/system-prompts-and-models-of-ai-tools) | Tool | 2026-01-26 | Redundant with official sources. Re-evaluated 2026-02-13 (Opus 4.6 update): still 2/5. | Anthropic confirms CLI prompts not published |
| [o16g — Outcome Engineering](https://o16g.com/) | Manifesto | 2026-02-13 | Emerging framework by Cory Ondrejka (CTO Onebrief, co-creator Second Life, ex-VP Google/Meta). 16 principles for shifting from "code writing" to "outcome engineering". Honeycomb endorsement. No Claude Code-specific content yet. Memetic potential (naming follows i18n/k8s pattern). | Term adopted in >3 independent AI engineering resources OR author publishes tool-specific implementation |

## Graduated

Resources that moved from watch to integrated in the guide.

| Resource | Graduated | Evaluation |
|----------|-----------|------------|

## Dropped

Resources removed from watch after re-evaluation.

| Resource | Dropped | Reason |
|----------|---------|--------|
