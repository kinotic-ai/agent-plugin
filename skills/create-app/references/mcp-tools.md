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
  `Project Service Retry Repo Initialization`, which reaches the repository host.
- Authenticated users act as their Kinotic **organization**, so the tools below operate
  on that organization's applications and projects.

## The full tool surface

These services carry a type-level `@McpTool`, so **every** function on them is a tool:

| Title prefix | Service |
|---|---|
| `Application Service` | The organization's applications |
| `Project Service` | Projects, their Git repositories, deployments, deployment history and SBOM (`… Find Deployment`, `… Find Deployment History`, `… Find All Deployment History`, `… Find Dependencies`) |
| `Project Repo Credential Service` | Tokens for writing a project's repository |
| `Tenant Service` | The tenants of the organization's applications |
| `Owned Oidc Configuration Service` | The organization's own OIDC sign-in providers (`… Find All`, `… Find Managed Providers`, `… Find By Id`, `… Find Callback Url`, `… Save`, `… Delete Client Secret`, `… Copy`, `… Delete By Id`) |
| `Owned Password Configuration Service` | The organization's email-and-password sign-in configurations (`… Find All`, `… Save`, `… Delete By Id`) |

Two more services expose a subset of their functions:

| Title | Purpose |
|---|---|
| `Microservice Deployment Service Find All For Project` / `… Find History` / `… Restart` | A project's microservice deployments, their history, and a restart in a fresh VM |
| `Ui Deployment Service Find All For Project` / `… Find All For Application` / `… Find History` | A project's or an application's UI deployments and their history |

Removing a microservice or UI deployment is not a tool: it stays with the console. The
`deploying` skill documents what these return.

`Application Service` and `Project Service` also expose the functions they inherit from the CRUD hierarchy:

| Inherited from | Tools |
|---|---|
| `CrudService` | `… Save`, `… Save Sync`, `… Find By Id`, `… Count`, `… Delete By Id`, `… Delete By Id Sync`, `… Find All`, `… Search`, `… Sync Index` |
| `IdentifiableCrudService` | `… Create`, `… Create Sync` |
| `ApplicationScopedCrudService` (Project Service) | `… Count For Application`, `… Find All For Application` |

`Tenant Service` inherits nothing: a tenant id is unique within its application, not the
organization, so every one of its tools names the application — `Create Tenant If Not Exist`
(`{"applicationId", "name", "description"}`), `Find By Id` and `Delete By Id`
(`{"applicationId", "tenantId"}`), `Count For Application`, `Find All For Application` and
`Find For Application By Prefix`.

`Application Service Count` answers how many applications the organization has, and
`Application Service Find All` lists them a page at a time (the `pageable` argument's
shape is in the tool's input schema). `Project Service Find All For Application` with
`{"applicationId": ..., "pageable": ...}` lists an application's projects. Stick to the
documented tools for the onboarding workflow, and never call a destructive tool unless
the user explicitly asks for that exact operation:

- `Application Service Delete By Id` is guarded server-side
  (`Cannot delete an application with projects in it.`).
- `Project Service Delete By Id` has no such guard, and deleting a project **cascades**:
  its deployment's worker removes every one of its microservice deployments (stopping each
  VM and deleting its machine identity) and UI deployments (taking each site down and
  deleting its files), then stops the deployment's own workloads, removes the sync machine
  identity and deletes the deployment and SBOM records.

## ApplicationService

Qualified name: `management-api~org.kinotic.management.api.services.ApplicationService`

### `Application Service Create Application If Not Exist`

Creates an application, or returns the existing one whose id matches the slugified name.
Idempotent.

Arguments:

```json
{ "name": "Inventory App", "description": "Tracks warehouse inventory", "tenantManagement": "SYSTEM" }
```

Any human-readable `name` works — the server slugifies it into the id (lowercase
letters, digits, interior dashes). It fails when the name is blank, slugifies to
nothing (all punctuation), slugifies to the platform-reserved id `system-api`, or is too
long: the id must leave room for a UI name in the sites' hostname labels, so an id over 34
characters is refused with a message ending `shorten the application name`.

`tenantManagement` decides who creates the application's tenants:

| Value | Meaning |
|---|---|
| `SYSTEM` | Every user gets a tenant of their own when the user is created, and the rows the user writes to a `SHARED` entity are isolated in it. The application's sign-up, when turned on, admits new people at its own login |
| `TENANT_ADMINISTRATORS` (also what `null` means) | The organization creates each tenant, and every user of the application belongs to the tenant it was added to or signed up into through the tenant's own sign-in provider |

See the create-app workflow for choosing it, and the entities-and-persistence skill for
the entity side that must agree.

It applies **only when the application is created**. An idempotent call that returns an
existing application returns it with the `tenantManagement` it already had; the argument
is ignored. Changing it afterwards is a portal action (Application → Settings →
**Tenants**), and `SYSTEM` only assigns tenants to users created from then on, so it is
settled before the application has users. `Application Service Save` can also write it,
but it replaces the whole document and is `destructiveHint: true`; prefer the portal.

Result (Application):

```json
{
  "id": "inventory-app",
  "organizationId": "acme",
  "name": "Inventory App",
  "description": "Tracks warehouse inventory",
  "tenantManagement": "SYSTEM",
  "primaryUiId": null,
  "primaryUiUrl": null,
  "updated": "2026-07-29T00:00:00.000Z"
}
```

The create-app workflow passes the value its Step 1 picked from the app's tenant model. A
result with the other value means the application already existed with it; tell the user
before continuing.

`id` is the server-minted slug of `name`. `organizationId` is derived from the
authenticated user — pass both into project creation. `primaryUiId` and `primaryUiUrl` name
the first UI the application published, once it has one.

### `Application Service Get Login Methods`

Read-only. Returns the sign-in methods the application's login offers, in the order to
present them (used by the frontend skill when wiring the login and sign-up pages): a
`PASSWORD` entry for the email and password form while the application has the **Email and
password** provider, and an `OIDC` entry per provider button. `signUp` is true for a method
new people may sign up through. Fails when the id does not name an application in the
caller's organization.

Arguments:

```json
{ "applicationId": "inventory-app" }
```

Result (a list of ApplicationLoginProvider):

```json
[
  { "configurationId": "my-org:inventory-app", "type": "PASSWORD", "name": "Email and password", "provider": null, "signUp": true },
  { "configurationId": "…", "type": "OIDC", "name": "Google", "provider": "google", "signUp": false }
]
```

## ProjectService

Qualified name: `management-api~org.kinotic.management.api.services.ProjectService`

### `Project Service Get Repository Host`

Read-only, no arguments. Where the server keeps every project's repository:

```json
{ "type": "GITEA", "webBaseUrl": "http://localhost:3001" }
```

`type` is `GITHUB` (Kinotic Cloud: an organization links GitHub before it creates a
project) or `GITEA` (a local Kinotic's own Git server: no linking). A repository's web page
is `webBaseUrl` + `/` + its `repoFullName`.

### `Project Service Create Project If Not Exist`

Creates a project and provisions its repository from the Kinotic template on the repository
host — on GitHub through the organization's GitHub App installation. Nothing is deployed
until the first push. Returns the existing project unchanged if one with the same id
exists. Idempotent.

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
- `name` must stay the application's name, not `main`: the repo is named from the
  slugified **name**, so naming the project `main` creates a repo called `main` that the
  next application in the same account collides with.
- `organizationId` is **required**, not just constrained: omitting it fails with
  `Organization id must be set on Project`, and a mismatch fails with
  `Cannot save Project with organizationId '<x>' while authenticated as organization '<y>'`.
  That check runs on the save, after the repository has already been created on the host,
  so a re-run with the id corrected finds no project and tries to create the repository
  again, failing on its taken name (below). Get `organizationId` right the first time.
- The repository is named after the slugified project **name** (truncated to
  100 chars), created under the account holding the GitHub App installation, or on a
  local Kinotic under the organization's id — record `repoFullName` from the result rather
  than deriving it.
- `sourceOfTruth` accepts only `"TYPESCRIPT"` today (it is not validated server-side).
- `repoPrivate` controls the repository's visibility at creation.

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
| `DISCONNECTED` | GitHub revoked the system's access to the repo | User must re-link in the portal |

Errors:

- `"GitHub is not linked for this organization. Link GitHub before creating a project."`
  (`GITHUB` only) — the organization has no GitHub App installation. The user links GitHub in the
  Kinotic OS portal (Organization settings → Integrations → GitHub → Link GitHub), then
  the same call is re-run.
- `"A repository named '<name>' already exists in the GitHub account '<owner>' …"` — the
  slugified project name is taken on the host by a repository that is not this project's
  (another application's project of the same name, or one left by an earlier failed
  creation). Choose another project name, or have the user rename or delete that
  repository; never delete it yourself.
- `"Project for id <id> already exists"` — the id is taken by a project that did not match
  the lookup; report it to the user.

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
  `repoDefaultBranch`; a `GITEA` host checks only the last two) — provisioning never got
  far enough for a retry; re-run project creation instead.
- `"GitHub is not linked for this organization. …"` — the installation disappeared
  between create and retry; back to GitHub linking.

### `Project Service Find Deployment`

Read-only. Returns the project's deployment record — where its code is running, the
artifacts of the synced commit, and what the deployment should be beside what it is — or
`null` when the project has never been deployed. This is how a push is verified; see the
`deploying` skill for the whole workflow.

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
  "syncWorkloadId": "wl-…",
  "uiPublishWorkloadId": "wl-…",
  "sbomWorkloadId": null,
  "syncMachineIdentityId": "…",
  "artifacts": {
    "commitSha": "9f2c1ab…",
    "microservices": [{ "name": "main", "dir": "packages/microservices/main", "entry": "src/main.ts" }],
    "uis": [{ "name": "admin", "dir": "packages/ui/admin" }],
    "dependencyHash": "…"
  },
  "sbomGenerated": true,
  "lastJobRunId": "…",
  "failureMessage": null,
  "state": {
    "desired": { "phase": "RUNNING", "commitSha": "9f2c1ab…" },
    "observed": { "phase": "RUNNING", "commitSha": "9f2c1ab…" },
    "generation": 3,
    "desiredAt": 1785000000000,
    "observedGeneration": 3,
    "reconciled": true,
    "conditions": [],
    "parent": null,
    "dirty": false,
    "dirtyAt": 0,
    "unrecorded": {},
    "deletionRequested": null
  },
  "created": "2026-07-29T00:00:00.000Z",
  "updated": "2026-07-29T00:00:00.000Z"
}
```

`state.desired` is the commit the last default-branch push asked for, always with the phase
`RUNNING`. `state.observed.phase` is `DEPLOYING` (a job runs; `observed.commitSha` is the
commit still live), `RUNNING` (`observed.commitSha` is live) or `FAILED` (`failureMessage`
carries the reason; `observed.commitSha` is still the previous commit). The pushed commit is
live when `observed` is `RUNNING` at that commit. Errors with `"projectId must not be blank"`.

### `Project Service Find Deployment History`

Read-only. `{"projectId": …, "pageable": …}` returns a page of ledger entries (`WatchEvent`)
of what happened to the project's deployment and to the deployment jobs, build VMs,
microservice deployments and UI deployments it made, newest first, with what caused each;
empty when the project has never been deployed.

### `Project Service Find Dependencies`

Read-only. `{"projectId": …}` returns the project's SBOM — every package version its
`bun.lock` installs and which depends on which — or `null` while the project has no SBOM of
the dependencies its last sync reported (`sbomGenerated` on the deployment record says
whether it does).

### `Project Service Find by GitHub Repo`

Read-only. Looks up projects in the caller's organization whose backing repository, on
either host, has the given `owner/repo` full name (function `findByRepoFullName`). Returns `[]` when none
match.

Arguments:

```json
{ "repoFullName": "acme-gh-org/inventory-app" }
```

## ProjectRepoCredentialService

Qualified name: `management-api~org.kinotic.management.api.services.github.ProjectRepoCredentialService`

### `Project Repo Credential Service Issue Write Token`

Issues a token that may write the project's repository — commits, branches and tags — on
either repository host. A push to the default branch deploys the project.

Arguments:

```json
{ "projectId": "inventory-app-main" }
```

Result (ProjectRepoToken):

```json
{
  "token": "…",
  "expiresAt": "2026-07-29T01:00:00.000Z",
  "cloneUrl": "https://github.com/acme-gh-org/inventory-app.git",
  "defaultBranch": "main"
}
```

The token is good for an hour. Git presents it over HTTP as the password of the user
`x-access-token`; the `create-app` skill's Step 3 shows the clone and the push. Request a
fresh token for each push. Errors with `"projectId is required"`, with a message starting
`Project <projectId> of organization <organizationId> has no` when the project is not one
of the caller's organization's or has no repository, and on GitHub with
`"GitHub install for organization <organizationId> no longer exists"` when the installation
was removed — back to GitHub linking.
