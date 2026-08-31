# Relationship Lens

A client-relationship page for Salesforce, published as a live Claude artifact. One glass
page per account: hero, 12 key facts, the case list, and case creation from inside the
page. All Salesforce I/O runs through the connected **sObject read/write MCP connector**,
nothing else. Works in claude.ai and Cowork.

- **Read**: the agent pulls the real Account, Opportunity, Case and Contact records via
  live SOQL and fills the page's data slot; once open, the page re-reads live through the
  viewer's own connector (with a Refresh button).
- **Write**: describe a case in plain language inside the page; the viewer's Claude drafts
  subject + priority; one confirm creates it in the org via `createSobjectRecord`, the
  page re-reads it by Id and the new row lands with a violet wash. Writes also work from
  chat in natural language ("create a case for the failed portal login, high priority").

## Install

In claude.ai / Cowork / Claude Code, add the marketplace and install:

```
/plugin marketplace add Connectry-io/relationship-lens
/plugin install relationship-lens@relationship-lens
```

Requirements in the workspace:

1. The **sObject** Salesforce MCP connector, connected and authorized. The connector's
   display name must be `sObject`; if the workspace named it differently, say so when you
   open a relationship and the agent will adjust the page's manifest.
2. Artifacts enabled (the page uses the `mcp` and `sample` runtime capabilities).

No other connectors. No middleware. Standard objects only.

## Demo runbook (2 prompts)

**Before the meeting**: open a fresh conversation in the workspace with the sObject
connector enabled, and keep the demo org open in a browser tab, logged in.

1. **"Open the Hartwell Precision relationship"** (any account name works, fuzzy match).
   The agent reads the org through the connector and publishes the glass page filled with
   the real book: pipeline, opportunities, cases, contacts.
   - First open, claude.ai asks the viewer to allow the page to use the sObject
     connector. Say yes on stage: *that pop-up is the governance story - the page acts
     with the viewer's own credentials, scoped to two tools, nothing more.*
   - Beat: "This is live Salesforce, read through the model. No integration project."
2. **In the page: New case** > type *"Client reports the borrowing base upload keeps
   failing since Friday, blocking the month-end certificate"* > **Draft case** (the
   viewer's Claude writes the subject and sets priority) > **Create in Salesforce**.
   The row lands with the violet wash and the counters tick.
   - Beat: "And it wrote back." Switch to the org tab, refresh: the case is there.
3. Optional reverse proof: create a record in the Salesforce UI, hit **Refresh** in the
   page top bar; it appears in the lens.

Natural-language alternative for step 2, from chat: *"Create a case: portal login
failures for the treasury team, high priority"* - the agent creates it via the same
connector and republishes the page with the row landed.

## Architecture

Deliberately simple. The page is a single HTML file rendered from a JSON slot
(`#lens-data`). The agent (skill: `relationship-lens`) fills the slot and publishes; the
published page uses two artifact runtime capabilities:

- `mcp` - scoped to `{server: "sObject", tools: ["soqlQuery", "createSobjectRecord"]}`,
  running with the viewer's credentials.
- `sample` - the viewer's own Claude drafts case fields from a plain-language
  description. If unavailable, the composer falls back to plain subject/priority fields.

No fetch, no external resources, no page-side API keys. Failure states are designed:
reconnect copy per error code, one retry on read timeouts, never a blind retry on writes.

## Repo layout

```
.claude-plugin/plugin.json        plugin manifest
.claude-plugin/marketplace.json   marketplace manifest (install via this repo)
skills/relationship-lens/SKILL.md the agent methodology (read, fill, publish, write back)
skills/relationship-lens/assets/relationship-lens.html  the page template (sample data)
```

The template ships with fictional sample data (Meridian Components Ltd) and renders
standalone as a safe dry run - no org, no connector, no capabilities needed.
