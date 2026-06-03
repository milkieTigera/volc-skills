---
name: data-connection-guide
description: Use when choosing a data-transfer strategy across local machines, object storage, offline GPU hosts, direct-download GPU hosts, model or dataset mirrors, VEPFS/shared filesystems, package wheelhouses, or cache placement. Use when requests mention moving models, datasets, Python packages, checkpoints, or logs between a Mac/workstation, TOS or other object storage, Volcengine/A100-style offline training storage, and RTX/Isaac-style direct-download storage.
---

# Data Connection Router

This skill routes data movement without embedding private infrastructure details. Treat all hostnames, users, ports, bucket names, filesystem roots, mirror URLs, and credentials as local configuration supplied by the user, environment, SSH config, or project notes.

## Required Local Context

Before running transfer commands, identify only the fields needed for the target path:

- Offline GPU host: SSH alias or `<OFFLINE_GPU_USER>@<OFFLINE_GPU_HOST>`, optional `<OFFLINE_GPU_SSH_PORT>`, and `<OFFLINE_GPU_STORAGE_ROOT>`.
- Object storage handoff: `<OBJECT_STORAGE_URI>` such as `tos://<bucket>/<prefix>` and the configured upload/download tool.
- Direct-download GPU host: SSH alias or `<DIRECT_GPU_USER>@<DIRECT_GPU_HOST>`, optional `<DIRECT_GPU_SSH_PORT>`, and `<DIRECT_GPU_STORAGE_ROOT>`.
- Mirrors: `<HF_MIRROR_ENDPOINT>`, `<MODEL_REGISTRY_MIRROR>`, and `<PYPI_MIRROR_URL>` when the environment requires mirrors.
- Secrets: use the existing credential provider, shell environment, or host configuration; never write access keys, tokens, passwords, or private keys into files, logs, commands that will be shared, or final answers.

Prefer SSH config aliases and named storage roots over raw private values in prompts and artifacts.

## Route Selection

Use the offline GPU route when the target host cannot reliably access public internet resources or when the request mentions offline training, object storage handoff, VEPFS/shared training storage, A100-style training, or Volcengine ML tasks.

Use the direct-download GPU route when the target host has suitable internet access and local high-capacity storage, or when the request mentions Isaac, Omniverse, RTX rendering, direct model download, or a direct-download storage root.

If the request only says to download or move an asset and no target host is clear, ask whether the target is the offline GPU route or the direct-download GPU route.

## Offline GPU Route

Use a staged lifecycle:

1. Download or build the asset on a local machine or another internet-connected staging host.
2. Verify that the source endpoint is the intended mirror or registry, not an accidental fallback to a blocked or disallowed public endpoint.
3. Upload the completed asset to `<OBJECT_STORAGE_URI>`.
4. Pull the asset from object storage into `<OFFLINE_GPU_STORAGE_ROOT>` on the offline GPU host.
5. Verify size, checksum when practical, and file count on object storage and the offline GPU host.
6. Delete only the local staging copy for that asset after remote verification succeeds.
7. Repeat one asset at a time for large transfers so local disk cleanup is deterministic.

For public Hugging Face assets, use the configured mirror endpoint when required by the environment. Clear proxy variables before mirror downloads if proxies could cause a fallback path:

```bash
unset all_proxy ALL_PROXY http_proxy HTTP_PROXY https_proxy HTTPS_PROXY ftp_proxy FTP_PROXY no_proxy NO_PROXY
HF_ENDPOINT=<HF_MIRROR_ENDPOINT> hfd <org-or-repo> --local-dir <local-staging-dir>
```

Use object storage tooling configured for the environment, for example:

```bash
<object-storage-cli> cp -r <local-staging-dir> <OBJECT_STORAGE_URI>/<asset-name>
ssh <OFFLINE_GPU_SSH_ALIAS> '<object-storage-cli> cp -r <OBJECT_STORAGE_URI>/<asset-name> <OFFLINE_GPU_STORAGE_ROOT>/<asset-name>'
```

Keep large files, caches, environments, checkpoints, temporary files, and logs under `<OFFLINE_GPU_STORAGE_ROOT>`. Do not place large work artifacts under `/`, `/root`, or the login user's small home directory.

For Python packages on offline GPU hosts, prefer the configured private or platform mirror. If the required package is unavailable, build a compatible wheelhouse on an internet-connected machine, stage it through object storage, and install from the wheelhouse on the offline host. Do not fall back to public PyPI from the offline host unless the user explicitly confirms that public egress is allowed.

## Direct-Download GPU Route

Download directly on the target GPU host when the host has network access and enough storage:

1. SSH to `<DIRECT_GPU_SSH_ALIAS>`.
2. Put models, datasets, caches, packages, logs, checkpoints, and environments under `<DIRECT_GPU_STORAGE_ROOT>`.
3. Use configured mirrors such as `<HF_MIRROR_ENDPOINT>` or `<PYPI_MIRROR_URL>` when required.
4. Verify size, checksum when practical, and expected file count.
5. Do not route through local staging or object storage unless the user explicitly asks or direct download fails for a reason staging can solve.

Example:

```bash
ssh <DIRECT_GPU_SSH_ALIAS>
mkdir -p <DIRECT_GPU_STORAGE_ROOT>/models <DIRECT_GPU_STORAGE_ROOT>/cache/pip
HF_ENDPOINT=<HF_MIRROR_ENDPOINT> hfd <org-or-repo> --local-dir <DIRECT_GPU_STORAGE_ROOT>/models/<asset-name>
python -m pip install -i <PYPI_MIRROR_URL> --cache-dir <DIRECT_GPU_STORAGE_ROOT>/cache/pip <package>
```

## Gated Models And Private Assets

For gated Hugging Face assets, prefer an approved model registry mirror or organization-approved export path. If a token is required, pass it through the secure mechanism already used by the environment and avoid printing it. Never paste tokens into a skill, README, shared command transcript, or final answer.

If the direct mirror path is unavailable, ask the user which approved source should be used. Do not invent credentials, bypass access controls, or share private model artifacts outside the intended storage boundary.

## Long Runs

For multi-hour downloads or transfers:

- Run work inside `screen`, `tmux`, a managed job, or another resumable execution environment.
- Log progress to a file under the target storage root, not under `/root` or a small home directory.
- Track expected bytes and file counts before and after each hop.
- Attach a lightweight heartbeat monitor when useful to report progress, speed, remote verification, and cleanup status.
- Clean local staging only after object storage and remote-host verification succeed.

## Safety Checklist

- Never record access keys, HF tokens, passwords, or private keys in shared files, commands, logs, or final answers.
- Do not keep local archives after object-storage handoff and remote verification.
- Keep large work artifacts off `/`, `/root`, and small home directories.
- Keep offline GPU assets on the configured shared training storage root.
- Keep direct-download GPU assets on the configured high-capacity storage root.
- Do not use direct public downloads on offline hosts unless the user confirms public egress is allowed.
- Do not fall back to public PyPI from offline hosts unless the user confirms it is allowed.
- Ask before deleting any remote artifact unless it is an explicitly identified temporary staging file from the current task.
