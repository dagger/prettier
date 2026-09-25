# Prettier Dagger Module

Check and fix formatting with [Prettier](https://prettier.io).

## Requirements

Dagger engine `v1.0.0-beta.15` or later. That release is not out yet, so for
now this module only runs on a dev engine.

## Install

```sh
dagger install github.com/dagger/prettier
```

## Projects

Every directory holding a Prettier config file is a project. The config files
are `.prettierrc`, `.prettierrc.{json,json5,yaml,yml,toml,js,mjs,cjs,ts,mts,cts}`
and `prettier.config.{js,mjs,cjs,ts,mts,cts}`.

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

- **Inside a project's subdirectory:** only the enclosing project.
- **At a project root:** that project and the projects below it.
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
dagger check --prettier-project=packages/ui            # one project
dagger check prettier/projects/format-check --prettier-project=app
```

Flags from `dagger check --help`:

| Flag | Selects |
| --- | --- |
| `--prettier`, `--by-prettier` | checks from this module |
| `--format-check`, `--check-format-check` | checks named `format-check` |
| `--prettier-project PATH` | one project (repeatable) |
| `--prettier-projects` | every project |

The selected projects are checked concurrently. A failure lists every
project with issues, with Prettier's report for each.

## Fixing formatting

`format` runs `prettier --write` and returns the changes as a changeset.
`dagger call` cannot select an item from a collection yet, so use `project`
to look up the project that contains a path. The path is relative to your
working directory:

```sh
dagger call prettier project --path=. format            # the project you're in
dagger call prettier project --path=packages/ui format
```

The changes are rooted at your working directory. When you run from inside a
project's subdirectory, only files below that directory change.

`format` is not a `@generate` generator, so `dagger generate` does not run
it. A generator adds a staleness check to `dagger check`, which would run
Prettier a second time next to `format-check`.

## Dependencies

Prettier runs from the project directory. When there is a `package.json` at
or above the project, the nearest one's dependencies are installed with the
configured package manager, so the project's own Prettier version and plugins
are used. Without one, `npx` fetches the latest Prettier on demand, so a
standalone Prettier config works without a Node project.

The directory holding that `package.json`, or the project itself, is mounted
without `node_modules` and without files ignored by `.gitignore`.

## Settings

Set them with `dagger settings`, or in `dagger.toml`:

```sh
dagger settings prettier baseImageAddress node:22-alpine
```

```toml
[modules.prettier.settings]
baseImageAddress = "node:22-alpine"   # default: node:25-alpine; any image with node and npx
packageManager = "pnpm"               # default: npm; also yarn, pnpm or bun
```

pnpm is enabled through corepack. The default `node:25-alpine` image does not
include corepack, so pair pnpm with an image that does, such as
`node:22-alpine`.

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
