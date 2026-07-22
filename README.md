# Prettier Dagger Toolchain

## Installation

```
dagger toolchain install github.com/dagger/prettier
```

## Functions

- `check`: Check that if the files are formatted.
- `write`: Rewrite all processed files in place.

## Working directory awareness

Functions run from your current working directory within the workspace. The
whole workspace is mounted — so shared configuration like a root
`.prettierrc` still resolves — but prettier itself runs from the directory
you invoke `dagger` from. Run from the workspace root to cover everything,
or from a subdirectory to scope `check` and `write` to that subtree.

## Customization

The toolchain can be customized in your `dagger.json` to meet your needs:

```json
{
  "name": "my-module",
  "engineVersion": "...",
  "toolchains": [
    {
      "name": "prettier",
      "source": "github.com/dagger/prettier@main",
      "pin": "...",
      "customizations": [
        {
          "argument": "baseImageAddress",
          "default": "node:22"       # default: node:25-alpine; use any container image 
        },
        {
          "argument": "packageManager",
          "default": "yarn"          # default: npm; alternatively use yarn, pnpm, or bun
        }
      ]
    }
  ]
}
```
