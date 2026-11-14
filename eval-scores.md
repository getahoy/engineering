# Eval scores

The last recorded score per set, updated after each end-of-batch run (see `CLAUDE.md`, AI
evals). A set that drops below its row is rerun twice more and judged on the middle score.

| Set | Score | Recorded | Notes |
|---|---|---|---|
| Tuning | 65/68 | 2026-11-15 | End of the email batch. Long-standing misses: "find on this page", "search in notion", "what's the weather in bangalore" (drops "in") |
| Held-out | 39/41 | 2026-11-15 | Dropped from 40; reruns 40 and 39, middle 39. Misses: "press command q" (always, the shortcut filter blocks it anyway), "thoda aur" (the Mac's rules catch it first) |
| Mac actions | 47/48 | 2026-11-15 | Miss: "neeche karo" read as volume down (the Mac's rules catch it first) |
| Addresses | 10/10 | 2026-11-15 | |
| Email | 14/15 | 2026-11-15 | Miss: "Hi team," for team@ — correct under the new greeting rule since "team" was said; the case needs updating |
