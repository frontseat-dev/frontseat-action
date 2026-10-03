# Hacks

## The runner's AppArmor policy is relaxed for the sandbox

- **Where:** `action.yml`, "Ready the sandbox"
- **Why:** Ubuntu 24.04 denies unprivileged user namespaces through AppArmor (`kernel.apparmor_restrict_unprivileged_userns=1`), and bubblewrap needs one to confine an action; the action turns the restriction off on the runner it owns.
- **Remove when:** Ubuntu ships an AppArmor profile that grants bubblewrap its namespaces, or GitHub's images allow them.

## Preinstalled toolchains are deleted for disk

- **Where:** `action.yml`, "Free disk"
- **Why:** GitHub's Ubuntu images leave about 14 GiB free, and the embedded grid will not start an action below 8 GiB; the toolchains the images preinstall are the space there is to take.
- **Remove when:** GitHub's images leave room for a grid store, or frontseat runs on a grid of its own.
