# `@artk0de/tree-sitter-swift` — prebuilt N-API package

This branch (`prebuild/0.7.3`) packages the upstream tree-sitter-swift 0.7.3
grammar as `@artk0de/tree-sitter-swift@0.7.3-prebuild.1`, with N-API prebuilds
for six platforms. [TeaRAGs](https://github.com/artk0de/TeaRAGs) loads it
through `@artk0de/tree-sitter` 0.25.

## Why a fork package

Upstream tagged 0.7.3 on GitHub but never published it to npm (the latest
there is 0.7.1). 0.7.3 is the release that parses `@unchecked Sendable`, `#if`
inside a type body and `nonisolated(unsafe)`. Upstream's npm package also has
no prebuilds and regenerates the parser during `node-gyp` builds, which needs
`tree-sitter-cli` at install time.

## Provenance

- Source: the upstream **release asset**
  `https://github.com/alex-pinkus/tree-sitter-swift/releases/download/0.7.3/tree-sitter-swift.tar.gz`,
  sha256 `c595b41459b0816f246ec27f60e08392ff453135e12ef47a1852ed37fe6705fe`.
- The asset's `src/parser.c` is generated at ABI 15 (`LANGUAGE_VERSION 15`).
  The generated files reachable from the git tag `0.7.3` are ABI 14, so this
  branch uses the asset, not the tag.
- The branch starts at tag `0.7.3`; the tree is then replaced with the asset
  contents as staged by `scripts/prepare-from-release.js`.

## What differs from the release asset

- `package.json` — renamed to `@artk0de/tree-sitter-swift`, version
  `0.7.3-prebuild.1`, `repository`/`bugs`/`homepage` pointed at this fork.
  `tree-sitter-cli` and `which` removed from `dependencies`: only the parser
  generator needs them, and a consumer of prebuilds never generates. The
  install script is just `node-gyp-build`. The `tree-sitter` peer is optional
  (upstream misspells the meta key as `tree_sitter`, so upstream enforces it).
- `binding.gyp` — the `actions` block that runs `tree-sitter generate` on every
  native build is removed. The asset already carries the generated `src/`.
- `package-lock.json` — regenerated for the reduced dependency set.
- `.gitignore` — upstream ignores the generated parser; here it is committed.
- `.github/workflows/` — upstream's workflows (grammar CI, crates/PyPI/npm
  release on every tag) are removed from this branch; only `prebuild.yml`
  remains.

`src/`, `grammar.js`, `queries/` and `bindings/node/` are byte-identical to the
release asset.

## Build

Pushing a tag matching `v*-prebuild*` (or dispatching the workflow manually)
runs `.github/workflows/prebuild.yml`:

1. `prebuild` — on macos-14 (darwin-arm64), macos-15-intel (darwin-x64),
   ubuntu-24.04-arm (linux-arm64), ubuntu-latest (linux-x64), windows-latest
   (win32-x64) and windows-11-arm (win32-arm64): `npm install --ignore-scripts`
   then `npx prebuildify --napi --strip`, uploaded as `prebuild-<platform>`.
2. `pack` — downloads all prebuilds, fails unless all six
   `prebuilds/<platform>/*.node` exist, runs `npm pack`, prints the tarball's
   sha512, and uploads the `.tgz` plus per-platform
   `tree-sitter-swift-<platform>.node` copies as the `package` artifact.

## Publish

Nothing is published by CI. With the `package` artifact downloaded:

```bash
gh run download <run-id> -R artk0de/tree-sitter-swift -n package -D ts-swift-pkg
tar -tzf ts-swift-pkg/artk0de-tree-sitter-swift-0.7.3-prebuild.1.tgz | grep prebuilds/
npm publish ts-swift-pkg/artk0de-tree-sitter-swift-0.7.3-prebuild.1.tgz --access public --tag latest
```

npm 11 refuses a prerelease version without an explicit `--tag`; this package
has no non-prerelease line, so `latest` is the honest tag.

A new build of the same grammar bumps the `-prebuild.N` suffix in
`package.json` and pushes a matching `v0.7.3-prebuild.N` tag.
