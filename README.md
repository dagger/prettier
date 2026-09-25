# Prettier Dagger Toolchain

Check and fix formatting with [Prettier](https://prettier.io).

Requires Dagger engine `v1.0.0-beta.15` or later.

## Installation

```
dagger toolchain install github.com/dagger/prettier
```

## Projects

Every directory holding a Prettier config file is a project: `.prettierrc`,
`.prettierrc.{json,json5,yaml,yml,toml,js,mjs,cjs,ts,mts,cts}` or
`prettier.config.{js,mjs,cjs,ts,mts,cts}`. `node_modules` is skipped.

Projects form a collection keyed by their directory, relative to the workspace
root. From a subdirectory, the collection holds the projects at or below it,
plus the project enclosing it.

```
dagger list prettier-projects -a
```

A project covers its directory and everything below it, except nested
projects: those are checked on their own, from their own directory, so their
own config and `.prettierignore` apply and nothing is checked twice.

## Checks

| Check | Address | Description |
| --- | --- | --- |
| `format-check` | `prettier/projects/format-check` | Check formatting of this project (`prettier --check`). |

```
dagger check --prettier                                    # every project
dagger check --prettier-project=packages/web               # one project
dagger check -l --all --prettier                           # list them
```

Selected projects are checked concurrently, and a failure lists every project
with issues along with Prettier's report.

## Fixing formatting

`format` runs `prettier --write` and returns the changes. Look up the project
containing a path, relative to your working directory:

```
dagger call prettier project --path=. format
dagger call prettier project --path=packages/web format
```

From other modules, format a project or a selection:

```
prettier.projects(ws).get(key: "packages/web").format(ws)
prettier.projects(ws).batch.format(ws)
```

The changes are rooted at your working directory. An enclosing project only
changes files below it.

`format` is not a `@generate` generator: a generator adds a staleness check
to `dagger check`, which would run Prettier a second time next to
`format-check`.

## Dependencies

Prettier runs from the project directory. When there is a `package.json` at
or above the project, the nearest one's dependencies are installed with the
configured package manager, so the project's own Prettier version and plugins
are used. Otherwise `npx` fetches the latest Prettier on demand, so a
standalone Prettier config works without a Node project.

Files ignored by `.gitignore` are not mounted.

## Customization

Settings go in your workspace `dagger.toml`:

```toml
[modules.prettier]
source = "github.com/dagger/prettier@main"
settings.baseImageAddress = "node:22"   # default: node:25-alpine; any image with node and npx
settings.packageManager = "yarn"        # default: npm; also yarn, pnpm or bun
```
