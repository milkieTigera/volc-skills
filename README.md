# volc-skills

Shared Codex skills for Volcengine and GPU workflow operations.

## Layout

```text
volc-skills/
├── README.md
├── skills/
│   ├── benchmark-eval/
│   │   ├── SKILL.md
│   │   └── agents/
│   │       └── openai.yaml
│   ├── data-connection-guide/
│   │   ├── SKILL.md
│   │   └── agents/
│   │       └── openai.yaml
│   └── gpu-repo-sync/
│       ├── SKILL.md
│       └── agents/
│           └── openai.yaml
└── .gitignore
```

Each installable skill lives under `skills/<skill-name>/` and keeps its required `SKILL.md` plus optional `agents/`, `scripts/`, `references/`, and `assets/` resources.

## Included Skills

- `benchmark-eval`: a generalized high-throughput benchmark evaluation planner for task sharding, worker launch, monitoring, and result verification.
- `data-connection-guide`: a sanitized router for choosing data-transfer paths between local staging, object storage, offline GPU storage, and direct-download GPU hosts.
- `gpu-repo-sync`: a sanitized router for syncing Git repositories and worktrees across local workstations, object storage, offline GPU hosts, and direct-download GPU hosts.

## Installation

### With Skill Installer

After this repository is pushed to GitHub, install all shared skills from Codex with the `skill-installer` skill:

```text
Use $skill-installer to install benchmark-eval, data-connection-guide, and gpu-repo-sync from <github-owner>/volc-skills at skills/benchmark-eval, skills/data-connection-guide, and skills/gpu-repo-sync
```

Equivalent helper command:

```bash
python3 "${CODEX_HOME:-$HOME/.codex}/skills/.system/skill-installer/scripts/install-skill-from-github.py" \
  --repo <github-owner>/volc-skills \
  --path skills/benchmark-eval skills/data-connection-guide skills/gpu-repo-sync
```

To install only one skill, pass only that path:

```bash
python3 "${CODEX_HOME:-$HOME/.codex}/skills/.system/skill-installer/scripts/install-skill-from-github.py" \
  --repo <github-owner>/volc-skills \
  --path skills/benchmark-eval
```

For a branch or tag other than `main`, add `--ref <branch-or-tag>`.

For a private GitHub repo, make sure existing git credentials work or set `GITHUB_TOKEN`/`GH_TOKEN` before installing. Restart Codex after installation so the new skill is discovered.

No extra repository manifest is required. The important part is that the installer path points to a directory containing `SKILL.md`.

## Manual TOS Setup

Configure `tosutil` manually on each machine that needs to read from or write to TOS. Local workstations, offline GPU hosts, and direct-download GPU hosts each have their own home directory and their own `~/.tosutilconfig`.

1. Install `tosutil` using the official package or binary for the target OS, then make it executable on Linux or macOS:

   ```bash
   chmod +x ./tosutil
   ./tosutil version
   ```

2. Collect the non-secret routing values for the target bucket:

   ```text
   <TOS_ENDPOINT>    # TOS protocol endpoint, not an S3-compatible endpoint
   <TOS_REGION>      # Region for the bucket, for example cn-beijing
   <TOS_BUCKET_URI>  # Bucket or prefix, for example tos://<bucket>/<prefix>
   ```

3. Configure permanent credentials in a private shell. Do not paste real keys into shared docs, logs, issue comments, or chat transcripts:

   ```bash
   ./tosutil config \
     -i '<ACCESS_KEY_ID>' \
     -k '<SECRET_ACCESS_KEY>' \
     -e '<TOS_ENDPOINT>' \
     -re '<TOS_REGION>'
   ```

   For temporary credentials, include the security token:

   ```bash
   ./tosutil config \
     -i '<TEMP_ACCESS_KEY_ID>' \
     -k '<TEMP_SECRET_ACCESS_KEY>' \
     -t '<SECURITY_TOKEN>' \
     -e '<TOS_ENDPOINT>' \
     -re '<TOS_REGION>'
   ```

4. Verify the config without exposing secrets:

   ```bash
   ./tosutil version
   ./tosutil ls '<TOS_BUCKET_URI>'
   ```

5. Keep credentials local:

   - Do not commit `~/.tosutilconfig`, shell history, transfer logs, or credential screenshots.
   - Prefer temporary credentials for shared or short-lived machines.
   - If a key was pasted into a shared channel or committed by mistake, revoke and rotate it before continuing.

Official references: [tosutil quick start](https://www.volcengine.com/docs/6349/152743) and [configuration file fields](https://www.volcengine.com/docs/6349/152766).

### Manual Install

For local development before pushing to GitHub, copy or symlink a skill directory into your Codex skills directory:

```bash
mkdir -p "${CODEX_HOME:-$HOME/.codex}/skills"
ln -s "$PWD/skills/benchmark-eval" "${CODEX_HOME:-$HOME/.codex}/skills/benchmark-eval"
ln -s "$PWD/skills/data-connection-guide" "${CODEX_HOME:-$HOME/.codex}/skills/data-connection-guide"
ln -s "$PWD/skills/gpu-repo-sync" "${CODEX_HOME:-$HOME/.codex}/skills/gpu-repo-sync"
```

The shared skills intentionally use placeholders for hosts, paths, buckets, mirrors, and credentials. Keep site-specific values in local environment variables, SSH config, project notes, or a private overlay that is not committed.
