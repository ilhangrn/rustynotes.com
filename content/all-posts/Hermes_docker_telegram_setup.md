+++
title = "Hermes Agent with Docker and Telegram"
date = 2026-06-26
[taxonomies]
categories=["Guides"]
tags=["Hermes", "Docker", "Telegram", "AI", "Setup", "Guide"]
+++
---
<br>

## (*Eng*) Hermes Agent with Docker and Telegram

> Hermes Agent is a terminal-based AI assistant from Nous Research. It runs models locally or through providers like Nous, OpenRouter, Anthropic. This guide covers containerized setup with Telegram integration.

**Why Docker.**

Hermes needs persistent storage for memories, skills, cron jobs, and gateway state. Running it raw on the host works, but Docker gives you clean filesystem isolation, reproducible mounts, and easy migration between machines.

**The directory structure.**

```
hermes-docker/
└── docker-compose.yml         # single file, everything else is mounted
```

Hermes reads config from `~/.hermes/config.yaml` by default. In Docker, you mount your `.hermes` directory into the container so the agent sees the same state you configured on the host.

**docker-compose.yml breakdown.**

```yaml
services:
  hermes:
    image: hermes-agent:latest
    container_name: hermes
    restart: unless-stopped
    volumes:
      - /home/user/.hermes:/opt/data
      - /home/user/.git-credentials:/opt/data/.git-credentials
      - /home/user/.ssh:/opt/data/.ssh
      - /home/user/ig_one:/ig_one
    environment:
      - HOME=/opt/data/profiles/teacher/home
    stdin_open: true
    tty: true
```

**Key mounts.**

| Mount | Container path | Purpose |
|---|---|---|
| `.hermes` | `/opt/data` | Config, skills, cron jobs, gateway state |
| `.git-credentials` | `/opt/data/.git-credentials` | Auth for git push to your repos |
| `.ssh` | `/opt/data/.ssh` | SSH keys for git or server access |
| project files | `/ig_one` | Your code, learning repos, blog content |

**The HOME trick.**

Hermes profiles override `HOME` to `$HERMES_HOME/profiles/<name>/home`. Git looks for `.gitconfig` and `.git-credentials` in `$HOME`. Mounting the credential file directly is not enough — git looks in the profile's home, not the container root.

Solution: symlink the mounted files into the profile home.

```bash
ln -sf /opt/data/.gitconfig /opt/data/profiles/teacher/home/.gitconfig
ln -sf /opt/data/.git-credentials /opt/data/profiles/teacher/home/.git-credentials
```

Or pass them per-command:

```bash
git -c credential.helper='store --file=/opt/data/.git-credentials' push
```

**Telegram gateway.**

Hermes connects to Telegram through its gateway system. Configure in `config.yaml`:

```yaml
gateway:
  telegram:
    enabled: true
    bot_token: "YOUR_BOT_TOKEN"
    allowed_users:
      - "your_telegram_id"
```

The bot token comes from [@BotFather](https://t.me/BotFather) on Telegram. `allowed_users` restricts who can talk to your agent — set this to your Telegram user ID (get it from [@userinfobot](https://t.me/userinfobot)).

**Why Telegram over CLI.**

- The agent can notify you asynchronously — cron job results, watchdogs, daily briefings
- You can send commands from your phone and get results back
- Long-running tasks (delegated subagents, builds) deliver results to your chat instead of a terminal you closed

**Cron jobs with delivery.**

```bash
hermes cron create \
  --schedule "0 9 * * *" \
  --prompt "Check my GitHub for new PRs and summarize" \
  --deliver telegram
```

The `--deliver` flag routes the output to your configured gateway instead of the terminal. Combine with `deliver=all` to send everywhere.

**Practical workflow.**

Start the container, attach to the terminal session for interactive work, and let Telegram handle async notifications.

```bash
docker compose up -d
docker attach hermes  # interactive session
```

When you detach (Ctrl+P Ctrl+Q), the agent is still running. Cron jobs fire on schedule. Gateway messages arrive in Telegram. Reattach later to continue.

**The credential problem.**

Docker bind mounts are owned by root inside the container by default. Git's credential store tries to write-lock the file when reading or updating it. This produces:

```
fatal: unable to write credential store: Device or resource busy
```

The credential is still read successfully — the push or fetch works. The warning is cosmetic. If you want to silence it, switch your git remotes to SSH instead of HTTPS.

**What you end up with.**

- A reproducible Hermes environment you can tear down and recreate in one command
- Telegram as your notification channel — cron results, build completions, delegated task results
- Your project files accessible from inside the container at predictable paths
- Git push working through mounted credentials or SSH keys
- The same config, skills, and cron jobs survive container restarts because they live on the host filesystem

**Next steps.**

Once the container is running and Telegram is connected, try:

1. Ask the agent to set a cron job for a daily briefing delivered to Telegram
2. Mount a project repo and have the agent review your code
3. Write a custom skill that the agent auto-loads by name

The combination of Docker isolation + Telegram async access + terminal depth makes Hermes feel like an always-on junior developer you can ping from anywhere.
