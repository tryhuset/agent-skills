# Catalog conventions

How to model a product in `backstage.yaml`: entity kinds, ordering, naming, ownership, shared references, relations and links. Annotations live in [`annotations.md`](annotations.md).

## Header comment

Every file starts with the header comment from [`template.yaml`](template.yaml): a quick primer on the four entity kinds, the four house-convention lines (`consumesApis`, `providesApis`, `dependsOn`, `owner`) and the reference catalog link. It's the first thing every reader sees and the biggest onboarding lift, so it's not optional. Adapt the wording to the product, but keep all four convention lines.

Below it, add one comment line for anything intentionally not modeled (skipped deployables, internal workspace libraries).

## Entity ordering

Always use this order, separated by `---`:

1. **System** — exactly one per file
2. **API** entries that are _local_ to this product (e.g. an internal `/api/revalidate` provided by our web app and called by our worker)
3. **Component** entries — the codebases we build and deploy: web, worker, mobile, studio
4. **Resource** entries — queues, durable objects, KV namespaces, buckets, anything we provision

Only **local** entities go in the product's `backstage.yaml`. Shared services are referenced, not redefined (see _Shared catalog references_).

Local API entities fit when the product exposes an HTTP surface that another component in the same product (or another TRY product) calls. They're **not** for documenting what you call — callees are referenced through `consumesApis`.

## Owner

`spec.owner` always resolves to a `Group`. Groups come from two places:

1. **GitHub teams in the `tryhuset` org** — auto-imported as `group:default/<gh-team-slug>`. This is the source of truth.
2. **Manually defined groups** in `try-backstage-catalog/org/*.yaml` — typically partner orgs or non-GitHub business units.

Always write the owner fully qualified, as `group:default/<gh-team-slug>`. Groups always live in the `default` namespace, and spelling it out makes the reference resolve whatever namespace the containing entity uses.

- ✅ `owner: group:default/ecom-tech`
- ✅ `owner: group:default/creative-tech`
- ⚠️ `owner: group:ecom-tech` — only works if the entity is also in `default`; fragile
- ❌ `owner: ecommerce` — friendly name, may not resolve
- ❌ `owner: ecom-tech` — ambiguous, could be a User

If you don't know the team, ask. Don't guess.

## Shared catalog references

Third-party services, hosting platforms, observability backends and other shared infrastructure live in **`try-backstage-catalog`** (https://github.com/tryhuset/try-backstage-catalog). Don't clone or read that repo. Find shared entities with the Backstage MCP when it's available (see below); the canonical name pattern is:

- Resource: `resource:tryhuset/<service-slug>` — e.g. `resource:tryhuset/vercel`, `resource:tryhuset/cloudflare`, `resource:tryhuset/betterstack`
- API: `tryhuset/<service-slug>-api` — e.g. `tryhuset/storyblok-api`, `tryhuset/centra-api`
- Public API: `public/<api-name>` — e.g. `public/udir-api`
- Partner-owned: `resource:<org>/<name>` or `<org>/<name>-api` — e.g. `<partner-org>/<service>-api`

`<service-slug>` is the kebab-case brand name: `google-maps`, `azure-openai`, `aws-s3`, `cloudflare`.

Don't redefine shared services as local entities. When the Backstage MCP is available, look the service up first (SKILL.md Step 1.7) and use the existing `entityRef` exactly. An existing entity may not follow the brand-name pattern (e.g. `tryhuset/gcp-maps-api` for Google Maps), and guessing a second slug creates a duplicate. When nothing matches, or you couldn't look it up, **reference the canonical slug** and list it in the hand-off. If it doesn't exist yet, Backstage shows a temporary "entity not found" warning, and a catalog admin reconciles it later (adds the entity, or asks for a rename if the slug is non-standard). That keeps product PRs unblocked.

## Naming and namespaces

- **kebab-case** for all `metadata.name` values.
- **Local product entities** (System, Components, Resources, local APIs) live in the **default namespace** — don't set `metadata.namespace`. That avoids verbose refs like `acme/acme-web`.
- **Shared services** live in the `tryhuset`, `public` or partner namespaces. You reference them; you don't define them.

## Relation rules

The core model. Get this right and most other things follow.

### Pick the right entity kind first

- **API** = something your code _calls_ over the network (HTTP, GraphQL, webhooks). Referenced via `consumesApis`. Examples: Storyblok, Centra, Mapbox, any third-party HTTP endpoint.
- **Resource** = something your code _runs on_ or is bound to at runtime. Referenced via `dependsOn`. Examples: Vercel, Cloudflare, BetterStack, queues, durable objects.

For a given service, pick the one that matches what you do with it, and don't reference both. The field you're filling dictates the kind: `consumesApis` takes APIs, `dependsOn` takes Resources.

### `consumesApis` — who calls what (HTTP direction)

Use for any outbound HTTP call:

- Third-party APIs: `tryhuset/storyblok-api`, `tryhuset/centra-api`, `public/udir-api`
- **Internal APIs in the same product**: if the worker calls the web app's `/api/revalidate`, model the endpoint as a local `API` entity and put it in the worker's `consumesApis`.

### `providesApis` — who exposes the API

Use on the producing Component. Every entry needs a matching `API` entity, local in this file or in `try-backstage-catalog`.

### `dependsOn` — runtime and platform bindings

Use for **non-HTTP** relations:

- Hosting platforms: `resource:tryhuset/vercel`, `resource:tryhuset/cloudflare`
- Data bindings: `resource:my-product-queue` (a Cloudflare Queue bound to the worker), `resource:my-product-durable-object`
- Log drains and observability backends, e.g. `resource:tryhuset/betterstack`
- Sibling Components only when there's a real runtime data binding (rare — usually it's HTTP, so `consumesApis`)

### Don't duplicate edges

Pick **one** edge type per relationship:

- HTTP calls to a service that also has a resource → `consumesApis: tryhuset/storyblok-api` **only**. No `dependsOn: resource:tryhuset/storyblok`.
- A sibling component called over HTTP → `consumesApis: <our-local-api>` **only**. No `dependsOn: component:other-component`.

A second edge for the same relationship is noise: it makes the graph misleading and the file harder to maintain.

### Common mistake: cross-component `dependsOn`

If a subcomponent (GUI, mobile, studio) calls a server over HTTP, that's **`consumesApis` of the server's API entity**, not `dependsOn: component:server`. `dependsOn` implies a runtime data binding (shared queue, shared DB), which usually isn't what's happening.

## Entity conventions

Every entity needs `metadata.name`, `metadata.title`, a narrative `metadata.description`, `metadata.tags` (domain, tech and platform), `spec.owner` (as `group:default/<slug>`), `spec.lifecycle`, and `spec.system` where it applies.

### System

Top-level product grouping, exactly one per file.

- `spec.owner` and `spec.lifecycle`
- Rich `metadata.links` (see _Links_)
- Project-wide annotations such as `github.com/project-slug` and `try.no/sla-tier` (see [`annotations.md`](annotations.md))
- No `metadata.namespace` and no `spec.domain`

### Component

Deployable or buildable units.

- `spec.type`: `website` for anything serving pages to users, `service` for backends and workers, plus `mobile`, `library`, `tool` and so on. `website` matters: production websites are the components the operational baseline requires monitoring for.
- `spec.system`: the System name
- `metadata.links`: per-component dashboards and URLs
- `backstage.io/source-location` pointing at the monorepo subdirectory
- BetterStack monitoring annotations on production websites (see [`annotations.md`](annotations.md))
- `providesApis`, `consumesApis` and `dependsOn` per the relation rules

### Resource

Infrastructure we provision: queues, durable objects, KV, buckets, databases.

- `spec.type`: `database`, `storage`, `infrastructure`, `service`, `deployment`
- `dependsOn`: the shared platform it lives on, e.g. `resource:tryhuset/cloudflare`
- `providesApis` only if it genuinely exposes an HTTP or protocol API that consumers reference

### API

Use sparingly in the local file — the common case is an **internal API between sibling Components** (e.g. a Next.js `/api/revalidate` endpoint called by a worker).

- `spec.type`: `rest`, `openapi`, `graphql`, `websocket`, `grpc`, `asyncapi`
- `spec.lifecycle`, `spec.owner`, `spec.system`
- `definition`: an inline note with the method, path, auth model and source file. It doesn't need to be a full OpenAPI document.

## Links

Put `metadata.links` on the System **and** on each Component. Links are where developers actually click, so be generous: an extra link costs nothing, a missing daily-click link is real friction.

### Required System links

Include each of these whenever you can identify the service from the repo. A missing one is a defect, not a style choice.

| Link | When | How to derive |
| --- | --- | --- |
| **Production URL** | Always | From the README, `vercel.json`, framework config or `try.no/sla-client-domain`. Ask if it isn't clear. |
| **Hosting project** | When `dependsOn` has `resource:tryhuset/vercel` or `resource:tryhuset/cloudflare` | Vercel: `https://vercel.com/<team>/<project>`. Cloudflare: `https://dash.cloudflare.com/<account_id>` with `account_id` from `wrangler.jsonc`. |
| **CMS admin** | When the product uses a CMS (Storyblok, Sanity, Contentful…) | Storyblok: `https://app.storyblok.com/#/me/spaces/<space_id>/dashboard`, with the space ID from `wrangler.jsonc` or `.env.example`. |
| **Commerce admin** | When the product uses Centra, Shopify, etc. | Centra: `https://<store>.centra.com/<admin-path>`. Derive `<store>` from e.g. `CENTRA_INTEGRATION_URL` in `wrangler.jsonc`. The admin path varies per client — **always ask**, never borrow it from another project. |

### Required Component links

Every Component gets the dashboard URL matching its `dependsOn`:

- `resource:tryhuset/vercel` → `https://vercel.com/<team>/<project>`
- `resource:tryhuset/cloudflare` → `https://dash.cloudflare.com/<account_id>/workers/services/view/<worker_name>/production` (`name` and `account_id` from `wrangler.jsonc`)
- A Cloudflare Queue the worker produces to or consumes → `https://dash.cloudflare.com/<account_id>/workers/queues/<queue_name>`

Add the public production URL only on the Component that serves traffic (usually the web Component), not on workers.

```yaml
links:
  - url: https://www.example.no
    title: Production
    icon: web
  - url: https://vercel.com/tryhuset/example
    title: Vercel project
    icon: dashboard
```

### Shape vs. values

**Safe to take from vendor URL conventions** (public, documented, the same for every client) — the _shape_ of the URL:

- Vercel: `https://vercel.com/<team>/<project>`
- Cloudflare worker: `https://dash.cloudflare.com/<account_id>/workers/services/view/<worker_name>/production`
- Cloudflare queue: `https://dash.cloudflare.com/<account_id>/workers/queues/<queue_name>`
- Storyblok space: `https://app.storyblok.com/#/me/spaces/<space_id>/dashboard`
- Centra admin: `https://<store>.centra.com/<admin-path>`

**Must come from this repo or the user** — every variable inside those URLs: project, account ID, worker name, queue name, store slug, Centra admin path, Storyblok space ID. Look in `wrangler.jsonc`, `vercel.json`, `.env.example`, `README*` and deploy scripts; if it isn't there, ask.

**House constants** — use these, don't ask:

- Vercel team: `tryhuset`
- GitHub org: `tryhuset`
