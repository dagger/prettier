# Prettier Dagger Toolchain

## Installation

```
dagger toolchain install github.com/dagger/prettier
```

## Functions

- `check`: Check that if the files are formatted.
- `write`: Rewrite all processed files in place.

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
          "argument": "source",
          "defaultPath": "/src",         # default: /; custom default path
          "ignore": ["**/node_modules"]  # custom ignore filter
        },
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
