---
name: gpu-repo-sync
description: Use when choosing where or how to sync Git repositories, worktrees, patches, bundles, checkpoints-related code, or project source across local workstations, object storage, offline GPU hosts, and direct-download GPU hosts. Use when a request ambiguously mentions GPU machines, A100-style offline training hosts, RTX/Isaac-style simulation hosts, repo sync, git sync, code deployment, or deciding which GPU host should receive a repository update.
---

# GPU Repo Sync Router

This skill routes repository synchronization without embedding private infrastructure details. Treat all hostnames, users, ports, bucket names, filesystem roots, repository URLs, and credentials as local configuration supplied by the user, environment, SSH config, git config, or private project notes.

If machine-specific companion skills are installed, prefer them after selecting the target route. If they are not installed, use this skill's sanitized workflow directly.

## Required Local Context

Before syncing code, identify only the fields needed for the target path:

- Source repo: `<LOCAL_REPO_PATH>`, `<REPO_REMOTE_URL>`, `<BRANCH_OR_TAG>`, and the commit SHA to deploy.
- Offline GPU host: SSH alias or `<OFFLINE_GPU_USER>@<OFFLINE_GPU_HOST>`, optional `<OFFLINE_GPU_SSH_PORT>`, and `<OFFLINE_GPU_STORAGE_ROOT>`.
- Object storage handoff: `<OBJECT_STORAGE_URI>` and `tosutil` when the offline host cannot fetch directly and TOS is the selected backend.
- Direct-download GPU host: SSH alias or `<DIRECT_GPU_USER>@<DIRECT_GPU_HOST>`, optional `<DIRECT_GPU_SSH_PORT>`, and `<DIRECT_GPU_STORAGE_ROOT>`.
- Credentials: use existing SSH agents, git credential helpers, deploy keys, or environment variables. Never write tokens, private keys, passwords, or access keys into shared files, logs, commands, or final answers.

Prefer SSH config aliases and named storage roots over raw private values in prompts and artifacts.

## Route Selection

Use the offline GPU route when the request mentions offline training, object storage handoff, VEPFS/shared training storage, A100-style training, platform jobs, or a host that should not perform public internet downloads.

Use the direct-download GPU route when the request mentions Isaac, Omniverse, RTX rendering, simulation, direct internet access, or a direct-download storage root.

If the user only says "GPU machine", "sync the repo", or "deploy the code" without enough target context:

- Default to the direct-download GPU route for Isaac, simulation, rendering, or direct internet tasks.
- Default to the offline GPU route for training jobs, object storage, shared filesystems, or offline storage tasks.
- Ask one concise target-host question when neither default is defensible.

## Pre-Sync Checks

Before touching any remote checkout:

1. Inspect local status and avoid overwriting uncommitted work:

   ```bash
   git -C <LOCAL_REPO_PATH> status --short --branch
   git -C <LOCAL_REPO_PATH> rev-parse HEAD
   ```

2. Decide whether the target should receive a branch, tag, exact commit, patch, bundle, or full working tree.
3. Prefer syncing a committed commit SHA. Use patches or archives only when the user explicitly wants uncommitted local changes copied.
4. Keep remote checkouts, build artifacts, caches, virtual environments, checkpoints, and logs under the configured storage root, not under `/`, `/root`, or a small home directory.

## Offline GPU Git Sync

Use direct git access only when the offline host can reach the repository remote through an approved network path. Otherwise stage a git bundle or archive through object storage. `tosutil` is the public TOS CLI name and can appear in shared examples; actual bucket names, prefixes, and credentials remain private.

Preferred committed-code flow:

1. On the local machine, ensure the desired commit is present and record its SHA.
2. Create a git bundle for the branch, tag, or all refs needed by the target.
3. Upload the bundle to `<OBJECT_STORAGE_URI>`.
4. Pull the bundle into `<OFFLINE_GPU_STORAGE_ROOT>/repos` on the offline host.
5. Clone or fetch from the bundle, then check out the recorded commit.
6. Verify the remote checkout commit SHA matches the intended source SHA.

Example skeleton:

```bash
git -C <LOCAL_REPO_PATH> bundle create <repo-name>.bundle --all
tosutil cp <repo-name>.bundle <OBJECT_STORAGE_URI>/<repo-name>.bundle
ssh <OFFLINE_GPU_SSH_ALIAS> 'mkdir -p <OFFLINE_GPU_STORAGE_ROOT>/repos && tosutil cp <OBJECT_STORAGE_URI>/<repo-name>.bundle <OFFLINE_GPU_STORAGE_ROOT>/repos/'
```

For uncommitted changes, create a patch or archive only after the user confirms that local dirty state should be copied. Record the base commit and patch file name so the remote state is auditable.

## Direct-Download GPU Git Sync

Use normal git clone/fetch on the target GPU host when the host has approved network access to the repository remote:

```bash
ssh <DIRECT_GPU_SSH_ALIAS>
mkdir -p <DIRECT_GPU_STORAGE_ROOT>/repos
cd <DIRECT_GPU_STORAGE_ROOT>/repos
git clone <REPO_REMOTE_URL> <repo-name>
cd <repo-name>
git fetch --all --tags --prune
git checkout <BRANCH_OR_TAG_OR_COMMIT>
git rev-parse HEAD
```

If the repository already exists remotely, inspect its dirty state before fetching or checking out:

```bash
ssh <DIRECT_GPU_SSH_ALIAS> 'git -C <DIRECT_GPU_STORAGE_ROOT>/repos/<repo-name> status --short --branch'
```

Do not delete or reset remote worktrees unless the user explicitly asks and the target path has been verified.

## Do Not Mix

- Do not route direct-download GPU repo sync through object storage unless the user asks or direct git access fails.
- Do not run public internet git operations on offline hosts unless the user confirms public egress is allowed.
- Do not use an Isaac/simulation host for offline training storage work unless the user explicitly chooses it.
- Do not place remote repositories, downloads, caches, environments, package builds, checkpoints, temporary files, or logs under `/`, `/root`, or small home directories.
- Keep offline GPU repositories under `<OFFLINE_GPU_STORAGE_ROOT>/repos`.
- Keep direct-download GPU repositories under `<DIRECT_GPU_STORAGE_ROOT>/repos`.
- Never expose deploy keys, personal access tokens, private repository URLs with embedded credentials, or SSH private keys in shared artifacts.
