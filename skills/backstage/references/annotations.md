# Annotations

Which annotations to set, where, and with what values. Use what applies and leave the rest out — never write a placeholder.

```yaml
annotations:
  # Source and docs
  github.com/project-slug: tryhuset/repo-name
  backstage.io/source-location: url:https://github.com/tryhuset/repo-name/tree/main/apps/web
  try.no/documentation: https://... # external docs (Notion, Confluence, Storybook…)

  # Project trackers
  jira.com/project-key: PROJ
  busy.no/project-id: "ACME"
  hubspot.com/company-id: "12345678"
  try.no/project-owner: "<PM name>"

  # SLA (System)
  try.no/sla-tier: standard
  try.no/sla-agreement: "<agreement reference>"
  try.no/sla-client-domain: example.no

  # Observability
  try.no/log-drain: betterstack
  sentry.io/project-slug: try-apt-as/project-name

  # Operational baseline (each production website Component)
  betterstack.com/monitor-ids: '1234567'
  betterstack.com/source-ids: '2345678,3456789'
  # try.no/unmonitored: <reason>  # instead of the two above, when TRY can't monitor it
```

- **System:** `github.com/project-slug` is strongly recommended.
- **Each Component in a monorepo:** `backstage.io/source-location` pointing at its subdirectory is strongly recommended.

## The Project card

The Backstage entity page has a **Project** card that shows the commercial and operational metadata: Busy ID, Jira key, SLA, log drain, project owner and external docs. It inherits annotations from the System down to its Components, so set them once on the **System**. Only override on a Component when it differs (e.g. the web app's logs go to BetterStack while a worker's stay in Cloudflare).

- `busy.no/project-id` — Busy time-tracking ID, conventionally the **uppercase project key** (e.g. `ACME`). Not derivable from the repo; ask.
- `jira.com/project-key` — Jira board key, auto-linked to `tryoslo.atlassian.net/browse/<key>`.
- `hubspot.com/company-id` — numeric ID of the client's HubSpot company record. Not derivable from the repo; ask (it's in the HubSpot URL when viewing the company).
- `try.no/sla-tier` — see _SLA tier_ below.
- `try.no/sla-agreement` — free-text reference to the SLA contract. The CS team is the source of truth.
- `try.no/sla-client-domain` — the client's production domain, usually obvious from the System's production link.
- `try.no/project-owner` — the PM or project owner (a person, not a GitHub team).
- `try.no/documentation` — external docs URL (Notion, Confluence, Storybook…).
- `try.no/log-drain` — where logs go, e.g. `betterstack` or `vercel`. Only `betterstack` counts towards the operational baseline.

## SLA tier

`try.no/sla-tier` goes on the **System** and must be one of:

| Value | Tier |
| --- | --- |
| `basic` | Basic |
| `standard` | Standard |
| `standard-plus` | Standard+ |

- **Default to `standard`** for new files. The CS team is the source of truth for anything else, so only write a non-default value when the user confirms it.
- The legacy value `on-call` means `standard-plus`. Replace it when you touch the file.

## Operational baseline (BetterStack)

Every client project records its monitoring, so coverage can be checked against BetterStack instead of taken on trust. The **Monitoring & SLA** card on the System page shows the result per component.

| Requirement | Where | Annotation | Value |
| --- | --- | --- | --- |
| SLA tier | `System` | `try.no/sla-tier` | `basic`, `standard` or `standard-plus` |
| Uptime monitoring | Each production `website` Component | `betterstack.com/monitor-ids` | Comma-separated BetterStack Uptime monitor IDs |
| Logs and traces | Each production `website` Component | `betterstack.com/source-ids` | Comma-separated BetterStack Telemetry source IDs |

A project meets the baseline when the System has a valid SLA tier and every production `website` Component is both monitored and logged.

- These annotations go on the **Component**, not the System — they don't inherit.
- **Quote the values** (`'1234567'`) so YAML keeps them as strings. Several IDs are comma-separated: `'2345678,3456789'`. Only digits, commas and spaces are valid.
- A component counts as covered when the IDs are present **and** resolve to active monitors and sources that receive data. A source with no recent logs or spans is a gap, even though the annotation is valid.
- Production services and workers are welcome to have the same annotations; they don't count towards the baseline yet.
- If TRY genuinely can't monitor a component (for example it runs in the client's own account), set `try.no/unmonitored: <reason>` instead of leaving the annotations out. Missing annotations mean "not monitored".
- Typical sources for a web app: a Vercel log integration (runtime logs) and an OpenTelemetry source (traces, sometimes logs). List both if both are in use.

Example:

```yaml
kind: System
metadata:
  name: acme
  annotations:
    try.no/sla-tier: standard
---
kind: Component
metadata:
  name: acme-web
  annotations:
    try.no/log-drain: betterstack
    betterstack.com/monitor-ids: '1234567'
    betterstack.com/source-ids: '2345678,3456789'
spec:
  type: website
  lifecycle: production
  system: acme
```

### Finding the IDs by hand

- **Monitor ID:** open the monitor in BetterStack Uptime; it's the number after `/monitors/` in the URL.
- **Source ID:** open the source in BetterStack Telemetry; it's the number after `/sources/` in the URL.
