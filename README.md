# LLM_Cron

Keep your **Claude Code** and **Codex** subscription usage windows warm with a
scheduled GitHub Actions cron — no laptop, no API keys, no extra billing.

## Why this exists

Both Claude Code (Anthropic subscription) and Codex (ChatGPT subscription) run
usage on a **rolling window that starts on your first message of the period**
(commonly ~5 hours). If that window only starts when you happen to sit down and
type something, you waste the first chunk of it just getting started.

This repo sends one tiny "hi" to each assistant on a schedule (e.g. right before
your workday starts), so the window is already warm by the time you're ready to
work — and any later resets tend to land inside your working hours instead of
overnight.

**No API keys required.** Both jobs authenticate with your existing subscription
(OAuth token / CLI login), not pay-per-token API billing.

---

## How it works

One workflow, three jobs:

| Job          | Does |
|--------------|------|
| `warm-claude`| Installs the Claude Code CLI, sends `"hi"` to `claude-haiku-4-5` using your subscription token |
| `warm-codex` | Installs the Codex CLI, restores your ChatGPT login, sends `"hi"` to `gpt-5.4-mini` at low reasoning effort |
| `heartbeat`  | Commits a timestamp file each run, so GitHub never auto-disables the schedule (see "60-day rule" below) |

The schedule fires **twice** in UTC to cover both sides of Daylight Saving Time,
but each job checks the real local hour at runtime and only sends a real message
when it's actually your target hour — the other trigger is a harmless no-op. Net
result: **exactly one Claude call + one Codex call per day**, always at the right
local time, year-round, with zero manual DST bookkeeping.

Manually triggering the workflow (`workflow_dispatch`) always sends for real,
regardless of the time — useful for testing.

---

## Setup

### 1. Use this repo
Click **Use this template** (or fork/clone) and push it to your own GitHub account.
It must be its **own repository** — scheduled workflows only run on the default
branch of the repo that defines them.

```bash
git clone <your-fork-url>
cd LLM_Cron
git push origin main
```

### 2. Add secrets

Repo → **Settings → Secrets and variables → Actions → New repository secret**.

| Secret                    | Required | How to get it |
|---------------------------|----------|----------------|
| `CLAUDE_CODE_OAUTH_TOKEN` | yes      | `claude setup-token` (run locally in a real terminal — opens a browser) |
| `CODEX_AUTH_JSON`         | yes      | `codex login` (run locally — opens a browser), then copy `~/.codex/auth.json` |
| `GH_PAT`                  | optional | Personal Access Token with **Contents: read/write** on this repo — lets the Codex job write back its refreshed token (see "Codex fragility" below) |

Setting them:

```bash
# Claude — run setup-token in your OWN terminal, not through an agent/CI shell
claude setup-token
gh secret set CLAUDE_CODE_OAUTH_TOKEN --repo <owner>/<repo>

# Codex — run login in your OWN terminal
codex login
gh secret set CODEX_AUTH_JSON --repo <owner>/<repo> < ~/.codex/auth.json
# PowerShell equivalent (no `<` redirection support):
# Get-Content -Raw "$env:USERPROFILE\.codex\auth.json" | gh secret set CODEX_AUTH_JSON --repo <owner>/<repo>
```

`setup-token` / `login` need a real interactive terminal (they open a browser and
wait for the OAuth callback) — they will hang if run from a non-interactive shell
or a nested/agent session.

### 3. Set your local timezone in the workflow

Both `warm-claude` and `warm-codex` jobs guard on:
```bash
hour=$(TZ="America/Chicago" date +%H)
```
Change `America/Chicago` to your [IANA timezone](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones)
(e.g. `America/New_York`, `Europe/London`, `Asia/Kolkata`), and change the `"06"`
comparison to your target hour (24h, zero-padded).

### 4. Test it

```bash
gh workflow run warm.yml --repo <owner>/<repo>
gh run watch --repo <owner>/<repo>
```

Manual runs always send for real, so this is a true end-to-end test regardless
of the current time. Green `warm-claude` + `warm-codex` = both subscriptions
received a real message = window warmed.

---

## Schedule reference

```yaml
on:
  schedule:
    - cron: "0 11 * * *"
    - cron: "0 12 * * *"
  workflow_dispatch:
```

- GitHub Actions cron is **UTC-only** with no DST awareness — that's why there
  are two lines an hour apart, one of which will always land on your target
  local hour depending on the season. The runtime guard (`TZ=... date +%H`)
  decides which one actually fires.
- Adjust the two `cron:` lines so they bracket your target local hour in UTC
  across both DST states. Format is `minute hour * * *`.
- Scheduled runs are commonly 5–15 minutes late; don't rely on second-level
  precision.

---

## The 60-day auto-disable rule

GitHub automatically disables a scheduled workflow after **60 days with no
repository activity**. A repo whose only purpose is running cron jobs would
otherwise silently stop with no error. The `heartbeat` job commits a small
timestamp file on every run, which counts as activity and keeps the schedule
alive indefinitely.

---

## Codex fragility (read before relying on this)

Anthropic ships `claude setup-token` specifically for headless/CI use — that
path is solid. OpenAI has **no equivalent** for Codex; there's no official
"subscription token for CI." This repo works around that by restoring
`~/.codex/auth.json` from a secret each run.

That file contains a short-lived access token and a refresh token. When Codex
refreshes the access token, the OAuth server may **rotate** the refresh token,
invalidating the old one. Since CI runners are ephemeral, a rotated token is
lost unless something writes it back — so `warm-codex` does that automatically
via `GH_PAT`, if you set it.

- **With `GH_PAT`:** should keep authenticating indefinitely.
- **Without `GH_PAT`:** if the refresh token turns out to be single-use, you'll
  need to re-run `codex login` and update `CODEX_AUTH_JSON` every so often.

If this proves too flaky for your use, delete the `warm-codex` job — `warm-claude`
is fully independent and doesn't share this problem.

---

## Security notes

- No API keys, no long-lived plaintext credentials in the repo — only encrypted
  GitHub Actions secrets.
- `.gitignore` blocks local `auth.json` / `.env` / credential files from ever
  being committed.
- GitHub automatically masks secret values in Action logs.
- Consider adding a branch protection ruleset on `main` (Settings → Rules →
  Rulesets) blocking force-push and deletion, since the `heartbeat` job pushes
  directly to it.

---

## Extending

Add another scheduled assistant/provider by copying the `warm-claude` job:
install its CLI, restore its auth from a secret, add the same local-hour guard,
and list it in `needs:` on the `heartbeat` job.
