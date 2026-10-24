---
name: frontend
description: >
  How a frontend or Node client talks to a Kinotic application: how the UI resolves which
  server URL to connect to (same-origin vs an explicit server override, and the dev proxy
  that makes them equivalent), Kinotic.connect, the three VITE_KINOTIC_* build variables
  the deployment sets, browser sign-in through the gateway's login routes and the session
  cookie it establishes, BasicCredentialsResolver and BearerCredentialsResolver for Bun
  and Node clients, machine identities via environment credentials, OIDC login, creating
  application users, invoking published services and entity repositories from the UI, and
  the vite.config.ts a UI needs to bundle the domain package and its entity decorators. Use when building a
  Kinotic app frontend or SPA, working out what host or URL the UI should point at, wiring
  up login or authentication, connecting any client to a Kinotic server, or when a UI
  bundle fails to load with a SyntaxError or a blank page.
---

# Frontends and Clients

Every client — browser SPA, Node client, script — speaks the same STOMP-over-WebSocket
protocol to the Kinotic RPC gateway. There is no REST layer for application services;
the only REST endpoints are platform auth (`/api/auth/*`) and MCP (`/mcp`). Frontends
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

Answer this before writing any connect code — it decides whether the frontend needs
configuration at all, and it is the question the resolution ladder above does not settle
on its own.

**The env-var rung does not exist in a browser.** Core reads it as
`typeof process !== 'undefined' ? process.env : undefined`, and bundlers do not provide
`process` in browser builds (Vite exposes `import.meta.env` instead). So putting
`KINOTIC_SERVER_HOST` in a `.env` does nothing for an SPA — the ladder skips straight
from an explicit `server` option to `window.location`. That leaves exactly two shapes:

**Cross-origin — pass `server` explicitly.** This is the platform's path. A published UI
is served from its own site host, not the gateway, and the deployment sets
`VITE_KINOTIC_HOST`, `VITE_KINOTIC_PORT` and `VITE_KINOTIC_USE_SSL` for the build; `vite dev`
takes the same three from the app's `.env` (next section). Left alone, the client would open
a socket against the page's own origin, so `connect()` is handed a `server` built from those
variables, and the REST calls are built from the same values and sent with
`credentials: 'include'` so the session cookie travels. Recipe 1's `serverOptions()` /
`apiUrl()` are that code, and the platform's own consoles connect the same way. Nothing is
hand-set, and the code does not change between local and published.

**Same-origin — configure nothing.** The alternative, for a UI served from the gateway's own
webroot or from a dev server that proxies to it. `window.location` is already the right
answer, so a bare `Kinotic.connect()` and bare REST paths
(`fetch('/api/auth/me', { credentials: 'include' })`) are correct, and the three variables
stay unset. A Vite dev proxy has to forward both the REST and the WebSocket paths to the
app server under the application's API host (the app server identifies the application by
that hostname), and `/v1` needs `ws: true` — that is the broker path the STOMP client opens:

```typescript
// vite.config.ts
const APP_API_HOST = 'acme--orders.localhost:58505'   // <organizationId>--<applicationId> under the app server's API base URL

proxy: {
    '/api': { target: 'http://localhost:58505', headers: { host: APP_API_HOST } },
    '/v1':  { target: 'http://localhost:58505', headers: { host: APP_API_HOST }, ws: true }
}
```

Cross-origin does not push the SPA off the session cookie. The cookie is `SameSite=Lax`,
which is sent whenever the API and the site share a site — as the application's API host
and its sites, both under `kinotic.ai`, do in production. A published UI therefore uses the
cookie flow (Recipe 1), cross-origin or not.

## The UI build contract

Every UI package under `packages/ui` that declares a `build` script is built during the
deployment with `bun run build` and published to its own site. The build is handed its
application's API host, `<organizationId>--<applicationId>` under the platform's application
API domain, as exactly three variables, and Vite exposes them to the page on its own:

| Variable | Value | What the UI does with it |
|---|---|---|
| `VITE_KINOTIC_HOST` | e.g. `acme--orders.apps-api.kinotic.ai` | The host the UI connects to Kinotic on from the browser |
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

For `vite dev` on a developer's machine, the UI reaches a local app server through the dev
server itself: the three variables in the app's `.env.development` point at the dev server,
and the dev server proxies `/api` and `/v1` to the app server under the application's API
host (the proxy in "How the UI knows where the server is" above), so the page and its API
share an origin and a session cookie:

```
VITE_KINOTIC_HOST=localhost
VITE_KINOTIC_PORT=5173
VITE_KINOTIC_USE_SSL=false
```

The app server's development profile admits a page on any `http://localhost` port as a UI of
every application. The code does not change between local and published. The gateway sets
the session cookie `Secure` (it is a `__Host-` cookie), so signing in over plain
`http://localhost` relies on the browser treating localhost as a secure context: Chrome and
Firefox do; Safari does not store the cookie, and the upgrade after a `204` login arrives
unauthenticated.

Kinotic projects run on Bun, whose built-in WebSocket accepts the upgrade headers
credentials travel on. Only a **Node** process sending credentials must first call
`ensureNodeWebSocket()` from `@kinotic-ai/core/node`, which swaps in `ws` because Node's
own WebSocket ignores a headers option. It is a no-op under Bun and unnecessary in the
browser — do not add it to a Bun entry point.

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
it produces an unauthenticated upgrade and `Max number of reconnection attempts reached`.
In a browser the session cookie is the credential, and the gateway's login route
establishes it.

- The app server identifies the application from the request's hostname, the API host the
  UI connects to (`<organizationId>--<applicationId>.<apps API domain>`), so the login routes
  carry no organization or application id in their path.
- The app-scope password login is `POST /api/auth/app/login` with `{ email, password }`. It
  answers `204` and sets the cookie, `401` with `{ "error": "Invalid credentials" }` for a
  wrong email or password, or `403` when the page that sent it is not one of the
  application's UIs (its published sites, or `http://localhost` under the app server's
  development profile). The user must exist in **that application's** user base (portal,
  Application → **Users**); the route does not authenticate organization members.
- `GET /api/auth/me` answers `204` when the browser holds a live session and `401`
  otherwise. `POST /api/auth/logout` ends the session.
- Every request carries `credentials: 'include'`, and `Kinotic.connect` takes only the
  server: the STOMP upgrade rides the cookie.

```ts
// src/kinotic/session.ts
//
// Sign-in for a Kinotic app UI. The deploy builds the UI with three variables naming the
// platform it was published against, and Vite exposes them to the page on its own:
//   VITE_KINOTIC_HOST     e.g. acme--orders.apps-api.kinotic.ai, the application's API host
//   VITE_KINOTIC_PORT     e.g. 443
//   VITE_KINOTIC_USE_SSL  "true" | "false"
// For `vite dev` on a laptop, put the same three in the app's .env.development, pointing at
// the dev server (localhost, 5173, false), which proxies to the local app server.
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

## Application users and OIDC login

An application's users are its own user base, apart from the organization's members, and
an identity at a provider never grants access by itself. A user comes from one of:

- **An invitation.** An organization admin invites the email from the portal's
  Application → **Users** page; the invitee accepts the emailed link with a password and
  from then on signs in through Recipe 1's `login()`.
- **First sign-in through the organization's own provider.** An
  `ApplicationOidcConfiguration` with `createUserOnSignIn` on turns an identity it verifies
  that no user is bound to into a user, from the provider's verified email and name — only
  while the application's `tenantManagement` is `SYSTEM` (the default is
  `TENANT_ADMINISTRATORS`), which gives each such user a tenant of its own. A
  `TenantOidcConfiguration` with `createUserOnSignIn` does the same into its tenant, at
  that tenant's login.

The system's shared Google and Microsoft registrations (named in the application's
`socialProviders`) create no users: they sign in only a user already bound to that
identity. An identity with no user and nothing to create one ends at `no_account`; one
whose email the application already holds under another method ends at `account_exists`.

The login page offers every method at once — the password form (Recipe 1's `login()`)
and a "Continue with <name>" button per provider — and never asks the server about an
email first; there is no lookup route.

1. To see what the login offers, call the MCP tool whose description begins with
   `Application Service Get Login Providers`, with `{"applicationId": "..."}` (Kinotic
   MCP tool names are opaque hashes; match on the description). It answers
   `[{configurationId, name, provider, type}]` in presentation order: the organization's
   own `APPLICATION` configurations, then the `APP_SOCIAL` registrations the application
   names. An empty list means the login is password only — tell the user rather than
   writing provider buttons nobody can use.
2. At runtime the page reads the same list from its API host, with
   `credentials: 'include'` as in Recipe 1:
   - `GET /api/auth/app/login/providers` answers `[{ id, name, provider }]` — `id` is the
     configuration id, `provider` the kind's key (`google`, `azure-ad`, `okta`, …) for
     branding.
   - A button posts to `POST /api/auth/app/login/oidc/start/<id>` (no body) and answers
     `{ "redirect": "…" }`; send the browser there. `400` for a configuration the login
     does not offer, `403` from a page that is not one of the application's UIs. The flow's
     state lives in the API host's session, so the start needs `credentials: 'include'`.
   - The provider returns to the API host (a shared registration via the app server's
     shared host), which sets the session cookie and sends the browser to `/` on the site
     that started the sign-in. Recipe 1's `resume()` then sees the live session and opens
     the connection.
   - A failed sign-in lands on `<site>/login?error=<code>` (`no_account`,
     `account_disabled`, `account_exists`, `email_not_verified`, `handoff_invalid`, …), so
     the UI needs a `/login` route that shows the error and the methods again. One whose
     flow session is lost lands on the application's primary UI's `/login` instead.
3. A page served at one of a tenant's sites signs in to that tenant with the same calls:
   the list adds the tenant's own providers ahead of the application's, and
   `/api/auth/app/login` admits only the tenant's users. A page that is not a tenant site
   (a UI on localhost, say) names the tenant with `?tenantId=` on the providers `GET` and
   the start, and `tenantId` in the password body; `404` for a tenant the application
   lacks. The MCP tool lists the application's own login only.

```ts
// src/kinotic/session.ts, beside Recipe 1's login()

/** A provider the login offers, as GET /api/auth/app/login/providers lists it. */
export interface LoginProvider { id: string; name: string; provider: string }

/** The providers to render a "Continue with <name>" button for, in order. */
export async function loginProviders(): Promise<LoginProvider[]> {
    const res = await fetch(apiUrl('/api/auth/app/login/providers'), { credentials: 'include' })
    if (!res.ok) {
        throw new Error(await errorMessage(res, 'Could not load sign-in providers'))
    }
    return await res.json() as LoginProvider[]
}

/** Starts a sign-in through one provider; the browser leaves the page. */
export async function continueWith(provider: LoginProvider): Promise<void> {
    const res = await fetch(apiUrl(`/api/auth/app/login/oidc/start/${encodeURIComponent(provider.id)}`), {
        method: 'POST',
        credentials: 'include',            // the flow's state rides the API host's session
    })
    if (!res.ok) {
        throw new Error(await errorMessage(res, 'Could not start sign-in'))
    }
    const { redirect } = await res.json() as { redirect: string }
    window.location.assign(redirect)
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
