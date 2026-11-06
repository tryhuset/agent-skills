---
name: backstage
description: Create, update or check a repo's backstage.yaml catalog file following TRY conventions, including ownership, relations, links, SLA tier and BetterStack monitoring (uptime monitor and log source IDs). Use when the user asks to create, scaffold, update or check a backstage.yaml, register a repo in Backstage, or add monitoring, logging, SLA or ownership details to a Backstage catalog entry. Uses the Backstage and BetterStack MCPs when they're available.
license: MIT
metadata:
  author: eirikeikaas
  version: "1.2.0"
---

# Backstage Catalog Generator

Creates or updates a multi-document `backstage.yaml` in the repo root, so the product shows up in TRY's Backstage with the right owner, relations, links, SLA tier and BetterStack monitoring.

Write for a developer who has never opened Backstage: narrative descriptions over boilerplate, the links people click every day, and relations that encode how the system actually works.

## Ground rules

- **Every specific comes from this repo or from the user.** TRY products share libraries and stack choices — same Storyblok + Centra + Cloudflare-revalidate-worker shape, same `@frend-digital/cache/next`. That similarity is exactly when it's tempting to fill in env var names, HTTP methods, binding names, Centra admin paths, Storyblok space IDs, Cloudflare account IDs, queue and worker names, or BetterStack IDs from memory of another repo. **Don't.** Two products can share the entire stack and still differ in every one of those values. Each one must trace back to a file you read in this session, a read-only lookup you ran, or an explicit answer from the user.
- **Never read secret files.** Only `.env.example` / `.env.sample` are allowed, and secret files stay out of searches too.
- **Never copy a secret** into the file or the chat. If you spot one, say which file it's in.
- **Read-only shell:** `git` and text tools only. No vendor CLIs, no process env. From the Git remote, keep only `org/repo`.
- **Only touch `backstage.yaml`.** Don't commit or push.
- **No placeholders in the output.** If a value is unknown and the user skips it, leave the annotation or link out and list it in the hand-off. Never write `<placeholder>` or `TODO` values into the file.
- **Both MCPs are optional. Use them when they're there, read-only.** The Backstage MCP shows what the catalog already has, so you can reuse existing entities instead of inventing slugs that duplicate them (Step 1.7). The BetterStack MCP finds monitors and sources (Step 2). If either is missing, use the fallback in that step and carry on. Never stop, or ask the user to install one, because it's missing. Never write to Backstage or BetterStack.
- **Few, good questions.** Discover first, then ask in at most two batches with recommended answers (Step 3).

## Reference files

Read these when a step needs them:

- [`references/conventions.md`](references/conventions.md) — entity kinds and ordering, naming, owner, shared catalog references, relation rules, links. Read before writing a new file or changing entities or relations.
- [`references/annotations.md`](references/annotations.md) — annotations, SLA tiers, and the operational baseline (BetterStack monitors and sources). Read before adding or changing annotations.
- [`references/template.yaml`](references/template.yaml) — starting point for a new file, with the required header comment.

## Workflow

### Step 0: Check the MCPs and the existing file

Note which MCPs are available: Backstage (a tool named like `search-catalog-entities`) and BetterStack (tools named like `monitors` and `sources`). Either may be deferred, so search for them before concluding they're missing. This only decides how Step 1.7 and Step 2 run. Continue either way.

Then look for `backstage.yaml` in the repo root, and read all of it if it exists. Every run follows the same steps. Two rules adjust them:

- **If the file exists, edit it in place.** Don't regenerate it, reorder entities or reformat untouched parts, and keep existing values and comments unless this repo shows they're wrong.
- **If the user only asked for a review** ("is this right?", "what's missing?"), stop after the gap list in Step 3. Don't ask questions or write anything.

### Step 1: Discover — no questions yet

1. **Repo identity.** Read the Git remote (`git remote get-url origin`) for `github.com/project-slug` and the source-location URLs. If the remote's org isn't `tryhuset` (for example a mirror), confirm the slug with the user in Step 3.
2. **Inventory.** List every buildable or deployable unit — don't spot-check. In a monorepo, take the list from the workspace or build config (e.g. `pnpm-workspace.yaml`, `package.json` `workspaces`, `*.sln`, `settings.gradle`, `go.work`) and go through every directory it matches. A single-app repo is one Component at the root.
   - Deployables (web apps, services, workers, mobile apps, CLIs, build scripts) → candidate Components. If you skip one, leave a one-line YAML comment saying why (e.g. `# parser/ is a one-shot CSV generator script, not modeled`).
   - Internal libraries in the workspace → don't model them and don't add `dependsOn` edges to them, unless the user asks or the library is shared across separate products (then it belongs in `try-backstage-catalog`, not here). Say in the header comment that they're intentionally left out.
   - Build manifests, container files and platform config (e.g. `package.json`, `*.csproj`, `go.mod`, `Dockerfile`, `wrangler.jsonc`, `vercel.json`, Terraform) → component types and platforms.
   - Databases, migrations, queues, caches, key-value stores, buckets → Resources.
   - HTTP route handlers, OpenAPI specs, websocket handlers → local APIs (`providesApis` on the host Component).
   - Outbound HTTP (SDKs, service URLs in config) → `consumesApis`.
3. **Read the specifics that will appear in the file.**
   - For every local API: open the handler and capture the HTTP method, the path and the source file.
   - For every infra config: use the literal database, queue, binding and bucket names. If you pick a tidier catalog name, put the real name in the Resource's `description`.
   - When one component calls another's API, confirm from config that the caller points at the route you modeled. That closes the `consumesApis` ↔ `providesApis` loop.
   - Look for IDs and slugs in platform and infra config, `.env.example`, `README*`, deploy scripts and CI workflows.
4. **Production domain.** From the README, platform config, framework config (e.g. a site URL or sitemap setting) or the existing file. Note domains that redirect to it, too. If none of these give a clear answer, ask the user in Step 3.
5. **Owner hints.** `CODEOWNERS`, the existing file, or other catalog files the user points you to.
6. **Telemetry wiring.** Note how each component ships logs and traces: OpenTelemetry setup and service names, logging settings in platform config, log drains mentioned in docs. You'll match these to BetterStack sources in Step 2.
7. **Look up the Backstage catalog.** If the Backstage MCP is available, use it to check what already exists, read-only:
   - **The product itself.** Search by the `github.com/project-slug` annotation and by `term` (product, repo and domain names). If a System or Component for this repo is already registered (for example from another repo or a manual catalog file), don't create a parallel one — tell the user in Step 3 and reuse its names.
   - **Shared APIs and Resources.** List `kind: API` and `kind: Resource` with `namespace: tryhuset` (`limit: 100`). Map every outbound service and platform from item 2 to an existing `entityRef`, and use that exact slug even when it isn't the one you'd guess (e.g. `tryhuset/gcp-maps-api`, not `google-maps-api`). A sub-product of a listed vendor (e.g. Cloudflare Turnstile) goes under the vendor's API rather than a new slug. Only fall back to a new canonical slug when nothing matches, and list each new one in the hand-off.
   - **Owner groups.** List `kind: Group`. Use the result to offer owner options in Step 3, and flag any team (from `CODEOWNERS`, the existing file or the user) that has no matching `group:default/<slug>`.
   - Results for whole kinds are large and may spill to a file. Extract just the refs, e.g. `jq -r '.items[].entityRef'`.

   **Without the Backstage MCP**, skip this lookup. Use the canonical slug patterns in `references/conventions.md` for shared services, and take the owner from `CODEOWNERS`, the existing file or the user. Say in the hand-off that the product, shared refs and owner group weren't checked against the catalog.

### Step 2: Find BetterStack monitoring

The operational baseline (see `references/annotations.md`) expects every production `website` Component to record an uptime monitor and a log/trace source. Production services and workers should have them too, but aren't required yet. The baseline is about the file, not the tooling: the IDs can come from the BetterStack MCP or from the user, and a gap is fine as long as it's listed in the hand-off.

**If the user has the BetterStack MCP configured** (tools named like `monitors`, `sources`, `source`, `query`), use it to look the IDs up, read-only:

1. **Monitors.** List all monitors (`per_page: 250`, page until done) and match on the production host, ignoring scheme, `www.` and a trailing slash. Check redirecting domains too. Don't rely on the name or URL filters — they behave like exact matches and miss obvious hits.
2. **Sources.** Search with the `name` filter (partial match) for the product, repo and component names. Map each source to a component with what you found in Step 1: a Vercel integration source belongs to the Vercel-hosted web app, an OpenTelemetry source usually matches the OTel service name, and a Cloudflare source belongs to the worker.
3. **Check that data is arriving.** For each candidate source, count rows over the last 7 days — logs, plus spans for OpenTelemetry sources:
   ```sql
   SELECT count() AS rows, max(dt) AS latest FROM {{source}}
   WHERE dt >= {{start_time}} AND dt < {{end_time}}
   ```
   Run it with `source_type: logs` (then `spans`), `start_time: now-7d` and `end_time: now`. A source with no recent rows isn't real coverage: still offer it, but flag it in Step 3 and in the hand-off.
4. Source details include ingestion tokens. Never copy a token into the file, the chat or anywhere else.

**If the BetterStack MCP isn't configured**, skip the lookup and don't treat it as an error. In Step 3, ask whether the user wants to paste the IDs, and tell them where to find them: the monitor ID is the number after `/monitors/` in the monitor's BetterStack URL, and the source ID is the number after `/sources/`. You can't check whether data is arriving, so say so in the hand-off.

### Step 3: Summarize, then ask

Start with a short summary (5–10 lines) of what you found: the components you'll model and anything you'll skip, the production domain, the platforms, and the BetterStack monitors and sources with whether data is arriving (or that you couldn't look them up without the BetterStack MCP). This lets the user catch a wrong assumption before answering anything.

If `backstage.yaml` exists, follow the summary with a gap list: compare the file with what you found and with the Step 5 checklist, and list what's missing, wrong or out of date, one line per gap.

If the user only asked for a review, stop here.

Otherwise ask with `AskUserQuestion`:

- At most 4 questions per call and 2–4 options per question. The user can always pick "Other" and type an answer.
- Put the inferred answer first, labelled "(Recommended)", and say in its description where it came from.
- Don't ask what the repo or the existing file already answers — for example a clear `CODEOWNERS` team, or an existing `try.no/sla-tier` in the file.
- Don't ask for the System name, title or description. Derive them; the user corrects them in review.
- Two rounds at most.

**Round 1 — decisions.** Ask the ones that apply:

| Question                                                 | Options                                                                                                                                                                                                                                                   |
| -------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Owner team (GitHub team slug)                            | Teams found in `CODEOWNERS` or the existing file, recommended first. If you found none, offer Groups from the catalog lookup (Step 1.7), if you ran it, and rely on "Other". Don't offer a slug the catalog doesn't have unless you say so.         |
| Production domain                                        | Only when Step 1.4 found no clear answer. Offer the candidates you found (e.g. from config or BetterStack monitor URLs), most likely first, plus **No production domain yet**. Match monitors (Step 2) once you have the answer. |
| Lifecycle                                                | `production` (Recommended when there's a live production domain), `development`, `experimental`                                                                                                                                                           |
| SLA tier                                                 | `standard` (Recommended), `basic`, `standard-plus`. Say that the CS team owns non-standard values.                                                                                                                                                        |
| Monitoring for `<component>`, one per production website | Use found IDs (list them) · I'll paste the IDs · Not monitored by TRY (adds `try.no/unmonitored`; ask for the reason) · Set up later. Recommend the first when the lookup found IDs, otherwise the second.                                                |
| Also fix these gaps?                                     | Only when the file exists and the gap list has items outside what the user asked for. Use `multiSelect`, one option per gap or group of related gaps. Unselected gaps go in the hand-off. Skip it when the user asked to bring the whole file up to date. |

If there are more production websites than fit, ask about the rest in round 2. For services and workers, show any found IDs in the summary and add them, rather than spending a question, unless the user asked about their monitoring.

**Round 2 — typed values.** Only the ones that are missing and apply: HubSpot company ID, Busy project ID, Jira project key, Centra admin path (always ask when the product uses Centra — it varies per client), project owner (PM), and pasted BetterStack IDs. Give each question the options **Skip for now** and **Not applicable**; the user types the value via "Other". Skipped values go in the hand-off; not-applicable ones are simply left out.

Example monitoring question:

```json
{
  "question": "Which BetterStack monitoring should acme-web record?",
  "header": "Monitoring",
  "multiSelect": false,
  "options": [
    {
      "label": "Use found IDs (Recommended)",
      "description": "Monitor 1234567 (www.acme.no, up). Sources 2345678 (Vercel, logs today) and 3456789 (OpenTelemetry, spans today)."
    },
    {
      "label": "I'll paste the IDs",
      "description": "Ask for the monitor and source IDs in the next round."
    },
    {
      "label": "Not monitored by TRY",
      "description": "Adds try.no/unmonitored with a reason, e.g. it runs in the client's own account."
    },
    {
      "label": "Set up later",
      "description": "Leaves the annotations out. Backstage will show the component as missing monitoring."
    }
  ]
}
```

### Step 4: Write

- **No file yet:** start from `references/template.yaml`. Keep the header comment and the entity order (System → local APIs → Components → Resources), and remove sections that don't apply.
- **Existing file:** make targeted edits for the request and the gaps the user chose to fix, and keep the file's existing style.
- Quote ID annotation values (`'1234567'`) so YAML keeps them as strings.

### Step 5: Verify

Read the file back and check:

**Factual specifics** (the bugs that survive review):

- [ ] Every HTTP method, path and source file in a local API's `definition` was read from the handler in this session
- [ ] Every Resource `metadata.name` matches the literal infra name, or the real name is in its `description`
- [ ] Every deployable unit from the inventory is modeled or skipped with a comment
- [ ] `github.com/project-slug` matches the Git remote, or the user confirmed it
- [ ] Every BetterStack ID came from a lookup or from the user, and belongs to this product

**Operational baseline:**

- [ ] The System has `try.no/sla-tier` set to `basic`, `standard` or `standard-plus` (replace the legacy `on-call` with `standard-plus`)
- [ ] Each production `website` Component has `betterstack.com/monitor-ids` and `betterstack.com/source-ids` (quoted, comma-separated numbers) or `try.no/unmonitored` with a reason — or the gap is listed in the hand-off

**Structure:**

- [ ] Header comment present
- [ ] Valid multi-document YAML with `---` separators, exactly one System, and the System first
- [ ] `kind:` values in TitleCase (`System`, `Component`, `Resource`, `API`) — Backstage rejects lowercase
- [ ] kebab-case `metadata.name`, no duplicates, no `metadata.namespace` on local entities
- [ ] `owner` everywhere is `group:default/<gh-team-slug>`, and the Group exists in the catalog (when you could check)
- [ ] Every entity has `name`, `title`, `description`, `tags`, `owner` and `lifecycle`, plus `system` where it applies
- [ ] `consumesApis` for all HTTP, `providesApis` only with a matching API entity, `dependsOn` only for platforms and runtime bindings, and no relation in both `dependsOn` and `consumesApis`
- [ ] Shared services referenced by their canonical slug, not redefined — the existing catalog `entityRef` when the lookup found one, and every new or unchecked slug listed in the hand-off
- [ ] `backstage.io/source-location` on each Component in a monorepo
- [ ] System links include the production URL, hosting project, CMS admin and commerce admin where they apply; each Component has the dashboard link matching its `dependsOn`
- [ ] No placeholders left

Fix anything that fails in what you wrote before handing off. In an existing file, failures the user chose not to fix go in the hand-off instead.

### Step 6: Hand off for human review

Tell the user what you created or changed, then where the values came from, in three short lists:

- **From the repo or BetterStack** — e.g. components, bindings, monitor and source IDs.
- **From your answers** — e.g. owner, SLA tier, HubSpot ID.
- **Skipped or missing** — values left out, baseline gaps, gaps the user chose not to fix, sources that aren't receiving data, and anything you couldn't check because an MCP was missing.

Then:

> Please read `backstage.yaml` top to bottom before committing — other people will rely on what it says about ownership, infrastructure, integrations and monitoring. Check especially:
>
> - Entity names and the `dependsOn` / `consumesApis` graph — does it match how the system actually works?
> - Real-world specifics (HTTP methods, binding names, source paths, account/space/queue IDs) — do they match the running system?
> - Owner team — is it the team that should be contacted for this product?
> - Annotations (Busy ID, SLA tier, Centra admin path, HubSpot ID, BetterStack IDs) — are the values correct?

Don't treat silence as approval. The skill drafts; the developer verifies and commits.
