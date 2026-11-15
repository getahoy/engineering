# Eval scores

The last recorded score per set, updated after each end-of-batch run (see `CLAUDE.md`, AI
evals). A set that drops below its row is rerun twice more and judged on the middle score.

| Set | Score | Recorded | Notes |
|---|---|---|---|
| Tuning | 65/68 | 2026-11-15 | Middle of 3, email step wired. Long-standing misses: "find on this page", "search in notion" |
| Held-out | 40/41 | 2026-11-15 | Middle of 3. Long-standing miss: "press command q" (the Mac's shortcut filter blocks it anyway) |
| Mac actions | 47/48 | 2026-11-15 | Middle of 3 (48, 47, 45 with two API errors) |
| Addresses | 9/10 | 2026-11-15 | Middle of 3; the miss is a flaky empty answer for "fourteen oh four" that passes 3/3 when called directly |
| Email | 11/15 | 2026-11-15 | Middle of 3. Misses: a name guessed from the address ("Hi team,", "Hi Dev,"), 2-word subjects |
