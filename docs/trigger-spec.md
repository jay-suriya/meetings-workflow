# Build: the trigger step of the Meetings workflow-rule builder

You are working in the Zoho CRM codebase. This brief covers **one thing**: the *when* step of
a workflow rule on the **Meetings** module — the part that decides what event starts the rule.

**The condition step and the action step are out of scope. Do not touch them.** They already
work and are expected to keep working exactly as they do today: whatever the trigger is, the
rule then narrows through the existing criteria builder and runs the existing instant and
scheduled actions. Your changes should add and reshape trigger options only, and must leave
every downstream step behaving as before.

Everything you need is in this document. There is nothing else to read.

---

## Background you need first

Meetings in CRM can record **check-in** and **check-out**: a host physically arriving at a
customer's location taps Check In on the record, and the system stores the time, the address,
the coordinates and the distance from the meeting's own Location. When they leave, they check
out, and the gap between the two is the time they spent on site. The point of the feature is
that none of it is typed by hand — it is recorded by the system at the moment it happens, so
a visit has evidence rather than a claim.

Five facts about it shape every trigger below.

**Only the host checks in.** On Meetings the structure is fixed: check-in is anchored to the
meeting's From and To, validated against its Location, and **the host is the only person
eligible**. A meeting therefore holds at most one check-in. Anything in the wider feature that
deals with several people checking in — a partially-checked-in state, counts of who has and
has not arrived, one row per person — does not apply to Meetings and must not appear here.
This is why the trigger names the host rather than naming the check-in.

**The window.** Check-in is not possible at any time. Each meeting has a window that opens a
configured amount *before* its start time and closes a configured amount *after* its end time
— an hour either side, typically. Inside the window a capture is allowed; outside it,
impossible. The window is the clock behind the "missed" triggers, and it is not the same as
the meeting time: a meeting that ran 10:00–11:00 can still be checked into at 11:45.

**The status.** Every meeting carries a *Check-In/Out Status* field with five values it can
reach: **Not Checked In**, **Checked In**, **Checked Out**, **Missed Check-In** and
**Missed Check-Out**. The same field, under the same name, is shown on the record header, in
the list view, in Kanban, in filters and on the calendar.

- *Missed Check-In* means the window closed and nobody ever checked in.
- *Missed Check-Out* means the host checked in but was still checked in when the window
  closed, so their time on site was never measured.
- The status is **derived, not stored** — it recomputes whenever the record changes. A meeting
  that reads Missed Check-In and is then rescheduled into the future stops being missed,
  because its new window has not closed yet.

**Check-out is optional.** An administrator decides per layout whether check-out is captured
at all. When it is off, every check-out option below must be absent.

**Captures can be undone, in two different ways, and the difference matters.**

- *Revert* removes one capture. The host can undo their own check-in or check-out any time
  until the window closes. While a meeting holds both, only the check-out can be reverted;
  removing it makes the check-in revertable again. Reverting a check-out also empties the
  duration, since that is only calculated at check-out. Nothing notifies anyone — it is
  written only to the record's timeline.
- *Clear* destroys everything. Editing a meeting's Location, start time, end time or host when
  it already holds captures raises a prompt asking whether to keep or clear the check-in data.
  Choosing **Clear it** discards the times, the addresses, the coordinates, the distances and
  every captured answer, for **both halves together**, and returns the meeting to Not Checked
  In. There is no way to clear only the check-out. This is why there is a single
  cleared-data trigger rather than one per half.

---

## The screen

The step opens with one question — **"Execute this workflow rule based on"** — offering
**Record Action**, **Date/Time Field** and **Record Notes**.

**Date/Time Field** runs the rule relative to a date on the record rather than in response to
an event: pick the field, then On / Before / After it, then the time of day to run at. The
field list should include the meeting's own **From** and **To**, the record's Created and
Modified times, and — because the feature adds them — **Check-In Time** and **Check-Out
Time**.

**Record Action** is where the work is. Choosing it reveals a second picklist naming what the
rule watches, and further controls that depend on it. There are five subjects:

1. When a meeting
2. When a meeting participant
3. When a meeting Invite
4. **When the meeting host** ← the check-in and check-out triggers
5. When any action happens in a meeting

Each combination must produce a plain-English sentence, shown once the step is answered, so
the user can read back what they built.

---

### 1. When a meeting is…

The record's own lifecycle: **Scheduled**, **Canceled**, **Rescheduled**, **Modified**,
**Completed**, **Deleted**.

*Scheduled* is the record-created event — a meeting comes into existence by being scheduled.
*Completed* fires after the meeting's end time, which is the natural moment to write the
outcome back to the customer record and raise whatever follows.

*Modified* needs one more choice: **any field gets modified**, or **specific field(s) gets
modified** — the latter revealing a field picker and an "is modified to [value]" row.

---

### 2. When a meeting participant is…

Two options — **Added** and **Removed** — each with a second picklist.

**Added** asks when: *While a meeting is scheduled*, *After the meeting is scheduled*, or
*Anytime*.

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

The catch-all drops the clause from the sentence entirely — *"…is Removed."* — rather than
producing the ungrammatical *"…who had any reply status."*

---

### 3. When a meeting Invite…

The invite's reply state: **Is Accepted**, **Is Rejected**, **Is Replied May be**,
**Got No Reply**. Each is followed by **by [threshold] of the Participants** — All, 10%, 25%,
50%, 75%, or a custom percentage.

**Got No Reply** reveals one more row, because silence only means something relative to a
clock:

> within **[N] [Minute(s) | Hour(s) | Day(s)]** **[after the invite was sent | before the
> meeting starts]**

Both anchors are needed, and neither substitutes for the other. *After the invite was sent*
measures **responsiveness** — it is how you express a policy like "people should reply within
48 hours of being invited". *Before the meeting starts* measures **risk to the meeting** — it
chases while there is still time to act.

The reason for offering both is a real failure in each direction. Anchored only to the invite,
a meeting booked an hour before it happens gets its 24-hour chase a day *after* the meeting is
over. Anchored only to the start, an invite sent three months ahead can never enforce a
48-hour reply policy.

---

### 4. When the meeting host has…

This subject carries every check-in and check-out trigger in **one list**. The connector reads
**has**, so the row is *"When the meeting host has […]"* and each option completes it.

| # | Option | Extra control it reveals | What it is for |
|---|---|---|---|
| 1 | **checked-in** | — | The host arrived and confirmed on site. The meeting reads Checked In. |
| 2 | **check-in after** | `[0] [Minute(s) \| Hour(s)] from the meeting start time` | A late arrival. `0` means any check-in after the meeting has begun; a larger number is the grace period before lateness counts. |
| 3 | **checked-out** | — | The visit is finished and its duration is now known. |
| 4 | **checked-out before** | `[0] [Minute(s) \| Hour(s)] before the meeting end time` | They left before the meeting was due to end. |
| 5 | **missed check-in** | — | The window closed and nobody checked in. |
| 6 | **missed check-out** | — | Still checked in when the window closed, so time on site was never measured. |
| 7 | **reverted check-in** | — | The host undid their own check-in. Evidence of the visit was deleted and only the timeline records it. |
| 8 | **reverted check-out** | — | The check-out was undone: the meeting returns to Checked In and its duration is emptied. |
| 9 | **cleared checkin/checkout data** | — | An edit to the Location, start, end or host was saved with *Clear it*, destroying every captured value on the record. |

Options 3, 4, 6 and 8 appear only when check-out is being captured; with it off, the list is
five entries.

Three things to get right here:

- **Lateness and earliness are typed numbers, not fixed rules.** The window already decides
  whether a capture is *allowed*, but says nothing about whether it was *punctual*: inside the
  window, a check-in five minutes early and one fifty minutes late are both simply Checked In.
  The threshold is policy — fifteen minutes for one team, an hour for another — so the user
  types it.
- **A check-out is never validated against the schedule.** Leaving early is *measured*, not
  policed: option 4 reports a departure, it does not prevent one.
- **There is one cleared-data option, not two.** Clearing removes whole check-ins, both halves
  together, so a per-half pair would imply a distinction that does not exist.

---

### 5. When any action happens in a meeting

A catch-all with no further options.

---

## Interaction details

- The two timed options — *check-in after* and *checked-out before* — **reset their number to
  0** when newly selected, but a value the user has typed must survive re-picking the same
  option. (Reading the number only at save time is the obvious bug here: re-rendering the row
  after any later change then silently discards what was typed.)
- Switching subject must clear every dependent control belonging to the previous one, not
  merely hide the row it sat in.
- Nothing in this step may change what the condition or action steps receive. A rule built
  from any trigger above must flow into the existing criteria builder and the existing action
  menus unchanged.

## Two naming rules, both learned the hard way

1. **Keep the product's vocabulary in the option text.** The user sees *Missed Check-In* on
   the record, in the list, in Kanban and in every filter, so the trigger that fires on it must
   contain those words. Earlier drafts used phrasings that read better in isolation and left
   the workflow builder as the only place in CRM calling the same thing something else.
2. **Name the actor, not the artefact.** "When a meeting check-in" was tried and abandoned:
   it names a thing that only ever has one actor, and once check-out options were added to the
   same list the label claimed one half while the menu held both. Naming the host is both
   truer and shorter.
