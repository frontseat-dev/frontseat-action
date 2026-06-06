# frontseat-action

The official GitHub Action for [Frontseat](https://github.com/frontseat-dev/frontseat).
Installs the Frontseat CLI plus declared plugins, restores the REAPI build cache
between runs, and drives any lifecycle command (`build`, `ship`, `lint`, `test`,
`conform`, …) on the workspace.

## Quick start

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: frontseat-dev/frontseat-action@v1
        with:
          mode: build
```

That single step replaces a manual `curl mise.run | sh` + tool install + REAPI
cache restoration block — `frontseat build //` runs in the same job, and a
hashed cache of `.frontseat/cache/` survives across runs.

## Shipping a release

```yaml
- uses: frontseat-dev/frontseat-action@v1
  with:
    mode: ship
    args: --prerelease alpha
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

`mode: ship` runs the full Frontseat ship lifecycle (`assemble → changelog →
catalog → sign → deploy → upload → release → prepare → publish → announce`).
GitHub release creation needs `GITHUB_TOKEN` exposed to the step.

## Inputs

| Name | Default | Description |
|---|---|---|
| `mode` | `build` | Lifecycle command to run (`build`, `ship`, `lint`, `test`, `package`, `verify`, `conform`, `sync`). Empty string installs Frontseat and exits. |
| `entities` | `//` | Entity selector passed to the command. |
| `version` | `latest` | Frontseat CLI version. Resolved against the [frontseat releases](https://github.com/frontseat-dev/frontseat/releases) via the `frontseat-mise` plugin. |
| `plugins` | `""` | Comma-separated plugin names (e.g. `go,maven,docker`). When empty, the action defers to whatever the workspace's `mise.toml` declares. |
| `args` | `""` | Extra arguments appended to the Frontseat command. |
| `exec-mode` | `local` | Passed as `--exec-mode`. |
| `cache-mode` | `offline` | Passed as `--cache-mode`. |
| `cache` | `"true"` | Enable REAPI local-cache persistence via `actions/cache`. |
| `working-directory` | `.` | Directory containing `frontseat.yaml`. |
| `mise-version` | `""` | Override the mise CLI version (passed to `jdx/mise-action`'s `version` input). |

## What it does, in order

1. **Configures `frontseat-mise`** if your workspace doesn't already declare it.
   Drops a small `mise.toml` overlay into `.mise/` so this action works in
   workspaces that haven't pinned Frontseat themselves.
2. **Installs mise and all declared tools** via [`jdx/mise-action`](https://github.com/jdx/mise-action)
   with caching enabled. This downloads the Frontseat CLI, every plugin tarball
   for the runner's OS/arch, and the language toolchains declared in your
   workspace's `mise.toml`. On a warm cache this is seconds.
3. **Restores the REAPI cache** from `actions/cache` keyed on the hash of your
   workspace manifests (`go.sum`, `go.work.sum`, `pom.xml`, `package-lock.json`,
   `Cargo.lock`). Subsequent runs that don't change dependencies skip
   already-completed actions.
4. **Runs the Frontseat command** with the configured `--exec-mode` and
   `--cache-mode`.

## Why not just call `frontseat` directly

You can — and for a quick experiment that's the right answer. The action exists
to encapsulate three things that are tedious to get right by hand:

- **Auth wiring** for `gh` (used by `frontseat-mise` for version discovery)
  and the GitHub release step in `mode: ship`.
- **REAPI cache persistence** between ephemeral runner instances. Without it,
  every `frontseat build` rebuilds everything from scratch.
- **Plugin installation** through `frontseat-mise`'s `plugins` option, which
  needs a token configured correctly to talk to the GitHub API.

## License

Apache 2.0. See [LICENSE](./LICENSE).