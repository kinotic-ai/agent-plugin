---
name: deploying
description: >
  How a Kinotic project runs: pushing to the default branch builds and deploys it, the
  deployment job's six steps, checking deployment status and the deployed commit with
  the Project Service Find Deployment MCP tool and the per-microservice and per-UI
  deployment tools, diagnosing a FAILED deployment, the machine identities the deployment
  provisions, and what can and cannot be run locally.
  Use when deploying, running, restarting, or releasing a Kinotic app, checking whether
  a service is live, or working out why a push did not take effect.
---

# Running and Deploying a Kinotic Project

Pushing to the project repository's **default branch** is the deployment. There is no CI
to configure, no pipeline file, and nothing to run from a developer machine.
Docs: <https://kinotic.ai/apps/deployment/push-to-deploy>.

Only default-branch pushes deploy. Pushes to other branches, branch deletions, and tag
operations are ignored. Creating a project deploys nothing: a new project waits for its
first push.

## Pushing

Push with a token Kinotic issues, on GitHub and a local Kinotic's Git server alike — never
with `gh` or the user's own GitHub credentials, which cannot reach a local Kinotic's
repositories. Before every push, call the tool titled
`Project Repo Credential Service Issue Write Token` with `{"projectId": "<project id>"}`; a
token is good for an hour, so never reuse one from an earlier push or the clone. Present it
as the password of the user `x-access-token`, through a credential helper scoped to the one
command:

```bash
KINOTIC_REPO_TOKEN='<token>' git -c credential.helper= \
  -c 'credential.helper=!f() { echo username=x-access-token; echo "password=$KINOTIC_REPO_TOKEN"; }; f' \
  push origin <defaultBranch>
```

The empty `credential.helper=` comes first so no stored helper (the macOS keychain, a
credential manager) saves the token. Never put the token in the remote URL, a file or git
config, and never echo it back to the user. Commits are authored by the user's own
`git config user.name` and `user.email`.

## What the platform actually runs

| Artifact | Status |
|---|---|
| Entities and their generated repositories | Synchronized and published on every push |
| Every directory under `packages/microservices` with a `package.json` | Gets its own long-lived runtime workload |
| Every directory under `packages/ui` whose `package.json` declares a `build` script | Built with `bun run build` and published to its own site |
| A `packages/ui` package with no `build` script | Treated as a library — never published |
| Batch/scheduled jobs | Not implemented |

**Artifacts are discovered by where they sit and what their `package.json` says**, so
adding one is adding a package — there is no registry to update:

- A microservice is a directory directly under `packages/microservices` holding a
  `package.json`. Its entry is that file's `main`, or `src/main.ts` when it declares none.
  Each gets its own workload, so a second microservice really does run.
- A UI is a directory directly under `packages/ui` whose `package.json` declares a `build`
  script. It must honor the UI build contract (`frontend` skill) or it publishes assets
  nothing can load.
- An artifact's identity is the **unscoped part of its `package.json` name** (`@acme/orders`
  is `orders`), which must be lowercase letters, digits, and interior dashes, and unique
  among artifacts of its kind. The directory name never matters. A missing or invalid name,
  or two artifacts of one kind sharing a name, fails the deployment naming the package.
- A UI's name must not contain `--`, and `<applicationId>--<uiName>` must fit in 37
  characters, since it leads the site's hostname. A UI name belongs to one project of the
  application: a second project publishing a UI of the same name fails its deployment until
  the first project's UI deployment is removed in the console.

## The deployment job

Each qualifying push runs one tracked job, `Deploy project <id> at commit <sha>`, with six
steps:

1. **Resolve deployment target** — the first deployment picks a node with capacity and
   creates the project's checkout directory on it; later deployments reuse the same node
   and directory.
2. **Sync project source** — a short-lived sandboxed build VM fetches the pushed commit into
   the checkout (incremental, so installs stay warm), runs `bun install`, finds the
   artifacts, then runs `kinotic sync`, which regenerates from the entity sources, pushes
   the entity definitions and named queries, publishes every entity it introduces (which
   creates the backing storage), and applies pending migrations from `./migrations`. Then
   it builds every UI with `bun run build` and the three build variables; a build that
   writes no `dist/index.html` fails the step naming the UI. **This step is the build
   gate**: it fails when generation fails, when the server rejects an entity definition, a
   named query or a migration, when a package is not named validly, or when a UI build
   fails, and nothing of the commit reaches the running services. Only after it fully
   succeeds does it write the reload sentinel. The VM is destroyed when its run ends,
   whatever the outcome; its log stays readable from the job run.
3. **Resolve artifacts** — binds the artifacts the build VM reported into the run; the
   step's row on the job run page lists each microservice with its entry module and each UI.
4. **Ensure runtime workloads** — every microservice of the commit ends with a VM serving
   it, one per microservice, with a machine identity of its own: a new microservice gets a
   VM on its first appearance, and a running one is kept, its supervisor restarting the
   process onto the new commit. A VM whose entry module the commit moved, or whose run has
   ended, is replaced by a fresh one. A microservice that cannot be left running is
   recorded `FAILED` with the reason, the others still deploy, and the step fails naming it.
5. **Publish UIs** — uploads every UI the build VM built to its site through a short-lived
   publish VM that holds one upload URL per site and no Kinotic credentials. A UI's first
   publish mints its site's hostname; later publishes switch the site to the new commit.
6. **Generate SBOM** — when the dependencies installed from `bun.lock` changed since the
   last SBOM, a short-lived SBOM VM reads the dependency tree and records it; otherwise no
   VM runs. A checkout without a `bun.lock` fails this step, and so the deployment, although
   the microservices and UIs already serve the new commit.

Consequences worth stating to a user:

- A failed build leaves the **previously deployed commit running**. Nothing is torn down.
- Deployments serialize per project, **latest-wins**: pushes arriving during a deployment
  collapse to the newest commit; intermediate commits are skipped, not queued.
- Redelivering the same push is harmless — syncing a commit twice converges.
- The runtime restarts your services as whole processes. There is no hot reload of a
  single class. A process that exits is started again: inside the VM with a backoff from one
  second to thirty, and when the VM's run itself ends, the platform replaces the VM after a
  wait that doubles with each failure since the last deployment, from thirty seconds to ten
  minutes (`restartAt` on the microservice's deployment says when). A new push or a restart
  starts it again at once.
- An artifact that disappears from a commit is marked **orphaned**, not deleted: its
  workload keeps running and its site keeps serving. One that comes back is adopted rather
  than re-provisioned. Removing it for real is a console action.
- **The build gate does not type-check microservice code.** The sync step generates from the
  entity sources and builds the UIs, but a microservice is only run, by its VM, so a type
  error or a crash on startup in a service surfaces as a `FAILED` microservice deployment
  with its process restarting, not as a failed job. Run `bun run type-check` locally before
  pushing.

## Checking a deployment

Call the MCP tool titled `Project Service Find Deployment` with
`{"projectId": "<project id>"}` (Kinotic OS tool names are opaque hashes — pick the tool
whose description starts with that title). It returns `null` when the project has never
deployed. The record keeps what the deployment should be beside what it is, under `state`:

```json
{
  "id": "inventory-app-main",
  "nodeId": "node-1",
  "artifacts": { "commitSha": "9f2c1ab…", "microservices": [{ "name": "main", "dir": "packages/microservices/main", "entry": "src/main.ts" }], "uis": [{ "name": "admin", "dir": "packages/ui/admin" }], "dependencyHash": "…" },
  "sbomGenerated": true,
  "syncWorkloadId": "wl-…",
  "syncMachineIdentityId": "…",
  "lastJobRunId": "…",
  "failureMessage": null,
  "state": {
    "desired": { "phase": "RUNNING", "commitSha": "9f2c1ab…" },
    "observed": { "phase": "RUNNING", "commitSha": "9f2c1ab…" },
    "generation": 3, "observedGeneration": 3, "reconciled": true,
    "conditions": []
  }
}
```

`state.desired` is what the last default-branch push asked for, always the phase `RUNNING`
with the pushed commit. `state.observed` is what the deployment is:

| `state.observed.phase` | Meaning | What to tell the user |
|---|---|---|
| `DEPLOYING` | A job is running for the latest qualifying push; `observed.commitSha` is the commit still live | Wait and re-check; the job page shows live steps |
| `RUNNING` | Every step passed; `observed.commitSha` is live | Compare `observed.commitSha` to the pushed commit |
| `FAILED` | The last deployment failed | `failureMessage` carries the reason; `observed.commitSha` is still the previous commit |

The pushed commit is live exactly when `state.observed.phase` is `RUNNING` and
`state.observed.commitSha` is the pushed commit. Poll rather than assume: after pushing,
`findDeployment` moves `DEPLOYING` → `RUNNING`/`FAILED`. A `state.desired.commitSha` that
still names the previous commit means the push was not received — check that it landed on
the default branch.

Each microservice and each UI has a deployment record of its own, in the same `state` shape:

- `Microservice Deployment Service Find All For Project` (`{"projectId": …}`) lists the
  project's microservices with `workloadId`, `machineIdentityId`, `entryPoint`,
  `failureMessage` and `restartAt`. `state.observed.phase` is `DEPLOYED` (the VM runs the
  microservice at `observed.commitSha`), `FAILED` (`failureMessage` says why) or `ORPHANED`
  (the last commit dropped it; the VM keeps running). `state.conditions` carries
  `NODE_UNREACHABLE` while the platform cannot reach the node.
  `Microservice Deployment Service Restart` (`{"deploymentId": …}`) runs a microservice in a
  fresh VM from the current deployment.
- `Ui Deployment Service Find All For Project` lists the project's UIs with the site's
  `url`, the phase `PROVISIONING` (the site does not yet serve the published commit;
  `observation` says what it answered), `READY` or `ORPHANED`.
- `Project Service Find Deployment History` (`{"projectId": …, "pageable": …}`) lists what
  happened to the deployment and the records it made, newest first, and
  `Microservice Deployment Service Find History` / `Ui Deployment Service Find History`
  (`{"deploymentId": …, "pageable": …}`) the same for one record — how a deployment got to
  where it is.

## When a deployment fails

`failureMessage` is the failure reason recorded from the job. The most common cause is
the build gate: generation failed, the server rejected an entity change, or a UI build
failed. Reproduce it locally first — `bun install && bun run generate` runs the same
generation the sync step does, and `bun run type-check` catches what the sync step does not
— then fix and push again. A `RUNNING` project whose service does not answer is a
microservice that crashed on startup: `Microservice Deployment Service Find All For Project`
shows it `FAILED` with its `failureMessage`.

Deployment logs are **not** reachable through MCP: `LogService` streams workload logs but
is not exposed as a tool. Send the user to the portal for anything the messages do not
answer:

- **Project → Deployment page** (`/application/<applicationId>/project/<projectId>/deployment`)
  — status, live commit, the latest deployment run's steps live, each microservice with its
  log, restart and removal, each UI with its site, and the project's machine identities.
- **Jobs page** (`/jobs`, then the run) — the same job among all organization jobs; a step
  opens into its VM's log.
- The build VM of a failed sync is destroyed like any other, but its workload record and log
  stay readable from the job run.

Never try to "fix" a deployment by deleting and recreating the project — that removes
the project's machine identities, deployment records, microservice VMs and UI sites along
with it.

## Machine identities

The deployment provisions the project one machine identity for the build VM (type
`PROJECT_SYNC`, shared by the SBOM VM) and one per microservice (type `APP_RUNTIME`), listed
on the Deployment page. All are **organization scope**, because publishing services into
the application's zone and synchronizing entity definitions are things the organization
does on its own behalf — an application-scope identity cannot do either. The UI publish VM
carries no identity at all.

Kinotic stores only a hash of a machine's secret, so a secret is disclosed exactly once:

- The **sync** identity gets a freshly issued secret on every deployment; the previous one
  stops working at that moment. The SBOM VM that runs after the build VM gets a newer one
  again.
- A **microservice** VM's secret is issued once with the VM and lives for that VM's life,
  since later deployments restart the process inside the running VM.

Do not rotate a project's runtime machine secret to borrow it for local work — the
running deployment is using it, and rotating cuts it off at its next reconnect. Deleting
a project machine is recoverable: the next deployment provisions a replacement.

## Running locally

What is possible locally follows directly from connection scope (see the `services`
skill for the full table):

- **Calling services and using entity repositories** — works. Create a machine identity
  on the portal's **Application → Machines** page (application scope), and run with
  `KINOTIC_CLIENT_ID` / `KINOTIC_CLIENT_SECRET` set, or connect as an application user
  with `BasicCredentialsResolver(email, password, organizationId, applicationId)`. This
  is how a one-off script talks to the deployed application.
- **Hosting a `@Publish` service** — needs organization scope, which the Machines page
  does not mint (it provisions application-scope machines). Push to run your services.

So the working loop is: edit → `bun run generate` (when entities changed) → export from
`packages/domain/index.ts` → `bun run type-check` → commit → push (see **Pushing**) →
`findDeployment` until `state.observed` is `RUNNING` at the pushed commit. Verify behavior by
calling the deployed service through a proxy from a local script, or on the published UI
(`frontend` skill, **Verifying a UI**), not by running it locally.

A local Kinotic (`kinotic local up`) runs the whole of Kinotic OS on the user's machine,
deployments included, and a project works against it as against Kinotic Cloud, through its
`kinotic-local` MCP server: push, then `findDeployment` until `RUNNING`. With more than one
Kinotic OS server connected, every call goes to the server that holds the project.
