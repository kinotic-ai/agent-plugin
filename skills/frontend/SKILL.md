---
name: frontend
description: >
  How a frontend or Node client talks to a Kinotic application: how the UI resolves which
  server URL to connect to, how to verify a UI on its published site, Kinotic.connect, the three VITE_KINOTIC_* build variables
  the deployment sets, browser sign-in through the gateway's login routes and the session
  cookie it establishes, BasicCredentialsResolver and BearerCredentialsResolver for Bun
  and Node clients, machine identities via environment credentials, OIDC login, sign-up pages and
  application users, invoking published services and entity repositories from the UI, and
  the vite.config.ts a UI needs to bundle the domain package and its entity decorators. Use when building a
  Kinotic app frontend or SPA, working out what host or URL the UI should point at, wiring
  up login or authentication, connecting any client to a Kinotic server, or when a UI
  bundle fails to load with a SyntaxError or a blank page.
---

# Frontends and Clients

Every client — browser SPA, Node client, script — speaks the same STOMP-over-WebSocket
protocol to the Kinotic RPC gateway. There is no REST layer for application services;
the only REST endpoints are platform auth and OAuth (`/api/auth/*`), MCP (`/mcp`) and
`/health`. Frontends
live in `packages/ui` of the project scaffold and may use any framework that builds to
static assets. Docs: <https://kinotic.ai/apps/security/authentication>.

A UI under `packages/ui` **is** built and published by the deployment and served at its own
site — see "The UI build contract" below for what the build is handed and what it must
produce.

Authentication happens at the **WebSocket upgrade**, and every authentication carries a
scope. Application users always authenticate at APPLICATION scope — both
`organizationId` and `applicationId` are supplied (directly or via JWT claims). Scope
isolation means a user of one application can never reach another application, even
with the same identity provider.

An application-scope connection may **call** services in its own
`app.<org>.<app>` zone and use entity repositories (`app-api`), which is everything a
frontend needs. It cannot **host** a `@Publish` service — that requires organization
scope, and is what the deployment's runtime does (see the `services` and `deploying`
skills).

`Kinotic.connect(options?)` takes an optional plain `ConnectOptions` literal; every
field is optional. Unset server fields resolve in order: explicit value →
`KINOTIC_SERVER_HOST` / `KINOTIC_SERVER_PORT` / `KINOTIC_SERVER_USE_SSL` → the browser's
own location → `https://api.kinotic.ai`. Credentials resolve through a resolver chain:
the environment variables, then the browser session.

## How the UI knows where the server is

Answer this before writing any connect code — it decides what the connect code passes, and
it is the question the resolution ladder above does not settle on its own.

**The env-var rung does not exist in a browser.** Core reads it as
`typeof process !== 'undefined' ? process.env : undefined`, and bundlers do not provide
`process` in browser builds (Vite exposes `import.meta.env` instead). So putting
`KINOTIC_SERVER_HOST` in a `.env` does nothing for an SPA — the ladder skips straight
from an explicit `server` option to `window.location`, which for a published UI is its own
site, not Kinotic. So the UI passes `server` explicitly.

A published UI is served from its own site host, and the deployment sets `VITE_KINOTIC_HOST`,
`VITE_KINOTIC_PORT` and `VITE_KINOTIC_USE_SSL` for the build to the application's API host on
the server it deploys to. `connect()` is handed a `server` built from those variables, and the
REST calls are built from the same values and sent with `credentials: 'include'` so the session
cookie travels. Recipe 1's `serverOptions()` / `apiUrl()` are that code, and the platform's own
consoles connect the same way. Nothing is hand-set, and the code is the same on every server.

The session cookie is `SameSite=Lax`, which is sent whenever the API and the site share a
site, as an application's API host and its sites do: both under `kinotic.io` on Kinotic Cloud
(the API host `<organizationId>--<applicationId>.apps-api.<zone>`, the sites
`<applicationId>--<uiName>.<organizationId>.<zone>`; `kinotic.ai` holds only the platform's
own hosts), and both under `kinotic.localhost` on a local Kinotic. A published UI therefore
uses the cookie flow (Recipe 1).

## The UI build contract

Every UI package under `packages/ui` that declares a `build` script is built during the
deployment with `bun run build` and published to its own site. The build is handed its
application's API host, `<organizationId>--<applicationId>` under the platform's application
API domain, as exactly three variables, and Vite exposes them to the page on its own:

| Variable | Value | What the UI does with it |
|---|---|---|
| `VITE_KINOTIC_HOST` | e.g. `acme--orders.apps-api.kinotic.io` | The host the UI connects to Kinotic on from the browser |
| `VITE_KINOTIC_PORT` | e.g. `443` | Its port |
| `VITE_KINOTIC_USE_SSL` | `true` or `false` | Whether to connect over TLS |

Publishing needs **no `vite.config.ts` changes** — no `base`, no `define`. A UI on another
build tool must pass the three through to the page itself. A UI that imports the domain package
does need a domain alias and a decorator plugin in its Vite config; see below. The
deployment hands the build no base path and no commit, and the publish uploads `dist` under
the site's root as it is.

Declare the UI's ambient typing once, so `import.meta.env.VITE_KINOTIC_*` type-checks
(Vite's `vite/client` types merge with it):

```ts
// src/vite-env.d.ts
/// <reference types="vite/client" />

interface ImportMetaEnv {
    readonly VITE_KINOTIC_HOST?: string
    readonly VITE_KINOTIC_PORT?: string
    readonly VITE_KINOTIC_USE_SSL?: string
}
```

Files under `assets/` carry Vite's content hash in their name and are cached for a year;
everything else, `index.html` included, is never cached. A file outside `assets/` that
changes between publishes is therefore safe.

Each site publishes `version.json` (`{ "commitSha": "<commit>" }`) next to `index.html`.
The platform reads it to decide when a site is ready. A publish deletes the previous
commit's files.

A build that leaves no `dist/index.html` **fails the deployment**, naming the UI — so a UI
whose build writes elsewhere never publishes.

A package under `packages/ui` **without** a `build` script is treated as a library and is
never published — that is how shared component packages live there.

Kinotic projects run on Bun, whose built-in WebSocket accepts the upgrade headers
credentials travel on. Only a **Node** process sending credentials must first call
`ensureNodeWebSocket()` from `@kinotic-ai/core/node`, which swaps in `ws` because Node's
own WebSocket ignores a headers option. It is a no-op under Bun and unnecessary in the
browser — do not add it to a Bun entry point.

## Verifying a UI

A UI is tried on its published site, never on a dev server or a proxy in front of Kinotic: the
deployment builds it with the variables for the server it deploys to, so the site is the only
place it runs as its users see it. After changing a UI, commit and push (`deploying` skill),
then:

1. Call `Project Service Find Deployment` with `{"projectId": "<project id>"}` until
   `state.observed.phase` is `RUNNING` with `state.observed.commitSha` at the pushed commit,
   or `FAILED`, whose `failureMessage` says why.
2. Call `Ui Deployment Service Find All For Project` with `{"projectId": "<project id>"}`. Each
   UI deployment carries the site's `url`
   (`https://<applicationId>--<uiName>.<organizationId>.<sites domain>`), its `state`
   (`observed.phase` `PROVISIONING` until the site serves the commit, then `READY`), and an
   `observation` saying what the site answered while it does not yet serve the commit.
3. Fetch `<url>/version.json`; the site serves the push once its `commitSha` is the pushed
   commit.
4. Give the user the `url` to open, and say what to try there: signing up, signing in, the
   screens the change touched.

Kinotic OS MCP tool names are opaque hashes; pick the tool whose description starts with the
title. A local Kinotic serves its sites over plain `http`, and the session cookie is `Secure`
(a `__Host-` cookie), so signing in there relies on the browser treating `*.localhost` as a
secure context: Chrome and Firefox do, Safari does not store the cookie.

## Using the domain package in a Vite UI

A UI that imports the domain package (`@<project slug>/domain`, for its repositories or entity
types) needs two things in its `vite.config.ts`:

- **An alias to the domain package's source.** The deployment runs `bun run build` in the UI's
  own directory and never builds the domain package. A production Vite build resolves the
  package to its `dist/index.js`, which does not exist, and fails with `failed to resolve
  import "@<project slug>/domain"`. Aliasing the package to its `index.ts` bundles the source.
- **A plugin that compiles the decorators**, when the UI uses an entity class as a value. The
  persistence decorators (`@Entity`, `@Id`, `@TenantId`, `@Query`, …) are standard TC39
  decorators, and Vite 8 copies them into the bundle unchanged. No browser parses that
  syntax, so the bundle fails to load and the page stays blank.

The decorators reach the bundle only when the UI uses an entity class as a value: `new Todo()`,
`instanceof Todo`, or reading a static off the class. The template's domain package declares
`"sideEffects": false`, so a UI that imports a repository and uses `Todo` only as a type
(`import { TodoRepository, type Todo } from '@<project slug>/domain'`) has the entity module
dropped from its bundle. A project created before the template declared it lacks the field;
add it to `packages/domain/package.json`. When unsure which case a UI is in, add the plugin.

Write the plugin into the UI's own `vite.config.ts`, and add `typescript` to the UI's
`devDependencies`. For a UI at `packages/ui/<name>`:

```typescript
// vite.config.ts
import { fileURLToPath } from 'node:url'
import { defineConfig, type Plugin } from 'vite'
import ts from 'typescript'

// Compiles the TC39 decorators on entity and repository classes, which Vite leaves in the bundle
function kinoticDecorators(): Plugin {
    return {
        name: 'kinotic-decorators',
        enforce: 'pre',
        transform(code, id) {
            const file = id.split('?', 1)[0]
            if (!file.endsWith('.ts') || file.includes('/node_modules/')
                || !/from\s*['"]@kinotic-ai\/persistence['"]/.test(code)) {
                return
            }
            const result = ts.transpileModule(code, {
                fileName: file,
                compilerOptions: {
                    target: ts.ScriptTarget.ES2022,
                    module: ts.ModuleKind.ESNext,
                    useDefineForClassFields: true,
                    sourceMap: true
                }
            })
            return { code: result.outputText, map: result.sourceMapText }
        }
    }
}

export default defineConfig({
    plugins: [kinoticDecorators()],
    resolve: {
        // The deployment builds this UI without building the domain package, so bundle its source
        alias: {
            '@<project slug>/domain': fileURLToPath(new URL('../../domain/index.ts', import.meta.url))
        }
    }
})
```

The plugin compiles every `.ts` module outside `node_modules` that imports from
`@kinotic-ai/persistence` (the entities and repositories) and leaves the rest of the UI to
Vite; add a framework plugin such as `vue()` after it. Verify the build with `bun run build`
in the UI's directory, then parse the bundle, for example `node --check` on each file in
`dist/assets/*.js`; searching the bundle for `@Entity` misses minified output.

## Recipe 1 — Browser SPA (session cookie)

A browser **cannot** set WebSocket upgrade headers. `BasicCredentialsResolver` puts the
credentials in upgrade headers, which only the Node and Bun sockets accept, so in a browser
the connection never authenticates (`Credential resolution failed`, or with
`maxConnectionAttempts` set, `Max number of reconnection attempts reached`). In a browser
the session cookie is the credential, and the gateway's login route establishes it.

- The app server identifies the application from the request's hostname, the API host the
  UI connects to (`<organizationId>--<applicationId>.<apps API domain>`), so the login routes
  carry no organization or application id in their path.
- The app-scope password login is `POST /api/auth/app/login` with `{ email, password }`. It
  answers `204` and sets the cookie, `401` with `{ "error": "Invalid credentials" }` for a
  wrong email or password, `400` for a blank field, or `403` when the page that sent it is
  not one of the application's UIs (its published sites, plus any origin matching the app
  server's `kinotic.appServer.localUiOriginPattern`, which a local Kinotic sets to
  `http://localhost:<any port>` so a Vite dev server can sign in; Kinotic Cloud sets none).
  The user must exist in **that application's** user base (portal, Application →
  **Users**); the route does not authenticate organization members.
- `GET /api/auth/me` answers `204` when the browser holds a live session and `401`
  otherwise. `POST /api/auth/logout` ends the session.
- Every request carries `credentials: 'include'`, and `Kinotic.connect` takes only the
  server: the STOMP upgrade rides the cookie.

```ts
// src/kinotic/session.ts
//
// Sign-in for a Kinotic app UI. The deploy builds the UI with three variables naming the
// platform it was published against, and Vite exposes them to the page on its own:
//   VITE_KINOTIC_HOST     e.g. acme--orders.apps-api.kinotic.io, the application's API host
//   VITE_KINOTIC_PORT     e.g. 443
//   VITE_KINOTIC_USE_SSL  "true" | "false"
//
// A browser cannot set WebSocket upgrade headers, so credentials never go to Kinotic.connect
// here. The gateway's login route establishes the session cookie, and the STOMP upgrade rides
// that cookie; connect() takes only the server.

import { Kinotic, type ServerInfo } from '@kinotic-ai/core'

/** The platform this UI was built against. */
export function serverOptions(): ServerInfo {
    const host = import.meta.env.VITE_KINOTIC_HOST
    if (!host) {
        throw new Error('VITE_KINOTIC_HOST is not set: the build was not handed the platform address')
    }
    const useSSL = import.meta.env.VITE_KINOTIC_USE_SSL === 'true'
    const port = parseInt(import.meta.env.VITE_KINOTIC_PORT || (useSSL ? '443' : '80'))
    return { host, port, useSSL }
}

/** Absolute URL of a gateway REST path on that platform. */
export function apiUrl(path: string): string {
    const { host, port, useSSL } = serverOptions()
    return `${useSSL ? 'https' : 'http'}://${host}:${port}${path.startsWith('/') ? path : `/${path}`}`
}

/**
 * Signs in with an application user's email and password, then opens the connection.
 * Rejects with the gateway's message ("Invalid credentials" when it gives none).
 */
export async function login(email: string, password: string): Promise<void> {
    const res = await fetch(apiUrl('/api/auth/app/login'), {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        credentials: 'include',            // stores the Set-Cookie from the API's origin
        body: JSON.stringify({ email, password }),
    })
    if (!res.ok) {
        throw new Error(await errorMessage(res, 'Invalid credentials'))
    }
    await connect()
}

/**
 * Reconnects a returning tab: true when the browser still holds a live session and the
 * connection is open, false when the user must sign in. Rejects when the session is live
 * but the connection cannot be opened, so a caller can tell the two apart.
 */
export async function resume(): Promise<boolean> {
    const res = await fetch(apiUrl('/api/auth/me'), { credentials: 'include' })
    if (!res.ok) {
        return false
    }
    await connect()
    return true
}

/** Closes the connection and ends the browser session. */
export async function logout(): Promise<void> {
    await Kinotic.disconnect(true)
    await fetch(apiUrl('/api/auth/logout'), { method: 'POST', credentials: 'include' })
}

async function connect(): Promise<void> {
    await Kinotic.connect({
        server: serverOptions(),
        // bounded so a session the gateway no longer honors fails fast instead of retrying
        // the upgrade forever with no feedback on the login button
        maxConnectionAttempts: 3,
    })
}

async function errorMessage(res: Response, fallback: string): Promise<string> {
    try {
        const body = await res.json() as { error?: string }
        return body.error ?? fallback
    } catch {
        return fallback
    }
}
```

One condition applies only to a **self-hosted** server; on Kinotic Cloud it is already
settled. The cookie is `SameSite=Lax`, sent when the API and the site share a site — which
the hosted platform does. A self-hosted environment whose API sits on a different site from
its published UIs (a tunnel origin such as ngrok, say) needs
`kinotic.apiGateway.sessionCookieSameSite: NONE` in the server's profile, which sends the
cookie from any origin the CORS pattern admits.

## Recipe 2 — Machine identity, zero ceremony (environment credentials)

The default credential resolver chain reads the environment, so a script or local
process normally passes nothing:

```bash
KINOTIC_CLIENT_ID=<machine identity id> KINOTIC_CLIENT_SECRET=<secret> \
bun run src/main.ts
```

```typescript
import { Kinotic } from '@kinotic-ai/core'

await Kinotic.connect()
```

Create the machine in the portal: **Application → Machines → create machine**. The
secret is disclosed exactly once (only a hash is stored), so it can be rotated but never
re-read. Machines created there are **application scope** — they call services and use
repositories, and cannot publish a service. Optionally `KINOTIC_ORGANIZATION_ID` /
`KINOTIC_APPLICATION_ID` pin the scope explicitly; `KINOTIC_TOKEN` resolves to bearer
auth instead. A `clientId` containing `@` is a user email; anything else resolves only
to a machine identity. When nothing resolves, connect fails with
`No Kinotic credentials found; consulted: <resolvers>`.

Do not borrow a project deployment's machine secret for this — rotating it cuts the
running deployment off.

## Recipe 3 — explicit credentials in code

For **Bun and Node clients** — scripts, tests, machines. Both resolvers below put the
credentials in WebSocket upgrade headers, which a browser cannot send; a browser signs in
through Recipe 1.

Email/password (an application user):

```typescript
import { BasicCredentialsResolver, Kinotic } from '@kinotic-ai/core'

await Kinotic.connect({
    credentials: new BasicCredentialsResolver('user@example.com', 'password', 'my-organization', 'my-application')
})
```

The arguments are `(clientId, clientSecret, organizationId?, applicationId?)` — scope
is structural: neither id → SYSTEM, organization only → ORGANIZATION, both →
APPLICATION. (In Node, call `ensureNodeWebSocket()` first; on Bun, don't.)

Bearer token (OIDC):

```typescript
import { BearerCredentialsResolver, Kinotic } from '@kinotic-ai/core'

await Kinotic.connect({ credentials: new BearerCredentialsResolver(async () => await getToken()) })
```

The supplier runs before every (re)connect, so short-lived tokens refresh
automatically. A Kinotic-minted JWT already carries the organization/application
claims — no extra scope headers.

## Application users, sign-up and OIDC login

An application's users are its own user base, apart from the organization's members, and
an identity at a provider never grants access by itself. A user comes from one of:

- **An invitation.** An organization admin invites the email from the portal's
  Application → **Users** page; the invitee accepts the emailed link with a password and
  from then on signs in through Recipe 1's `login()`.
- **Sign-up with an email and a password.** While the application's **Email and
  password** provider has `createUserOnSignIn` on, and only while the application's
  `tenantManagement` is `SYSTEM`, a new person signs up on the UI's `/signup` page
  (below), which gives them a tenant of their own.
- **First sign-in through a provider.** An `ApplicationOidcConfiguration` with
  `createUserOnSignIn` on turns an identity it verifies that no user is bound to into a
  user, from the provider's verified email and name — only while the application's
  `tenantManagement` is `SYSTEM` (the default is `TENANT_ADMINISTRATORS`), which gives
  each such user a tenant of its own. A `TenantOidcConfiguration` with
  `createUserOnSignIn` does the same into its tenant, at that tenant's login.

A Google or Microsoft provider may use Kinotic's own registration with the provider
(`clientRegistrationType` `MANAGED`, no client of the organization's own) or the
organization's (`OWN`); sign-in and sign-up behave the same either way. An identity with
no user and nothing to create one ends at `no_user`; one whose email the application
already holds under another method ends at `user_exists`; a Microsoft account whose email
the token does not vouch for ends at `email_not_verified`.

The login page offers every method at once — the password form (Recipe 1's `login()`)
and a "Continue with <name>" button per provider — and never asks the server about an
email first; there is no lookup route.

1. To see what the login offers, call the MCP tool whose description begins with
   `Application Service Get Login Methods`, with `{"applicationId": "..."}` (Kinotic
   MCP tool names are opaque hashes; match on the description). It answers
   `[{configurationId, type, name, provider, signUp}]` in presentation order. A `PASSWORD`
   entry is the email and password form; an `OIDC` entry is a provider button. `signUp`
   true means new people may sign up through it. A list with no `OIDC` entry means the
   login is password only — tell the user rather than writing provider buttons nobody can
   use.
2. At runtime the page reads the same list from its API host, with
   `credentials: 'include'` as in Recipe 1:
   - `GET /api/auth/app/login/methods` answers the same
     `[{ configurationId, type, name, provider, signUp }]` — `provider` is the kind's key
     (`google`, `azure-ad`, `okta`, …) for branding, `null` for `PASSWORD`.
   - A button posts to `POST /api/auth/app/login/oidc/start/<configurationId>` (no body)
     and answers `{ "redirect": "…" }`; send the browser there. `400` for a configuration
     the login does not offer, `403` from a page that is not one of the application's UIs.
     The flow's state lives in the API host's session, so the start needs
     `credentials: 'include'`.
   - The provider returns to the API host (a shared registration via the app server's
     shared host), which sets the session cookie and sends the browser to `/` on the site
     that started the sign-in. Recipe 1's `resume()` then sees the live session and opens
     the connection.
   - A failed sign-in lands on `<site>/login?error=<code>` (`no_user`, `user_disabled`,
     `user_exists`, `email_not_verified`, `handoff_invalid`, `state_mismatch`,
     `exchange_failed`, …; `no_user` and `email_not_verified` add `&provider=<key>` naming
     the provider that verified the identity), so the UI needs a `/login` route that shows
     the error and the methods again. One whose flow session is lost lands on the
     application's primary UI's `/login` instead.
3. A page served at one of a tenant's sites signs in to that tenant with the same calls:
   the list adds the tenant's own providers ahead of the application's, and
   `/api/auth/app/login` admits only the tenant's users. A page that is not a tenant site
   (a UI on localhost, say) names the tenant with `?tenantId=` on the methods `GET` and
   the start, and `tenantId` in the password body; `404` for a tenant the application
   lacks. The MCP tool lists the application's own login only.

### Sign-up with an email and a password

While the `PASSWORD` entry's `signUp` is true, the UI serves a `/signup` page and the
`/signup/verify` page the link opens; a tenant's site signs no one up.

- `/signup` posts `{ email }` to `POST /api/auth/app/signup`. The answer depends on whether
  the server sends email, and the page must handle both, since the same app runs on every
  server:
  - `202` — the link was emailed. Tell the person to check their inbox. It answers `202`
    for every email, including one that already has a user, so it reveals nothing.
  - `200 { verificationUrl }` — the server sends no email (a local server does not), so
    show the link on the page for the person to open. Never tell them to check their
    inbox here: nothing was sent.
  - `409` — the server sends no email and the email already has a user; offer sign-in.

  Returning the link is intended, not a hole for the app to close. A Kinotic server sends
  no email only when its operator turned email off (`kinotic.domain.email.enabled: false`),
  as the local Kinotic `kinotic local up` starts on the developer's own machine does so they
  can try their app without an email service; only the developer can reach it, so handing
  them the link exposes nothing. A server that sends email, Kinotic Cloud included, answers
  `202` and never puts the link in a response. The server decides which answer to give, so
  the page shows whatever it receives: never hide the link, check which server it runs on,
  or treat the `200` as a misconfiguration.
- `/signup/verify?token=…` asks for a password and an optional name (at most 200
  characters) and posts `{ token, password, displayName }` to
  `POST /api/auth/app/signup/complete`, which answers `204` with the session cookie;
  then open the connection as Recipe 1's `login()` does. `400` for a token that is
  unknown, used or expired; `409` when the email already has a user, who signs in.
  The `204` creates the account and spends the link, so when opening the connection fails
  after it, send the person to `/login` to sign in with their new password instead of
  leaving them on a page whose link no longer works.

```ts
// src/kinotic/session.ts, beside Recipe 1's login()

/** A sign-in method the login offers, as GET /api/auth/app/login/methods lists it. */
export interface LoginMethod {
    configurationId: string
    type: 'PASSWORD' | 'OIDC'
    name: string
    provider: string | null
    signUp: boolean
}

/** The methods to render, in order: the password form for PASSWORD, a "Continue with <name>" button per OIDC. */
export async function loginMethods(): Promise<LoginMethod[]> {
    const res = await fetch(apiUrl('/api/auth/app/login/methods'), { credentials: 'include' })
    if (!res.ok) {
        throw new Error(await errorMessage(res, 'Could not load sign-in methods'))
    }
    return await res.json() as LoginMethod[]
}

/** Starts a sign-in through one provider; the browser leaves the page. */
export async function continueWith(method: LoginMethod): Promise<void> {
    const res = await fetch(apiUrl(`/api/auth/app/login/oidc/start/${encodeURIComponent(method.configurationId)}`), {
        method: 'POST',
        credentials: 'include',            // the flow's state rides the API host's session
    })
    if (!res.ok) {
        throw new Error(await errorMessage(res, 'Could not start sign-in'))
    }
    const { redirect } = await res.json() as { redirect: string }
    window.location.assign(redirect)
}

/** What the sign-up page shows once the email is accepted. */
export type SignUpStarted =
    | { kind: 'emailed' }                              // tell the person to check their inbox
    | { kind: 'link', verificationUrl: string }        // no email is sent: show this link to open

/** Starts a sign-up; rejects with 'An account already exists' when the server reports the email has a user. */
export async function startSignUp(email: string): Promise<SignUpStarted> {
    const res = await fetch(apiUrl('/api/auth/app/signup'), {
        method: 'POST',
        credentials: 'include',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ email }),
    })
    if (res.status === 409) {
        throw new Error('An account already exists for this email. Sign in instead.')
    }
    if (!res.ok) {
        throw new Error(await errorMessage(res, 'Could not start sign-up'))
    }
    let ret: SignUpStarted
    if (res.status === 200) {
        const { verificationUrl } = await res.json() as { verificationUrl: string }
        ret = { kind: 'link', verificationUrl }
    } else {
        ret = { kind: 'emailed' }
    }
    return ret
}

/** Completes a sign-up from /signup/verify?token=…, then opens the connection as login() does. */
export async function completeSignUp(token: string, password: string, displayName?: string): Promise<void> {
    const res = await fetch(apiUrl('/api/auth/app/signup/complete'), {
        method: 'POST',
        credentials: 'include',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ token, password, displayName }),
    })
    if (!res.ok) {
        throw new Error(await errorMessage(res, 'Could not finish sign-up'))
    }
    try {
        await connect()
    } catch {
        // the account exists and the link is spent, so the person continues by signing in
        window.location.assign('/login?created=1')
    }
}
```

## Calling services from the frontend

Once connected, the frontend calls published services exactly like any client — the
hand-written proxy pattern from the services skill (note the `~` between zone and
qualified name):

```typescript
const proxy = Kinotic.serviceProxy('app.my-organization.my-application~com.example.OrderService')
const orders = await proxy.invoke('findOpenOrders', [customerId])
proxy.invokeStream('watchOrder', [orderId]).subscribe(update => render(update))
```

A proxy call reaches a service only while that service is running — which, for an
application's own services, means the project has been deployed. If calls fail to
resolve, check the deployment before debugging the address (`deploying` skill).

Generated entity repositories (entities-and-persistence skill) also work in the
browser after `Kinotic.use(PersistencePlugin)` — data access needs no bespoke backend
endpoints.

## Further reading

- Authentication: <https://kinotic.ai/apps/security/authentication>
- Service proxies: <https://kinotic.ai/apps/services/service-proxies>
- Platform auth model: <https://kinotic.ai/platform/system-security>
