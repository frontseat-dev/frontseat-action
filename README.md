# frontseat-action

Readies a runner, GitHub-hosted or the workspace's own, to run [frontseat](https://github.com/frontseat-dev/frontseat).

You do not write the workflow that uses it. frontseat's github plugin renders
`.github/workflows/frontseat.yml` from `frontseat.yaml`, and `frontseat
generate --fix` keeps it in step. The workflow keeps the triggers and the
decision whether a run releases; the action does the rest:

```yaml
steps:
  - uses: frontseat-dev/frontseat-action@v1
    with:
      release: ${{ github.event_name == 'workflow_dispatch' && inputs.release }}
```

frontseat runs on the grid `frontseat.yaml` names, any REAPI grid, and the
action readies the runner for it. What the runner needs to reach that grid,
a VPN client for one, is the repository's own: the github plugin's `setup`
steps run before the action. The action:

1. checks the repository out with its history and tags;
2. installs the repository's mise toolchain, frontseat included;
3. reads the grid the run selects from `frontseat.yaml`, as frontseat
   selects it: `grid`'s address, else the one of `grids` that
   `FRONTSEAT_GRID` names, else the one marked `default`, else the only
   one. It reads the file with yq, which mise runs at a pinned version, so
   a runner needs only mise and bash;
4. without a grid address, frees disk, since the embedded grid then runs
   on the runner and will not start an action with less than 8 GiB free;
5. readies the sandbox: the embedded grid confines every action under
   bubblewrap, and effects such as publishes run on an embedded grid of
   their own even when a remote grid builds. A runner that has `bwrap` is
   left as it is; where it is missing and `apt-get` exists, the action
   installs it and allows the user namespaces it needs; anywhere else it
   fails, and the runner's image must carry bubblewrap;
6. restores the workspace's memos and, without a named grid, the embedded
   grid's store, and saves them after the job;
7. runs `frontseat verify`, then `frontseat publish` when `release` is
   `true`, logged in to `ghcr.io` with `github-token` first: the publish's
   tools push with the Docker config's credentials, and the action writes
   them to a config of the job's own, beside what the runner's holds. The
   workflow grants the token `packages: write` (the github plugin's
   `permissions`).

## Inputs

| name | default | |
|---|---|---|
| `release` | `false` | Publish after verifying. |
| `github-token` | the job's token | The token tools are installed with from GitHub releases, and a publish uses for the forge and GitHub's registry. |
| `working-directory` | `.` | Directory holding `frontseat.yaml` and `mise.toml`. |
| `free-disk` | `true` | Remove preinstalled toolchains frontseat never uses, when the grid is the embedded one. Turn it off on a self-hosted runner. |
| `cache` | `true` | Restore and save the memos, the plugins and their compiled wasm, and the embedded grid's store. |

Workarounds the action carries are in [HACKS.md](HACKS.md).
