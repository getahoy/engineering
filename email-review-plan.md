# Email review: the draft on the Mac, the choices on the phone

Status: planned (2026-11-16). Build right after the screen-aware click work ("send the mail").

## The idea

Every email Ahoy writes becomes a real draft on the Mac (Gmail compose or the mail app, as
today), so anything said later still works. At the same time the phone shows a draft card with
choices, so the person can send, change or leave it from the couch without looking at the Mac.

## Every scenario

| The person | Ahoy |
|---|---|
| "mail Rahul about moving our call to 5" | Opens the draft on the Mac, in front. The phone shows the draft card |
| Taps **Send** on the card, or says its number | Brings the draft to the front if needed and clicks Send. The tap is the confirmation, so there's no second ask. Card: "Sent to Rahul" |
| "don't send", "leave it", or dismisses the card | Nothing is sent. The draft stays open on the Mac (Gmail also keeps it in Drafts). Reply "Left as a draft." |
| Later: "send it", "send the mail", "click send" | The remembered draft: brought to the front, then the screen-aware click on Send, with the usual phone confirm. No remembered draft: the screen-aware click on whatever is in front |
| **Change recipient** (card) or "send it to priya instead" | Hold and say the new address or name. Ahoy closes the old draft, opens a new one with the new To, and updates the card |
| **Change the message** (card) or "make it shorter", "more formal", "add that I'll be late" | The email step rewrites it from the current draft plus the change. The old draft is replaced and the card updates |
| **More** | Add Cc/Bcc, translate, casual or formal, discard |
| **Discard** | Closes and deletes Ahoy's own draft, with no extra confirm because Ahoy made it. Reply "Discarded." |
| Edits the draft by hand on the Mac | Fine. Ahoy treats the open draft as the truth from then on; "send it" sends what's on screen |
| Two drafts open | "send it" means the one in front; if neither is in front, the numbered choice ("1 to Rahul, 2 to Priya") |
| Sent it by hand already, or closed it | "send it" finds no Send button: "I don't see a button for that." |
| 30 minutes later, or after the phone disconnected | The memory is gone; "send it" falls back to the screen-aware click on whatever is in front |
| Mail app instead of Gmail | The same: send and discard through the screen-aware click on the mail app's own buttons |

Never sent without an explicit tap or an explicit "send" from the person.

## The draft card (phone)

- To (and Cc/Bcc when set), the subject, and the message (first lines, tap to expand).
- Choices: **1 Send**, **2 Change recipient**, **3 Change the message**, **4 More**, plus
  **Leave as draft** (closes the card).
- Answer by tapping, or hold and say a number. Holding and saying a change instead ("make it
  shorter") goes straight to Change the message.
- The card stays until it's acted on, dismissed, or a new command starts. It doesn't use the
  20-second choice timer.
- Design: Claude Design prompt below, in the app's existing style.

## How it works

- **Draft memory (Mac):** the last 3 drafts Ahoy made (to, cc, bcc, subject, body, which app,
  which browser tab or window), in memory only, for 30 minutes. Never written to disk, never in
  History, never logged, and cleared when the phone disconnects. History keeps only "Drafted an
  email to Rahul".
- **Replacing a draft:** close the old one through its own Discard button (screen-aware click;
  no confirm for Ahoy's own draft), then open the new compose as today.
- **Sending:** bring the draft's tab or window to the front (tab switch by its compose title),
  then the screen-aware click on Send.
- **Rewriting:** the email step gets a revise mode: the current draft plus the requested change
  give the new draft. Same rules: the person's words win, nothing invented, the greeting check
  stays.
- **Protocol (additive):** a Mac → phone frame with the draft for the card, and a phone → Mac
  frame for the choice (send, discard, revise with the change text, change recipient with the
  spoken address).

## Later (with agent mode)

The same card pattern for anything that needs a look before acting: a Slack or WhatsApp
message, a form, a payment.

## Decided for later: an explicit "send" skipping the confirm

Idea: when the person says "send" outright ("send the mail"), the click on Send could skip the
phone confirm, since the words were the confirmation. Parked on 2026-11-17, no code: for now
Send stays on the destructive list and always asks "Send can't be undone. Confirm on your
phone." Revisit with the draft card, where a tap on **Send** already counts as the confirm.

## Checks (to write with the build)

One row per scenario in the table above, run from the phone with real addresses.

## Claude Design prompt (for the card)

> Ahoy drafts an email from what the person said. Design the phone card that shows the draft
> and lets them act on it from across the room: send it, change who it goes to, change the
> message, more (Cc/Bcc, translate, tone, discard), or leave it as a draft. It must be easy by
> voice too (numbered choices, like our choice sheet). Match the existing Ahoy iPhone design.
> Show: the card with a short draft, a long draft, after "change the message" (updating), sent,
> and left as a draft. Light and dark.
