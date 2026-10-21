# Kinotic OS MCP tool contracts

The kinotic-os server is a stateless streamable-HTTP MCP endpoint at `POST <server url>`
secured by OAuth 2.1. Behavior common to every tool:

- **Tool names are opaque hashes — never hardcode or invent one.** A tool's `name` is
  the base-36 XXH3-128 of `<zone>~<package>.<ServiceName>/<functionName>` (≤ 25 chars of
  `[0-9a-z]`), e.g.
  `management-api~org.kinotic.management.api.services.ProjectService/findById` hashes
  to something like `5nwldv2gqljeygvmd1ywjl2z7`. **Pick a tool by its description**:
  every description begins with the tool's title and a colon
  (`Application Service Count: Returns the number of entities available.`), and the
  titles used below are those prefixes. Match the whole prefix, service half included —
  a function inherited from the CRUD base (`Count`, `Find All`, `Save`, …) has the same
  text after the colon on every service, so only the prefix says whose applications,
  projects or tenants it reads. If the tools are deferred, search for the title's words
  (`Application Service Count`). A stale or invented name fails as a JSON-RPC protocol
  error (`-32602`, `Unknown tool: <name>`) — not as a tool result.
- **Results are raw JSON in a single text content block.** The tool result contains one
  `text` item whose text is the JSON serialization of the service's return value — parse
  it. An empty list result is the literal text `[]`, and a `null` result is the literal
  text `null`.
- **Unknown argument keys fail the call** with `isError: true`
  (`Received argument '<key>' which matches no parameter of <function>`). Send exactly
  the argument names documented below.
- **Service errors arrive as `isError: true`** with the exception message as text
  content. Match on the message substrings documented below.
- **Every tool carries an `annotations` block** with all four hints written explicitly
  (`readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint`). Treat
  `destructiveHint: true` (`… Save`, `… Delete By Id`, and their `Sync` twins) as
  never-call-speculatively. `openWorldHint` is `false` on everything except
  `Project Service Retry Repo Initialization`, which reaches GitHub.
- Authenticated users act as their Kinotic **organization**, so the tools below operate
  on that organization's applications and projects.

## The full tool surface

Four services carry a type-level `@McpTool`, so **every** function on them is a tool:

| Title prefix | Service |
|---|---|
| `Application Service` | The organization's applications |
| `Project Service` | Projects, their GitHub repos and deployments |
| `Tenant Service` | The tenants of the organization's applications |
| `Owned Oidc Configuration Service` | The organization's own sign-in providers |

The first three also expose the functions they inherit from the CRUD hierarchy:

| Inherited from | Tools |
|---|---|
| `CrudService` | `… Save`, `… Save Sync`, `… Find By Id`, `… Count`, `… Delete By Id`, `… Delete By Id Sync`, `… Find All`, `… Search`, `… Sync Index` |
| `IdentifiableCrudService` | `… Create`, `… Create Sync` |
| `ApplicationScopedCrudService` (Project and Tenant services) | `… Count For Application`, `… Find All For Application` |

`Application Service Count` answers how many applications the organization has, and
`Application Service Find All` lists them a page at a time (the `pageable` argument's
shape is in the tool's input schema). `Project Service Find All For Application` with
`{"applicationId": ..., "pageable": ...}` lists an application's projects. Stick to the
documented tools for the onboarding workflow, and never call a destructive tool unless
the user explicitly asks for that exact operation:

- `Application Service Delete By Id` is guarded server-side
  (`Cannot delete an application with projects in it.`).
- `Project Service Delete By Id` has no such guard, and deleting a project **cascades**:
  its deployment record and both of its machine identities go with it, cutting off any
  workload still running. The checkout and runtime VM on the node are not reachable from
  the management plane; deleting the runtime machine is what strands them.

## ApplicationService

Qualified name: `management-api~org.kinotic.management.api.services.ApplicationService`

### `Application Service Create Application If Not Exist`

Creates an application, or returns the existing one whose id matches the slugified name.
Idempotent.

Arguments:

```json
{ "name": "Inventory App", "description": "Tracks warehouse inventory", "tenantManagement": "PLATFORM" }
```

Any human-readable `name` works — the server slugifies it into the id (lowercase
letters, digits, interior dashes). It fails only when the name is blank, slugifies to
nothing (all punctuation), or slugifies to the platform-reserved id `system-api`.

`tenantManagement` decides who creates the application's tenants:

| Value | Meaning |
|---|---|
| `PLATFORM` | Every user gets a tenant of their own when the user is created, and the rows the user writes to a `SHARED` entity are isolated in it |
| `TENANT_ADMINISTRATORS` (also what `null` means) | A customer signing up, or the organization in the portal, creates a tenant, and its administrator decides who joins. Users created into no tenant have none and share the application's data |

See the create-app workflow for choosing it, and the entities-and-persistence skill for
the entity side that must agree.

It applies **only when the application is created**. An idempotent call that returns an
existing application returns it with the `tenantManagement` it already had; the argument
is ignored. Changing it afterwards is a portal action (Application → Settings →
**Tenants**), and `PLATFORM` only assigns tenants to users created from then on, so it is
settled before the application has users. `Application Service Save` can also write it,
but it replaces the whole document and is `destructiveHint: true`; prefer the portal.

Result (Application):

```json
{
  "id": "inventory-app",
  "organizationId": "acme",
  "name": "Inventory App",
  "description": "Tracks warehouse inventory",
  "tenantManagement": "PLATFORM",
  "primaryUiId": null,
  "primaryUiUrl": null,
  "socialProviders": [],
  "updated": "2026-07-29T00:00:00.000Z"
}
```

The create-app workflow always passes `tenantManagement: "PLATFORM"`. A result with
`TENANT_ADMINISTRATORS` means the application already existed with administrator-managed
tenants; tell the user before continuing.

`id` is the server-minted slug of `name`. `organizationId` is derived from the
authenticated user — pass both into project creation.

### `Application Service Get Login Providers`

Read-only. Returns the sign-in providers the application's login offers besides email and
password, in the order to present them (used by the frontend skill when wiring login).
Returns `[]` when the application's login offers password only. Fails when the id does
not name an application in the caller's organization.

Arguments:

```json
{ "applicationId": "inventory-app" }
```

Result (a list of LoginProviderSummary):

```json
[
  { "configurationId": "…", "name": "Google", "provider": "GOOGLE", "type": "APP_SOCIAL" }
]
```

## ProjectService

Qualified name: `management-api~org.kinotic.management.api.services.ProjectService`

### `Project Service Create Project If Not Exist`

Creates a project and provisions its GitHub repository from the Kinotic template through
the organization's GitHub App installation. Returns the existing project unchanged if
one with the same id exists. Idempotent.

The single argument key is exactly `project`:

```json
{
  "project": {
    "id": "inventory-app-main",
    "applicationId": "inventory-app",
    "organizationId": "acme",
    "name": "Inventory App",
    "description": "Tracks warehouse inventory",
    "repoPrivate": true,
    "sourceOfTruth": "TYPESCRIPT"
  }
}
```

- `applicationId` and `name` are required. A supplied `id` is used as given; left unset
  the server derives `<applicationId>-<slugified name>` (`inventory-app-inventory-app`).
  Pass `<applicationId>-main` so an application's default project is its `main` project.
- `name` must stay the application's name, not `main`: the GitHub repo is named from the
  slugified **name**, so naming the project `main` creates a repo called `main` that the
  next application in the same account collides with.
- `organizationId` is **required**, not just constrained: omitting it fails with
  `Organization id must be set on Project`, and a mismatch fails with
  `Cannot save Project with organizationId '<x>' while authenticated as organization '<y>'`.
- The GitHub repository is named after the slugified project **name** (truncated to
  100 chars), created under the account holding the GitHub App installation — record
  `repoFullName` from the result rather than deriving it.
- `sourceOfTruth` accepts only `"TYPESCRIPT"` today (it is not validated server-side).
- `repoPrivate` controls the GitHub repository visibility at creation.

Result (Project) — fields beyond the input:

```json
{
  "id": "inventory-app-main",
  "repoFullName": "acme-gh-org/inventory-app",
  "repoId": 123456789,
  "repoDefaultBranch": "main",
  "repoConnectionStatus": "CONNECTED",
  "updated": "2026-07-29T00:00:00.000Z"
}
```

`repoConnectionStatus` values:

| Status | Meaning | Action |
|---|---|---|
| `CONNECTED` | Repo provisioned and baseline committed | Proceed |
| `INITIALIZATION_FAILED` | Repo exists but the baseline commit failed | Call `Project Service Retry Repo Initialization` |
| `DISCONNECTED` | GitHub revoked the platform's access to the repo | User must re-link in the portal |

Errors:

- `"GitHub is not linked for this organization. Link GitHub before creating a project."`
  — the organization has no GitHub App installation. The user links GitHub in the
  Kinotic OS portal (Organization settings → Integrations → GitHub → Link GitHub), then
  the same call is re-run.

### `Project Service Retry Repo Initialization`

Re-runs repository initialization for a project left in `INITIALIZATION_FAILED`.
Succeeds with the project marked `CONNECTED` once the baseline is committed; the write
is read-your-write consistent, so the returned project reflects the outcome.

Arguments:

```json
{ "projectId": "inventory-app-main" }
```

Errors:

- `"projectId must not be blank"`
- `"Project for id <projectId> does not exist"`
- `"Project <projectId> is not awaiting initialization retry (status <status>)"` — the
  project is not in `INITIALIZATION_FAILED`; never call this tool speculatively.
- `"Project repoId must be set to reinitialize"` (likewise `repoFullName` /
  `repoDefaultBranch`) — provisioning never got far enough for a retry; re-run project
  creation instead.
- `"GitHub is not linked for this organization. …"` — the installation disappeared
  between create and retry; back to GitHub linking.

### `Project Service Find Deployment`

Read-only. Returns the project's deployment record — where its code is running and which
commit is live — or `null` when the project has never been deployed. This is how a push
is verified; see the `deploying` skill for the whole workflow.

Arguments:

```json
{ "projectId": "inventory-app-main" }
```

Result (ProjectDeployment), `id` always equal to the project id:

```json
{
  "id": "inventory-app-main",
  "organizationId": "acme",
  "applicationId": "inventory-app",
  "nodeId": "node-1",
  "hostDir": "<node workload data dir>/projects/inventory-app-main",
  "runtimeWorkloadId": "wl-…",
  "syncMachineIdentityId": "…",
  "runtimeMachineIdentityId": "…",
  "commitSha": "9f2c1ab…",
  "lastJobRunId": "…",
  "status": { "type": "RUNNING", "message": null },
  "created": "2026-07-29T00:00:00.000Z",
  "updated": "2026-07-29T00:00:00.000Z"
}
```

`status.type` is `DEPLOYING`, `RUNNING`, or `FAILED`; `status.message` carries the last
job's failure reason when there is one. Errors with `"projectId must not be blank"`.

### `Project Service Find by GitHub Repo`

Read-only. Looks up projects in the caller's organization whose backing GitHub repo has
the given `owner/repo` full name (function `findByRepoFullName`). Returns `[]` when none
match.

Arguments:

```json
{ "repoFullName": "acme-gh-org/inventory-app" }
```
