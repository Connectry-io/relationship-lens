---
name: relationship-lens
description: >
  Open a client relationship as a live, light-glass artifact page fed by the connected
  Salesforce sObject MCP (read + write). Trigger on "open the <name> relationship",
  "pull up <account>", "relationship lens", or "refresh the relationship". The page shows
  a hero, a 12-fact key information block and the case list, and can create cases from
  inside the page (AI-drafted via the viewer's Claude, or plain fields). ONE connector
  only: the sObject read/write MCP. No Customer 360, no other connectors.
---

# Relationship Lens

A single-file HTML artifact rendered from a JSON data slot, wired live to the viewer's
**sObject** connector. The agent fills the slot with real org data and publishes; the page
then keeps itself current (Refresh) and writes cases back on its own via the `mcp`
capability. Standard objects only: Account, Opportunity, Case, Contact.

## Hard rules

1. **Only the sObject connector.** The capabilities manifest declares exactly one server.
   Never add Customer 360 or any other connector to the manifest or the flow.
2. **Same artifact, always.** Re-publish with the same file path (and `url` if resuming a
   prior conversation's artifact). A new path forks a second page and breaks the demo.
3. **Never invent records.** Empty result = honest empty state. Missing field = "-".
   Null Amount renders "-" and is excluded from the pipeline sum (the page handles this).
4. **Design is locked.** The template carries the light-glass system (ground #F5F5F7,
   solid white content cards, lens-glass chrome, ink buttons, purple #7500C0 as identity
   only, no em dashes). Edit the slot JSON, not the design, unless explicitly asked.

## Opening a relationship

1. Resolve the account by fuzzy match. Prefer SOQL LIKE:
   `SELECT Id, Name, Industry, Type, Website, Phone, Owner.Name FROM Account WHERE Name LIKE '%<term>%' LIMIT 5`
   If several match, list them and ask. If a name contains a single quote, ask for the
   exact name rather than retrying blind (escape `'` as `\'` in SOQL).
2. Query the book (three queries, LIMIT 30 each):
   - `SELECT Id, Name, StageName, Amount, CloseDate, IsClosed FROM Opportunity WHERE AccountId = '<id>' ORDER BY IsClosed ASC, CloseDate ASC`
   - `SELECT Id, Subject, Priority, Status, CreatedDate, IsClosed FROM Case WHERE AccountId = '<id>' ORDER BY CreatedDate DESC`
   - `SELECT Id, Name, Title, Email FROM Contact WHERE AccountId = '<id>' ORDER BY Name`
   The sObject server sometimes times out; retry a failed READ once. Never blind-retry a
   failed WRITE (create): re-read first, the record may exist.
3. Fill the `#lens-data` slot in `assets/relationship-lens.html` (copy it to a working
   file; keep the template pristine). Schema:

   ```json
   {
     "accountId": "001...", "name": "...", "industry": "...", "type": "...",
     "owner": "...", "website": "...", "phone": "...",
     "opportunities": [{"id","name","stage","amount","closeDate","isClosed"}],
     "cases": [{"id","subject","priority","status","createdDate","isClosed"}],
     "contacts": [{"id","name","title","email"}],
     "instanceUrl": "https://<mydomain>.lightning.force.com",
     "source": "live", "asOf": "YYYY-MM-DD"
   }
   ```

   `instanceUrl` makes the account name and every case row a direct Lightning link
   (`/lightning/r/<Type>/<Id>/view`), including cases created from inside the page.
   Resolve the My Domain once per org: `SELECT Domain, HttpsOption FROM Domain LIMIT 10`
   and derive it from any `...my.salesforce-sites.com` / `...my.site.com` row (the part
   before `.my.` + `.lightning.force.com`), or ask the user for their org URL. Omit the
   field if unknown; rows then render unlinked.

   `accountId` must be the real Id: the page keys its live queries and case creation on it.
   A slot with `"accountId": "SAMPLE"` renders but disables live fetch (safe dry run).
4. Publish with the Artifact tool:
   - `capabilities`: `{"mcp": {"servers": [{"server": "sObject", "tools": ["soqlQuery", "createSobjectRecord"]}]}, "sample": {}}`
   - `favicon`: keep stable (e.g. "🔍"); title stays "Relationship Lens".
   - The `server` value must equal the connector's display name in the viewer's
     workspace. If the workspace named it something other than "sObject", update the
     manifest AND the `SERVER` constant at the top of the page script to match.

## Observed tool contracts (verified 2026-09-01, bankinggpt org)

- `soqlQuery` input `{q}` → payload `{"totalSize", "done", "records": [{...fields, "attributes"}]}`
- `createSobjectRecord` input `{"sobject-name": "Case", "body": {"Subject", "Priority", "AccountId", "Origin"}}`
  → payload `{"id": "500...", "success": true, "errors": []}`
- `deleteSobjectRecord` input `{"sobject-name", "id"}` → `{"value": ""}` (agent-side only;
  the page never deletes)

## Writes from chat (alternative path)

When the user asks in chat to create an opportunity or case, collect what is missing
(opportunity: Name, Amount, CloseDate, StageName default "Prospecting"; case: Subject,
Priority default "Medium"), confirm in ONE short line, create via `createSobjectRecord`,
re-read by Id, add the record to the slot with `"landed": true` (clear `landed` everywhere
else), and republish the SAME artifact. Tell the user the new Id.

## Demo script

1. Warm up: "open the <real account> relationship" once before the meeting.
2. Live: open the relationship; the glass page fills with real org data.
3. In the page: New case → describe it → AI drafts subject + priority → confirm →
   the row lands with the violet wash. Have the org open in a tab; refresh it: the case
   is there. That refresh is the wow.
4. Reverse proof: create something in the Salesforce UI, hit Refresh in the page.
