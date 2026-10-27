---
name: create-app
description: >
  Set up a new Kinotic application end to end: authenticate with Kinotic OS through the
  kinotic-os MCP server (signing up if needed), create the Application and Project with
  the Kinotic MCP tools, wait for the project's Git repository to be provisioned from the
  template, clone it with a token Kinotic issues, and verify the scaffold. Use when
  the user wants to create, start, bootstrap, or set up a new Kinotic app or project,
  sign up for Kinotic OS, or connect the coding agent to Kinotic OS.
---

# Creating a Kinotic Application

Follow these steps in order. Exact MCP tool contracts (argument and result JSON shapes,
error strings) are in `references/mcp-tools.md`. The expected repository layout is in
`references/project-scaffold.md`.

An application holds one or more projects; the workflow below creates its default `main`
project, scaffolded as a Bun-workspace mono repo. The project's Git repository is created by
Kinotic OS — never create the repository yourself. Kinotic Cloud hosts it on GitHub; a local
Kinotic (`kinotic local up`) hosts it on its own Git server, so never assume GitHub — Step 2
asks the server where its repositories live.

**MCP tool names are opaque hashes.** Every Kinotic OS tool's description begins with
its title and a colon (e.g. `Application Service Create Application If Not Exist: Creates
a new application …`) — pick the tool whose description starts with the title you need,
matching the service half too, since inherited functions like `Count` and `Find All`
carry the same text after the colon on every service. Never call a tool by a guessed or
remembered name, and never parse the hash. `references/mcp-tools.md` documents each tool
by title.

**More than one Kinotic OS server can be connected**, such as the plugin's `kinotic-os` and a
local Kinotic's `kinotic-local`. They offer the same tool titles but are different servers
holding different organizations, applications and projects, so every tool call of a workflow
goes to the one server the work belongs to — never create an application on one and its
project, token or deployment check on another. Choose the server per task:
- **A new application** — ask the user which Kinotic to create it on, naming the connected
  servers, unless they already said.
- **An existing project** — use the server that holds it: the one whose
  `Project Service Get Repository Host` `webBaseUrl` is the host of the clone's
  `git remote get-url origin`, and whose `Project Service Find By Id` returns the project
  named by `projectId` in `.config/kinotic.config.ts`. Ask the user when that still leaves
  more than one.

## Step 0 — Sign up or sign in to Kinotic OS

Check whether the kinotic-os MCP tools are available — look for tools whose
descriptions start with `Application Service` / `Project Service` (e.g.
`Application Service Create Application If Not Exist`). If they are present, the user
is already connected — skip to Step 1.

If the tools are missing, the user has not authenticated the Kinotic OS MCP server
yet. The plugin's own server is named `kinotic-os`; a manually added server carries the
name it was given (`kinotic-os-test` in the README's example) — the steps below say
`kinotic-os`; substitute that name. Ask whether they already have a Kinotic OS account, then walk them through the
matching path — the whole flow happens in their browser, so narrate what they will see:

1. Tell the user to start the server's sign-in from their coding agent:
   - **Claude Code** — run `/mcp`, select `kinotic-os`, and authenticate.
   - **Codex** — installing the plugin prompts for sign-in; to sign in again, run
     `codex mcp login kinotic-os` in a terminal.
   - **Grok Build** — run `/mcps`, select `kinotic-os`, and press `i` to authenticate.

   A browser opens on the Kinotic OS **portal** login page — the one with a "New to Kinotic?
   **Create an organization**" link. The plugin targets the Kinotic OS dev server
   (`https://dev-api.kinotic.ai/mcp`, portal `https://dev-portal.kinotic.ai`); for a
   different Kinotic OS see "Other servers" below.
2. **Existing account** — sign in, then on the consent screen ("Authorize …", with a
   "Verified as <host>" line) click **Approve**. **Deny** is a real decision that
   returns a denial to the coding agent — the tools will not appear. If the page shows
   "Authorization request unavailable", the link is stale: start the sign-in again to
   mint a fresh one.
3. **No account yet** — click **Create an organization** on the login page:
   - On Kinotic Cloud, sign up with GitHub — that is the only sign-up path (Kinotic projects are backed
     by GitHub repositories, so the organization starts from the user's GitHub
     identity). Users invited to an existing organization accept the emailed
     invitation link instead of creating a new organization.
   - Pick an organization name (and optional description) on the registration page and
     click **Create organization**. This creates the Kinotic organization that owns
     every application and project.
   - The page then shows **"Your organization is ready"** with a **Continue to
     GitHub** button — this installs the Kinotic GitHub App, which authorizes
     repository access. The user chooses the GitHub account or org to install into;
     that is where project repositories will be created. GitHub may interrupt with a
     sudo email code or a 2FA prompt — normal, not a Kinotic error.
   - Signup ends on the Applications page. It does **not** return to the OAuth consent
     screen — the authorization request is not resumed. Tell the user to go back to
     their coding agent and start the sign-in again; now signed in, they land directly
     on the consent screen — click **Approve**.
4. Back in the coding agent, confirm the kinotic-os tools are now available before moving on.

If tools still do not appear right after a server restart, wait and retry: the service
directory publishes tools shortly after startup, not instantly.

The user can review or revoke this connection later under **Account → Connected apps**
in the portal.

**Other servers.** A staging, self-hosted, or local Kinotic OS is added as its own MCP
server — the tools work the same from either:
- **Claude Code** — `claude mcp add -s user --transport http kinotic-os-test <url>`
  (`-s user` so it follows the user into every project directory).
- **Codex** — `codex mcp add kinotic-os-test --url <url>`, then
  `codex mcp login kinotic-os-test`.
- **Grok Build** — add the server to `~/.grok/config.toml` with Kinotic's client ID for
  Grok Build, then authenticate it from `/mcps`:

  ```toml
  [mcp_servers.kinotic-os-test]
  url = "<url>"
  oauth_client_id = "https://kinotic.ai/oauth/grok-build-client.json"
  ```

**A local Kinotic** (`kinotic local up`) is added this way as `kinotic-local`, at
`http://localhost:58503/mcp`; the portal's Quickstart shows the command. Its portal at
`http://localhost:9090` signs up with an email and a password, with no GitHub. The
plugin's own `kinotic-os` server stays beside it, authenticated or not as the user also uses
Kinotic Cloud; with both connected, choose the server as described above.

Two settings on that server gate this, set by whoever runs it (Kinotic Cloud has them
in place): the server must list the coding agent's client-metadata URL in
`kinotic.domain.oauth.allowedClientIds` (there is no dynamic client registration) —
`https://claude.ai/oauth/claude-code-client-metadata` for Claude Code,
`https://chatgpt.com/oauth/codex/client.json` for Codex,
`https://kinotic.ai/oauth/grok-build-client.json` for Grok Build — and a server whose OAuth
surface is reached at a different host than the browser uses (e.g. localhost behind a
tunnel) must set `kinotic.managementServer.issuerBaseUrl`. Missing either looks like
"the server never appeared" after OAuth.

## Step 1 — Create the Application

1. Ask the user for an application name and one-line description if not already known.
   Any human-readable name works — the server slugifies it into the application id. It
   is rejected only when blank, all punctuation, or slugifying to the platform-reserved
   id `system-api`.
2. **Ask whether people sign in.** Ask in plain English, and never show the user the
   `tenantManagement` setting, JSON, or code:

   > Should people have to sign in to use your app?

   If they say **no**, tell them that apps anyone can use without signing in are not
   supported yet, and that it's coming soon 🙂. Ask whether they want to continue with an
   app people sign in to. Do not create the application unless they do.

   If they say **yes**, create the application with `tenantManagement: "SYSTEM"`, which
   gives every user a tenant of their own. Each entity then decides who sees its rows:
   `@Entity(MultiTenancyType.SHARED)` keeps each user's rows private to that user, and
   `@Entity(MultiTenancyType.NONE)` is data every user of the app sees. A `@TenantId` field is
   optional, added only when the app's logic needs it. The `entities-and-persistence` skill
   has the full table.

3. Call the tool titled `Application Service Create Application If Not Exist` with
   `{"name": ..., "description": ..., "tenantManagement": "SYSTEM"}`. The call is
   idempotent — if the application already exists it is returned unchanged, **including
   its existing `tenantManagement`**: the argument only applies when the application is
   created, because changing it later would split the users into tenanted and untenanted
   halves. An application that already exists with `TENANT_ADMINISTRATORS` is changed in
   the portal (**Application → Settings → Tenants**), and only while it still has no users.
4. From the result, record `id` (the server-minted slug of the name, e.g.
   `Inventory App` → `inventory-app`) and `organizationId`. Both are needed in Step 2.
   The result's `tenantManagement` confirms what the application was created with.

## Step 2 — Create the Project (provisions the Git repository)

First call the tool titled `Project Service Get Repository Host` (no arguments). Its
`type` says where the server keeps projects' repositories: `GITHUB` (Kinotic Cloud), where
an organization links GitHub before it creates a project, or `GITEA` (a local Kinotic),
which needs no linking. Its `webBaseUrl` followed by `/<repoFullName>` is a repository's web
page, the link to give the user.

Every application gets a **`main` project** by default — do not ask the user to name it.
Call the tool titled `Project Service Create Project If Not Exist`. The single argument
key is exactly `project`:

```json
{
  "project": {
    "id": "<application id from Step 1>-main",
    "applicationId": "<application id from Step 1>",
    "organizationId": "<organizationId from Step 1>",
    "name": "<the application name>",
    "description": "<project description>",
    "repoPrivate": true,
    "sourceOfTruth": "TYPESCRIPT"
  }
}
```

Setting `id` explicitly is what makes it the `main` project: left unset the server derives
`<applicationId>-<slugified name>`, giving the doubled `inventory-app-inventory-app`.
**Do not put `main` in `name`** — the repository is named after the slugified
project name, so a project named `main` produces a repo called `main`, and the second
application in the same account then collides with it. The name carries the app's
identity for the repo; the id carries the `main` convention.

`organizationId` is required — the server rejects a project without it. Creation
provisions the repository from the Kinotic template — on GitHub through the organization's
GitHub App installation — commits a rendered baseline, and deploys that baseline. The call is
idempotent: re-running it returns the existing project unchanged.

Handle the outcomes:

- **Error containing "GitHub is not linked for this organization"** (`GITHUB` only) — the
  organization has no GitHub App installation. Fresh signups normally completed this during Step 0
  ("Continue to GitHub"), so this error usually means that install was skipped or
  abandoned. Direct the user to the Kinotic OS portal → sidebar **Organization →
  Organization settings** → **Integrations** → the **GitHub** card → **Link GitHub**.
  They choose the GitHub account or org to install the Kinotic GitHub App into — that
  is where project repositories are created; on success the card reads "Linked as
  <account>". An organization holds one installation at a time (moving to a different
  account requires **Unlink** first). Then re-run the exact same tool call.
- **GitHub App installs can fail visibly** (`GITHUB` only). The install returns to a callback page; on
  failure it shows "Couldn't finish linking GitHub" plus a reason. Common ones:
  *state is missing, expired, or already used* (the install state is single-use and
  expires in 10 minutes — start the link again from Organization settings; never
  refresh or re-open the callback URL); *the GitHub account that authorized this
  install does not have access to the requested installation* (they authorized GitHub
  as a different user than the owner/admin of the target account — retry signed in as
  the right GitHub user); *already linked to installation N* (Unlink first).
- **Result with `repoConnectionStatus: "CONNECTED"`** — proceed to Step 3.
- **Result with `repoConnectionStatus: "INITIALIZATION_FAILED"`** — the repository was
  created but the baseline commit failed. Call the tool titled
  `Project Service Retry Repo Initialization` with `{"projectId": "<project id>"}`.
  Never call it in any other status — it fails unless the project is awaiting a retry.
  If the retry also fails, report the error to the user and stop; do not delete or
  recreate anything. Deleting a project cascades to its deployment and machine
  identities, so it is never a recovery step.

Record `repoFullName` (`owner/repo`) from the result — the repo is named after the
slugified project name, not the project id, so never derive it. If you are ever unsure
whether a project already exists for a repository, call the tool titled
`Project Service Find by GitHub Repo` with `{"repoFullName": "owner/repo"}`, which finds
a repository on either host despite its title.

## Step 3 — Clone with a Kinotic token

Clone and push with a token Kinotic issues, on GitHub and the local Git server alike — never
with `gh` or the user's own GitHub credentials, which cannot reach a local Kinotic's
repositories.

1. Call the tool titled `Project Repo Credential Service Issue Write Token` with
   `{"projectId": "<project id>"}`. The result carries `token`, `expiresAt` (an hour
   out), `cloneUrl` and `defaultBranch`. The token may write this one repository: commits,
   branches and tags.
2. Clone with the token as the password of the user `x-access-token`, through a credential
   helper scoped to the one command:

   ```bash
   KINOTIC_REPO_TOKEN='<token>' git -c credential.helper= \
     -c 'credential.helper=!f() { echo username=x-access-token; echo "password=$KINOTIC_REPO_TOKEN"; }; f' \
     clone <cloneUrl>
   ```

   The empty `credential.helper=` comes first so no stored helper (the macOS keychain, a
   credential manager) saves the token. Never put the token in the remote URL, a file or
   git config, and never echo it back to the user.
3. Push as the `deploying` skill's **Pushing** section shows, with a fresh token for every
   push.

## Step 4 — Verify the scaffold

Compare the cloned repository against `references/project-scaffold.md`. In short: a Bun
workspace with `packages/domain`, `packages/microservices/main`, `packages/ui`, and a
`.config/kinotic.config.ts` whose `organizationId` and `applicationId` match Step 1 and whose `projectId` matches Step 2.

Run `bun install`, then `bun run type-check`. Check which `@kinotic-ai/*` versions
actually resolved (`bun pm ls | grep @kinotic-ai`) — the current SDK line is `5.x`
(`@kinotic-ai/core` 5.0.0-beta and up, `@kinotic-ai/management-api` alongside it). A
resolution on the old `4.x` line, or any `@kinotic-ai/os-api` (renamed to
`@kinotic-ai/management-api`), means the template's catalog pins are stale — report that
to the user rather than working around it. Report any mismatch between the clone and the
expected shape to the user instead of silently changing files — the repository contents
are the source of truth.

## Step 5 — First entity, first push

1. Define a first entity under the path listed in `.config/kinotic.config.ts`
   `entitiesPaths` (see the entities-and-persistence skill), choosing its multi-tenancy by
   who should see its rows: `SHARED` for data private to each user, `NONE` for data every
   user sees.
2. Run `bun run generate` from the project root to generate the typed repository
   classes. The script wraps the Kinotic CLI vendored as a project dependency and runs
   locally — no server connection or login. If the script is missing from
   `package.json`, tell the user instead of improvising.
3. Export the entity and its repository from `packages/domain/index.ts` — nothing else
   in the workspace can import them until you do — then run `bun run type-check`.
4. Commit and push (Step 3's push) everything generate produced: entity sources, the generated
   repository classes, and `.config/c3` (a committed generation artifact — keep it in
   git so schema changes show up in diffs; never gitignore it). Never run
   `kinotic login` or `kinotic sync` yourself.
5. **The push is the deployment.** A push to the default branch builds the project,
   synchronizes and publishes the entity definitions, and starts (or reloads) the runtime
   that runs `packages/microservices/main/src/main.ts`. Verify it landed with the tool titled
   `Project Service Find Deployment` (`{"projectId": "<project id>"}`) — poll until
   `status.type` is `RUNNING` with the pushed `commitSha`, and report a `FAILED`
   status's `message` to the user. Full workflow and failure handling: the `deploying`
   skill. Publication is what creates the entity's storage, and it makes the definition
   additive-only from then on — see `entities-and-persistence` before reshaping it.

From here, hand off to the other kinotic skills: entity modeling and persistence →
`entities-and-persistence`; business logic and APIs → `services`; UI and client
connections → `frontend`; running, releasing, and diagnosing → `deploying`.
