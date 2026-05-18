---
name: backstage
description: Generate a backstage.yaml catalog file for a product or system following internal conventions. Use when the user asks to create, scaffold, or update a backstage.yaml, Backstage catalog, or service catalog entry.
tools: Read, Glob, Grep, Write, AskUserQuestion, SemanticSearch
license: MIT
metadata:
  author: eirikeikaas
  version: "1.1.0"
---

# Backstage Catalog Generator

Generate a multi-document `backstage.yaml` for a product/system following internal catalog conventions.

The output should be readable by a developer who has never opened Backstage before. Prefer narrative descriptions over boilerplate, lean on links and annotations that make the page useful day-to-day, and use the relations that encode reality — not approximations.

## Workflow

### Step 1: Gather Context

Before generating anything, understand the system:

1. **Check for an existing `backstage.yaml`** in the repo root using Glob. If one exists, read it — you may be updating, not creating from scratch.
2. **Scan the codebase** to infer components, tech stack, and structure:
   - Monorepo? Look at `apps/*`, `packages/*`, `services/*` — each deployable typically becomes its own Component.
   - `package.json`, `Dockerfile`, `*.csproj`, `build.gradle`, `go.mod`, `wrangler.jsonc` — identify component types and platforms.
   - DB/ORM config, migrations, queue config, KV/Durable Object bindings — these become Resources.
   - API route handlers, OpenAPI specs, websocket handlers — these become APIs (and `providesApis` on the host Component).
   - Outbound HTTP calls (env vars like `*_URL`, SDKs for Storyblok/Centra/etc.) — these become `consumesApis`.
3. **Check `try-backstage-catalog`** before inventing entities. Shared services (Vercel, Cloudflare, Sentry, Storyblok, Centra, HyperDX…) already exist in `entities/internal.yaml`. Reference them as `resource:tryhuset/<name>` — do not duplicate.
4. **Ask the user** (via AskUserQuestion) for anything you cannot infer:
   - System name, title, description
   - Owning **GitHub team slug** (not a friendly name — see *Owner convention* below)
   - Lifecycle (`production`, `development`, `experimental`)
   - SLA tier/agreement (if applicable)
   - Any external/partner APIs consumed that aren't already in the shared catalog

### Step 2: Generate the backstage.yaml

Write the file to the repository root as `backstage.yaml`.

**Always start the file with a header comment block** explaining the entity kinds and house conventions (see *Header comment* below). This is non-negotiable — it's the first thing every reader sees and is the single biggest onboarding lift.

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
#   - owner         → always a GitHub team slug, written as group:<slug>.
#
# Reference catalog: https://github.com/tryhuset/try-backstage-catalog
```

---

## Entity Ordering

Always use this order in the file, separated by `---`:

1. **System** (exactly one per file)
2. **API** entries that are *local* to this product (e.g. an internal `/api/revalidate` provided by our web app and consumed by our worker)
3. **Component** entries (the codebases we build/deploy: web, worker, mobile, studio)
4. **Resource** entries (queues, durable objects, KV namespaces, deployments — anything we provision)

> Shared infra and third-party services (Vercel, Cloudflare, Sentry, Storyblok, Centra, HyperDX…) live in `try-backstage-catalog`. Reference them — do **not** redefine them here.

> Local API entities are appropriate when your product exposes an HTTP surface that another component in the same product (or another TRY product) calls. They are **not** appropriate for documenting your callees — those are referenced through `consumesApis`.

---

## Owner convention

`spec.owner` always resolves to a `Group`. In our setup, Groups come from two places:

1. **GitHub teams under the `tryhuset` org** — auto-imported by the `githubOrg` catalog provider as `group:default/<gh-team-slug>`. This is the source of truth.
2. **Manually-defined groups** in `try-backstage-catalog/org/*.yaml` — typically partner orgs (`back`, `partner`) or non-GitHub business units.

**Always write owner as `group:<gh-team-slug>`** — explicit kind, slug from the GitHub team. Examples:

- ✅ `owner: group:ecom-tech`
- ✅ `owner: group:creative-tech`
- ❌ `owner: ecommerce` (friendly name — may not resolve)
- ❌ `owner: ecom-tech` (works, but ambiguous — looks like it could be a User)

If you don't know the GitHub team, ask the user. Do not guess.

---

## Naming and Namespaces

- Use **kebab-case** for all `metadata.name` values.
- **Local product entities** (your System, Components, Resources, internal APIs) live in the **default namespace** — don't set `metadata.namespace`. Avoids verbose refs like `hest/hest-web`.
- **Internally managed shared services/APIs** live in the `tryhuset` namespace and are defined in `try-backstage-catalog`. Reference them as `resource:tryhuset/<name>`, `api:tryhuset/<name>`. Examples:
  - `resource:tryhuset/vercel`
  - `resource:tryhuset/cloudflare`
  - `resource:tryhuset/sentry`
  - `resource:tryhuset/hyperdx`
- **Public APIs** set `metadata.namespace: public` and are referenced as `public/<api-name>` in `consumesApis`.
- **Partner/external-org resources** use their org namespace, e.g. `resource:back/knox`.

> If an internally managed service does not already exist in `try-backstage-catalog`, create it there first. Then reference it from the product `backstage.yaml`.

---

## The relation rules (most-violated, most-important)

This is the core conceptual model. Get this right and most other things follow.

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
- `spec.owner` (as `group:<slug>`), `spec.lifecycle`, and `spec.system` (where applicable)

### Annotations

The following annotations are commonly useful. Use what applies; don't include placeholders.

```yaml
annotations:
  # Source / docs
  github.com/project-slug: tryhuset/repo-name
  backstage.io/source-location: url:https://github.com/tryhuset/repo-name/tree/main/apps/web
  try.no/documentation: https://...                               # external link to docs (Notion, Confluence, etc.)
  # backstage.io/techdocs-ref: dir:.                              # only if the repo has a mkdocs.yml and we want docs rendered IN Backstage

  # Project trackers
  jira.com/project-key: PROJ
  busy.no/project-id: "HEST"
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

- `busy.no/project-id` — Busy time-tracking ID. The conventional value is the **uppercase project key** (e.g. `HEST`).
- `jira.com/project-key` — Jira board key. Auto-linked to `tryoslo.atlassian.net/browse/<key>`.
- `try.no/sla-tier` — SLA tier. **Default to `standard`** when generating. The CS team is the source of truth for anything other than standard — ask the user (who will check with CS) before writing a non-default value.
- `try.no/sla-agreement` — Free-text reference to the SLA contract. CS team is the source of truth.
- `try.no/sla-client-domain` — Production domain for the client. Usually obvious from the System's production link.
- `try.no/project-owner` — The PM / project owner (person, not GitHub team).
- `try.no/documentation` — External docs URL (Notion, Confluence, Storybook…). Separate from `backstage.io/techdocs-ref` which renders in-repo Markdown inside Backstage.
- `try.no/log-drain` — Where logs go (e.g. `hyperdx`, `vercel`). If both Components share the same drain, set once on the System.

Put these on the **System** entity by default. Only override on a Component when that Component differs (e.g. web logs go to Vercel while the worker exports OTEL to HyperDX).

### Links

Put `metadata.links` on both the System AND on each Component. Links are where developers actually click — they pay off every day.

- **System links**: Production URL, staging, CMS admin, hosting dashboard, e-commerce admin
- **Component links**: that component's Vercel project, Cloudflare worker dashboard, queue dashboard, etc.

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

- `spec.owner` (as `group:<gh-team-slug>`)
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
- Third-party and shared internal APIs (Storyblok, Centra, Vercel, etc.) belong in `try-backstage-catalog`, not in the product file. Reference them via `consumesApis`.

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
#   - owner         → GitHub team slug, written as group:<slug>.
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
    try.no/sla-tier: standard                          # default; CS team owns non-standard values
    try.no/log-drain: hyperdx                          # only set on System if all Components share it
    # backstage.io/techdocs-ref: dir:.                 # add only if the repo has a mkdocs.yml
spec:
  owner: group:<gh-team-slug>
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
  owner: group:<gh-team-slug>
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
  owner: group:<gh-team-slug>
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
  owner: group:<gh-team-slug>
  system: <system-name>
  dependsOn:
    - resource:tryhuset/<shared-platform>
```

---

## Checklist

Before finishing, verify:

- [ ] **Header comment** present, with the conventions block
- [ ] System entity comes first; entity order: System → local APIs → Components → Resources
- [ ] All `metadata.name` values use kebab-case
- [ ] No `metadata.namespace` on local product entities (they live in `default`)
- [ ] Shared internal services referenced as `resource:tryhuset/<name>`, not redefined
- [ ] Public APIs referenced as `public/<api-name>`
- [ ] `owner` everywhere is written as `group:<gh-team-slug>`
- [ ] `consumesApis` used for ALL HTTP (third-party AND sibling components)
- [ ] `providesApis` only used when a matching API entity exists
- [ ] `dependsOn` is only platforms / runtime bindings — no HTTP relations
- [ ] No relation duplicated across `dependsOn` and `consumesApis`
- [ ] `backstage.io/techdocs-ref: dir:.` on the System
- [ ] `backstage.io/source-location` on each Component in a monorepo
- [ ] `metadata.links` on the System AND on each Component
- [ ] Every entity has `name`, `title`, `description`, `tags`, `owner`, `lifecycle`
- [ ] `kind:` values use TitleCase (`System`, `Component`, `Resource`, `API`)
- [ ] File uses `---` separators between entities
- [ ] No duplicate entity names
- [ ] At minimum: one System, one Component for the core deployable, one Resource for the hosting platform binding
