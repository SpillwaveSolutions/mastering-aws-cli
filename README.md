# Mastering AWS CLI

A comprehensive Claude Code skill for AWS CLI v2 quick-reference, designed for experienced developers.

## Overview

This skill provides instant access to AWS CLI commands, patterns, and best practices. It covers compute, storage, networking, security, and CI/CD integration with GitHub Actions.

## Installing with Skilz (Universal Installer)

The recommended way to install this skill across different AI coding agents is using the **skilz** universal installer.

### Install Skilz

```bash
pip install skilz
```

This skill supports [Agent Skill Standard](https://agentskills.io/) which means it supports 14 plus coding agents including Claude Code, OpenAI Codex, Cursor and Gemini.


### Git URL Options

You can use either `-g` or `--git` with HTTPS or SSH URLs:

```bash
# HTTPS URL
skilz install -g https://github.com/SpillwaveSolutions/mastering-aws-cli

# SSH URL
skilz install --git git@github.com:SpillwaveSolutions/mastering-aws-cli.git
```

### Claude Code

Install to user home (available in all projects):
```bash
skilz install -g https://github.com/SpillwaveSolutions/mastering-aws-cli
```

Install to current project only:
```bash
skilz install -g https://github.com/SpillwaveSolutions/mastering-aws-cli --project
```

### OpenCode

Install for [OpenCode](https://opencode.ai):
```bash
skilz install -g https://github.com/SpillwaveSolutions/mastering-aws-cli --agent opencode
```

Project-level install:
```bash
skilz install -g https://github.com/SpillwaveSolutions/mastering-aws-cli --project --agent opencode
```

### Gemini

Project-level install for Gemini:
```bash
skilz install -g https://github.com/SpillwaveSolutions/mastering-aws-cli --agent gemini
```

### OpenAI Codex

Install for OpenAI Codex:
```bash
skilz install -g https://github.com/SpillwaveSolutions/mastering-aws-cli --agent codex
```

Project-level install:
```bash
skilz install -g https://github.com/SpillwaveSolutions/mastering-aws-cli --project --agent codex
```


### Install from Skillzwave Marketplace
```
# Claude to user home dir ~/.claude/skills
skilz install SpillwaveSolutions_mastering-aws-cli/mastering-aws-cli

# Claude skill in project folder ./claude/skills
skilz install SpillwaveSolutions_mastering-aws-cli/mastering-aws-cli --project

# OpenCode install to user home dir ~/.config/opencode/skills
skilz install SpillwaveSolutions_mastering-aws-cli/mastering-aws-cli --agent opencode

# OpenCode project level
skilz install SpillwaveSolutions_mastering-aws-cli/mastering-aws-cli --agent opencode --project

# OpenAI Codex install to user home dir ~/.codex/skills
skilz install SpillwaveSolutions_mastering-aws-cli/mastering-aws-cli

# OpenAI Codex project level ./.codex/skills
skilz install SpillwaveSolutions_mastering-aws-cli/mastering-aws-cli --agent opencode --project


# Gemini CLI (project level) -- only works with project level
skilz install SpillwaveSolutions_mastering-aws-cli/mastering-aws-cli --agent gemini

```

See this site [skill Listing](https://skillzwave.ai/skill/SpillwaveSolutions__mastering-aws-cli__mastering-aws-cli__SKILL/) to see how to install this exact skill to 14+ different coding agents.


### Other Supported Agents

Skilz supports 14+ coding agents including Windsurf, Qwen Code, Aidr, and more.

For the full list of supported platforms, visit [SkillzWave.ai/platforms](https://skillzwave.ai/platforms/) or see the [skilz-cli GitHub repository](https://github.com/SpillwaveSolutions/skilz-cli)


## Manual Installation

Copy this skill to your Claude Code skills directory:

```bash
# User-level installation
cp -r mastering-aws-cli ~/.claude/skills/

# Or create a symlink
ln -s "$(pwd)" ~/.claude/skills/mastering-aws-cli
```

## Usage

The skill activates automatically when you mention AWS-related topics:

- "How do I assume an IAM role?"
- "Show me ECS deployment commands"
- "Set up GitHub Actions with AWS OIDC"

## Coverage

| Category | Services |
|:---------|:---------|
| **Compute** | Lambda, ECS, EKS, EC2 |
| **Storage** | S3, DynamoDB, Aurora/RDS |
| **Streaming** | MSK (Kafka), Kinesis |
| **Data** | Glue (ETL/Catalog) |
| **Security** | IAM, Secrets Manager, SSM Parameter Store |
| **Networking** | VPC, Security Groups, SSM Tunneling |
| **CI/CD** | GitHub Actions, OIDC Federation |

## Structure

```
mastering-aws-cli/
├── SKILL.md                    # Main skill definition
├── README.md                   # This file
└── references/
    ├── setup.md                # Installation, SSO, profiles
    ├── iam-security.md         # Roles, policies, STS
    ├── lambda.md               # Serverless functions
    ├── ecs.md                  # Container orchestration
    ├── eks.md                  # Kubernetes
    ├── ecr.md                  # Container registry
    ├── s3.md                   # Object storage
    ├── dynamodb.md             # NoSQL database
    ├── aurora.md               # Relational databases
    ├── glue.md                 # ETL and data catalog
    ├── msk.md                  # Managed Kafka
    ├── kinesis.md              # Data streams
    ├── vpc-networking.md       # VPC and networking
    ├── bastion-tunneling.md    # SSM tunneling
    ├── github-cicd.md          # GitHub Actions integration
    └── advanced-patterns.md    # JMESPath, waiters, aliases
```

## Triggers

The skill responds to these keywords:
- Services: `lambda`, `ecs`, `eks`, `ecr`, `s3`, `dynamodb`, `aurora`, `rds`, `glue`, `msk`, `kinesis`
- Security: `iam`, `sts`, `assume role`, `secrets manager`, `parameter store`
- Networking: `vpc`, `bastion`, `ssm tunnel`
- Setup: `aws configure`, `aws sso`
- CI/CD: `github actions aws`, `oidc aws`

## Version

- **Version:** 2.0.0
- **Author:** Spillwave
- **License:** MIT

## Contributing

1. Fork this repository
2. Add or update reference files in `references/`
3. Update `SKILL.md` navigation if adding new files
4. Submit a pull request

## Related Skills

- `mastering-gcloud-cli` - Google Cloud CLI reference
- `gcloud-expert` - Advanced GCP patterns

---

<a href="https://skillzwave.ai/">Largest Agentic Marketplace for AI Agent Skills</a> and
<a href="https://spillwave.com/">SpillWave: Leaders in AI Agent Development.</a>
