---
name: backstage
description: Generate a backstage.yaml catalog file for a product or system following internal conventions. Use when the user asks to create, scaffold, or update a backstage.yaml, Backstage catalog, or service catalog entry.
tools: Read, Glob, Grep, Write, AskUserQuestion
license: MIT
metadata:
  author: eirikeikaas
  version: "1.1.1"
---

# Backstage Catalog Generator

Generate a multi-document `backstage.yaml` for a product/system following internal catalog conventions.

The output should be readable by a developer who has never opened Backstage before. Prefer narrative descriptions over boilerplate, lean on links and annotations that make the page useful day-to-day, and use the relations that encode reality — not approximations.

## Workflow

### Step 1: Gather Context

Before generating anything, understand the system:

1. **Check for an existing `backstage.yaml`** in the repo root using Glob. If one exists, read it — you may be updating, not creating from scratch.
2. **Inventory the codebase** to infer components, tech stack, and structure:
   - Monorepo? Use Glob/`ls` on `apps/*`, `packages/*`, `services/*`. **Enumerate every subdirectory** explicitly before deciding what to model — don't just spot-check the obvious ones.
   - `apps/*` and `services/*` directories → **each one is a candidate Component**. Includes deployables (web, worker, mobile), CLI tools, build-time scripts, anything with its own `package.json`. If you skip one, leave a one-line YAML comment in the file explaining why (e.g. `# parser/ is a one-shot CSV generator script, not modeled`).
   - `packages/*` directories → **internal workspace libraries. Do NOT model these as Components by default.** They're implementation detail and clutter the catalog. Skip them unless: (a) they're explicitly consumed across multiple separate products (a shared TRY-wide library — in which case it should live in `try-backstage-catalog`, not the product file), or (b) the user explicitly asks. **Do NOT add `dependsOn: component:<workspace-lib>` edges** — workspace package deps stay invisible to the catalog.
   - `package.json`, `Dockerfile`, `*.csproj`, `build.gradle`, `go.mod`, `wrangler.jsonc`, `vercel.json`/`vercel.ts` — identify component types and platforms.
   - DB/ORM config, migrations, queue config, KV/Durable Object bindings — these become Resources.
   - API route handlers, OpenAPI specs, websocket handlers — these become APIs (and `providesApis` on the host Component).
   - Outbound HTTP calls (env vars like `*_URL`, SDKs for Storyblok/Centra/etc.) — these become `consumesApis`.
3. **Read source for the specifics that will appear in the file.** Inventory tells you what entities exist; this step extracts the literal strings you'll write down. Do not infer these from memory of a similar product — verify each one against this repo.
   - **For every local `API` entity:** open the route file (e.g. `apps/<app>/src/app/api/<path>/route.ts`) with the Read tool and capture: the actual HTTP method exported (`GET`, `POST`, …), the actual env-var name used for auth (e.g. `REVALIDATE_SECRET` vs `REVALIDATE_TOKEN`), the handler library, and the source path. These go straight into the API entity's `definition`.
   - **For every infra-config file** (`wrangler.jsonc`, `vercel.json`/`vercel.ts`, Terraform, etc.): use the _literal_ binding/queue/DO/KV names as the basis for Resource `metadata.name`. If you choose to use a tidier catalog name, put the real infra name in the Resource's `description` so devs can cross-reference dashboards.
   - **For caller-side env vars** (e.g. the worker's `REVALIDATE_URL`): confirm the URL points at the route you modeled — that's what closes the `consumesApis` ↔ `providesApis` loop.

   > ⚠️ **Anti-pattern: borrowing from sibling projects.** TRY products share libraries and stack choices — same Storyblok+Centra+Cloudflare-revalidate-worker shape, same `@frend-digital/cache/next`. Strong pattern-similarity is exactly when you're most tempted to fill in **env var names, HTTP methods, resource binding names, Centra admin paths, Storyblok space IDs, Cloudflare account IDs, queue names, and worker names** from memory of another repo. **Don't.** Same library family ≠ same secrets, routes, slugs, or IDs — two products from the same agency can share the entire stack and still have entirely different values for every one of those fields. Every specific in the generated file must trace back to a file you read in _this_ session, in _this_ repo — or to an explicit answer from the user.

4. **Reference shared services by the canonical naming convention** — don't redefine third-party services locally. See _Shared catalog references_ below for the patterns.
5. **Ask the user** (via AskUserQuestion) for anything you cannot infer:
   - System name, title, description
   - Owning **GitHub team slug** (not a friendly name — see _Owner convention_ below)
   - Lifecycle (`production`, `development`, `experimental`)
   - SLA tier/agreement (if applicable)
   - Any external/partner APIs consumed that aren't already in the shared catalog
   - **Centra admin path** — if the product uses Centra, the admin URL is `https://<store>.centra.com/<admin-path>` and the path varies per client (`ams2019`, `ams2020`, etc.). Always ask; do not assume.
   - **HubSpot company ID** — the numeric ID for the client's HubSpot company record, used for the `hubspot.com/company-id` annotation. Not derivable from the repo; ask the user (they'll usually find it in HubSpot's URL when viewing the company).

### Step 2: Generate the backstage.yaml

Write the file to the repository root as `backstage.yaml`.

**Always start the file with a header comment block** explaining the entity kinds and house conventions (see _Header comment_ below). This is non-negotiable — it's the first thing every reader sees and is the single biggest onboarding lift.

Use the entity ordering, naming, and conventions described below.

### Step 3: Verify

After writing, read the file back. Check:

- Valid multi-document YAML with `---` separators.
- No `kind: resource` / `kind: component` (capitalization matters — Backstage rejects lowercase).
- No `dependsOn` + `consumesApis` pointing at the same target.
- Owner references resolve to actual GitHub teams (or other groups in `try-backstage-catalog/org/`).
- Every annotation that requires a real value has one (no `<placeholder>` left behind).

---

## Header comment

Every generated file MUST start with a short header that teaches the model. Adapt the wording to the product, but keep all four rule lines:

```yaml
# Backstage catalog manifest for the <Product> product.
#
# Quick primer:
#   System    = the product as a whole.
#   Component = something we build and deploy (web app, worker, mobile app).
#   Resource  = infrastructure we depend on (queues, durable objects, log drains, hosting).
#   API       = an HTTP/websocket surface — third-party or one of our own.
#
# House conventions:
#   - consumesApis  → who CALLS an API (HTTP direction). Use for third-party AND
#                     our own component-to-component calls.
#   - providesApis  → who EXPOSES an API. Put on the producing component, with a
#                     matching API entity in this file (or in try-backstage-catalog).
#   - dependsOn     → platforms we run on, or runtime/data bindings (queue, durable
#                     object, log drain). NOT for HTTP — that's consumesApis.
#   - owner         → always a GitHub team slug, written as group:default/<slug>.
#
# Reference catalog: https://github.com/tryhuset/try-backstage-catalog
```

---

## Entity Ordering

Always use this order in the file, separated by `---`:

1. **System** (exactly one per file)
2. **API** entries that are _local_ to this product (e.g. an internal `/api/revalidate` provided by our web app and consumed by our worker)
3. **Component** entries (the codebases we build/deploy: web, worker, mobile, studio)
4. **Resource** entries (queues, durable objects, KV namespaces, deployments — anything we provision)

> Only **local** entities go in your product's `backstage.yaml`. Shared services are referenced, not redefined — see _Shared catalog references_.

> Local API entities are appropriate when your product exposes an HTTP surface that another component in the same product (or another TRY product) calls. They are **not** appropriate for documenting your callees — those are referenced through `consumesApis`.

---

## Owner convention

`spec.owner` always resolves to a `Group`. In our setup, Groups come from two places:

1. **GitHub teams under the `tryhuset` org** — auto-imported by the `githubOrg` catalog provider as `group:default/<gh-team-slug>`. This is the source of truth.
2. **Manually-defined groups** in `try-backstage-catalog/org/*.yaml` — typically partner orgs or non-GitHub business units.

**Always write owner as the fully-qualified `group:default/<gh-team-slug>`** — explicit kind and namespace. Groups always live in the `default` namespace; spelling it out means the reference resolves correctly no matter what namespace the containing entity uses. Examples:

- ✅ `owner: group:default/ecom-tech`
- ✅ `owner: group:default/creative-tech`
- ⚠️ `owner: group:ecom-tech` (works only if the entity is also in `default` — fragile)
- ❌ `owner: ecommerce` (friendly name — may not resolve)
- ❌ `owner: ecom-tech` (ambiguous — could be a User)

If you don't know the GitHub team, ask the user. Do not guess.

---

## Shared catalog references

Third-party services, hosting platforms, observability backends, and other shared infrastructure live in **`try-backstage-catalog`** (https://github.com/tryhuset/try-backstage-catalog). You do **not** clone, read, or otherwise check that repo. The skill has no access to it. Reference shared entities purely by the canonical name pattern:

- Resource: `resource:tryhuset/<service-slug>` — e.g. `resource:tryhuset/vercel`, `resource:tryhuset/cloudflare`, `resource:tryhuset/hyperdx`
- API: `tryhuset/<service-slug>-api` — e.g. `tryhuset/storyblok-api`, `tryhuset/centra-api`
- Public API: `public/<api-name>` — e.g. `public/udir-api`
- Partner-owned: `resource:<org>/<name>` or `<org>/<name>-api` — e.g. `<partner-org>/<service>-api`

`<service-slug>` is kebab-case of the service's brand name (examples: `google-maps`, `azure-openai`, `aws-s3`, `cloudflare`).

**Do not redefine shared services as local entities** in your product's `backstage.yaml`. Keep the shared-vs-product separation clean.

Because the skill does not check what's in the catalog, you can't know whether the entity exists. **Reference the canonical slug regardless.** If it doesn't exist there yet, Backstage will surface a temporary "entity not found" warning — a catalog admin reconciles it later (either by adding the entity, or by telling the dev to rename if the slug is non-standard). This keeps product PRs unblocked and lets the catalog grow organically.

---

## Naming and Namespaces

- Use **kebab-case** for all `metadata.name` values.
- **Local product entities** (your System, Components, Resources, internal APIs) live in the **default namespace** — don't set `metadata.namespace`. Avoids verbose refs like `acme/acme-web`.
- **Shared services** live in the `tryhuset`, `public`, or partner namespaces. You reference them; you don't define them. See _Shared catalog references_ for the patterns.

---

## The relation rules (most-violated, most-important)

This is the core conceptual model. Get this right and most other things follow.

### Before the field rules: pick the right entity kind

- **API** = something your code _calls_ over the network (HTTP, GraphQL, webhooks). Reference via `consumesApis`. Examples: Storyblok, Centra, Mapbox, any third-party HTTP endpoint.
- **Resource** = something your code _runs on_ or is bound to at runtime. Reference via `dependsOn`. Examples: Vercel, Cloudflare, HyperDX, queues, durable objects.

For any given service, pick the one that matches what you do with it. **Don't reference both for the same service** — the field you're writing dictates the entity kind. If you're filling in `consumesApis`, you're referencing an API; if you're filling in `dependsOn`, you're referencing a Resource. They're not paired in product `backstage.yaml` files.

### `consumesApis` — who calls what (HTTP direction)

Use for any outbound HTTP call:

- Third-party APIs: `tryhuset/storyblok-api`, `tryhuset/centra-api`, `public/udir-api`
- **Internal APIs in the same product**: if your worker calls your web app's `/api/revalidate`, model the revalidate endpoint as a local `API` entity and put it in the worker's `consumesApis`.

### `providesApis` — who exposes the API

Use on the producing Component. Every entry must have a corresponding `API` entity (either local in this file or in `try-backstage-catalog`).

### `dependsOn` — runtime/platform binding

Use for **non-HTTP** relations:

- Hosting platforms: `resource:tryhuset/vercel`, `resource:tryhuset/cloudflare`
- Data bindings: `resource:my-product-queue` (a Cloudflare Queue bound to the worker), `resource:my-product-durable-object`
- Log drains, observability hooks
- Sibling Components only when there's a real runtime data binding (rare — usually it's HTTP, so use `consumesApis`)

### Don't duplicate edges

For a given relationship, pick **one** edge type:

- HTTP call to a service that also has a resource? → `consumesApis: tryhuset/storyblok-api` **only**. Don't add `dependsOn: resource:tryhuset/storyblok`.
- Sibling component that we call over HTTP? → `consumesApis: <our-local-api>` **only**. Don't add `dependsOn: component:other-component`.

> A second edge that encodes the same relationship is noise; it makes the graph misleading and the file harder to maintain.

### Common mistake: cross-component `dependsOn`

If a subcomponent (GUI, mobile, studio) calls a server over HTTP, that's **`consumesApis` of the server's API entity**, not `dependsOn: component:server`. The `dependsOn` would imply a runtime data binding (shared queue, shared DB) — which is usually not what's happening.

---

## Required Metadata

Every entity must include:

- `metadata.name`, `metadata.title`, `metadata.description` — narrative, not boilerplate
- `metadata.tags` — combine domain, tech, platform tags
- `spec.owner` (as `group:default/<slug>`), `spec.lifecycle`, and `spec.system` (where applicable)

### Annotations

The following annotations are commonly useful. Use what applies; don't include placeholders.

```yaml
annotations:
  # Source / docs
  github.com/project-slug: tryhuset/repo-name
  backstage.io/source-location: url:https://github.com/tryhuset/repo-name/tree/main/apps/web
  try.no/documentation: https://... # external link to docs (Notion, Confluence, etc.)
  # backstage.io/techdocs-ref: dir:.                              # only if the repo has a mkdocs.yml and we want docs rendered IN Backstage

  # Project trackers
  jira.com/project-key: PROJ
  busy.no/project-id: "ACME"
  hubspot.com/company-id: "12345678"
  try.no/project-owner: "<PM name>"

  # Observability
  sentry.io/project-slug: try-apt-as/project-name
  try.no/log-drain: hyperdx

  # SLA (see "Project card" below — these populate a UI block)
  try.no/sla-tier: standard
  try.no/sla-agreement: "<agreement reference>"
  try.no/sla-client-domain: example.no
```

**Strongly recommended on the System:** `github.com/project-slug`.
**Strongly recommended on each Component in a monorepo:** `backstage.io/source-location` pointing at the subdir.
**Only add `backstage.io/techdocs-ref: dir:.`** when the repo also has a `mkdocs.yml` and you actually want docs rendered in Backstage. The annotation alone does nothing — without `mkdocs.yml` the Docs tab just sits empty.

### The Project card

The Backstage entity page shows a custom **"Project" card** that aggregates the commercial/operational metadata above (Busy ID, Jira key, SLA, log drain, project owner, external docs link). It inherits annotations from the System down to its Components — set them once on the System and every Component shows the same values.

Annotations it reads:

- `busy.no/project-id` — Busy time-tracking ID. The conventional value is the **uppercase project key** (e.g. `ACME`).
- `jira.com/project-key` — Jira board key. Auto-linked to `tryoslo.atlassian.net/browse/<key>`.
- `try.no/sla-tier` — SLA tier. **Default to `standard`** when generating. The CS team is the source of truth for anything other than standard — ask the user (who will check with CS) before writing a non-default value.
- `try.no/sla-agreement` — Free-text reference to the SLA contract. CS team is the source of truth.
- `try.no/sla-client-domain` — Production domain for the client. Usually obvious from the System's production link.
- `try.no/project-owner` — The PM / project owner (person, not GitHub team).
- `try.no/documentation` — External docs URL (Notion, Confluence, Storybook…). Separate from `backstage.io/techdocs-ref` which renders in-repo Markdown inside Backstage.
- `try.no/log-drain` — Where logs go (e.g. `hyperdx`, `vercel`). If both Components share the same drain, set once on the System.

Put these on the **System** entity by default. Only override on a Component when that Component differs (e.g. web logs go to Vercel while the worker exports OTEL to HyperDX).

### Links

Put `metadata.links` on both the System AND on each Component. Links are where developers actually click — they pay off every day. Be generous; the cost of an extra link is near-zero, the cost of a missing daily-click link is real friction.

#### Required System links

For every System, **always** include these links if you can identify the underlying service from the repo:

| Link                          | When to include                                                        | How to derive                                                                                                                                                                                                                                                                                                                                            |
| ----------------------------- | ---------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Production URL**            | Always                                                                 | Ask the user if not obvious from the repo (often in `vercel.json` rewrites, README, or `try.no/sla-client-domain`).                                                                                                                                                                                                                                      |
| **Hosting platform project**  | When `dependsOn: resource:tryhuset/vercel` or `:cloudflare` is present | Vercel: `https://vercel.com/<team>/<project>` (team is usually `tryhuset` or `FrendDigital` — confirm from `vercel.json` or the preview URL slug). CF: `https://dash.cloudflare.com/<account_id>` from `wrangler.jsonc`.                                                                                                                                 |
| **CMS admin**                 | When the product uses a CMS (Storyblok, Sanity, Contentful…)           | Storyblok: `https://app.storyblok.com/#/me/spaces/<SB_SPACE_ID>/dashboard` — pull `SB_SPACE_ID` from `wrangler.jsonc` or `.env*`.                                                                                                                                                                                                                        |
| **Commerce/e-commerce admin** | When the product uses Centra, Shopify, etc.                            | Centra: `https://<store>.centra.com/<admin-path>` — derive `<store>` from `CENTRA_INTEGRATION_URL` in `wrangler.jsonc` or similar. **`<admin-path>` is NOT a constant** (it's a per-instance Centra path like `ams2019`, but varies between clients). **Always ask the user** what the Centra admin path is — do not borrow it from another TRY project. |

> **Shape vs. values — that's the line.**
>
> **What's safe to infer from vendor URL conventions** (public, documented, don't change per client) — the _shape_ of the URL:
>
> - Vercel: `https://vercel.com/<team>/<project>`
> - Cloudflare workers: `https://dash.cloudflare.com/<account_id>/workers/services/view/<worker_name>/production`
> - Cloudflare queues: `https://dash.cloudflare.com/<account_id>/workers/queues/<queue_name>`
> - Storyblok space: `https://app.storyblok.com/#/me/spaces/<space_id>/dashboard`
> - Centra admin: `https://<store>.centra.com/<admin-path>`
>
> **What must be grounded in this repo** (read it out of a file, don't assume) — every variable inside those URLs: the **project, account ID, worker name, queue name, store slug, Centra admin path, Storyblok space ID**. For each one, look in this order: `wrangler.jsonc`, `vercel.json`, `.env*` / `.env.example`, `README*`, deploy scripts (`package.json` scripts, CI workflows). If the slug isn't in any file you can read from _this_ repo, **ask the user**.
>
> **House constants** (don't ask, don't borrow per-project, just use these):
>
> - Vercel team: `tryhuset`
> - GitHub org: `tryhuset`

#### Required Component links

For every Component, **always** include the dashboard URLs that match its `dependsOn`:

- `dependsOn: resource:tryhuset/vercel` → add `https://vercel.com/<team>/<project>` on the Component.
- `dependsOn: resource:tryhuset/cloudflare` → add the worker dashboard URL: `https://dash.cloudflare.com/<account_id>/workers/services/view/<worker_name>/production` (worker name is `name` in `wrangler.jsonc`, account ID is `account_id`).
- `dependsOn: resource:<your-queue>` (Cloudflare Queue) → add the queue dashboard if the worker is the producer/consumer: `https://dash.cloudflare.com/<account_id>/workers/queues/<queue-name>`.

Add the public production URL only on the Component that actually serves traffic (typically the web Component) — not on workers.

```yaml
links:
  - url: https://www.example.no
    title: Production
    icon: web
  - url: https://vercel.com/tryhuset/example
    title: Vercel project
    icon: dashboard
```

---

## Entity Conventions

### System

Top-level product grouping. Exactly one per file.

- `spec.owner` (as `group:default/<gh-team-slug>`)
- `spec.lifecycle`
- Rich `metadata.links` (production, staging, CMS, hosting, admin)
- Add `backstage.io/techdocs-ref: dir:.` so the README renders in Backstage
- Do **not** set `metadata.namespace` on the System (or its child entities) — local product entities live in `default`
- Do **not** set `spec.domain`

### Component

Deployable or buildable units.

- `spec.type`: `service`, `website`, `mobile`, `library`, `tool`, etc.
- `spec.system`: the System name
- `metadata.links`: per-component dashboards and URLs
- `backstage.io/source-location` annotation pointing at the monorepo subdir
- `providesApis`: APIs this component exposes (must have a matching API entity)
- `consumesApis`: HTTP services this component calls (third-party AND sibling components)
- `dependsOn`: platforms and runtime bindings — **never** an HTTP relation

### Resource

Infrastructure we provision: queues, durable objects, KV, deployments, hardware.

- `spec.type`: `database`, `storage`, `infrastructure`, `service`, `deployment`
- `dependsOn`: the shared platform it lives on, e.g. `resource:tryhuset/cloudflare`
- Use `providesApis` only if the resource genuinely exposes an HTTP/protocol API consumers reference

### API

Integration surfaces. Use sparingly in the local file — the only common case is **internal APIs between sibling Components in the same product** (e.g. a Next.js `/api/revalidate` endpoint called by a worker).

- `spec.type`: `rest`, `openapi`, `graphql`, `websocket`, `grpc`, `asyncapi`
- `spec.lifecycle`, `spec.owner` (group), `spec.system`
- `definition`: an inline note pointing to method, auth model, source file. Doesn't need to be a full OpenAPI doc.
- Third-party and shared internal APIs are referenced via `consumesApis`, not redefined locally. See _Shared catalog references_.

---

## Template

Use this as the starting point. Remove sections that don't apply, add sections as needed. Keep the header comment and entity order intact.

```yaml
# Backstage catalog manifest for the <Product> product.
#
# Quick primer:
#   System    = the product as a whole.
#   Component = something we build and deploy.
#   Resource  = infrastructure we depend on.
#   API       = an HTTP/websocket surface — third-party or one of our own.
#
# House conventions:
#   - consumesApis  → who calls an API (HTTP direction). Third-party AND component-to-component.
#   - providesApis  → who exposes an API. Needs a matching API entity.
#   - dependsOn     → platforms / runtime bindings. NOT for HTTP.
#   - owner         → GitHub team slug, written as group:default/<slug>.
#
# Reference catalog: https://github.com/tryhuset/try-backstage-catalog
---
apiVersion: backstage.io/v1alpha1
kind: System
metadata:
  name: <system-name>
  title: <System Title>
  description: <One- or two-sentence narrative of what this product is and what it runs on.>
  links:
    - url: https://www.<production>.no
      title: Production
      icon: web
    - url: https://vercel.com/tryhuset/<project>
      title: Vercel
      icon: dashboard
  tags:
    - <domain-tag>
    - <framework-tag>
    - <tech-tag>
  annotations:
    github.com/project-slug: tryhuset/<repo-name>
    busy.no/project-id: <UPPERCASE-KEY>
    try.no/sla-tier: standard # default; CS team owns non-standard values
    try.no/log-drain: hyperdx # only set on System if all Components share it
    # backstage.io/techdocs-ref: dir:.                 # add only if the repo has a mkdocs.yml
spec:
  owner: group:default/<gh-team-slug>
  lifecycle: production
---
# Local API — only when a sibling Component in this product calls it.
# Third-party / shared APIs live in try-backstage-catalog.
apiVersion: backstage.io/v1alpha1
kind: API
metadata:
  name: <product>-<api-name>
  title: <API Title>
  description: <What this endpoint does and who calls it.>
spec:
  type: rest
  lifecycle: production
  owner: group:default/<gh-team-slug>
  system: <system-name>
  definition: |
    <METHOD> /<path>
    Auth: <auth model>
    Source: <file path>
---
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: <component-name>
  title: <Component Title>
  description: <What this component does — narrative, not boilerplate.>
  tags:
    - <tech-tags>
  links:
    - url: https://vercel.com/tryhuset/<project>
      title: Vercel project
      icon: dashboard
  annotations:
    backstage.io/source-location: url:https://github.com/tryhuset/<repo>/tree/main/apps/<subdir>
spec:
  type: service
  lifecycle: production
  owner: group:default/<gh-team-slug>
  system: <system-name>
  providesApis:
    - <product>-<api-name>
  consumesApis:
    - tryhuset/<external-api>
    - public/<public-api>
  dependsOn:
    - resource:tryhuset/<hosting-platform>
    - resource:<product>-<local-resource>
---
apiVersion: backstage.io/v1alpha1
kind: Resource
metadata:
  name: <product>-<resource-name>
  title: <Resource Title>
  description: <What this resource is and what it's used for.>
  tags:
    - <tech-tags>
spec:
  type: infrastructure
  lifecycle: production
  owner: group:default/<gh-team-slug>
  system: <system-name>
  dependsOn:
    - resource:tryhuset/<shared-platform>
```

---

## Checklist

Before finishing, verify:

**Factual specifics (most important — these are the bugs that survive review):**

- [ ] Every HTTP method, auth env-var name, and source path written into a local `API` entity's `definition` was read out of the actual route file in _this_ session — not borrowed from a similar product
- [ ] Every Resource `metadata.name` either matches the literal infra binding name from `wrangler.jsonc` / `vercel.json` / equivalent, OR the real name appears in the Resource's `description`
- [ ] Every directory under `apps/*` (and `packages/*`, `services/*`) was either modeled as an entity or explicitly skipped with a reason
- [ ] `github.com/project-slug` matches the actual GitHub remote (or has been confirmed with the user if the repo was moved/mirrored)

**Structural:**

- [ ] **Header comment** present, with the conventions block
- [ ] System entity comes first; entity order: System → local APIs → Components → Resources
- [ ] All `metadata.name` values use kebab-case
- [ ] No `metadata.namespace` on local product entities (they live in `default`)
- [ ] Shared internal services referenced as `resource:tryhuset/<name>`, not redefined
- [ ] Public APIs referenced as `public/<api-name>`
- [ ] `owner` everywhere is written as `group:default/<gh-team-slug>`
- [ ] `consumesApis` used for ALL HTTP (third-party AND sibling components)
- [ ] `providesApis` only used when a matching API entity exists
- [ ] `dependsOn` is only platforms / runtime bindings — no HTTP relations
- [ ] No relation duplicated across `dependsOn` and `consumesApis`
- [ ] `backstage.io/techdocs-ref: dir:.` on the System **only if** a `mkdocs.yml` exists in the repo (don't promise rendering that won't happen)
- [ ] `backstage.io/source-location` on each Component in a monorepo
- [ ] **System `metadata.links` includes the daily-click set:** production URL, hosting platform project (Vercel/CF), CMS admin (if applicable), commerce admin (if applicable). Missing one is a defect, not a stylistic choice.
- [ ] **Each Component has the dashboard URL matching its `dependsOn`**: Vercel project for Vercel-hosted Components, Cloudflare worker dashboard for Workers, etc.
- [ ] Every entity has `name`, `title`, `description`, `tags`, `owner`, `lifecycle`
- [ ] `kind:` values use TitleCase (`System`, `Component`, `Resource`, `API`)
- [ ] File uses `---` separators between entities
- [ ] No duplicate entity names
- [ ] At minimum: one System, one Component for the core deployable, one Resource for the hosting platform binding

---

## Final step: hand off for human review

After writing the file, tell the user explicitly:

> Generated `backstage.yaml` is ready. **Please read it top-to-bottom before committing** — the catalog records facts about ownership, infrastructure, and integrations that other people will rely on. Pay particular attention to:
>
> - Entity names and the `dependsOn` / `consumesApis` graph — does it match how the system actually works?
> - Real-world specifics (HTTP methods, env var names, resource binding names, source paths, account/space/queue IDs) — do they match the running system?
> - Owner team — is it the right team that should be paged for this product?
> - Annotations (Busy ID, SLA tier, Centra admin path, HubSpot ID, etc.) — are the values correct?
>
> Edit anything that's wrong before committing. The skill makes its best inference from this repo's source, but you are the source of truth.

Do not assume silence is approval. The skill's job is to draft; the human's job is to verify.
