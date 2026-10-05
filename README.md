# Prettier Dagger Module

Check and fix formatting with [Prettier](https://prettier.io).

## Requirements

Requires Dagger v1.0.0-beta.15 or later.

## Install

```sh
dagger install github.com/dagger/prettier
```

## Projects

Every directory holding a Prettier config file is a project. The config files
are `.prettierrc`, `.prettierrc.{json,json5,yaml,yml,toml,js,mjs,cjs,ts,mts,cts}`
and `prettier.config.{js,mjs,cjs,ts,mts,cts}`. A TypeScript config loads
only when Node can strip its types, which Prettier needs `"type": "module"`
in the nearest `package.json` for.

Projects form a collection keyed by directory, relative to the workspace root:

```sh
dagger list prettier-projects -a
```

A project covers its directory and everything below it, except nested
projects. Those are checked from their own directory, so their own config and
`.prettierignore` apply and no file is checked twice.

### Discovery

Projects are found by config file name alone, in one walk of the workspace
that skips `node_modules`. Listing runs no container and no Prettier.
Because only file names are checked:

- A config that lives only in the `prettier` key of `package.json` is not
  found. Add a `.prettierrc` to make that directory a project.
- A config file inside a directory ignored by `.gitignore` (other than
  `node_modules`) still makes that directory a project.

### Your working directory selects projects

Which projects you see depends on where you run `dagger`:

- **Inside a project's subdirectory:** the enclosing project, plus any
  projects nested below that directory.
- **At a project root:** that project and the projects below it, never the
  ones above it.
- **In a directory that belongs to no project:** the projects below it.

Given `app/.prettierrc`, `packages/ui/.prettierrc` and
`packages/ui/legacy/.prettierrc`:

```sh
cd app/src && dagger check           # checks app
cd packages/ui && dagger check       # checks packages/ui and packages/ui/legacy
cd packages && dagger check          # checks packages/ui and packages/ui/legacy
```

## Checks

| Check | Address | Description |
| --- | --- | --- |
| `format-check` | `prettier/projects/format-check` | Check formatting of this project (`prettier --check`). |

```sh
dagger check -l --all --prettier                       # one line per project
dagger check --prettier                                # every project in view
dagger check --check format-check                      # checks named format-check, any module
dagger check --prettier-project=packages/ui            # one project
dagger check prettier/projects/format-check --prettier-project=app
```

Run checks with `dagger check`, including in CI. `dagger call` on a check
function such as `format-check` does not fail the command when the check
fails.

Flags from `dagger check --help`:

| Flag | Selects |
| --- | --- |
| `--prettier`, `--by-prettier` | checks from this module |
| `--check format-check` | checks named `format-check` |
| `--prettier-project PATH` | one project (repeatable) |
| `--prettier-projects` | every project |

The selected projects are checked concurrently. Each line of Prettier's
output starts with its project, e.g. `[packages/ui] [warn] src/a.ts`. A
failure names every failing project and the step that failed, for example:

```
Prettier failed in 3 project(s):
- a: install failed (npm install, exit 1):
  npm error 404 Not Found - GET https://registry.npmjs.org/...
- b: prettier --check failed (exit 1):
  [b] [warn] index.js
- c: prettier is not installed: add it to the devDependencies of c/package.json (...)
```

## Fixing formatting

`format` runs `prettier --write` and returns the changes as a changeset.
`dagger call` cannot select an item from a collection yet, so use `project`
to look up the project that contains a path. The path is relative to your
working directory:

```sh
dagger call -y prettier project --path=. format            # the project you're in
dagger call -y prettier project --path=packages/ui format
```

`dagger call` asks before applying the changes. Without a terminal to ask in
(scripts, CI, agents) it fails unless you pass `-y`. A project can also be
picked by its key in the Dagger shell. `export .` writes the changes:

```sh
dagger -c 'prettier | projects | get packages/ui | format | export .'
```

The changes are rooted at your working directory. When you run from inside a
project's subdirectory, only files below that directory change.

`format` is not a `@generate` generator, so `dagger generate` does not run
it. A generator adds a staleness check to `dagger check`, which would run
Prettier a second time next to `format-check`.

## Dependencies

Prettier runs from the project directory.

**Without a `package.json`** at or above the project, `npx` fetches the latest
Prettier, so a standalone Prettier config works without a Node project.

**With one**, dependencies are installed and the project's own Prettier runs:
the nearest `node_modules/.bin/prettier` between the project and the install
root, or yarn's under Plug'n'Play. If there is none, the check fails with
`prettier is not installed: add it to the devDependencies of ...` rather than
fetching a different version.

- **Install root.** The nearest workspace root at or above the project: a
  directory with `pnpm-workspace.yaml`, or a `package.json` with
  `"workspaces"`. Failing that, the nearest lockfile's directory, then the
  nearest `package.json`'s. A package inside a monorepo therefore installs
  with the whole workspace, so `workspace:` and `catalog:` dependencies
  resolve.
- **Package manager.** The `packageManager` setting if set. Otherwise the
  `packageManager` field of the install root's `package.json`, then its
  lockfile (`pnpm-lock.yaml` or `pnpm-workspace.yaml`: pnpm, `yarn.lock`:
  yarn, `bun.lock`/`bun.lockb`: bun), then npm. pnpm and yarn run through
  corepack, installed when the image lacks it, at the version the
  `packageManager` field pins.
- **Caching.** The install sees only what it reads: every `package.json`,
  lockfiles, `pnpm-workspace.yaml`, `.npmrc`, `.yarnrc*`, `.yarn/{releases,plugins,patches}`,
  `.pnpmfile.cjs`, `bunfig.toml`, `patches/`, and the directories that
  `file:`, `link:` and `portal:` dependencies point at (package managers copy
  those). The rest of the source is laid over the result, so editing a
  source file does not reinstall. If a `package.json` can't be read, or pnpm
  injects a workspace package (`dependenciesMeta.*.injected`), the install
  gets the full source instead. Package manager caches, the pnpm store
  (passed with `--store-dir`) and corepack live on cache volumes.
- **Less noise.** Browser downloads (Playwright, Puppeteer, Cypress) and git
  hook installers (husky, simple-git-hooks) are switched off. Install scripts
  still run. They see only the install inputs, so a script that needs source
  files fails; pass `--ignore-scripts` through `installFlags`.
- **Install output stays out.** Prettier skips what the install writes into
  the tree (`node_modules`, and yarn Plug'n'Play's `.pnp.cjs`,
  `.pnp.loader.mjs` and `.yarn/`), and `format` never returns those files.

The install root, or the project itself, is mounted without `node_modules`
and without files ignored by any `.gitignore` in the tree. Prettier run
locally reads only the `.gitignore` in its working directory. So a file
ignored by a parent directory's `.gitignore` (e.g. a root `dist/` ignoring
`packages/ui/dist/`) is checked by a local `prettier --check .` but not by
`dagger check`.

## Settings

Set them with `dagger settings`, or in `dagger.toml`:

```sh
dagger settings prettier packageManager pnpm
dagger settings -u prettier packageManager      # back to the default
```

```toml
[modules.prettier.settings]
baseImageAddress = "node:22-alpine"      # default: node:25-alpine; any image with node and npm
packageManager = "pnpm"                  # default: "" (detect); npm, yarn, pnpm or bun
installFlags = ["--ignore-scripts"]      # default: []; appended to the install command
environment = ["NODE_OPTIONS=--max-old-space-size=4096"]  # default: []; KEY=VALUE for Prettier
```

## Use from another module

`projects(ws)` returns the collection. `keys`, `get(key:)`, `subset(keys:)`
and `batch` work on it:

```dang
let projects = prettier.projects(ws)
projects.keys                                             # ["app", "packages/ui", ...]
projects.get(key: "app").format(ws)                       # Changeset for one project
projects.subset(keys: ["app", "packages/ui"]).batch.format(ws)
prettier.project(ws, "packages/ui/src").path              # "packages/ui"
```

A check called through a dependency returns a `Check` that has not run yet.
Run it and raise on failure:

```dang
let run(check: Check!): Void {
  if (check.pass == false) {
    raise check.error.message ?? "check failed"
  }
  null
}

run(prettier.projects(ws).batch.formatCheck(ws))
run(prettier.projects(ws).get(key: "app").formatCheck(ws))
```

The end-to-end tests in `.dagger/modules/e2e` exercise all of this.
