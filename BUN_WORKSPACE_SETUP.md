# Adding a New Bun Workspace Member to the Monorepo

Steps to make a subproject a Bun workspace member in the `@axhxrx` monorepo. This is separate from the Deno workspace (root `deno.jsonc`), which most projects are already part of via the bot scaffolding tool.

## Prerequisites

- The subproject directory already exists (e.g., scaffolded by `bot`)
- The subproject is already in the Deno workspace (`deno.jsonc`) if it's a Deno project
- Bun is installed

## Steps

### 1. Create `package.json` in the subproject

```json
{
  "name": "@axhxrx/{PROJECT_NAME}",
  "private": true,
  "type": "module",
  "exports": {
    ".": "./mod.ts"
  },
  "devDependencies": {
    "@types/bun": "latest"
  },
  "peerDependencies": {
    "typescript": "^5"
  }
}
```

Adjust `exports` if the project has multiple entry points (e.g., `"./*": "./*.ts"`).

### 2. Create `tsconfig.json` in the subproject

```json
{
  "compilerOptions": {
    "lib": ["ESNext"],
    "target": "ESNext",
    "module": "Preserve",
    "moduleDetection": "force",
    "allowJs": true,
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "verbatimModuleSyntax": true,
    "noEmit": true,
    "strict": true,
    "skipLibCheck": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noUnusedLocals": false,
    "noUnusedParameters": false,
    "noPropertyAccessFromIndexSignature": false,
    "baseUrl": ".",
    "types": ["bun-types"]
  },
  "include": ["**/*.ts", "**/*.tsx"],
  "exclude": ["node_modules", "**/*.test.ts"]
}
```

Add `"jsx": "react-jsx"` and `"jsxImportSource": "react"` to `compilerOptions` if the project uses React/JSX.

### 3. Create `.npmrc` in the subproject

```
@jsr:registry=https://npm.jsr.io
```

This allows importing JSR packages via the `npm:@jsr/...` protocol that Bun uses.

### 4. Add to root `package.json` workspaces

Edit `/package.json` (monorepo root) and add `"./{PROJECT_NAME}"` to the `"workspaces"` array, **in alphabetical order**.

### 5. Run `bun install`

From the monorepo root:

```bash
bun install
```

This updates `bun.lock` and hoists dependencies.

## Adding dependencies later

```bash
# Add a regular dependency
cd {PROJECT_NAME}
bun add some-package

# Add a workspace dependency (another monorepo project)
bun add @axhxrx/ops

# This creates a "workspace:*" reference in package.json automatically

# Add a JSR package (use the npm: bridge)
# In package.json, add manually:
#   "@axhxrx/some-jsr-pkg": "npm:@jsr/axhxrx__some-jsr-pkg"
# Then run bun install
```

## Quick reference: one-liner checklist

```bash
PROJECT=exfiltrator

# 1. Create package.json, tsconfig.json, .npmrc (see templates above)
# 2. Add "./$PROJECT" to root package.json workspaces array (alphabetical)
# 3. bun install
```

## Note: bot tool

The `bot` scaffolding tool (`deno run -A ./bot/main.ts`) can do most of this automatically for new projects via the `TYPESCRIPT_LIBRARY` template. It runs `UpdatePackageJsonWorkspace` and `UpdateDenoWorkspace` ops to register the project in both workspace configs. Use that for greenfield projects. This doc is for when you need to manually add Bun workspace support to an existing project.
