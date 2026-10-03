# frontseat-action

Readies a GitHub-hosted runner to run [frontseat](https://github.com/frontseat-dev/frontseat).

You do not write the workflow that uses it. frontseat's github plugin renders
`.github/workflows/frontseat.yml` from `frontseat.yaml`, and `frontseat
generate --fix` keeps it in step:

```yaml
steps:
  - uses: actions/checkout@v4
    with:
      fetch-depth: 0
  - uses: frontseat-dev/frontseat-action@v1
  - run: mise exec -- frontseat verify --plain
```

frontseat runs on the grid `frontseat.yaml` names, and the action readies
the runner for that grid. It:

1. reads `grid.address` from `frontseat.yaml`;
2. without one, frees disk, since the embedded grid then runs on the runner
   and will not start an action with less than 8 GiB free;
3. installs bubblewrap and allows the user namespaces it needs: the embedded
   grid confines every action, and effects such as publishes run on an
   embedded grid of their own even when a remote grid builds;
4. installs the repository's mise toolchain, frontseat included;
5. restores the workspace's memos and, without a named grid, the embedded
   grid's store, and saves them after the job.

## Inputs

| name | default | |
|---|---|---|
| `working-directory` | `.` | Directory holding `frontseat.yaml` and `mise.toml`. |
| `free-disk` | `true` | Remove preinstalled toolchains frontseat never uses, when the grid is the embedded one. Turn it off on a self-hosted runner. |
| `cache` | `true` | Restore and save the memos, and the embedded grid's store. |

Workarounds the action carries are in [HACKS.md](HACKS.md).
