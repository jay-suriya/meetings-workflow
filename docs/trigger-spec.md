# Build: Meetings workflow-rule triggers for Check-In / Check-Out

You are working in the Zoho CRM codebase. Extend the **Workflow Rules builder for the
Meetings module** so it can express the triggers, conditions and actions below.

Source of truth for product behaviour is the **Check-In / Check-Out PRD** (section numbers
below refer to it). A working visual mock of everything described here exists at
github.com/jay-suriya/meetings-workflow → `meeting.html` — read it for the exact option
labels, ordering and summary wording, but implement against the real components and design
system, not the mock's markup.

## Ground rules that shape everything

1. **Use the PRD's own names.** The check-in status values — Not Checked In, Checked In,
   Checked Out, Missed Check-In, Missed Check-Out — appear on the record header, the list
   view, Kanban, filters and the calendar (§3.1). The builder must not invent different
   words for them.
2. **Offer only reachable values** (§3.1.3, §8.6). On Meetings, only the Host can check in
   (§2.2.5), so it is permanently the one-check-in case: `Partially Checked In`, the four
   count fields and the per-person rows of §3.3–3.4 never apply here and must not appear.
3. **Check-out is optional** (§2.2.3). Gate every check-out trigger, status value and the
   whole check-out subject behind whether check-out is captured for the layout.
4. **Missed is derived from the window, not the meeting time** (§6.10). The window opens a
   configured amount before `From` and closes a configured amount after `To`.
5. Every trigger must produce a **plain-English summary sentence** shown once the step is
   answered, e.g. "This rule will be executed when a meeting's Check-In/Out Status is
   Missed Check-In."

---

## Stage 1 — WHEN

The rule is based on one of: **Record Action**, **Date/Time Field**, **Record Notes**.

### Date/Time Field base

Offer the module's date/time fields — Created Time, Modified Time, Check-In Time,
**Check-Out Time**, From, To — with On / Before / After, and an execution time of either a
specific time or the field's own time.

### Record Action base — six subjects

**1. When a meeting is…**
`Scheduled` (this is the record-created event) · `Canceled` · `Rescheduled` · `Modified`
· `Completed` · `Deleted`.
`Modified` reveals *Any field gets modified* / *Specific field(s) gets modified*; the latter
reveals a field picker and an "is modified to [Value|Field] [____]" row.

**2. When a meeting participant is…**

- `Added` → second picklist: *While a meeting is scheduled* · *After the meeting is
  scheduled* · *Anytime*
- `Removed` → a standing label **who had** followed by a picklist: *accepted the invite* ·
  *rejected the invite* · *replied Maybe* · *not replied* · *any reply status* (default).
  "who had" is a label outside the picklist, not repeated inside each option.
  The catch-all drops the clause from the summary entirely: "…is Removed."

**3. When a meeting Invite…**
`Is Accepted` · `Is Rejected` · `Is Replied May be` · `Got No Reply`, each followed by
**by [threshold] of the Participants**.
`Got No Reply` additionally reveals: **within [N] [Minute(s)|Hour(s)|Day(s)]
[after the invite was sent | before the meeting starts]**.

The two anchors are not interchangeable and both are needed: *after the invite was sent*
measures responsiveness ("reply within 48 hours of being invited"); *before the meeting
starts* measures risk to the meeting. Anchored only to the invite, a meeting booked an hour
ahead gets its chase a day after the meeting happened.

**4. When a meeting check-in is…**

| Option | Extra control | Fires when |
|---|---|---|
| `Checked In` | — | the host checks in |
| `Check-In After` | `[0] [Minute(s)\|Hour(s)] from the meeting start time` | a late arrival; 0 means any check-in after the start |
| `Missed Check-In` | — | the window closes with no check-in (§3.1) |
| `Check-In Reverted` | — | the host undoes their own check-in (§6.9) |
| `Check-In Data Cleared` | — | an edit to Location/From/To/Host is saved with *Clear it* (§6.11) |

**5. When a meeting check-out is…** (only when check-out is captured)

| Option | Extra control |
|---|---|
| `Checked Out` | — |
| `Check-Out Before` | `[0] [Minute(s)\|Hour(s)] before the meeting end time` |
| `Missed Check-Out` | — |
| `Check-Out Reverted` | — |
| `Check-Out Data Cleared` | — |

**6. When any action happens in a meeting**

Behaviour notes for 4 and 5: the offset resets to 0 when one of the two timed options is
newly selected, but a typed value survives re-picking the same option. The three "event"
options (Reverted ×2, Data Cleared) drop the "is" connector so the row reads as a sentence
— "…when a meeting has its check-in reverted."

Correctness notes worth encoding in help text or tests:

- `Check-Out Data Cleared` and `Check-In Data Cleared` fire at the **same moment** — §6.11
  says Clear removes whole check-ins, both halves together. There is no edit that clears
  only a check-out.
- Reverting a check-out returns the record to Checked In and empties Checked-In Duration,
  and makes the check-in revertable again (§6.9).
- A **cancelled** meeting never derives a missed status, and a meeting **created after its
  window had passed** reads Not Checked In permanently and never becomes Missed (§6.12).

---

## Stage 2 — CONDITIONS

One card asking three things in order:

1. Heading: **"Which meetings would you like to apply the rule to?"**
2. **"Would you like to set conditions for meeting fields?"** — Yes / No, **No by default**.
   Yes reveals numbered criteria rows (field · operator · value, with + / − and a criteria
   pattern line).
3. **"Apply this rule to"** — All (default) · Contact · Lead · Account · Potential.
4. When a related module is chosen: **"Which Contacts would you like to apply this rule
   to?"** — *All Contacts* / *Contacts matching certain condition*. The second reveals a
   nested box: **"Contacts matching [all|any] of these conditions"** with its own rows. All
   labels follow the chosen module (pick Lead → "Leads" throughout, and Lead fields).

### Condition fields (Meetings, one-check-in case)

`Meeting Venue` · `Location` · `Meeting Invite` · `Meeting Title` · `Host` ·
`Meeting Status` · `Check-In/Out Status` · `Check-In Time` · `Check-In Location` ·
`Check-In Method`

### Operators — §8.2 and §8.3

Two fields carry **two presentations each**, and the presentation is chosen by which
operator group you pick, under a heading naming the comparison:

- `Check-In Time` → *As date and time* (Is, Isn't, Is Before, Is After, Between, Is Empty,
  Is Not Empty) · *As time relative to From (meeting start)* (Is exactly, Is more than,
  Is less than, Is between — value in minutes/hours, before/after, naming the field)
- `Check-In Location` → *As the address captured* (text operators) · *As distance from
  Location* (Is exactly / more than / less than / between, in metres)
- `Check-In/Out Status` → picklist operators over the reachable statuses only
- `Check-In Method` → Is / Isn't · **Manual** or **Automatic** (§2.5: an automatic capture
  fires from the device geofence with the record never opened — different evidence from
  someone tapping Confirm)

`Is between` reveals a second value input.

---

## Stage 3 — ACTIONS

**Instant Actions** menu, in this order, matching the live CRM:
Field Update · Assign Owner · Tags ▸ (Add Tag, Remove Tag) · Notify ▸ (Email Notification,
SMS Notification, Zoho Cliq Notification) · Task · Create Record · Webhook · Function ·
Actions By Zoho Flow · Zia Agent.

**Scheduled Actions** — a delay, a direction and an anchor, then an action:

```
Execute [N] [Minute(s)|Hour(s)|Day(s)] [before|after]
        [the rule is triggered | From (meeting start) | To (meeting end) | the check-in window closes]
```

plus an optional **"Only run it if the conditions still hold at that moment"**.

That re-check matters because check-in statuses are derived and can be undone: a meeting
that read Missed Check-In and is then rescheduled forward reverts to Not Checked In, so an
escalation queued 24 hours earlier would otherwise land on a healthy record (§3.1.2, §6.11).

---

## What to prioritise if you cannot do it all

1. The check-in and check-out trigger subjects (stages 1.4 and 1.5) — nothing else in the
   builder can express the exception handling the feature exists for.
2. The `Check-In/Out Status` and relative-time / distance conditions.
3. The conditions card's related-module scope.
4. Scheduled Actions with the anchor list and the re-check.

## How to work

- I will paste the URL of the workflow-rule page I am on; start from the file that serves it.
- Ask before inventing a label that is not in this brief or the PRD — the naming has been
  argued over and the PRD's own words win.
- Follow the repo's existing component and design conventions; do not hand-roll markup that
  duplicates an existing CRM control.
