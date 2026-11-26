# Expected Kinotic project scaffold

The repository provisioned by Kinotic OS is rendered from the Kinotic project template
(`kinotic-ai/kinotic-tpl-isomorphic-ts`, a Bun workspace mono repo). **The cloned
repository is the source of truth** — if it differs from this document, trust the clone,
report the difference to the user, and do not "fix" the clone to match this document.

## Layout

```
<repo root>/
├── README.md
├── package.json              # Bun workspace root
├── tsconfig.json             # `bun run generate` reads compilerOptions from here
├── tsconfig.base.json
├── bunup.config.ts           # bunup workspace: the domain package plus every microservice it finds
├── .config/
│   ├── kinotic.config.ts     # Kinotic project configuration (see below)
│   ├── templates/microservice/  # starter microservice, copied into packages/microservices when needed
│   ├── agent-plugin.json     # plugin version that last changed the project (written by the agent; see plugin-version.md)
│   └── c3/                   # generated schemas (written by `bun run generate`; committed)
├── .kinotic/                 # local incremental-generation cache (gitignored — never commit)
├── migrations/               # V<N>__<description>.sql files (absent until first used)
└── packages/
    ├── domain/               # entity model + generated repositories
    │   ├── package.json
    │   ├── tsconfig.json
    │   ├── index.ts          # package entry — export entities/repositories here
    │   ├── model/            # @Entity classes go here
    │   └── repositories/     # `bun run generate` writes repository classes here
    ├── microservices/        # empty until the application needs one; each package here gets its own workload
    └── ui/                   # UI packages; each one with a build script is published as a site
```

`packages/domain/model`, `packages/domain/repositories`, `packages/microservices`, and
`packages/ui` ship as empty directories holding a `.gitkeep`.

## Root `package.json`

```json
{
    "name": "<project slug>",
    "private": true,
    "scripts": {
        "build": "bunup",
        "dev": "bunup --watch",
        "generate": "kinotic generate",
        "type-check": "bun run --filter '*' type-check"
    },
    "devDependencies": {
        "@kinotic-ai/kinotic-cli": "<pinned version>",
        "@kinotic-ai/management-api": "<pinned version>",
        "@types/bun": "<pinned version>",
        "bunup": "<pinned version>",
        "typescript": "<pinned version>"
    },
    "catalog": {
        "@kinotic-ai/core": "<pinned version>",
        "@kinotic-ai/management-api": "<pinned version>",
        "@kinotic-ai/persistence": "<pinned version>"
    },
    "workspaces": ["packages/*", "packages/microservices/*", "packages/ui/*"],
    "type": "module"
}
```

Workspace packages reference the catalog (`"@kinotic-ai/core": "catalog:"`) so the
Kinotic SDK version is pinned once at the root. The versions are not written by the
template — Kinotic OS injects the ones the server ships with when it renders the repo, so
a fresh project always matches its server. Leave them as the template wrote them: never
bump an `@kinotic-ai/*` version past the one the template wrote, and never edit the
ranges by hand.

The current SDK line is `@kinotic-ai/core` and `@kinotic-ai/persistence` on
`5.0.0-beta.x`, `@kinotic-ai/management-api` on `5.0.0-beta.x`, and the CLI on `5.2.x`.
After `bun install`, check what actually resolved (`bun pm ls | grep @kinotic-ai`);
anything on the 4.x line, or any `@kinotic-ai/os-api` (the package was renamed to
`@kinotic-ai/management-api`), means the pins are stale — report that to the user rather
than working around it.

The Kinotic CLI ships as a project dependency with a `generate` script wired in
`package.json` — repository classes are generated with `bun run generate`, entirely
locally. Server synchronization happens inside the deployment (`kinotic sync` runs in the
build VM on every push); the globally-installed CLI (`kinotic login`, `kinotic sync`) is a
human-operator tool that the coding agent does not use.

## `.config/kinotic.config.ts`

A TypeScript module default-exporting a `KinoticProjectConfig`:

```typescript
import type { KinoticProjectConfig } from '@kinotic-ai/management-api'

const config: KinoticProjectConfig = {
  organizationId: "<organization id>",
  applicationId: "<application id>",
  projectId: "<project id>",
  entitiesPaths: [
    {
      path: "packages/domain/model",
      repositoryPath: "packages/domain/repositories",
      mirrorFolderStructure: true
    }
  ],
  fileExtensionForImports: ".js",
  validate: false
}

export default config
```

- `organizationId` and `applicationId` must match the Application created through the
  MCP tools (Step 1 of the workflow), and `projectId` the Project created in Step 2
  (`<application id>-main`). If they differ, synchronization targets the wrong
  application or fails because the project does not exist — surface this to the user
  before pushing.
- `entitiesPaths[].path` is where `@Entity` classes are discovered;
  `repositoryPath` is where repository classes are generated. Trust the values in the
  cloned config over the ones shown here.
- Optional fields the template leaves out: `name`, `description`, `servicesPaths` (the
  directories to search for `@Publish` classes) and `generatedPath` (where the
  `ServiceDefinitions.ts` module describing those services is written, and the fallback
  repository output for plain-string `entitiesPaths` entries). Setting `servicesPaths`
  without `generatedPath` fails generation; see the `services` skill for what the module
  does.

## Adding a microservice

The template ships no microservice: entities and their generated repositories need none,
so a project adds one only when the application needs code of its own on the server (the
`services` skill says when). First check that the application has no microservice yet, in
this project or another of its projects; if it has one, add the new services to it
instead. On Kinotic Cloud's beta an application can deploy only one microservice across
all of its projects, and a push that would give it a second fails its deployment.

The project carries a starter microservice at `.config/templates/microservice/`: a complete
package (`package.json`, `tsconfig.json`, and `src/main.ts`, **the file the platform runs**)
rendered for this project. It sits outside `packages/microservices/`, so it is never deployed
itself. The project's own `README.md` describes the same steps; trust it over this document if
they differ. To add a microservice named `<name>`:

```bash
cp -r .config/templates/microservice packages/microservices/<name>
```

In `packages/microservices/<name>/package.json`, change `name` from
`@<project slug>/microservice` to `@<project slug>/<name>`; its unscoped part is the
microservice's identity (lowercase letters, digits and interior dashes). Instantiate the
`@Publish` services in `src/main.ts` where its comment marks the spot. Leave
`.config/templates/microservice/` itself unchanged.

A project created before the template carried the starter has no `.config/templates/`. Take
the same three files from `.config/templates/microservice/` of the template repository,
`kinotic-ai/kinotic-tpl-isomorphic-ts`, replacing `{{ projectSlug }}` in `package.json.liquid`
with the slug in the project's root `package.json` `name`, and saving it as `package.json`.

`bunup.config.ts` finds every package directly under `packages/microservices` on its own, so a
new microservice needs no entry there. A project created before it did lists its packages by
hand: there, add an entry for the new microservice to the array passed to `defineWorkspace`,
after the `domain` entry, where every entry but the last ends with a comma:

```ts
{
    name: '<name>',
    root: 'packages/microservices/<name>',
    config: {
        entry: 'src/main.ts',
        dts: false
    }
}
```

Then run `bun install` so the workspace links it, and `bun run type-check`. The next push
starts its VM; see the `deploying` skill.

## The domain package

`packages/domain/index.ts` is the barrel other packages import through; the template
ships it as `export {}`. An entity or repository is only importable once it is exported
here:

```typescript
export * from './model/Person.js'
export * from './repositories/PersonRepository.js'
```

Its `package.json` declares `"sideEffects": false`, so a UI bundle that uses entities only
as types drops the decorated entity classes. Its `exports` map lists the source conditions (`development`, `bun`) ahead
of the `dist` fallbacks, so inside the repository the package resolves to its TypeScript
source and type-checks and runs without a prior build. Consumers installing it from a
registry get the built `dist/` output instead. `bunup.config.ts` keeps that map in sync
and must be edited by hand if a new workspace package is added.

## Adding a UI package

The template ships `packages/ui/` empty, so a UI is created from scratch. Each one is its
own workspace package under `packages/ui/` — the root workspace globs cover exactly that
depth. Run `bun install` so the workspace links it; a UI builds with its own `build` script,
not through `bunup.config.ts`.

Two things make it a publishable UI rather than a library:

1. Its `package.json` declares a **`build` script** and a `name` whose unscoped part is a
   valid label (lowercase letters, digits, interior dashes, no `--`), unique among the
   application's UIs across all its projects, and short enough that
   `<applicationId>--<uiName>` fits in 37 characters.
2. Its build writes **`dist/index.html`** — a build that leaves none fails the deployment,
   naming the UI.

A Vite project needs no `vite.config.ts` changes to be publishable: the publish uploads
`dist` under the site root as it is, and the deployment hands the build the three variables
`VITE_KINOTIC_HOST`, `VITE_KINOTIC_PORT` and `VITE_KINOTIC_USE_SSL`, which Vite exposes to
the page on its own. A UI that imports the domain package aliases it to its TypeScript source, since the
deployment does not build the domain package, and compiles the entity decorators, which
browsers cannot parse; the `frontend` skill has the `vite.config.ts`. Any framework that builds to static assets works. The full contract,
the ambient typing for those variables, and the browser sign-in module are in the
`frontend` skill.

```jsonc
// tsconfig.json — three levels up to the base config
{
    "extends": "../../../tsconfig.base.json",
    "files": [],
    "include": ["**/*"],
    "exclude": ["node_modules", "dist", "bin"]
}
```

Production single-file executables are built from a package directory with
`bun build --compile src/main.ts --outfile bin/<name>` — use `bun build` directly, not
bunup's `compile` option, which drops an import binding when bundling `@kinotic-ai/core`
and produces a binary that crashes on startup.

## Verification checklist

1. `package.json` at the root declares the workspace globs shown above and the
   `@kinotic-ai/*` catalog entries.
2. `.config/kinotic.config.ts` exists, default-exports a config, and its
   `organizationId`/`applicationId` match the created Application.
3. The directories named by `entitiesPaths` exist.
4. `bun install` succeeds and resolves `@kinotic-ai/*` on the current 5.x line, with no
   `@kinotic-ai/os-api` anywhere.
5. `bun run type-check` succeeds.
