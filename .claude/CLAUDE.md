# Meetings Workflow — working agreement

`meeting.html` is a single-file static mock of the Zoho CRM **workflow rule builder** for the
Meetings module, covering the Check-In / Check-Out feature. No build step, no dependencies.

## Push after every change

**Every edit to `meeting.html` gets committed and pushed to `origin/master` immediately** —
one commit per change, not batched at the end of a session. This applies to any file in the
repo, not just the mock: `.claude/`, docs, anything. If a push fails or is rejected, say so
rather than silently leaving commits local.

The commit message says what changed and why, in prose. The *why* matters more than the what:
the diff already shows the what.

## Check it in the browser before committing

Serve the folder and open the mock:

```
python3 -m http.server 8092      # from this folder
http://localhost:8092/meeting.html?v=N
```

**Cache-bust the URL** — `http.server` sends no cache headers and Chrome will serve a stale
copy. Increment `?v=` on every reload.

Verify the actual behaviour, not just that the file parses: drive the page's own functions
(`pickBase`, `pickSubject`, `pickStatus`, `goConditions`, `doneConditions`…) and read back the
DOM and `whenSummaryText()`. Collect `window.onerror` while doing it — a broken template
literal fails silently and leaves half the row unrendered.

## Source of truth

The product behaviour comes from **`Check-In / Check-Out — PRD`** (the .docx). Section numbers
in commit messages refer to it. Two rules that have come up repeatedly:

- **Use the PRD's own names.** The status field's values (Checked In, Checked Out, Missed
  Check-In, Missed Check-Out, Not Checked In) appear on the record header, the list, Kanban,
  filters and the calendar (§3.1), so the builder must not invent different words for them.
- **Offer only reachable values** (§3.1.3, §8.6). Meetings is always the one-check-in case —
  only the host can check in (§2.2.5) — so `Partially Checked In`, the four count fields and
  the per-person rows never apply here. `CHECKOUT_CAPTURED` gates everything check-out.

## Layout references

Screens are matched against screenshots of the real CRM where they exist. When replicating
one, measure it — take the control-column offset and row gaps off the image and compare with
`getBoundingClientRect()` rather than eyeballing.

Known trap: `.radios label` and `.relhead` are flex containers with a `gap`, so any bare text
node beside a `<span>` renders with extra space between them ("All&nbsp;&nbsp;Contacts").
Keep each label's text in a single element.
