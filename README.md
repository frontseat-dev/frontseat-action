# frontseat-action

GitHub Actions integration for [Frontseat](https://github.com/frontseat-dev/frontseat).

It ships in two layers. Pick the one that matches how much of the job you want
to own:

| | Use when |
|---|---|
| **Reusable workflow** — `frontseat-dev/frontseat-action/.github/workflows/frontseat.yml@v1` | You want the whole pipeline. Your workflow keeps triggers, concurrency, permissions and the release decision; everything else lives here. |
| **Composite actions** — `frontseat-dev/frontseat-action@v1` + `…/teardown@v1` | You want Frontseat as steps inside a job you control — a matrix leg, a job that does other work alongside the build. |

Both layers are the same code: the reusable workflow calls the composite
actions.

## Reusable workflow

```yaml
name: CI
on:
  pull_request:
    branches: [main, 'release/**']
  push:
    branches: [main, 'release/**']
  workflow_dispatch:
    inputs:
      release:
        type: boolean
        default: false

permissions:
  contents: write
  packages: write

jobs:
  ship:
    uses: frontseat-dev/frontseat-action/.github/workflows/frontseat.yml@v1
    with:
      release: ${{ github.event_name == 'workflow_dispatch' && github.event.inputs.release == 'true' }}
```

That runs `frontseat verify //...` on every trigger and `frontseat announce //...`
when `release` is true, on a runner with the toolchain installed, the REAPI grid
up, the daemon warm, and the grid store restored from and saved back to
`actions/cache`.

### Inputs

| Name | Default | Description |
|---|---|---|
| `release` | `false` | Whether this run cuts a release. Gates the ship command and the GPG import. |
| `runs-on` | `ubuntu-latest` | Runner label. |
| `working-directory` | `.` | Directory containing `frontseat.yaml`. |
| `build-command` | `frontseat verify //...` | Runs on every trigger. |
| `ship-command` | `frontseat announce //...` | Runs only when `release` is true. |
| `build-timeout-minutes` | `120` | Bound on the build so a stall fails into the log dump. |
| `bootstrap-packages` | `""` | Go packages to build from the checkout and put first on `PATH`. |
| `release-token-env` | `""` | Env var carrying a minted GitHub App token. Set it to enable minting. |
| `release-app-owner` | `""` | Installation owner for that token. |
| `release-app-repositories` | `""` | Comma-separated repositories to narrow it to. |
| `free-disk` | `true` | Reclaim preinstalled toolchains. Turn off on self-hosted runners. |
| `cache` | `true` | Restore and save the grid store and resolution memos. |
| `grid` | `true` | Start the embedded REAPI grid. Turn off only for an external `reapi:` endpoint. |
| `grid-max-size` | `4g` | Size the store is trimmed to before saving. |
| `mise-plugin-repo` | `frontseat-dev/frontseat-mise` | Repository holding the mise backend plugin. |

### Secrets

All optional. Map them explicitly — this workflow does not use `secrets: inherit`.

| Name | Description |
|---|---|
| `MISE_PLUGIN_TOKEN` | Read access to the mise plugin repository. Needed only while it is private and the mise cache is cold. |
| `GPG_PRIVATE_KEY` | Passphrase-less signing key, imported before ship. |
| `RELEASE_APP_ID` / `RELEASE_APP_PRIVATE_KEY` | The App that mints `release-token-env`. |
| `RELEASE_ENV` | Extra credentials for build and ship, as `NAME=VALUE` lines. |

`RELEASE_ENV` is one blob rather than a fixed list of named secrets, because the
names belong to the calling workspace, not to this workflow:

```yaml
    secrets:
      RELEASE_ENV: |
        MAVEN_CENTRAL_USERNAME=${{ secrets.MAVEN_CENTRAL_USERNAME }}
        MAVEN_CENTRAL_PASSWORD=${{ secrets.MAVEN_CENTRAL_PASSWORD }}
```

Each value is registered as a mask before it is exported, then written to
`$GITHUB_ENV` ahead of the daemon starting, so the daemon and everything it
spawns inherit it. Values must be single-line; base64-encode a PEM or JSON key
and decode it in the consuming task.

## Composite actions

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0

      - uses: frontseat-dev/frontseat-action@v1

      - run: frontseat verify //...

      - uses: frontseat-dev/frontseat-action/teardown@v1
        if: always()
```

Setup takes `working-directory`, `github-token`, `mise-plugin-token`,
`mise-plugin-repo`, `mise-version`, `free-disk`, `bootstrap-packages`, `cache`,
`go-module-cache`, `grid`, `grid-timeout-seconds`, `daemon` and
`daemon-timeout-seconds`. Teardown takes `working-directory`, `cache`,
`grid-max-size` and `dump-logs`. See each `action.yml` for the details.

**Teardown needs `if: always()`.** Composite actions cannot register a `post:`
hook, which is why the closing half is a second action rather than something
setup schedules for you. Saving the grid store on a *failed* run is the point:
the toolchains and dependency blobs that run fetched are exactly what the retry
needs to not re-download, and a warm store keeps upstream registries from
rate-limiting it. A single `actions/cache` step would not do — its post-save is
skipped when the job fails.

## What the pipeline actually does

1. **Free runner disk.** Hosted runners ship ~20GB of toolchains this pipeline
   never touches, leaving ~14GB — not enough for a grid store plus
   multi-platform build outputs.
2. **Install the toolchain via mise**, cached by version. Frontseat treats tool
   versions as user-owned: they come from your `mise.toml` and nowhere else.
3. **Restore the caches.** The grid store is the real dependency cache — warm
   CAS means tools and dependencies never re-download, warm action cache means
   unchanged actions never re-execute. The Go module download proxy is cached
   separately because the Go resolver reads it as a `file://` tier ahead of the
   network; no other language needs a host-side cache, since they provision
   sandboxes from the grid CAS.
4. **Start the REAPI grid.** Frontseat is remote-only: every hermetic action
   executes on a grid. CI uses the embedded no-Docker one, which serves the full
   REAPI surface in a single process on the runner.
5. **Start the daemon** and wait for it to report ready, rather than letting the
   first command start it implicitly — on a cold two-core runner, entity
   discovery outlasts the CLI's own readiness wait.
6. **Run build, then ship** if this run is a release.
7. **Trim and save the store, and dump logs on failure** — the daemon and
   per-action logs are the only place startup and sandboxed-task errors surface.

## Cache scope

GitHub shares Actions caches with a pull request only from its base branch, and
caches saved inside a PR run are invisible to every other PR. A repository that
only builds on pull requests therefore starts every run with a cold grid store.
Run the pipeline on pushes to your trunk too; that run's saved store is the one
every PR restores.

## Versioning

The reusable workflow and both composite actions ship under one tag and are
expected to move together — the workflow references the actions at its own major
tag. Pin to `@v1` for the moving major, or to a commit SHA to freeze.

The action's contract with Frontseat is the `frontseat grid` and
`frontseat daemon` command surface. A major bump here means that surface
changed.

## License

Apache 2.0. See [LICENSE](./LICENSE).
