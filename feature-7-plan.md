# Feature 7: Matching (plan)

**Exit:** saying "scroll down" and "click Save" works on whatever is on screen, and a
numbered overlay appears when Ahoy is unsure.

**Order for the 31st:** talk button redo → slim LLM tier → click and resolver → choices
and overlay → safety. If time runs short, the breaker and the blocklist slip a few days;
confirm-before-destructive never slips.

**Slim LLM tier first** (the `/api/interpret` plan): the AI proposes up to 3 actions from a
closed set (open, quit, hide, switch, search, openSite, keys, type); the Mac validates and
runs them. Step 7 then adds click as one more action the AI can choose.

## What gets built

1. **Grammar** (`agent/core/commands`, tests): "scroll up/down", "page up/down", "go to
   top/bottom", "click / press / tap {label}", added to the feature 6 parser. Scrolling is
   Page Down / Up through the existing `key` verb; note which apps ignore it during testing,
   a `scroll` verb comes later only if needed.
2. **Resolver** (`agent/core/resolver`, pure, tests): from the helper snapshot keep only
   elements that are enabled, visible, inside the window and not zero-size. Match the
   spoken label against title, description, help, placeholder and subrole, case-insensitive,
   with "…" normalised; tiers exact → whole word → contains. One match → click. Several →
   choices. None → miss.
3. **Before every click:** two identical snapshots ~100 ms apart, same frontmost app as when
   matching started, target re-checked (same label, same place) right before clicking.
   Changed → say so, never click. Click with `AXPress` first, coordinates only as fallback.
4. **Sparse trees (Chrome, Electron):** when a snapshot comes back under 10 elements, the
   helper sets `AXManualAccessibility` and retries; `AXEnhancedUserInterface` only as a
   fallback, since it can disturb window animations.
5. **Numbered overlay** (Swift helper `overlay` verb): non-activating and click-through, so it
   never takes focus. Stays until you pick or cancel, at most 20 s, and clears at once if the
   frontmost window changes.
6. **Choosing** (protocol + phone + Mac): `command.result` carries
   `choices: { number, label }[]` and `confirm: boolean`. The phone shows them as big
   numbered buttons in a bottom sheet matching the Mac's numbers. A pick is
   `command.choose { commandId, parentId, number }`: **a new `commandId` with the original as
   `parentId`**, never the original id, because the Mac dedupes ids. The Mac acts only if the
   screen is still the same window. Saying "number 2" comes later. The phone's choice
   sheet closes when the Mac overlay expires (20 s); a late tap gets "The screen changed,
   say it again", never a click.
7. **Confirm is a one-option choice with `confirm: true`**, not a separate flow. Destructive
   labels always confirm, matched case-insensitively as whole words: Delete, Remove, Erase,
   Discard, Trash, Empty Trash, Format, Uninstall, Don't Save, Sign Out, Log Out, Close
   Without Saving, Send, Pay, Buy, Purchase, Place Order, Transfer, Move to Trash, Delete
   Forever, Permanently Delete, Reset, Replace, Overwrite, Force Quit, Shut Down, Restart,
   Leave, Unsubscribe.
8. **Safety:** default app blocklist (password managers, Keychain, banking); breaker at 30
   actions a minute or 5 failures in a row, a tap to resume; never type into secure fields.

Not in 7: menu commands (10), screenshots/OCR (11), the full LLM tier with provider
fallback (12), saying "number 2". The slim LLM tier above comes before 7.

## Decisions taken

| # | Question | Answer |
|---|---|---|
| 1 | Scroll | Page Down / Up via `key` for now; `scroll` verb later if needed |
| 2 | Choosing | Tap on the phone |
| 3 | Overlay | Until pick or cancel, max 20 s, cleared when the frontmost window changes; non-activating, click-through |
| 4 | Destructive labels | The list in item 7 |
| 5 | Electron / Chrome | Automatic on sparse trees: `AXManualAccessibility`, then `AXEnhancedUserInterface` |
| 6 | Breaker | 30 a minute, stop after 5 failures in a row |
| 7 | "Push pause" | The talk button: a true hold |
| 8 | Push feature 6 | Backup on `feature/talk-voice-commands` |
