---
name: manage-cases
description: >
  Update Salesforce cases on the open relationship in plain language: close, reopen,
  escalate, put on hold, change priority, or delete. Trigger on "close the <x> case",
  "reopen", "escalate", "set priority to high", "put it on hold", "delete that case".
  Uses ONLY the sObject read/write MCP connector, confirms in one line before writing,
  and republishes the SAME Relationship Lens artifact so the updated row lands with the
  violet wash.
---

# Manage cases

The write-back half of the case lifecycle. The `relationship-lens` skill creates; this
skill moves cases through their life. Same rules: one connector (sObject), one-line
confirm before any write, honest ambiguity handling, same artifact on republish.

## Identify the case

1. Prefer the open lens context: match the user's words against the subjects in the
   current page's `#lens-data` slot (fuzzy, case-insensitive).
2. Otherwise query: `SELECT Id, Subject, Status, Priority, IsClosed FROM Case WHERE
   AccountId = '<id>' AND Subject LIKE '%<term>%' ORDER BY CreatedDate DESC LIMIT 5`
   (escape `'` as `\'`).
3. One match: proceed. Several: list them (subject, status, age) and ask. None: say so;
   never guess.

## Map intent to fields (org-verified picklists)

- **Status**: New (default), In Progress, On Hold, Escalated, Critical, Submitted,
  Waiting for Customer, Response Received, Merged, Closed
- **Priority**: Critical, High, Medium, Low
- "close" → `{"Status": "Closed"}`. Salesforce flips `IsClosed` and stamps `ClosedDate`
  automatically (verified); never write those fields directly.
- "reopen" → `{"Status": "New"}` ("start working on it" → `"In Progress"`).
- "escalate" → `{"Status": "Escalated"}`.
- "on hold" / "waiting for the client" → `"On Hold"` / `"Waiting for Customer"`.
- Priority changes are independent of status; combine freely in one update.
- If the org rejects a value (custom picklists differ per org), read the valid values
  with `getObjectSchema("Case")` and offer the closest ones.

## Execute

1. Confirm in ONE short line: "Close 'Treasury reporting change' (currently New)?"
2. `updateSobjectRecord` with `{"sobject-name": "Case", "id": "<id>", "body": {...}}`.
   Observed success response: `{"value": ""}` (empty body = success).
   **Never blind-retry a timed-out update**: re-read the record first; the write may
   have landed.
3. Re-read by Id (`Id, Subject, Priority, Status, CreatedDate, IsClosed`).
4. Update that row in the artifact's `#lens-data` slot (set `"landed": true` on it,
   clear `landed` everywhere else), republish the SAME artifact.
5. Report with the direct Lightning link:
   `<instanceUrl>/lightning/r/Case/<id>/view`.

## Delete

Only on an explicit ask, with a one-line confirm naming the exact subject.
`deleteSobjectRecord` (observed response: `{"value": ""}`). Deleted cases sit in the
Recycle Bin about 15 days. Remove the row from the slot and republish.

## Out of scope

Bulk updates across accounts, owner reassignment, record-type changes, and anything on
objects other than Case. Offer the Salesforce UI for those.
