# volc-skills

Shared Codex skills for Volcengine and GPU workflow operations.

## Layout

```text
volc-skills/
├── README.md
├── skills/
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

- `data-connection-guide`: a sanitized router for choosing data-transfer paths between local staging, object storage, offline GPU storage, and direct-download GPU hosts.
- `gpu-repo-sync`: a sanitized router for syncing Git repositories and worktrees across local workstations, object storage, offline GPU hosts, and direct-download GPU hosts.

## Installation

### With Skill Installer

After this repository is pushed to GitHub, install all shared skills from Codex with the `skill-installer` skill:

```text
Use $skill-installer to install data-connection-guide and gpu-repo-sync from <github-owner>/volc-skills at skills/data-connection-guide and skills/gpu-repo-sync
```

Equivalent helper command:

```bash
python3 "${CODEX_HOME:-$HOME/.codex}/skills/.system/skill-installer/scripts/install-skill-from-github.py" \
  --repo <github-owner>/volc-skills \
  --path skills/data-connection-guide skills/gpu-repo-sync
```

To install only one skill, pass only that path:

```bash
python3 "${CODEX_HOME:-$HOME/.codex}/skills/.system/skill-installer/scripts/install-skill-from-github.py" \
  --repo <github-owner>/volc-skills \
  --path skills/gpu-repo-sync
```

For a branch or tag other than `main`, add `--ref <branch-or-tag>`.

For a private GitHub repo, make sure existing git credentials work or set `GITHUB_TOKEN`/`GH_TOKEN` before installing. Restart Codex after installation so the new skill is discovered.

No extra repository manifest is required. The important part is that the installer path points to a directory containing `SKILL.md`.

### Manual Install

For local development before pushing to GitHub, copy or symlink a skill directory into your Codex skills directory:

```bash
mkdir -p "${CODEX_HOME:-$HOME/.codex}/skills"
ln -s "$PWD/skills/data-connection-guide" "${CODEX_HOME:-$HOME/.codex}/skills/data-connection-guide"
ln -s "$PWD/skills/gpu-repo-sync" "${CODEX_HOME:-$HOME/.codex}/skills/gpu-repo-sync"
```

The shared skills intentionally use placeholders for hosts, paths, buckets, mirrors, and credentials. Keep site-specific values in local environment variables, SSH config, project notes, or a private overlay that is not committed.
