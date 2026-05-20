# MCP Servers — Technical Specification

| | |
|---|---|
| **Status** | Draft (living document) |
| **Last updated** | 2026-05-20 |
| **Audience** | Platform engineers, integrators |
| **Scope** | The catalog, server, and preset entities that back Archestra's MCP server registry, plus their lifecycle, deployment, and configuration surfaces |

---

## 1. Abstract

Archestra's platform manages MCP (Model Context Protocol) servers as a first-class resource: organizations curate a **catalog** of server definitions, install them as concrete **servers** that an agent can call, and optionally parameterize those installs with **presets** for per-environment variation. This document specifies the data model, configuration shapes, lifecycle, cascade-reinstall semantics, runtime/deployment model, and API surface of those three entities and their dependencies.

## 2. Background and motivation

MCP servers expose tools to LLM agents. A given organization typically wants:

- a **central registry** of approved MCP servers (so installations are reviewable, not ad-hoc);
- the ability to **edit a server's definition** centrally and have running installations pick up the change (without quietly breaking them);
- a **preset mechanism** so the same server template can be installed against multiple environments (e.g., "production GitHub" vs "staging GitHub") with environment-specific values;
- a **runtime layer** that turns "local" MCP server definitions into actual K8s pods, with both single-tenant (one pod per install) and multi-tenant (one shared pod per catalog item) deployment models;
- a **gateway** that exposes installed servers to agents via JSON-RPC, with policy enforcement at the gateway layer.

This spec captures the contracts the codebase has converged on across these concerns. It is descriptive of the current implementation, not prescriptive — discrepancies between this doc and the code are bugs in the doc.

## 3. Goals and non-goals

**In scope.** The three core entities (`internal_mcp_catalog`, `mcp_server`, `mcp_preset_entry`), their configuration shapes (`localConfig`, `userConfig`, `oauthConfig`, `enterpriseManagedConfig`), the cascade-reinstall decision tree, the K8s runtime model, and the public HTTP surface for managing them.

**Out of scope.** The MCP wire protocol itself (covered by [`mcp-authentication.md`][user-mcp-auth] and [`platform-mcp-gateway.md`][user-mcp-gateway] user docs), agent invocation semantics, tool invocation policies, trusted data policies, secret-storage backends (Vault / BYOS), and the underlying RBAC permission model (see [`platform-access-control.md`][user-rbac]).

## 4. Glossary

| Term | Meaning |
|---|---|
| **Catalog item** | A row in `internal_mcp_catalog`. The template for an MCP server. May be a *parent* (top-level definition) or a *child preset* (a variant of a parent tied to a preset entry). |
| **MCP server** | A row in `mcp_server`. A concrete installation of a catalog item; for `serverType=local`, owns a K8s deployment. |
| **Preset entry** | A row in `mcp_preset_entry`. Org-wide named bucket ("Production", "Staging") that child catalog items reference. |
| **Child preset / child catalog item** | A row in `internal_mcp_catalog` with `parentCatalogItemId != null` and `presetEntryId != null`. Inherits template fields from its parent and overlays its own `presetFieldValues`. |
| **Default preset** | The parent catalog item itself, when its `presetFieldValues` are treated as the default overlay used by installs that don't pick a child. |
| **Cascade reinstall** | The process triggered by a parent catalog edit (or child preset edit) that visits every installed `mcp_server` for the affected catalog and either skips, marks-for-manual-reinstall, or auto-reinstalls. |
| **Manual reinstall path** | Cascade marks `mcp_server.reinstallRequired = true`. The pod keeps running on the old config; the user must click *Reinstall* and re-supply changed inputs. |
| **Auto reinstall path** | Cascade fires a background `setImmediate` restart; the pod restarts immediately without user prompting. |
| **Forward-compatible change** | A schema change that does not invalidate existing installs (e.g., adding an *optional* prompted env var, demoting required→optional). |
| **Re-prompt change** | A change that requires the user to provide new input on the affected installs (e.g., adding a *required* prompted env var, renaming the server). |
| **Multi-tenant catalog** | A `serverType=local` catalog with `multitenant=true`. All installations share a single K8s deployment + secret; caller identity is propagated via request headers. |
| **MCP Gateway** | `/v1/mcp/:profileId` — token-authenticated, stateless JSON-RPC endpoint that LLM clients call to discover and execute tools. |
| **MCP Proxy** | `/api/mcp/:agentId` — session-authenticated endpoint used by the in-browser AppRenderer to proxy JSON-RPC to a server. |
| **Secret bag** | A row in `secret` storing a JSON object of secret values, referenced by an FK column on the entity that owns the secrets. |

## 5. Domain model

```
            ┌─────────────────────────────┐
            │     mcp_preset_entry        │   Org-wide named buckets
            │  (id, organizationId, name, │   ("Production", "Staging")
            │   validationRegex)          │   with optional regex enforcement
            └──────────────┬──────────────┘
                           │ FK: presetEntryId
                           │ (cascades on delete)
                           │
            ┌──────────────▼──────────────┐
            │   internal_mcp_catalog      │   Catalog template OR child preset.
       ┌────┤   parentCatalogItemId       │   Self-reference forms a depth-1 tree:
       │    │   (self FK, set on child)   │   parent rows are the templates;
       └────►   localConfig / userConfig  │   children are preset overlays.
            │   oauthConfig / icon /...   │
            │   clientSecretId ───────────┼───► secret (OAuth client_secret)
            │   localConfigSecretId ──────┼───► secret (static env-var secrets)
            │   presetSecretId ───────────┼───► secret (sensitive preset values)
            └──────────────┬──────────────┘
                           │ FK: catalogId
                           │ (set null on delete)
                           │
            ┌──────────────▼──────────────┐
            │        mcp_server           │   One row per installation
            │  environmentValues (jsonb)  │   (single-tenant) or one per
            │  secretId ──────────────────┼───► secret (per-install creds)
            │  reinstallRequired (bool)   │   catalog (multi-tenant, aliased).
            │  localInstallationStatus    │
            └─────────────────────────────┘
```

The catalog item is a depth-1 self-referencing tree: the root row carries the template and a *default* preset overlay; child rows carry per-environment preset overlays only. An `mcp_server` references whichever catalog row was installed — parent or child.

## 6. Catalog item — `internal_mcp_catalog`

### 6.1 Purpose

A catalog item is the *definition* of an MCP server: how to deploy it, how it authenticates, what inputs the installer must supply, and (for child rows) what preset values overlay the parent.

### 6.2 Schema overview

Defined in [`backend/src/database/schemas/internal-mcp-catalog.ts`][schema-cat]. Selected columns:

| Column | Type | Role |
|---|---|---|
| `id` | uuid pk | |
| `name` | text | Display name. For child presets: composed from `parent.name` + DNS-normalised entry name. |
| `version` / `description` / `instructions` / `repository` / `docsUrl` / `icon` | text | Metadata; `description` is the *only* field treated as metadata-only by the cascade gate. |
| `serverType` | enum `local`/`remote`/`builtin` | Drives whether a K8s deployment is created at install time. |
| `multitenant` | bool, default false | Local-only; immutable after create. `true` ⇒ shared deployment across installations. |
| `serverUrl` | text nullable | Remote endpoint (for `serverType=remote`). |
| `localConfig` | jsonb | K8s deployment spec — command, args, env, image, transport, ports. See §9.1. |
| `userConfig` | jsonb | Field declarations for per-install / per-preset values. See §9.2. |
| `oauthConfig` | jsonb | OAuth flow + client config for remote servers. See §9.3. |
| `enterpriseManagedConfig` | jsonb | Enterprise credential resolution config. See §9.4. |
| `clientSecretId` | uuid → `secret` | OAuth `client_secret` storage. |
| `localConfigSecretId` | uuid → `secret` | Static secret env vars from `localConfig.environment[*]` (where `type=secret` AND `promptOnInstallation=false`). |
| `parentCatalogItemId` | uuid → self, cascade delete | NULL ⇒ parent (template). Non-NULL ⇒ child preset. |
| `presetEntryId` | uuid → `mcp_preset_entry`, cascade delete | Required on child rows; the org-wide bucket the child belongs to. |
| `presetFieldValues` | jsonb default `{}` | Map of `userConfig` field key → value for fields marked `promptOnPreset` (default-overlay on parent, child-overlay on children). |
| `presetSecretId` | uuid → `secret` | Sensitive subset of `presetFieldValues`. |
| `childName` | text nullable | Bare entry name on child rows (used to compose `name`). |
| `deploymentSpecYaml` | text nullable | Custom K8s manifest override; null ⇒ generated from `localConfig` at deploy time. |
| `organizationId` / `authorId` / `scope` | text + enum | Multi-tenancy and visibility (`personal`/`team`/`org`). |
| `createdAt` / `updatedAt` | timestamp | Standard audit. |

**Unique constraint:** `(parentCatalogItemId, name)` — child preset names are unique within a parent.

**Side tables:** `mcp_catalog_labels`, `mcp_catalog_team` join the catalog to label key/value pairs and to teams (for `scope=team`).

Drizzle-zod schemas live in [`backend/src/types/mcp-catalog.ts`][types-cat] (`SelectInternalMcpCatalogSchema`, `InsertInternalMcpCatalogSchema`, `UpdateInternalMcpCatalogSchema`, `CreateChildCatalogSchema`, `UpdateChildCatalogSchema`).

### 6.3 Lifecycle

```
       create                edit                              delete
    ─────────────► ◄────────────────────────────────────── ────────────►
                       │
                       │  for each affected installed server:
                       │   ┌─ skip   (metadata-only or forward-compat)
                       │   ├─ manual (re-prompt change; reinstallRequired=true)
                       │   └─ auto   (setImmediate restart in background)
                       │
                       └─ for each child preset: sync template fields,
                          re-partition preset values into the secret bag
                          if `sensitive` flipped, recurse cascade
```

**Create.** `POST /api/internal_mcp_catalog` (parent) or `POST /api/internal_mcp_catalog/:catalogId/children` (child). Creates the `secret` bag(s) for OAuth client_secret, static env-var secrets, and (for children) sensitive preset values. Non-admins may only create personal-scope items.

**Edit.** `PUT /api/internal_mcp_catalog/:id` (parent) or `PATCH /api/internal_mcp_catalog/:catalogId/children/:childId` (child). Triggers the cascade decision (§10) for every installed server pointing at the affected catalog rows.

**Delete.** `DELETE /api/internal_mcp_catalog/:id`. Cascades to child presets (via `parentCatalogItemId` FK). Installed `mcp_server` rows have `catalogId` set to NULL (set-null FK).

### 6.4 Server types

| `serverType` | Meaning | Deployment model |
|---|---|---|
| `local` | Server is a containerised process Archestra runs in its own K8s cluster. | One K8s Deployment per install (single-tenant) or one shared Deployment per catalog (multi-tenant). `localConfig` is required (enforced at the route level; `LocalConfigSchema` itself only requires that either `command` or `dockerImage` is set when `localConfig` *is* present). |
| `remote` | Server is an HTTP endpoint owned by a third party (e.g., a SaaS MCP). | No deployment; `serverUrl` + `oauthConfig` (or static auth) drive client connections. |
| `builtin` | Tools provided by Archestra itself (the [Archestra MCP server][user-archestra-mcp]). | Backed by an in-process module; no install action visible to most users. |

### 6.5 Multi-tenancy

`multitenant=true` is permitted only for `serverType=local` and is **immutable after create** (omitted from `UpdateInternalMcpCatalogSchema`). When set:

- One K8s Deployment named `mcp-mt-<catalogId8>-<slugged-name>` serves all installs of this catalog.
- One K8s Secret named `mcp-server-mt-<catalogId8>-secrets` holds catalog-level static secrets.
- Per-install credentials are still stored in per-`mcp_server` secret bags; the gateway propagates caller identity via headers at request time.
- Teardown is reference-counted: `stopServer()` only deletes the K8s resources when no other `mcp_server` row references the same catalog ([`backend/src/k8s/mcp-server-runtime/manager.ts:408-433`][runtime-manager]).

If `enterpriseManagedConfig` is set and `serverType=local`, `localConfig.transportType` MUST be `"streamable-http"` (validated in [`types/mcp-catalog.ts:267-289`][types-cat]).

### 6.6 Invariants

1. `(parentCatalogItemId, name)` is unique.
2. A child row has both `parentCatalogItemId` and `presetEntryId` non-null. A parent has both null.
3. `multitenant` cannot be changed after create.
4. A `userConfig` field cannot have both `promptOnInstallation=true` and `promptOnPreset=true`.
5. A header-mapped (`headerName` set) `userConfig` field with `sensitive=true` cannot carry a `default` value when the field is prompted (`promptOnInstallation` or `promptOnPreset` true); the value MUST come from a secret bag (preset-scoped) or the per-install secret bag. A separate validation rejects static (non-prompted) sensitive header-mapped fields entirely. Both enforced in [`backend/src/types/mcp-catalog.ts`][types-cat] (`superRefine` block on `userConfig`).
6. Built-in catalog items (except Playwright) cannot be modified or deleted.

## 7. Preset

The word "preset" has two distinct meanings in this codebase. Disambiguating them is the most error-prone part of working with the catalog.

### 7.1 Preset entry (`mcp_preset_entry`)

An *org-wide named bucket* — e.g., "Production", "Staging", "Development". Defined in [`backend/src/database/schemas/mcp-preset-entry.ts`][schema-preset-entry]:

| Column | Type | Role |
|---|---|---|
| `id` | uuid pk | |
| `organizationId` | text fk (cascade delete) | Owning org. |
| `name` | text | Bucket name. **Immutable.** Unique within org. Max 50 chars. |
| `sortOrder` | int default 0 | Display order. |
| `validationRegex` | text nullable | JS regex source (no delimiters/flags), max 1000 chars. When set, every preset-scoped value (i.e., `userConfig` field with `promptOnPreset=true`, prompted env vars on install) is validated against it. |

Preset entries are administered via `/api/organization/mcp-preset-entries` (admin-only). Deleting an entry cascades to every child catalog row that references it.

### 7.2 Child catalog item (a "preset" in casual usage)

A row in `internal_mcp_catalog` with `parentCatalogItemId` and `presetEntryId` non-null. Conceptually: *"this is the GitHub server as installed in our Production environment."*

- Composed name: `${parent.name}-${dns1123(entry.name)}`.
- Inherits all *syncable template fields* from its parent (see §10.5).
- Carries its own `presetFieldValues` overlay for fields marked `promptOnPreset` in the parent's `userConfig`.
- Sensitive preset values are stored in `presetSecretId`'s secret bag; non-sensitive ones in `presetFieldValues` JSON.

### 7.3 Default preset

When a user installs a parent catalog item directly (no child specified), the parent's own `presetFieldValues` serve as the overlay. The parent therefore plays a dual role: template AND default preset. Editing the default preset uses the same parent-`PUT` endpoint and triggers cascade.

### 7.4 Preset value flow

```
  CreateChildCatalogSchema {
    presetEntryId,          ─► picks the bucket
    presetFieldValues       ─► values for parent.userConfig fields marked promptOnPreset
  }
                │
                │  POST /api/internal_mcp_catalog/:catalogId/children
                │
                ▼
  partition by parent.userConfig.<key>.sensitive:
    ├─ sensitive=true    → presetSecretId (secret bag)
    └─ sensitive=false   → presetFieldValues (jsonb column)
                │
                │  on install:
                │
                ▼
  install-time form merges:
    parent.userConfig defaults
    ⊕ chosen child's presetFieldValues (or parent's own if installing parent)
    ⊕ per-install user inputs (for fields marked promptOnInstallation)
    = mcp_server.environmentValues + mcp_server.secretId bag
```

### 7.5 Validation regex

Introduced by [PR #4830][pr-4830]. Two enforcement sites:

- **Preset edit time** — `POST /api/internal_mcp_catalog/:catalogId/children` and `PATCH /api/internal_mcp_catalog/:catalogId/children/:childId` validate every value in the request's `presetFieldValues` against the child's preset-entry regex.
- **Install time** — `POST /api/mcp_server` validates the request's `userConfigValues` AND `environmentValues` against the *applicable* regex: the catalog's preset-entry regex if the catalog is a child preset, otherwise the org-wide `presetEntityDefaultValidationRegex` fallback (column on the `organization` table).

The regex is the same pattern for both sites; the install-time check covers the case where the user supplies a value at install that the preset author never set.

## 8. MCP server — `mcp_server`

### 8.1 Purpose

A `mcp_server` is one installation of a catalog item — the concrete instance the gateway routes to.

### 8.2 Schema overview

Defined in [`backend/src/database/schemas/mcp-server.ts`][schema-server]:

| Column | Type | Role |
|---|---|---|
| `id` | uuid pk | |
| `name` | text | Display name. For local servers it disambiguates the K8s deployment by appending the owner id (`${baseName}-${ownerId}`) for `scope=personal` or the team id (`${baseName}-${teamId}`) for `scope=team`; for `scope=org` (and for remote servers) it equals the catalog `baseName` verbatim. See [`backend/src/models/mcp-server.ts::constructServerName`][model-server]. |
| `catalogId` | uuid → `internal_mcp_catalog`, set null on delete | The catalog row this instance installs. |
| `serverType` | enum local/remote/builtin | Copied from catalog at install; immutable. |
| `secretId` | uuid → `secret`, set null on delete | Per-install secret bag — prompted secret env vars, OAuth access/refresh tokens, BYOS vault refs. |
| `environmentValues` | jsonb default `{}` | Per-install **plain** env vars: `{ KEY: "value" }`. |
| `ownerId` / `teamId` | text fks | Required by scope: `personal`⇒`ownerId`, `team`⇒`teamId`, `org`⇒neither. |
| `scope` | enum personal/team/org, default personal | Install-time only; immutable. |
| `reinstallRequired` | bool default false | Signals "pod is on stale config; restart needed before serving requests" — see §10. |
| `localInstallationStatus` | enum `idle`/`pending`/`discovering-tools`/`success`/`error` | Pod-deploy lifecycle state. |
| `localInstallationError` | text nullable | Error message when status is `error`. |
| `oauthRefreshError` / `oauthRefreshFailedAt` | enum + timestamp | Track failed OAuth token refresh state. |
| `createdAt` / `updatedAt` | timestamp | |

**Side table:** `mcp_server_user` joins servers to users for personal-auth tracking.

Drizzle-zod schemas: [`backend/src/types/mcp-server.ts`][types-server] (`SelectMcpServerSchema`, `InsertMcpServerSchema`, `UpdateMcpServerSchema`).

### 8.3 Installation status state machine

```
       install               discover tools         success
     ─────────►  pending  ────────────────►  discovering-tools  ────► success
                   │                             │
                   │                             │
                   ▼                             ▼
                 error                        error
                                                
       reinstall  ────────► resets status to "pending" (or "idle" for remote)
       reauthenticate  ──► no status change, pod restarted to pick up creds
       uninstall  ───────► deletes the row (and K8s deployment if last reference)
```

`reinstallRequired` is orthogonal to `localInstallationStatus`: a successfully-installed pod can be flagged `reinstallRequired=true` after a parent catalog edit, meaning "you're running fine but on stale config; user must re-confirm before next restart."

### 8.4 Lifecycle

**Install** (`POST /api/mcp_server`): resolves the catalog item, validates input, creates the per-install secret bag, creates the row, and (for `serverType=local`) triggers the K8s runtime manager's `startServer()`. Tool discovery runs once the pod reports ready.

**Operate**: agents invoke tools via the MCP Gateway (`/v1/mcp/:profileId`) or the MCP Proxy (`/api/mcp/:agentId`). See §12.4.

**Reauthenticate** (`PATCH /api/mcp_server/:id/reauthenticate`): swaps the secret bag (OAuth refresh, BYOS key rotation). For local servers, restarts the pod. Does NOT reset `localInstallationStatus`.

**Reinstall** (`POST /api/mcp_server/:id/reinstall`): preserves the row id, agent assignments, tool invocation policies, and trusted data policies. Updates the secret bag with re-supplied inputs and restarts the pod. Used both by the user (when `reinstallRequired=true`) and by the cascade's auto path.

**Uninstall** (`DELETE /api/mcp_server/:id`): deletes the row, the tools, the policies, and (for local single-tenant) the K8s Deployment and secrets. For multi-tenant, deletes only the row — the deployment is reference-counted (§6.5).

### 8.5 Tool discovery and the catalog ↔ tools join

Tools live in their own `tools` table keyed by `catalogId`. After install / reinstall, the runtime calls `tools/list` against the new server, inserts/updates `tools` rows, and broadcasts an installation-status event via WebSocket. Agent ↔ tool assignments survive reinstall (they reference `tools.id`).

## 9. Configuration shapes

All four configuration shapes are stored as JSONB on the catalog row and validated via Zod schemas in `shared/mcp-server-config.ts` and `backend/src/types/mcp-catalog.ts`.

### 9.1 `localConfig`

Defined in [`shared/mcp-server-config.ts`][shared-config]:

```ts
LocalConfig = {
  command?: string;                       // pod entrypoint
  arguments?: string[];                   // pod args
  environment?: EnvironmentVariable[];    // see §9.5
  envFrom?: { type: "secret"|"configMap", name, prefix? }[];
  dockerImage?: string;                   // overrides ARCHESTRA_ORCHESTRATOR_MCP_SERVER_BASE_IMAGE
  transportType?: "stdio" | "streamable-http"; // default "stdio"
  httpPort?: number;                      // default 8080; streamable-http only
  httpPath?: string;                      // default "/mcp"; streamable-http only
  nodePort?: number;                      // fixed K8s NodePort; local-dev only
  serviceAccount?: string;
  imagePullSecrets?: (
      | { source: "existing", name }
      | { source: "credentials", server, username, password?, email? }
    )[];
}
```

Constraint: at least one of `command` or `dockerImage` must be present.

### 9.2 `userConfig`

A map `fieldKey → UserConfigField`:

```ts
UserConfigField = {
  type: "string" | "number" | "boolean" | "directory" | "file";
  title: string;
  description: string;
  required?: boolean;
  default?: string | number | boolean | string[];
  multiple?: boolean;
  min?: number;
  max?: number;

  // Prompting (mutually exclusive)
  promptOnInstallation?: boolean;   // ask the installer
  promptOnPreset?: boolean;         // ask when creating a child preset

  // Storage / routing
  sensitive?: boolean;              // stored in secret bag, never plaintext
  headerName?: string;              // maps to an HTTP header for remote/HTTP servers
  valuePrefix?: string;             // e.g., "Bearer "
}
```

The frontend uses an `additionalHeaders` projection of `userConfig` for the subset where `headerName` is set; the two are interconverted by `transformCatalogItemToFormValues` and `transformFormToApiData`.

### 9.3 `oauthConfig` (remote servers)

Selected fields from [`OAuthConfigSchema`][shared-config]:

```ts
OAuthConfig = {
  name: string;
  server_url: string;
  grant_type?: "authorization_code" | "client_credentials";  // default authorization_code
  auth_server_url?: string;
  authorization_endpoint?: string;
  token_endpoint?: string;
  well_known_url?: string;
  resource_metadata_url?: string;
  client_id: string;
  client_secret?: string;     // stored in catalog.clientSecretId bag
  audience?: string;
  redirect_uris: string[];
  scopes: string[];
  default_scopes: string[];
  supports_resource_metadata: boolean;
  // ...flow modifiers (browser_auth, streamable_http_url, generic_oauth, ...)
}
```

Cross-field rules (from `superRefine`):

- For `authorization_code` flow: `authorization_endpoint` and `token_endpoint` must be set together (both or neither).
- For `client_credentials` flow: `token_endpoint` is required if `authorization_endpoint` is set.

### 9.4 `enterpriseManagedConfig`

Drives enterprise credential resolution (e.g., per-user token exchange through an external IdP). See [`backend/src/types/enterprise-managed-credentials.ts`][types-emc] and the user-facing [`platform-enterprise-managed-auth.md`][user-emc] doc for the full surface. Key fields:

```ts
EnterpriseManagedCredentialConfig = {
  identityProviderId?: string;
  assertionMode?: "exchange" | "passthrough";
  resourceType?: "mcp" | "oauth_protected_resource" | "secret" | "service_account" | "custom_http";
  requestedCredentialType?: "id_jag" | "bearer_token" | "secret" | "service_account" | "opaque_json";
  tokenInjectionMode?: "authorization_bearer" | "raw_authorization" | "header" | "env" | "body_field";
  headerName?: string;
  envVarName?: string;
  fallbackMode?: "fail_closed" | "fallback_to_dynamic" | "fallback_to_static";
  cacheTtlSeconds?: number;
  // ...resource identifiers, scopes, audience, fallback config
}
```

When set on a `serverType=local` catalog, the catalog's `localConfig.transportType` MUST be `"streamable-http"` (enforced by Zod refinement at [`types/mcp-catalog.ts:267-289`][types-cat]). The reason: enterprise-managed token injection happens at HTTP request time, which only works for HTTP transport.

### 9.5 Environment variables (prompted vs static)

Each entry in `localConfig.environment` is one of:

- **Static** (`promptOnInstallation=false`): value comes from `value` (plain) or the catalog-level `localConfigSecretId` secret bag (for `type=secret`).
- **Prompted on installation** (`promptOnInstallation=true`): the installer is asked at install time. Stored in the per-`mcp_server` secret bag (if `type=secret`) or `mcp_server.environmentValues` (otherwise).
- **Prompted on preset** (`promptOnPreset=true`): the preset-author is asked when creating a child preset. Stored as a key in `presetFieldValues` (or `presetSecretId` if `sensitive`).

`promptOnInstallation` and `promptOnPreset` are mutually exclusive.

### 9.6 Secret storage — the three catalog secret bags + the per-install bag

| Bag | FK column | Owner | Contents |
|---|---|---|---|
| OAuth client secret | `internal_mcp_catalog.clientSecretId` | catalog | `{ client_secret: "..." }` |
| Static env-var secrets | `internal_mcp_catalog.localConfigSecretId` | catalog | `{ DB_PASSWORD: "...", API_KEY: "..." }` from `localConfig.environment` entries where `type=secret` AND `promptOnInstallation=false` |
| Sensitive preset values | `internal_mcp_catalog.presetSecretId` | catalog (child or parent default) | `{ field_key: "..." }` for `userConfig` fields where `sensitive=true` |
| Per-install secrets | `mcp_server.secretId` | server | Prompted secret env vars, OAuth `access_token` / `refresh_token`, BYOS vault references |

The secret table (`backend/src/database/schemas/secret.ts`) supports three storage modes — DB-stored JSON, Archestra-managed Vault, BYOS Vault — distinguished by `isVault` / `isByosVault` flags. From the catalog/server perspective they are interchangeable; details are out of scope here.

## 10. Cascade reinstall

When a catalog item is edited, every installed `mcp_server` pointing at it must be reconciled with the new definition. The cascade is the algorithm that decides per-server whether to skip, mark-for-manual-reinstall, or auto-restart. It runs server-side as the source of truth, and is predicted client-side by the catalog edit form's confirm bar so the user can see the consequence of their save before they make it.

### 10.1 Decision tree

```
                              ┌──────────────────────────────┐
                              │  cascadeReinstallForCatalog  │
                              │  (server-side)               │
                              │  computeCascadeOutcome       │
                              │  (frontend mirror)           │
                              └──────────────┬───────────────┘
                                             │
                                  affectedServerCount == 0
                                             │
                                ┌────yes────► skip
                                │
                                no
                                │
                  requiresNewUserInputForReinstall?
                                │
                                ┌────yes────► manual  (reinstallRequired=true)
                                │
                                no
                                │
                  onlyForwardCompatibleEnvDiff?
                                │
                                ┌────yes────► skip
                                │
                                no
                                │
                                ▼
                              auto  (setImmediate background restart)
```

The three gates are:

1. **`requiresNewUserInputForReinstall`** — does the edit change a field that the user must re-supply on the affected installations?
2. **`onlyForwardCompatibleEnvDiff`** — are the remaining diffs entirely forward-compatible (additions of optional fields, demotion of required→optional, metadata-only)?
3. *(implicit)* Anything that survives both gates is a breaking change that needs a pod restart but no new user input → auto path.

### 10.2 `requiresNewUserInputForReinstall`

Defined in [`backend/src/services/mcp-reinstall.ts`][svc-reinstall]; mirrored in [`frontend/.../cascade-decision.ts`][fe-cascade].

**Local servers — re-prompt if:**

- `name` changed (affects secret paths and K8s resource names);
- `localExecutionConfigChanged` — any of `command` / `arguments` / `dockerImage` / `transportType` / `httpPort` / `httpPath` / `serviceAccount` changed (exactly these seven fields, per `getLocalExecutionConfig` in [`mcp-reinstall.ts`][svc-reinstall]). Note that `nodePort` and `imagePullSecrets` are deliberately *not* in this set: changes to them are caught by the auto path's catch-all whole-row comparison in `onlyForwardCompatibleEnvDiff`, so the cascade still fires — just as `auto` rather than `manual`.
- `promptedEnvVarsChanged` (see §10.4);
- `requiredUserConfigChanged` — a required `userConfig` field was added, removed, or had its `type` changed.

**Remote servers — re-prompt if:**

- The OAuth config was added or removed (`Boolean(oauthConfig)` flipped);
- `requiredUserConfigChanged`.

### 10.3 `onlyForwardCompatibleEnvDiff`

Returns true (⇒ skip) iff *all* of the following hold:

1. `promptedEnvVarsChanged` returns false (env-var schema evolution is forward-compatible);
2. `promptedEnvVarsRuntimeChanged` returns false — currently the `mounted` flag on an existing prompted env var. Flipping `mounted` swaps the pod spec between an env-var injection and a mounted secret file at `/secrets/<key>`; the user supplied the same value at install, so it's not a re-prompt — but the layout is a runtime concern that needs a restart. This check is what routes such flips to `auto` instead of letting them slip into `skip`.
3. Non-prompted env-var entries are byte-identical (their values are part of the catalog template; any change must propagate to pods);
4. `userConfigChangedBreakingly` returns false;
5. No other non-metadata field differs.

The fifth check is the historically fragile one. The backend implementation compares JSON-stringified strip-results of two `InternalMcpCatalog` rows; this is safe because both sides are model-shaped. The frontend implementation, however, compares `initialValues` (raw API response shape) against `transformFormToApiData(values)` (route-input shape) — these have different field sets, so the frontend uses an **explicit projection** of cascade-relevant fields rather than strip-then-stringify. See §10.6.

### 10.4 Schema evolution rules

**Prompted env vars** (`promptedEnvVarsChanged`):

| Change | Verdict |
|---|---|
| Added required var | breaking (re-prompt) |
| Added optional var | compatible (skip) |
| Removed var (any kind) | breaking (re-prompt) |
| `type` changed (plain ↔ secret ↔ number) | breaking (re-prompt) |
| `required: false → true` | breaking (re-prompt) |
| `required: true → false` | compatible (skip) |

**`userConfig` fields** (`userConfigChangedBreakingly`):

| Change | Verdict |
|---|---|
| Removed any field | breaking |
| Added required field | breaking |
| Demoted `required: true → false` | compatible |
| `type` changed | breaking |
| `headerName` changed | breaking (routing change) |
| `sensitive` flipped | breaking (storage bucket moves) |
| `default` changed on a **prompted** header-mapped field (`promptOnInstallation` or `promptOnPreset` is true) | compatible (it's just the placeholder shown at install/preset time; the actual runtime value comes from user input) |
| `default` changed on a **static** header-mapped field (both prompt flags false) | breaking → auto path (the form writes the admin's runtime header value into `userConfig[field].default` when `promptOnInstallation` is false; the next pod restart picks up the new value, no re-prompt) |
| `title`, `description`, `valuePrefix` change | compatible (cosmetic) |

### 10.5 Parent edit vs preset edit

**Parent edit** (`PUT /api/internal_mcp_catalog/:id`):

- The cascade visits every installed server whose `catalogId == parent.id`.
- Then, for each child preset under the parent, the cascade syncs *template* fields from parent to child (`version`, `description`, `instructions`, `repository`, `serverType`, `multitenant`, `serverUrl`, `docsUrl`, `localConfig`, `deploymentSpecYaml`, `userConfig`, `oauthConfig`, `enterpriseManagedConfig`, `clientSecretId`, `localConfigSecretId`) — preset values are preserved.
- If the parent's `userConfig` changed the `sensitive` flag of any field, child `presetFieldValues` are **re-partitioned** between the plaintext column and `presetSecretId` bag.
- The cascade then recurses for each child's installed servers.

**Child preset edit** (`PATCH /api/internal_mcp_catalog/:catalogId/children/:childId`):

- Only `presetFieldValues` is editable on a child row; the cascade visits every installed server whose `catalogId == child.id`.
- Preset value changes never trip `requiresNewUserInputForReinstall` (the preset author is not the installer), so the outcome is always either `skip` or `auto`.

### 10.6 Frontend mirror and the shape-mismatch gotcha

The catalog edit form's confirm-bar calls a pure `computeCascadeOutcome(prev, next, opts)` ([`cascade-decision.ts`][fe-cascade]) which mirrors the backend's gate. The contract is the shared scenario matrix in `cascade-scenarios.ts` — both sides assert against it.

There is one **non-obvious correctness condition** the design enforces. In real form usage, `prev` is the raw API response — with extra fields the backend exposes that are not part of the route's input shape (`id`, `organizationId`, `repository`, `instructions`, `updatedAt`, `presetSecretId`, `presetEntryId`, `parentCatalogItemId`, `version`, `requiresAuth`, `toolCount`, `teams`, `scope`, …). `next` is the output of `transformFormToApiData(values)` — only the route's input field set. The two have different key sets.

The frontend's `anyNonForwardCompatChange` must therefore use an **explicit projection** of cascade-relevant fields (`serverType`, `serverUrl`, `authMethod`, `authHeaderName`, `includeBearerPrefix`, `oauthConfig`, `enterpriseManagedConfig`, `multitenant`, `localConfig.envFrom`, `localConfig.imagePullSecrets`), *not* a `JSON.stringify(strip(prev)) !== JSON.stringify(strip(next))` whole-object compare. A whole-object compare over-fires on every edit because the prev-side has extra keys the next-side never produces.

The matrix sweep does not catch this on its own because the fixture-based scenarios use identical shapes for `prev` and `next` (`prev = CATALOG_SHAPES[id]`, `next = scenario.edit(CATALOG_SHAPES[id])`). The `cascade-decision.test.ts` file includes a dedicated *shape-mismatch regression* group that simulates the real form pipeline (API-shape `prev` + transform-shape `next`); every cascade-relevant change to the projection should add a regression case to that group.

### 10.7 The `reinstallRequired` flag

Set to `true` by:

- The manual path of the cascade (`requiresNewUserInputForReinstall` returned true);
- A failed auto-reinstall (the `setImmediate` restart raised);
- Other operations that leave the pod on stale config.

Cleared by:

- A successful `POST /api/mcp_server/:id/reinstall`;
- A successful auto-reinstall.

The frontend uses this flag to render the *"Reinstall required"* badge on the server card and to surface the reinstall confirmation flow.

## 11. Runtime and deployment

Scope: `serverType=local`. Remote and built-in servers have no runtime side.

### 11.1 K8s runtime manager

[`backend/src/k8s/mcp-server-runtime/manager.ts`][runtime-manager] owns the in-memory `mcpServerIdToDeploymentMap` and orchestrates pod lifecycle. On platform startup, `start()` queries all local `mcp_server` rows and calls `startServer()` for each in parallel.

`startServer(mcpServerId)`:

1. Fetch the catalog item and server row.
2. Resolve environment values by merging:
   - prompted secrets from the Archestra secret manager,
   - non-prompted catalog secrets from `localConfigSecretId`,
   - preset env values from `presetFieldValues`,
   - per-install plain env vars from `environmentValues`.
3. Construct a `K8sDeployment` object and register it in the map.
4. Create the K8s `Secret` (if any secret data) and any `docker-registry` secrets for `imagePullSecrets`.
5. Call `k8sDeployment.startOrCreateDeployment()` ([`k8s-deployment.ts:1488-1654`][k8s-deployment]).
6. For `transportType=streamable-http`, ensure the K8s `Service` is configured.

`stopServer`, `restartServer`, `removeMcpServer` are documented in §6.5 and §8.4.

### 11.2 Transports — `stdio` vs `streamable-http`

| | `stdio` (default) | `streamable-http` |
|---|---|---|
| **Pod ↔ platform channel** | `kubectl attach` to pod's stdin/stdout | Native HTTP/SSE to the pod's HTTP port |
| **K8s Service** | none | always created (`<deployment>-service`) |
| **Concurrency** | requests serialized (one at a time) | requests concurrent |
| **Routed via** | `/mcp_proxy/:id` (Archestra-side) | direct HTTP to the service URL |
| **Local-dev URL** | n/a (attach) | `http://localhost:<nodePort>` |
| **In-cluster URL** | n/a (attach) | `http://<service>.<ns>.svc.<clusterDomain>:<port><httpPath>` |
| **Use case** | simple / synchronous workloads, no service-DNS dependency | high concurrency, sessioned tools, enterprise-managed auth |

`streamable-http` is required when `enterpriseManagedConfig` is set, because per-request token injection needs an HTTP request to inject into.

### 11.3 K8s naming

Names are produced by `ensureStringIsRfc1123Compliant(...)` (RFC 1123 DNS-label sanitiser) and truncated to 253 chars. The naming inputs differ between modes:

| Resource | Single-tenant | Multi-tenant |
|---|---|---|
| Deployment | `mcp-<rfc1123(mcp_server.name)>` | `mcp-mt-<catalogId[:8]>-<rfc1123(catalog.name)>` |
| Secret (env vars) | `mcp-server-<serverId>-secrets` | `mcp-server-mt-<catalogId[:8]>-secrets` |
| Service (HTTP only) | `<deployment>-service` | `<deployment>-service` |

See [`constructDeploymentName` in `k8s-deployment.ts`][k8s-deployment]. The platform pod's `nodeSelector` and `tolerations` are inherited onto each MCP server pod so they schedule on the same node pool.

### 11.4 Deployment YAML override

`internal_mcp_catalog.deploymentSpecYaml` (text, nullable). When non-null, this raw K8s manifest is used instead of the generated spec. The route `POST /api/internal_mcp_catalog/validate-deployment-yaml` checks structure; `GET /api/internal_mcp_catalog/:id/deployment-yaml-preview` returns either the stored manifest or a freshly-generated template with `{server_id}` placeholder; `POST /api/internal_mcp_catalog/:id/reset-deployment-yaml` clears the override.

## 12. Gateways, authentication, authorization

### 12.1 MCP Gateway — `/v1/mcp/:profileId`

Token-authenticated, stateless JSON-RPC. The "profile" parameter identifies an Archestra agent profile; the token (`Bearer <platform-token>`) authenticates the caller and carries an agent identity. `GET` returns server discovery metadata (capabilities, agent id, token claims); `POST` accepts JSON-RPC requests (`initialize`, `tools/list`, `tools/call`, …) and dispatches them. Tool invocation policies and trusted data policies are enforced at this layer.

A fresh `Server` + `Transport` pair is created per request; sessions are not maintained. `WWW-Authenticate` is set per RFC 9728 on 401 responses for OAuth-aware MCP clients.

### 12.2 MCP Proxy — `/api/mcp/:agentId`

Session-authenticated alternative used by the in-browser AppRenderer. Verifies agent access via the regular session, then proxies JSON-RPC to the MCP server. Visibility filtering is enforced on `tools/call`: tools tagged `_meta.ui.visibility: ["model"]` (model-only) cannot be invoked through the proxy. Cached `Server` instances are reused across sequential requests (TTL defined in [`mcp-proxy.ts`][routes-mcp-proxy]).

### 12.3 RBAC

Resources and permissions are declared in [`shared/access-control.ts`][acl] (specifically `requiredEndpointPermissionsMap`) and enforced by the Fastify auth plugin on a per-route basis.

| Resource | Used by | Typical actions |
|---|---|---|
| `mcpRegistry` | catalog CRUD, preset CRUD, deployment-YAML endpoints, label endpoints, list preset entries | `read`, `create`, `update`, `delete` |
| `mcpServerInstallation` | server install/uninstall, reinstall, inspect, reauth, preset-entry admin (create/update/delete) | `read`, `create`, `update`, `delete`, `admin` |

**Gateway and proxy auth is not RBAC.** The three routes `McpGatewayGet`, `McpGatewayPost`, and `McpProxyPost` register with `{}` (no required permissions) in `requiredEndpointPermissionsMap`. Authentication is handled inside the route handlers themselves: the gateway routes validate a `Bearer <platform-token>` and resolve agent identity from it; the proxy uses the standard session cookie and then checks per-agent access.

Org-scope creation, preset-entry admin, and built-in catalog modification all require `admin`. Non-admins can create personal-scope catalog items and install servers (personal/team), but only with team-admin permission for team scope.

### 12.4 Two MCP entry points compared

| Aspect | MCP Gateway (`/v1/mcp/:profileId`) | MCP Proxy (`/api/mcp/:agentId`) |
|---|---|---|
| Auth | Bearer token | Session cookie |
| Audience | external LLM clients | in-browser AppRenderer |
| Session | stateless (fresh per request) | server instance cached ~30s |
| Policies enforced | tool invocation + trusted data | tool invocation + trusted data + tool visibility |
| Identifier | profile id (UUID or slug) | agent id (UUID) |

## 13. API reference

Routes are organized by resource. Auth column lists the `{resource: [action]}` entry from `requiredEndpointPermissionsMap`.

### 13.1 Catalog item

| Method | Path | Auth | Purpose |
|---|---|---|---|
| `GET` | `/api/internal_mcp_catalog` | `mcpRegistry:read` | List catalog items; `?includeChildren=true` for preset expansion |
| `POST` | `/api/internal_mcp_catalog` | `mcpRegistry:create` | Create catalog item |
| `GET` | `/api/internal_mcp_catalog/:id` | `mcpRegistry:read` | Get catalog item (with expanded secrets) |
| `PUT` | `/api/internal_mcp_catalog/:id` | `mcpRegistry:update` | Update catalog item; triggers cascade |
| `DELETE` | `/api/internal_mcp_catalog/:id` | `mcpRegistry:delete` | Delete catalog item (cascades to children) |
| `DELETE` | `/api/internal_mcp_catalog/by-name/:name` | `mcpRegistry:delete` | Delete by name |
| `GET` | `/api/internal_mcp_catalog/:id/tools` | `mcpRegistry:read` | List tools |
| `GET` | `/api/internal_mcp_catalog/:id/deployment-yaml-preview` | `mcpRegistry:read` | Render generated/stored K8s manifest |
| `POST` | `/api/internal_mcp_catalog/validate-deployment-yaml` | `mcpRegistry:read` | Validate a YAML body |
| `POST` | `/api/internal_mcp_catalog/:id/reset-deployment-yaml` | `mcpRegistry:update` | Clear stored manifest override |
| `GET` | `/api/internal_mcp_catalog/labels/keys` | `mcpRegistry:read` | List label keys |
| `GET` | `/api/internal_mcp_catalog/labels/values` | `mcpRegistry:read` | List label values, optionally filtered by key |
| `GET` | `/api/k8s/image-pull-secrets` | `mcpRegistry:read` | List available K8s docker-registry secrets |

### 13.2 Catalog children (presets)

| Method | Path | Auth | Purpose |
|---|---|---|---|
| `GET` | `/api/internal_mcp_catalog/:catalogId/children` | `mcpRegistry:read` | List child presets of a parent |
| `POST` | `/api/internal_mcp_catalog/:catalogId/children` | `mcpRegistry:create` | Create child preset (requires `presetEntryId`) |
| `PATCH` | `/api/internal_mcp_catalog/:catalogId/children/:childId` | `mcpRegistry:update` | Update child preset values; triggers cascade |

### 13.3 MCP server

| Method | Path | Auth | Purpose |
|---|---|---|---|
| `GET` | `/api/mcp_server` | `mcpServerInstallation:read` | List installed servers |
| `POST` | `/api/mcp_server` | `mcpServerInstallation:create` | Install a server |
| `GET` | `/api/mcp_server/:id` | `mcpServerInstallation:read` | Get a server |
| `GET` | `/api/mcp_server/:id/tools` | `mcpServerInstallation:read` | List server's tools |
| `GET` | `/api/mcp_server/:id/installation-status` | `mcpServerInstallation:read` | Poll local install progress |
| `POST` | `/api/mcp_server/:id/inspect` | `mcpServerInstallation:read` | Run `tools/list` or `tools/call` against the server |
| `PATCH` | `/api/mcp_server/:id/reauthenticate` | `mcpServerInstallation:update` | Swap credentials |
| `POST` | `/api/mcp_server/:id/reinstall` | `mcpServerInstallation:update` | Reinstall preserving id + policies |
| `DELETE` | `/api/mcp_server/:id` | `mcpServerInstallation:delete` | Uninstall |

### 13.4 Preset entries (org-level)

| Method | Path | Auth | Purpose |
|---|---|---|---|
| `GET` | `/api/organization/mcp-preset-entries` | `mcpRegistry:read` | List entries with `assignedCatalogCount` |
| `POST` | `/api/organization/mcp-preset-entries` | `mcpServerInstallation:admin` | Create entry (name immutable) |
| `PATCH` | `/api/organization/mcp-preset-entries/:id` | `mcpServerInstallation:admin` | Update entry's `validationRegex` |
| `DELETE` | `/api/organization/mcp-preset-entries/:id` | `mcpServerInstallation:admin` | Delete entry (cascades to child catalogs) |

### 13.5 Gateways

These routes register with no `requiredEndpointPermissionsMap` entry (auth is handled in-handler, not by the RBAC middleware — see §12.3).

| Method | Path | Auth | Purpose |
|---|---|---|---|
| `GET` | `/v1/mcp/:profileId` | Bearer token (in-handler) | Server discovery |
| `POST` | `/v1/mcp/:profileId` | Bearer token (in-handler) | Stateless JSON-RPC |
| `POST` | `/api/mcp/:agentId` | Session cookie + per-agent access check (in-handler) | AppRenderer-side JSON-RPC proxy |

## 14. Invariants and cross-cutting constraints

1. A catalog item is either a parent (`parentCatalogItemId IS NULL` AND `presetEntryId IS NULL`) or a child preset (both non-null). No other shape is valid.
2. `(parentCatalogItemId, name)` is unique. Child preset names are deterministic: `${parent.name}-${dns1123(entry.name)}`.
3. A `userConfig` field cannot have both `promptOnInstallation` and `promptOnPreset` set. The same rule applies to `localConfig.environment[*]`.
4. `multitenant` is immutable after create and only allowed for `serverType=local`.
5. `enterpriseManagedConfig` on `serverType=local` requires `localConfig.transportType="streamable-http"`.
6. The cascade decision tree applied by the backend gate (`cascadeReinstallForCatalog`) and predicted by the frontend confirm bar (`computeCascadeOutcome`) MUST agree for every scenario in `shared/cascade-scenarios.ts`. Divergence is treated as a bug.
7. `mcp_server.serverType` and `mcp_server.scope` are immutable after create.
8. The four secret bags (`clientSecretId`, `localConfigSecretId`, `presetSecretId`, `mcp_server.secretId`) are independent — each is created or replaced atomically with the entity that owns it.
9. Tool ↔ agent assignments and tool/data policies must survive `POST /api/mcp_server/:id/reinstall`. Only `DELETE /api/mcp_server/:id` removes them.

## 15. Known limitations and open questions

- The frontend's `anyNonForwardCompatChange` projection is an *enumerated allow-list* — every new cascade-relevant catalog field must be added to it explicitly. Forgetting to do so silently reverts the cascade to "skip" for that field. A future improvement would be a single source of truth for "cascade-relevant fields" reused by both gates.
- The K8s deployment-YAML override mechanism allows arbitrary manifests; only structural validation runs at save time. Operational drift between the override and the rest of the catalog template is not detected.
- The reference-counted multi-tenant teardown assumes the platform process owns the lifecycle; if a pod is deleted out-of-band, the in-memory map can drift from the cluster state until the next `start()`.
- The cascade auto path uses `setImmediate` and broadcasts via WebSocket; long restart sequences are not exposed as a tracked job, so a refresh during a cascade does not show progress beyond the per-server `localInstallationStatus`.
- Built-in catalog items (except Playwright) are deliberately immutable; this prevents customization but also makes them hard to disable in environments that don't want them.

## 16. References

### Code

| Concern | File |
|---|---|
| Catalog table | [`backend/src/database/schemas/internal-mcp-catalog.ts`][schema-cat] |
| Server table | [`backend/src/database/schemas/mcp-server.ts`][schema-server] |
| Preset entry table | [`backend/src/database/schemas/mcp-preset-entry.ts`][schema-preset-entry] |
| Catalog types | [`backend/src/types/mcp-catalog.ts`][types-cat] |
| Server types | [`backend/src/types/mcp-server.ts`][types-server] |
| Enterprise-managed credentials | [`backend/src/types/enterprise-managed-credentials.ts`][types-emc] |
| Shared config schemas (OAuth, LocalConfig, env vars) | [`shared/mcp-server-config.ts`][shared-config] |
| Metadata-only fields | `shared/catalog-runtime-fields.ts` |
| Cascade scenarios (contract) | [`shared/cascade-scenarios.ts`][shared-scenarios] |
| Backend cascade gate | [`backend/src/services/mcp-reinstall.ts`][svc-reinstall] |
| Backend cascade route hook | `backend/src/routes/internal-mcp-catalog.ts` |
| Frontend cascade mirror | [`frontend/src/app/mcp/registry/_parts/cascade-decision.ts`][fe-cascade] |
| K8s runtime manager | [`backend/src/k8s/mcp-server-runtime/manager.ts`][runtime-manager] |
| K8s deployment helper | [`backend/src/k8s/mcp-server-runtime/k8s-deployment.ts`][k8s-deployment] |
| Access control map | [`shared/access-control.ts`][acl] |
| Server name construction | [`backend/src/models/mcp-server.ts`][model-server] |

### Related user-facing docs

- [`docs/pages/platform-mcp.md`][user-platform-mcp]
- [`docs/pages/platform-mcp-gateway.md`][user-mcp-gateway]
- [`docs/pages/mcp-authentication.md`][user-mcp-auth]
- [`docs/pages/platform-archestra-mcp-server.md`][user-archestra-mcp]
- [`docs/pages/platform-enterprise-managed-auth.md`][user-emc]
- [`docs/pages/platform-access-control.md`][user-rbac]

[schema-cat]: ../../platform/backend/src/database/schemas/internal-mcp-catalog.ts
[schema-server]: ../../platform/backend/src/database/schemas/mcp-server.ts
[schema-preset-entry]: ../../platform/backend/src/database/schemas/mcp-preset-entry.ts
[types-cat]: ../../platform/backend/src/types/mcp-catalog.ts
[types-server]: ../../platform/backend/src/types/mcp-server.ts
[types-emc]: ../../platform/backend/src/types/enterprise-managed-credentials.ts
[shared-config]: ../../platform/shared/mcp-server-config.ts
[shared-scenarios]: ../../platform/shared/cascade-scenarios.ts
[svc-reinstall]: ../../platform/backend/src/services/mcp-reinstall.ts
[fe-cascade]: ../../platform/frontend/src/app/mcp/registry/_parts/cascade-decision.ts
[runtime-manager]: ../../platform/backend/src/k8s/mcp-server-runtime/manager.ts
[k8s-deployment]: ../../platform/backend/src/k8s/mcp-server-runtime/k8s-deployment.ts
[acl]: ../../platform/shared/access-control.ts
[model-server]: ../../platform/backend/src/models/mcp-server.ts
[routes-mcp-proxy]: ../../platform/backend/src/routes/mcp-proxy.ts
[pr-4830]: https://github.com/archestra-ai/archestra/pull/4830
[user-platform-mcp]: ../pages/platform-mcp.md
[user-mcp-gateway]: ../pages/platform-mcp-gateway.md
[user-mcp-auth]: ../pages/mcp-authentication.md
[user-archestra-mcp]: ../pages/platform-archestra-mcp-server.md
[user-emc]: ../pages/platform-enterprise-managed-auth.md
[user-rbac]: ../pages/platform-access-control.md
