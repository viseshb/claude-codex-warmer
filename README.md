# LLM_Cron — subscription usage-window warmer

Fires one throwaway prompt at **Claude Code** (Pro/Max subscription) and **Codex**
(ChatGPT subscription) every morning via GitHub Actions cron.

**Why:** both providers start their rolling **5-hour usage window on the first
message**. Warming early means the window is already ticking when you sit down to
work — and later resets land inside your workday.

**No API keys.** Uses subscription OAuth tokens only.

---

## What was created

```
.github/workflows/warm.yml   # cron + manual workflow, 2 warm jobs + heartbeat
.gitignore                   # blocks auth.json / secrets from being committed
README.md                    # this file
heartbeat.txt                # auto-created by the workflow each run (keepalive)
```

`warm.yml` has three jobs:

| Job          | What it does                                                            |
|--------------|------------------------------------------------------------------------|
| `warm-claude`| installs Claude Code CLI, sends one Haiku 4.5 prompt using your token   |
| `warm-codex` | installs Codex CLI, restores `auth.json`, sends one gpt-5.4-mini prompt |
| `heartbeat`  | commits a timestamp so GitHub never auto-disables the cron (60-day rule)|

---

## Secrets to add

Repo → **Settings → Secrets and variables → Actions → New repository secret**.

| Secret name              | Required | How to get it                                                        |
|--------------------------|----------|----------------------------------------------------------------------|
| `CLAUDE_CODE_OAUTH_TOKEN`| yes      | run `claude setup-token` locally, paste the printed token            |
| `CODEX_AUTH_JSON`        | yes      | run `codex login` locally, then paste the full contents of `~/.codex/auth.json` |
| `GH_PAT`                 | optional | PAT with **Secrets: write** — lets Codex write its refreshed token back (see below) |

> These come from **your own** subscription logins. Nobody — including this repo's
> author — can generate them for you; they require the interactive browser login.

### Getting the Claude token
```bash
npm install -g @anthropic-ai/claude-code
claude setup-token          # opens browser, prints a long-lived OAuth token
```

### Getting the Codex auth
```bash
npm install -g @openai/codex
codex login                 # opens browser, writes ~/.codex/auth.json
cat ~/.codex/auth.json      # copy the whole JSON into the CODEX_AUTH_JSON secret
```

---

## Push the repo

Already pushed if you used the automated setup. Manual version:

```bash
cd LLM_Cron
git init
git add .
git commit -m "init: subscription usage-window warmer"
git branch -M main
gh repo create LLM_Cron --private --source=. --push
```

Scheduled workflows run **only on the default branch** (`main`), so make sure this
is pushed to `main`.

---

## Test it manually

1. Add at least `CLAUDE_CODE_OAUTH_TOKEN` and `CODEX_AUTH_JSON`.
2. Repo → **Actions → warm-usage-windows → Run workflow**.
   - or CLI: `gh workflow run warm.yml`
3. Watch it: `gh run watch` (or the Actions tab).
4. Green `warm-claude` / `warm-codex` = a real message hit each subscription =
   window started. `heartbeat` commits a timestamp.

Before secrets are added the warm jobs fail at the auth step — that is expected and
proves the wiring is correct.

---

## How the schedule works

```yaml
on:
  schedule:
    - cron: "0 11 * * *"   # 06:00 CDT (summer, UTC-5)
    - cron: "0 12 * * *"   # 06:00 CST (winter, UTC-6)
  workflow_dispatch:
```

- **UTC only.** GitHub cron has no timezone/DST. The two lines cover both halves of
  the year; the extra warm during the "wrong" season is harmless (it just sends a
  second message inside an already-open window — it does **not** reset it).
- **Late runs.** GitHub scheduled jobs are routinely 5–15 min late and occasionally
  skipped under heavy load. Fine for warming; do not rely on second-level precision.
- **Change your time:** edit the `cron:` lines. Format is `minute hour * * *` in UTC.
  e.g. 6am in UTC-5 → `0 11 * * *`.

---

## The Codex fragility (read this)

Unlike Claude — which has a purpose-built `setup-token` for headless use — Codex has
**no official subscription-in-CI path**. We restore `~/.codex/auth.json` from a secret.

`auth.json` holds a short-lived access token plus a refresh token. When Codex
refreshes, the OAuth server may **rotate** (invalidate) the old refresh token. In CI
the runner is ephemeral, so the rotated token is lost — the next run would fail auth.

**Mitigation:** the `warm-codex` job writes the refreshed `auth.json` back to the
`CODEX_AUTH_JSON` secret after each run, using the optional `GH_PAT`. This keeps the
secret current across runs.

- **With `GH_PAT`:** cron should keep working unattended.
- **Without `GH_PAT`:** if the refresh token is single-use, you may need to re-run
  `codex login` and update `CODEX_AUTH_JSON` every few days.

If Codex proves too flaky, delete the `warm-codex` job — `warm-claude` is independent
and rock-solid.

---

## Extending

Add another provider = add another job to `warm.yml` (copy `warm-claude`, swap the
CLI + secret). Keep `heartbeat` as the final `needs: [...]` job.
