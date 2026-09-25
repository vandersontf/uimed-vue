# AGENTS.md

## Project

`@nexdom/uimed-vue` is a Vue 3 UI component library for NEXDOM applications, built on top of [Vuetify](https://vuetifyjs.com/) and [Material Design 3](https://m3.material.io/). It's published to npm and documented via VitePress at https://nexdom-healthtech.github.io/uimed-vue/.

Tooling is built around [Vite+](https://github.com/voidzero-dev/vite-plus) (`vp`/`vpr`/`vpx` CLIs), not raw `vite`/`vitest`/`eslint` invocations.

## Using this library in an app

These steps are for developers _consuming_ `@nexdom/uimed-vue` in a Vue application (not for contributing to this repo — see the rest of this document for that).

### Install

```bash
vp add @nexdom/uimed-vue
# But, if you're not using Vite+ yet...
npm i @nexdom/uimed-vue
# Or
pnpm add @nexdom/uimed-vue
# Or
yarn add @nexdom/uimed-vue
```

Peer dependencies (`@fontsource/roboto`, `@mdi/font`, `@nexdom/shared`, `resize-observer-polyfill`, `vite-plugin-vuetify`, `vite-plus`, `vue`, `vue-router`, `vuetify`) must be installed alongside it — most package managers install these automatically, but confirm versions match what's declared in `package.json`'s `peerDependencies`.

### Setup

Add the Vite plugin:

```ts
// vite.config.ts
import { vitePluginUimed } from "@nexdom/uimed-vue/plugins";

// ...

plugins: [vue(), vitePluginUimed()];

// ...
```

Register the plugin in the Vue app:

```ts
// main.js or main.ts
import { createApp } from "vue";
import { createUimed } from "@nexdom/uimed-vue";

import App from "./App.vue";

const uimed = createUimed();

createApp(App).use(uimed).mount("#app");
```

### Usage

Place the `UMain` component at the top of `App.vue`, then add other components as needed:

```vue
<!-- App.vue -->
<template>
  <u-main>
    <!-- ... -->
  </u-main>
</template>

<script setup lang="ts">
import { UMain } from "@nexdom/uimed-vue/components";
</script>
```

Never write CSS, classes or any kind of styling. Always use component props.

The library doesn't export prop types. Derive them with `ComponentProps` from `vue-component-type-helpers`:

```ts
import type { ComponentProps } from "vue-component-type-helpers";
import { UMain, USection } from "@nexdom/uimed-vue/components";

type MainProps = ComponentProps<typeof UMain>;
type AppBar = NonNullable<MainProps["appBar"]>;
type NavigationMenu = NonNullable<MainProps["navigationMenu"]>;
type SectionAction = NonNullable<ComponentProps<typeof USection>["actions"]>[number];
```

Available entry points: `@nexdom/uimed-vue` (root), `@nexdom/uimed-vue/components`, `@nexdom/uimed-vue/composables`, `@nexdom/uimed-vue/plugins`, `@nexdom/uimed-vue/unit-test` (test helpers, e.g. `vueTestUtilsPluginUimed()` for mounting components with Vuetify in Vitest).

Full component/composable reference lives in the [docs](https://nexdom-healthtech.github.io/uimed-vue/).

## Contributing to this repo

The sections below are for developers working _on_ this library itself.

### Dev environment setup

This project expects to be opened inside its devcontainer (VSCode). If commands below fail with missing shims/commands, run:

```bash
vp env doctor   # diagnose missing parts
vp env setup    # create shims like vpr and vpx
vp install      # install dependencies
```

On Windows, prefer VSCode's "Dev Containers: Clone Repository in Container Volume" over opening a folder cloned on the host:

- A host clone with `core.autocrlf=true` checks files out with CRLF, and `vpr check` then reports formatting issues on every file. `.gitattributes` enforces LF, but clones made before it was added need `git rm -rq --cached . && git reset --hard` to be re-normalized (commit or stash local changes first, since `reset --hard` discards them).
- A host folder bind-mounted into the container is roughly 10x slower (unit tests alone go from seconds to minutes), which makes Stryker's initial test run time out.

## Commands

- `vpr check` — lint, formatter, and type-check (builds the library first, since the type-check reads `dist`)
- `vp test --coverage` — unit tests with coverage (Vitest, jsdom, 100% coverage threshold enforced)
- `vpr test:mutations` — mutation tests (Stryker; thresholds: high 100, low 100, break 100). It starts one test runner per CPU core; on machines with limited memory, run `vpx stryker run --concurrency 4` instead
- `vpr test:e2e` — E2E tests (Playwright, runs against the built docs preview site)
- `vpr depcruise` — architecture/dependency rules (dependency-cruiser)
- `vp pack` / `vpr build` — build the library
- `vpr docs` / `vpr docs:dev` — run docs site (imports the lib from `dist`, not `src` — run `vpr dev` in a second terminal to keep `dist` updated while iterating)

CI (`.github/workflows/ci.yml`) runs, in order: commitlint on PR commits, `vp pack`, `vpr check`, `vpr depcruise`, `vp test --coverage`, `vpr test:mutations`, `vpr test:e2e`. Match this locally before opening a PR.

### Focused runs while iterating

The full suite takes minutes; while working on a single component or composable, scope each step to it (`button` below is an example) and run the full suite only before opening a PR:

```bash
# Unit tests of one or more folders
vp test src/components/button

# Same, with coverage limited to the files you changed. Without `--coverage.include`, files that
# are only imported (e.g. the children of `main.vue`) count as uncovered and fail the 100% threshold
vp test src/components/button --coverage --coverage.include="src/components/button/**"

# Mutation tests of specific files. List several files in a single comma-separated `--mutate`
# (repeating the flag keeps only the last one)
vpx stryker run --mutate "src/components/button/button.vue,src/composables/button/button.ts"

# One E2E spec
vpr test:e2e e2e/components/button.spec.ts
```

## Adding a new component

Use an existing component (e.g. `button`) as the reference, and deliver all of the following in the same PR:

1. `src/components/<name>/<name>.vue` (following the pattern in [Code conventions](#code-conventions)), `types.ts` and `__tests__/<name>.test.ts`.
2. The `U`-prefixed export in `src/components/index.ts`.
3. A usage guide at `docs/guide/components/<name>.md`, with `<demo>` examples (and a `<playground>` section when the component has configurable props), and an API reference at `docs/api/components/<name>.md` (props, events and slots tables). Both in Portuguese.
4. Both pages registered in the sidebar at `docs/.vitepress/config.ts`.
5. `e2e/components/<name>.spec.ts` covering the guide's interactive examples and a "UI consistency" screenshot check (plus accessible snapshots where relevant). Commit the generated files under `__snapshots__`/`__screenshot__`.
6. The full CI sequence passing locally.

## Adding a new composable

Use an existing composable (e.g. `use-toast`) as the reference, and deliver in the same PR:

1. `src/composables/<group>/<name>.ts` and `__tests__/<name>.test.ts`, plus the types in the group's `types.ts`. If it relies on an internal component (as `useToast` relies on the internal `Toast` rendered by `UMain`), that component follows the component layout but isn't exported.
2. The export in `src/composables/index.ts`, and its assertion in `src/composables/__tests__/index.test.ts`.
3. A usage guide at `docs/guide/composables/<name>.md` and an API reference at `docs/api/composables/<name>.md`, both in Portuguese, registered in both sidebars at `docs/.vitepress/config.ts` and listed in `docs/api/index.md`.
4. `e2e/composables/<name>.spec.ts` covering the guide's examples and a "UI consistency" screenshot check.
5. The full CI sequence passing locally.

## Code conventions

- All `src` code is written in English. `docs` content is written in Portuguese (aimed at Brazilian users), even though file/dir names stay in English.
  - `docs` must not mention Vuetify
- Never write CSS, classes or any kind of styling. Always use component props.
- Path aliases: `@/*` → `src/*`, `@e2e/*` → `e2e/*`.
- Every public component follows this pattern to block access to internals and give it an editor-hover description:

  ```vue
  <script lang="ts">
  /**
   * A short description of the component, shown when hovering it in the
   * editor.
   */
  export default {
    inheritAttrs: false,
  };
  </script>

  <script setup lang="ts">
  // component's implementation
  </script>
  ```

- Component layout: `src/components/<name>/<name>.vue`, `types.ts` for prop/option types, `__tests__/<name>.test.ts` for unit tests. Composables follow the same shape under `src/composables/<name>/`.
- Public components/composables are re-exported from `src/components/index.ts` and `src/composables/index.ts`. Prop types stay internal (`types.ts` is not re-exported); consumers derive them with `ComponentProps` (see [Usage](#usage)).
- Every publicly exported component must use the `U` prefix on its export identifier (e.g., `UButton`, `UMain`), while internal file names and component names remain unprefixed.
- Use [JSDoc](https://jsdoc.app/about-getting-started) on every method/prop/type intended to be part of the public API — it's the primary documentation surface and supports markdown/code examples.
- Known workarounds (see CONTRIBUTING.md before touching related config): `stryker-vue-ignorer` patches a Stryker/Vue macro-hoisting issue; `vue-tsc` is used for type-check instead of Vite+'s built-in one due to an oxlint/Vue support gap. Both are meant to be removed once their upstream issues are fixed — don't build further on top of them without checking if they're still needed.

## Testing conventions

- Unit tests use Vitest + `@vue/test-utils`, with `vueTestUtilsPluginUimed()` from `@/unit-test.ts` to mount a Vuetify instance.
- Global unit test setup (`src/__tests__/setup.ts`) stubs `visualViewport`, uses fake timers, and silences `console.error/warn/log`.
- With fake timers in jsdom, Vuetify transitions (e.g. of `VDialog`, `VMenu`) don't finish on their own. Emit the transition events on the Vuetify component (`wrapper.findComponent(VDialog).vm.$emit("afterLeave")`) or advance the timers with `await vi.runAllTimersAsync()`.
- Coverage threshold is 100%; mutation testing threshold is 100% (break at 100). Don't add code paths without covering tests.
- E2E tests (Playwright, `e2e/`) run against the built docs preview (`http://localhost:4173/uimed-vue/`). Snapshots/screenshots live under `__snapshots__`/`__screenshot__` next to each spec.

## Library documentation and dependencies

The stack is newer than most AI models' training data (Vue 3.5, Vuetify 4, Vue Router 5, Vite+ 0.2, Vitest 4, TypeScript 6, VitePress 2 alpha, Stryker 10, Playwright 1.62), so don't rely on memory for library APIs.

- Before using an API this repo doesn't use yet, or implementing something from scratch, look it up in the version declared in `package.json`. The project's `.mcp.json` provides two servers for that:
  - `context7`, with these library IDs: Vue `/websites/vuejs`, Vuetify `/websites/vuetifyjs_en`, Vue Router `/websites/router_vuejs`, Vite+ `/websites/viteplus_dev`, Vitest `/vitest-dev/vitest`, Vue Test Utils `/vuejs/test-utils`, Playwright `/microsoft/playwright`, VitePress `/vuejs/vitepress`, Stryker `/stryker-mutator/stryker-js`.
  - `vuetify`, Vuetify's own server, for component and composable APIs and release notes. Its tools that create or update bins, links, playgrounds or bug reports publish content outside the repo and are denied in `.claude/settings.json`.
  - Queries to both servers leave your machine: describe what you need in generic terms and never include source code or business rules.
- Before adding a dependency, check whether Vuetify, `@nexdom/shared` or the current dependencies already cover the need. If not, confirm with the requester, check the package's docs, maintenance and license, and declare runtime dependencies as `peerDependencies` (enforced by dependency-cruiser's `use-peer-deps` rule).
- Vuetify is an internal detail: consumers use the library's props, and `docs` never mention Vuetify.

## AI agents

Besides this file, the repo ships Claude Code subagents in `.claude/agents/`:

- `issue-planner`: turns an issue into an API proposal plus open questions, before any code is written.
- `issue-implementer`: implements an issue end to end (source, tests, docs, E2E).
- `code-reviewer`: reviews a branch or PR against these conventions, without changing code.
- `dependency-updater`: evaluates and applies dependency updates, such as Dependabot PRs.

Issues and PRs live in `nexdom-healthtech/uimed-vue`. Pass `-R nexdom-healthtech/uimed-vue` to `gh`, since clones from forks have issues disabled, and read issues with `gh issue view <number> -R nexdom-healthtech/uimed-vue --json title,body,comments` (`--comments` prints nothing in non-interactive shells).

To resolve an issue, run `/resolve-issue <number>` (`.claude/skills/resolve-issue/`). It chains planner, implementer and reviewer, asks you about open questions and waits for your approval of the public API before any code is written.

## Git workflow

- Trunk-based development. `main` is the only long-lived branch (`beta`/`alpha` exist only for pre-release/prototype work — see CONTRIBUTING.md for when to use them). `main`, `beta`, `alpha` are all protected against direct pushes.
- Short-lived branches are named after their intention (e.g. `fix-some-method-behavior`) and are deleted after merging back via PR.
- Commit messages MUST follow [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/) — enforced by commitlint in CI. Recent examples: `feat(components): ...`, `fix(composables): ...`, `test(e2e): ...`, `docs(root): ...`, `chore: ...`.
- Releases are automated via `semantic-release` (conventional-commits preset) — don't hand-edit versions/changelogs.
