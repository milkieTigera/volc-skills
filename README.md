# volc-skills

Shared Codex skills for Volcengine and GPU workflow operations.

## Layout

```text
volc-skills/
├── README.md
├── skills/
│   └── data-connection-guide/
│       ├── SKILL.md
│       └── agents/
│           └── openai.yaml
└── .gitignore
```

Each installable skill lives under `skills/<skill-name>/` and keeps its required `SKILL.md` plus optional `agents/`, `scripts/`, `references/`, and `assets/` resources.

## Included Skills

- `data-connection-guide`: a sanitized router for choosing data-transfer paths between local staging, object storage, offline GPU storage, and direct-download GPU hosts.

## Install Locally

Copy or symlink a skill directory into your Codex skills directory:

```bash
mkdir -p "${CODEX_HOME:-$HOME/.codex}/skills"
ln -s "$PWD/skills/data-connection-guide" "${CODEX_HOME:-$HOME/.codex}/skills/data-connection-guide"
```

The shared skill intentionally uses placeholders for hosts, paths, buckets, mirrors, and credentials. Keep site-specific values in local environment variables, SSH config, project notes, or a private overlay that is not committed.
