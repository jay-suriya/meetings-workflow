# Build: the trigger step of the Meetings workflow-rule builder

You are working in the Zoho CRM codebase. This brief covers **one screen**: the *when* step
of a workflow rule on the **Meetings** module — the part that decides what event starts the
rule. The condition and action steps that follow it are out of scope here.

Everything you need is in this document. There is nothing else to read.

---

## Background you need first

Meetings in CRM can now record **check-in** and **check-out**: a host physically arriving at
a customer's location taps Check In on the record, and the system stores the time, the
address, the coordinates and the distance from the meeting's own Location. When they leave,
they check out, and the gap between the two is the time they spent on site. The point of the
feature is that none of it is typed by hand — it is recorded by the system at the moment it
happens, so a visit has evidence rather than a claim.

Five facts about it shape every trigger below.

**The window.** Check-in is not possible at any time. Each meeting has a window that opens a
configured amount *before* its start time and closes a configured amount *after* its end
time — an hour either side, typically. Inside the window a capture is allowed; outside it,
impossible. The window is the clock behind most of these triggers, and it is not the same as
the meeting time: a meeting that ran 10:00–11:00 can still be checked into at 11:45.

**The status.** Every meeting carries a *Check-In/Out Status* field with five values it can
reach: **Not Checked In**, **Checked In**, **Checked Out**, **Missed Check-In** and
**Missed Check-Out**. This same field, under this same name, is shown on the record header,
in the list view, in Kanban, in filters and on the calendar — so the trigger list must use
these exact words and not invent alternatives.

- *Missed Check-In* means the window closed and nobody ever checked in.
- *Missed Check-Out* means someone checked in but was still checked in when the window
  closed, so their time on site was never measured.
- The status is **derived, not stored** — it recomputes whenever the record changes. A
  meeting that reads Missed Check-In and is then rescheduled into the future stops being
  missed, because its new window has not closed yet.

**Only the host checks in.** On Meetings, the host is the only person eligible, so a meeting
holds at most one check-in. Anything in the wider feature that deals with several people
checking in — a partially-checked-in state, counts of who has and has not arrived, one row
per person — does not apply to Meetings and must not appear in this UI.

**Check-out is optional.** An administrator decides per layout whether check-out is captured
at all. When it is off, no check-out trigger, and no checked-out status value, should be
offered anywhere.

**Captures can be undone, in two different ways, and the difference matters.**

- *Revert* removes one capture. The host can undo their own check-in or check-out any time
  until the window closes. While a meeting holds both, only the check-out can be reverted;
  removing it makes the check-in revertable again. Reverting a check-out also empties the
  duration, since that is only calculated at check-out. Nothing notifies anyone — it is
  written only to the record's timeline.
- *Clear* destroys everything. Editing a meeting's Location, start time, end time or host
  when it already holds captures raises a prompt asking whether to keep or clear the
  check-in data. Choosing **Clear it** discards the times, the addresses, the coordinates,
  the distances and every captured answer, for **both halves together**, and returns the
  meeting to Not Checked In. There is no way to clear only the check-out. This is the most
  destructive single action in the feature and, like revert, it tells nobody.

---

## The screen

The step opens with one question — **"Execute this workflow rule based on"** — offering
**Record Action**, **Date/Time Field** and **Record Notes**.

**Date/Time Field** runs the rule relative to a date on the record rather than in response
to an event: pick the field, then On / Before / After it, then the time of day to run at.
The field list should include the meeting's own **From** and **To**, the record's Created
and Modified times, and — because the feature adds them — **Check-In Time** and
**Check-Out Time**.

**Record Action** is where the work is. Choosing it reveals a second picklist naming what the
rule watches, and a third whose contents depend on the second. There are six subjects.

Each combination must produce a plain-English sentence, shown once the step is answered, so
the user can read back what they built — for example *"This rule will be executed when a
meeting's Check-In/Out Status is Missed Check-In."*

---

### 1. When a meeting is…

The meeting record's own lifecycle: **Scheduled**, **Canceled**, **Rescheduled**,
**Modified**, **Completed**, **Deleted**.

*Scheduled* is the record-created event — a meeting comes into existence by being scheduled.
*Completed* fires after the meeting's end time, which is the natural moment to write the
outcome back to the customer record and raise whatever follows.

*Modified* needs one more choice: **any field gets modified**, or **specific field(s) gets
modified** — the latter revealing a field picker and an "is modified to [value]" row, so a
rule can watch one field changing to one value.

Two of these are worth pairing with check-in in the user's mind, though they need no new
options: *Rescheduled* on a meeting that already holds a check-in means the recorded visit no
longer matches the schedule, and *Canceled* is how a meeting that was about to be marked
missed quietly stops being missed.

---

### 2. When a meeting participant is…

Two options — **Added** and **Removed** — each with a second picklist.

**Added** asks when: *While a meeting is scheduled*, *After the meeting is scheduled*, or
*Anytime*. The first means they were on the invite from the start; the second means they
were brought in later, which is usually a sign the meeting changed shape.

**Removed** is followed by a standing label — the words **who had** rendered *outside* the
picklist, not repeated inside every option — and then the reply the person had given before
they were taken off:

| Option | What it catches |
|---|---|
| accepted the invite | Someone who had confirmed they were coming has been cut. Either a mistake or a deliberate removal, and the organiser usually wants to know. |
| rejected the invite | They had already declined, so removing them is routine tidying. Separating this is what stops the rule crying wolf. |
| replied Maybe | An undecided attendee dropped before they made up their mind. |
| not replied | Removed before they ever answered — the invite may have gone to the wrong person. |
| any reply status | Every removal, whatever they had answered. The default. |

The catch-all should drop the clause from the sentence entirely — *"…is Removed."* — rather
than producing the ungrammatical *"…who had any reply status."*

---

### 3. When a meeting Invite…

The invite's reply state: **Is Accepted**, **Is Rejected**, **Is Replied May be**,
**Got No Reply**. Each is followed by **by [threshold] of the Participants**, so a rule can
require all of them or a proportion — All, 10%, 25%, 50%, 75%, or a custom percentage.

**Got No Reply** reveals one more row, because silence only means something relative to a
clock:

> within **[N] [Minute(s) | Hour(s) | Day(s)]** **[after the invite was sent | before the
> meeting starts]**

Both anchors are needed, and neither substitutes for the other:

- *after the invite was sent* measures **responsiveness** — it is how you express a policy
  like "people should reply within 48 hours of being invited".
- *before the meeting starts* measures **risk to the meeting** — it chases while there is
  still time to act.

The reason for offering both is a real failure in each direction. Anchored only to the
invite, a meeting booked an hour before it happens gets its 24-hour chase a day *after* the
meeting is over — useless. Anchored only to the start, an invite sent three months ahead can
never enforce a 48-hour reply policy.

---

### 4. When a meeting check-in is…

| Option | Extra control it reveals | What it is for |
|---|---|---|
| **Checked In** | — | The host arrived and confirmed on site. Stamp the customer record, tell the customer their rep has arrived. |
| **Check-In After** | `[0] [Minute(s) \| Hour(s)] from the meeting start time` | A late arrival. `0` means any check-in after the meeting has begun; a larger number is the grace period before lateness counts. |
| **Missed Check-In** | — | The window closed and nobody checked in. The same-day exception — re-book while the day can still be saved. |
| **Check-In Reverted** | — | The host undid their own check-in. Evidence of the visit was deleted and only the timeline records it. |
| **Check-In Data Cleared** | — | An edit to the Location, start, end or host was saved with *Clear it*, destroying every captured value on the record. |

A note on why **Check-In After** is a trigger rather than something the user expresses as a
condition: the window already decides whether a capture is *allowed*, but it says nothing
about whether it was *punctual*. Inside the window, a check-in five minutes early and one
fifty minutes late are both simply "Checked In". Lateness has to be asked for explicitly, and
the threshold is policy — fifteen minutes for one team, an hour for another — which is why it
is a number the user types rather than a fixed rule.

---

### 5. When a meeting check-out is…

Offered only when check-out is being captured.

| Option | Extra control it reveals | What it is for |
|---|---|---|
| **Checked Out** | — | The visit is finished and its duration is now known — the moment for follow-up tasks and the visit summary. |
| **Check-Out Before** | `[0] [Minute(s) \| Hour(s)] before the meeting end time` | They left before the meeting was due to end. |
| **Missed Check-Out** | — | Still checked in when the window closed, so time on site was never measured. Every one of these is a hole in the reporting. |
| **Check-Out Reverted** | — | The check-out was undone: the meeting returns to Checked In and its duration is emptied. |
| **Check-Out Data Cleared** | — | The same destructive edit as above, seen from the check-out side. |

Two things to encode, in help text or in tests:

- **Check-Out Data Cleared and Check-In Data Cleared fire at the same moment.** Clearing
  removes whole check-ins, both halves together, so there is no edit that clears only a
  check-out. They are one event seen from either side, and a user should not build a rule
  expecting them to be distinct.
- **A check-out is never validated against the schedule.** Leaving early is *measured*, not
  policed — Check-Out Before reports a departure, it does not prevent one.

---

### 6. When any action happens in a meeting

A catch-all with no further options.

---

## Interaction details

- The two timed options — Check-In After and Check-Out Before — **reset their number to 0**
  when newly selected, but a value the user has typed must survive re-picking the same
  option. (Reading the number only at save time is the obvious bug here: re-rendering the row
  after any later change then silently discards what was typed.)
- The four "event" options — the two Reverteds and the two Data Cleareds — read as complete
  predicates, so the row should drop the *is* connector for them: *"when a meeting has its
  check-in reverted"*, not *"is Check-In Reverted"*.
- Switching subject must clear every dependent control belonging to the previous one, not
  merely hide the row it sat in.

## Two naming rules, both learned the hard way

1. **Use the status names exactly as the product shows them.** Earlier drafts of this screen
   used phrasings like "doesn't check in" instead of *Missed Check-In*. It reads better in
   isolation and is wrong, because the user sees *Missed Check-In* on the record, in the
   list, in Kanban and in every filter — the workflow builder would be the only place in CRM
   calling it something else.
2. **Check-in and check-out are separate subjects.** They were combined once, under a single
   heading named for check-in, and it misled people twice over: the label claimed one half
   while the menu held both, and ten options in one list was too many to scan. Two subjects
   of five each, both named honestly, is the shape that worked.
