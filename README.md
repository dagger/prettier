# Prettier Dagger Module

Check and fix formatting with [Prettier](https://prettier.io).

## Requirements

Requires Dagger v1.0.0-beta.22 or later.

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
that skips `node_modules`. Listing (`dagger check -l`, `dagger list`) runs
no container: it installs nothing and runs no Prettier, so it works even
when a `package.json` or a config is broken. A config Prettier cannot read
fails only its own project's check, naming the file. Because only file
names are checked:

- A config that lives only in the `prettier` key of `package.json` is not
  found. Add a `.prettierrc` to make that directory a project.
- A config file inside a directory ignored by `.gitignore` (other than
  `node_modules`), or a config file that is itself ignored, still makes
  that directory a project. Prettier runs on the source without ignored
  files, so it would check nothing there, and that project's
  `format-check` fails with `Prettier did not run: ...` rather than
  passing.

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

### One check per project

Each selected project has its own `format-check`, and the selected projects
are checked concurrently. A project is one Prettier run: one
`prettier --check .` from the project directory, with its own config,
`.prettierignore` and Prettier, so the project is both the unit you select
and the unit that runs. There is no finer check: Prettier names the files
it found unformatted but not the ones it checked and passed, and which
files it checks depends on its ignore files, overrides and plugins, which
listing checks cannot evaluate without running it.

A failure fails only its own project's check and names the step that
failed. Each line of Prettier's output starts with its project, e.g.
`[packages/ui] [warn] src/a.ts`, and files Prettier names by absolute path
are given as workspace paths. A message keeps the last 32 KiB of each
output stream, marked when cut, however much the run printed:

```
install failed (npm install, exit 1):
npm error 404 Not Found - GET https://registry.npmjs.org/...

prettier --check failed (exit 1):
[b] [warn] index.js

prettier is not installed: add it to the devDependencies of c/package.json (...)

Prettier did not run: the config file that makes d a project is ignored by a .gitignore, ...
```

- **Shared install.** Projects with the same install root (see
  [Dependencies](#dependencies)) run the same install, so it runs once for
  all of them, and a failed install fails every one of them.
- **Nested projects.** A nested project's files are checked only by its own
  check, so an unformatted file there fails the nested project, never its
  parent, and selecting the parent alone does not check it.
- **Not run.** A selected project Prettier would never see fails rather
  than passing with nothing checked (see [Discovery](#discovery)).

The collection-level `formatCheck` is a plain helper, not a check (see
[Use from another module](#use-from-another-module)), so `dagger check`
never checks a project twice. There is no `.batch` namespace.

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

**Without a `package.json`** at or above the project, a pinned Prettier runs
(the `version` setting, 3.9.10 by default), so a standalone Prettier config
works without a Node project. It is installed once for the workspace in a
pinned image and mounted into the project's container; nothing is fetched
when a check runs.

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
version = "3.9.10"                       # default: 3.9.10; exact Prettier version for projects without a package.json
```

## Use from another module

`projects(ws)` returns the collection. `keys`, `get(key:)` and
`subset(keys:)` work on it, and so do two helpers over its projects,
`format` and `formatCheck`:

```dang
let projects = prettier.projects(ws)
projects.keys                                             # ["app", "packages/ui", ...]
projects.get(key: "app").format(ws)                       # Changeset for one project
projects.subset(keys: ["app", "packages/ui"]).format(ws)  # one Changeset for both
prettier.project(ws, "packages/ui/src").path              # "packages/ui"
```

A project's `formatCheck` is a check, so called through a dependency it
returns a `Check` that has not run yet. Run it and raise on failure:

```dang
let run(check: Check!): Void {
  if (check.pass == false) {
    raise check.error.message ?? "check failed"
  }
  null
}

run(prettier.projects(ws).get(key: "app").formatCheck(ws))
```

The collection's `formatCheck` is a plain function: it checks the projects
concurrently and raises one error naming every failing project and its
step, e.g. `Prettier failed in 2 project(s):\n- a: ...\n- b: ...`:

```dang
prettier.projects(ws).formatCheck(ws)
```

These calls do not see a `dagger check` selection: a direct call gets the
whole collection, or the `subset` you ask for.

The end-to-end tests in `.dagger/modules/e2e` exercise all of this. Its
`selected-*` checks run `dagger check` through the CLI on fixtures, since
only the CLI applies the selection.
