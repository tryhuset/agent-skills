---
name: backstage
description: Generate a backstage.yaml catalog file for a product or system following internal conventions. Use when the user asks to create, scaffold, or update a backstage.yaml, Backstage catalog, or service catalog entry.
tools: Read, Glob, Grep, Write, AskQuestion, SemanticSearch
license: MIT
metadata:
  author: eirikeikaas
  version: "1.0.0"
---

# Backstage Catalog Generator

Generate a multi-document `backstage.yaml` for a product/system following internal catalog conventions.

## Workflow

### Step 1: Gather Context

Before generating anything, understand the system:

1. **Check for an existing `backstage.yaml`** in the repo root using Glob. If one exists, read it — you may be updating, not creating from scratch.
2. **Read `.mcpcontext`** if it exists to discover project keys (Jira, Sentry, GitHub slug, Busy project ID, etc.).
3. **Scan the codebase** to infer components, tech stack, and structure:
   - Look for `package.json`, `Dockerfile`, `Podfile`, `*.csproj`, `build.gradle`, `go.mod`, or similar to identify component types and technologies.
   - Look for database config files, migration directories, or ORM setup to identify resources.
   - Check for API route definitions, OpenAPI specs, or socket handlers to identify APIs.
4. **Ask the user** (via AskQuestion when available) for anything you cannot infer:
   - System name, title, and description
   - Owner (group or user)
   - Lifecycle (`production`, `development`, `experimental`)
   - Domain (if applicable)
   - SLA tier and agreement (if applicable)
   - Any external APIs consumed

### Step 2: Generate the backstage.yaml

Write the file to the repository root as `backstage.yaml`. Use the entity ordering, naming, and conventions described below.

### Step 3: Verify

After writing, read the file back to confirm it is valid multi-document YAML with correct `---` separators and no syntax issues.

---

## Entity Ordering

Always use this order in the file, separated by `---`:

1. **System** (exactly one per file)
2. **Component** entries (the codebases where logic lives: server, GUI, mobile, studio, booking, etc.)
3. **Resource** entries (instances/environments: databases, infra, hardware, CMS, Docker instances, Vercel deployments, `.apk`/`.ipa`, etc.)
4. **API** entries (integration interfaces between components and/or external systems)
5. **Group** entries (only if defined in the same file)

> API entities should normally be created and managed in `try-backstage-catalog`, then referenced from local entities. Only create local API entities when its explicitly established that its managed by the client and is not using the internally managed services/resources/APIs. E.g: The hosting is managed by the client's own Vercel instance, and not by Tryhuset.'

---

## Naming and Namespaces

- Use **kebab-case** for all `metadata.name` values.
- **Internally managed shared services/resources/APIs** use the `tryhuset` namespace. Reference them as `resource:tryhuset/<name>`, `api:tryhuset/<name>`, etc. Common examples:
  - `resource:tryhuset/sentry`
  - `resource:tryhuset/github-registry`
  - `resource:tryhuset/azure-aci`
  - `resource:tryhuset/azure-storage`
- **Public APIs** set `metadata.namespace: public` and are referenced as `public/<api-name>` in `consumesApis`.
- **External partner/org resources** use their org namespace, e.g. `resource:back/knox`, `resource:norges-bank/sentry`.

> If an internally managed service, resource, or API does not already exist, create it in the `tryhuset` namespace in the `try-backstage-catalog` repository.

---

## Required Metadata

Every entity must include:

- `metadata.name`, `metadata.title`, `metadata.description`
- `metadata.tags` — combine domain, tech, platform, and environment tags
- `spec.owner`, `spec.lifecycle`, and `spec.system` (where applicable)

### Annotations (use when values are known)

```yaml
annotations:
  github.com/project-slug: tryhuset/repo-name
  jira.com/project-key: PROJ
  busy.no/project-id: "123"
  sentry.io/project-slug: try-apt-as/project-name
  try.no/log-drain: "target"
  try.no/sla-tier: "tier"
  try.no/sla-agreement: "agreement"
```

---

## Entity Conventions

### System

Top-level product grouping. One per file.

- Define ownership with `spec.owner` and define a dedicated namespace for the system.
- Set `spec.lifecycle`.
- Use `spec.type` for framework/shared platform systems when applicable.
- Include key links (docs, repo, production URLs) in `metadata.links`.
- Do not use `spec.domain` in the TRY Backstage catalog.

### Component

Deployable or buildable units.

- `spec.type`: `service`, `website`, `mobile`, `library`, `tool`, etc.
- `dependsOn`: explicit kind prefixes — `component:name`, `resource:tryhuset/sentry`.
- `providesApis`: APIs this component exposes (normally APIs that are defined centrally in `try-backstage-catalog`).
- `consumesApis`: APIs this component uses. Use `public/<api-name>` for public APIs.
- Subcomponents (e.g. GUI, mobile, studio) are components that list the server in `dependsOn`.

### Resource

Infrastructure, databases, hardware, or external services.

- `spec.type`: `database`, `storage`, `infrastructure`, `server`, `service`.
- Use `providesApis` when the resource exposes an API.
- Use `dependsOn` for linked infra (e.g. system-specific storage depending on `resource:tryhuset/azure-storage`).

### API

Integration surfaces.

- Internal APIs are referenced by name in the same system.
- Public APIs set `metadata.namespace: public` and are referenced as `public/<api-name>`.
- Websocket game APIs use `spec.type: websocket` and `parent: foundation-socket-api` when applicable.
- Provide `target` or `definition` for clarity.
- Prefer defining API entities in `try-backstage-catalog` and referencing them here to avoid duplication of shared services/resources/APIs.

---

## Template

Use this as the starting point. Remove sections that don't apply, add sections as needed. Keep entity order intact.

```yaml
apiVersion: backstage.io/v1alpha1
kind: System
metadata:
  name: <system-name>
  title: <System Title>
  namespace: <system-namespace>
  description: <System description>
  tags:
    - <domain-tag>
    - <framework-tag>
    - <tech-tag>
  annotations:
    github.com/project-slug: tryhuset/<repo-name>
    jira.com/project-key: <JIRA_KEY>
spec:
  type: <optional-system-type>
  owner: <owner-group>
  lifecycle: production
  domain: <domain>
---
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: <component-name>
  title: <Component Title>
  namespace: <system-namespace>
  description: <Component description>
  tags:
    - <tech-tags>
  annotations:
    sentry.io/project-slug: try-apt-as/<sentry-project>
spec:
  type: service
  lifecycle: production
  owner: <owner-group>
  system: <system-name>
  providesApis:
    - <api-name>
  consumesApis:
    - public/<external-api>
  dependsOn:
    - component:<parent-component>
    - resource:tryhuset/sentry
    - resource:tryhuset/github-registry
---
apiVersion: backstage.io/v1alpha1
kind: Resource
metadata:
  name: <resource-name>
  title: <Resource Title>
  namespace: <system-namespace>
  description: <Resource description>
  tags:
    - <tech-tags>
spec:
  type: database
  lifecycle: production
  owner: <owner-group>
  system: <system-name>
  dependsOn:
    - resource:tryhuset/<shared-resource>
---
apiVersion: backstage.io/v1alpha1
kind: API
metadata:
  name: <api-name>
  namespace: <system-namespace>
  description: <API description>
spec:
  type: websocket
  lifecycle: production
  owner: <owner-group>
  system: <system-name>
  parent: foundation-socket-api
  target: http://<host>:<port>
---
apiVersion: backstage.io/v1alpha1
kind: API
metadata:
  name: <public-api-name>
  namespace: public
  description: <Public API description>
spec:
  type: rest
  lifecycle: production
  owner: public
  definition: |
    <API definition summary>
---
apiVersion: backstage.io/v1alpha1
kind: resource
metadata:
  name: <deployment-name>
  title: <Deployment Title>
  namespace: <system-namespace>
  description: <Deployment description>
spec:
  type: deployment
  lifecycle: production
  owner: <owner-group>
  system: <system-name>
  dependsOn:
    - resource:tryhuset/<shared-resource-for-hosting>
    - component:<component-name>
```

> Note: API entities are typically owned in `try-backstage-catalog`. If they already exist there, do not duplicate definitions in this repository; reference them via `providesApis` and `consumesApis`.

---

## Checklist

Before finishing, verify:

- [ ] System entity comes first
- [ ] Entity order: System > Components > Resources > APIs
- [ ] All `metadata.name` values use kebab-case
- [ ] Shared internal services reference `tryhuset` namespace
- [ ] Public APIs use `metadata.namespace: public`
- [ ] API entities are created in `try-backstage-catalog` unless explicitly requested locally because they are managed by the client, not shared and thus not shared internally.
- [ ] All components and resources have `dependsOn` where applicable
- [ ] API relationships use `providesApis` and `consumesApis`
- [ ] Every entity has `name`, `title`, `description`, `tags`, `owner`, and `lifecycle`
- [ ] Annotations populated from `.mcpcontext` or user input
- [ ] File uses `---` separators between entities
- [ ] No duplicate entity names
- [ ] Excpect there to minimum be one system, one component that is the core service provided (website, application, etc.), one resource that is the hosting platform deployment (Vercel, AWS, Azure, etc.).
