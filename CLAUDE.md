# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Node.js bindings (a Node-API native addon) for libcmt, the Cartesi Machine guest rollup library. It lets Node.js dapps running inside a Cartesi Machine process rollup inputs and emit vouchers/notices/reports without the rollup HTTP server.

## Commands

```sh
git submodule update --init        # required once: libcmt sources live in deps/machine-guest-tools
npm install                        # compiles the addon (or picks up a prebuild)
npm test                           # node --test, runs test/ against the libcmt mock
node --test --test-name-pattern "voucher" test/rollup.test.mjs   # single test
npm run build                      # force recompile (node-gyp rebuild)
npm run check:package              # attw --pack: validates dual ESM+CJS packaging
npm run prebuild                   # prebuildify, writes prebuilds/<platform>-<arch>/
test/machine/run.sh                # e2e: riscv64 prebuild inside a real cartesi-machine (needs docker + cartesi-machine CLI; SKIP_PREBUILD=1/SKIP_ROOTFS=1 to reuse artifacts)
```

Documentation site (Vocs): MDX pages in `docs/pages/`, configured by `vocs.config.ts` at the repo root (`srcDir: 'docs'`); deps are root devDependencies. Run `npm run docs:dev|docs:build|docs:preview`. Quirks (all stem from vocs running vite with `configFile: false` plus this package being CJS-rooted):
- `docs:dev`/`docs:build` invoke `vite` directly so `vite.config.ts` is loaded; it replicates the vocs CLI (react + vocs plugins) and adds the `[name].js` entry-name + `dist/package.json` ESM-marker workaround (CJS root makes vite emit `.mjs`, but waku hardcodes `dist/server/build.js`).
- `react-server-dom-webpack` must stay a direct devDependency or `docs:dev` fails with a `react-server` condition error ([vocs#450](https://github.com/wevm/vocs/issues/450)).
- waku is pinned exactly (1.0.0-beta.1): beta.2 breaks vocs SSG (`contextMiddleware is not a function`).
- Docs code blocks use twoslash (```ts twoslash) and must typecheck — verify with `npx vocs twoslash`. Package imports resolve via self-reference (the root `exports` map), which needs `typescript` and `@types/node` as root devDependencies; without a root `typescript`, attw's pinned `typescript@5.6.1-rc` gets hoisted and the twoslash checker fails with a lib-file mismatch.
- Docs deploy to GitHub Pages (https://tuler.github.io/libcmt-node/) via `.github/workflows/docs.yml` on pushes to main: `renderStrategy: 'full-static'` emits HTML to `dist/public`, `BASE_PATH=/<repo>` sets the Pages subpath, and the job installs with `npm ci --ignore-scripts` (no addon compile, no submodules).

## Architecture

Three layers, one native instance:

- `src/addon.cc` — Node-API (node-addon-api) C++ wrapper exposing libcmt's C API as a `Rollup` ObjectWrap. **The whole API is synchronous on purpose**: blocking calls (`finish`, `gio`) yield the machine, pausing the entire guest including the Node event loop, so nothing could run concurrently anyway. Errors carry the negative errno in `error.errno`.
- `lib/index.js` (CJS, the real implementation) — argument conversion (0x-hex strings / `Uint8Array` / `bigint` ⇄ Buffers), the `Rollup` class wrapping the native one, and the `run()` handler loop. Loads the addon via `node-gyp-build` (prefers `build/Release`, falls back to `prebuilds/`).
- `lib/index.mjs` + `lib/index.d.mts` (ESM) — thin re-export wrappers over the CJS entry so both module systems share the single addon instance. The `exports` map in package.json routes `import`/`require` to the right pair. Any packaging change must keep `npm run check:package` (attw) green; CI enforces it.

### Dual-target build (binding.gyp)

The same addon source builds two flavors, selected by `target_arch`:

- **riscv64** (inside the Cartesi Machine): links a prebuilt real-IO-driver `libcmt.a` (`LIBCMT_LIB` override; defaults to the one installed by the machine-guest-tools `.deb`). Does not compile libcmt sources.
- **everything else** (development host): compiles libcmt's sources *including the mock IO driver* (`io-mock.c`) straight into the addon, from the `deps/machine-guest-tools` submodule (`LIBCMT_DIR` override, must resolve inside the package tree).

The mock is driven by env vars: `CMT_INPUTS="0:advance.bin,1:inspect.bin"` injects inputs (reason 0 = advance, 1 = inspect, other = gio reply), `CMT_DEBUG=yes` logs verbosely. The mock writes output files next to the inputs — tests `chdir` to a tmpdir to keep them out of the repo. `test/rollup.test.mjs` contains a pure-JS EVM-ABI encoder for crafting `EvmAdvance` inputs.

Only one `Rollup` instance may be open at a time (`-EBUSY`); `close()` the previous one first.

## CI / Releasing (.github/workflows/build.yml + release.yml)

- Prebuilds are built for linux-x64, linux-arm64, darwin-x64 (`macos-15-intel` runner), darwin-arm64, and linux-riscv64. The `test-machine` job then runs the riscv64 prebuild end-to-end inside an emulated Cartesi Machine (test/machine/run.sh with the `cartesi/machine-emulator` docker image; needs `docker/setup-qemu-action` for the riscv64 rootfs build). The riscv64 job cross-compiles: it needs the riscv64 cross toolchain, a libcmt cross-built with the Cartesi Linux headers (`LINUX_IMAGE_VERSION`/`LINUX_VERSION` workflow env must stay in sync with the submodule's Makefile `LINUX_HEADERS_*`), and `prebuildify --strip-bin riscv64-linux-gnu-strip` (host `strip` cannot process riscv64 ELF).
- Releasing is automated with changesets. **Every change that affects the published package must include a changeset**: run `npx changeset`, pick the bump level, describe the change, commit the generated `.changeset/*.md` file alongside the code. Docs-only/CI-only changes don't need one.
- Release flow (`.github/workflows/release.yml`): on pushes to main, `changesets/action` opens/updates a "Version Packages" PR (`changeset version`: bumps package.json, writes CHANGELOG.md, deletes consumed changesets). Merging that PR makes the action run `changeset tag` (single-package repo ⇒ tag is `vX.Y.Z`) and push the tag; since `GITHUB_TOKEN` tag pushes don't trigger workflows, a follow-up step dispatches build.yml on the tag (`gh workflow run`, needs `actions: write`). Never bump `version` in package.json by hand. Requires "Allow GitHub Actions to create and approve pull requests" in repo Actions settings.
- The publish job in build.yml (gated on `refs/tags/v*`, reached via that dispatch or a manually pushed tag) verifies the tag matches package.json, merges prebuild artifacts, smoke-tests the packed tarball, and publishes to npm via trusted publishing (OIDC — no token secret; needs `id-token: write`, npm >= 11.5.1, and the Trusted Publisher configured on npmjs.com for the package). The package is temporarily named `@tuler/node-libcmt` until it becomes an official Cartesi package.
