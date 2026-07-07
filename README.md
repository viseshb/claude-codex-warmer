# LLM_Cron

Keep your **Claude Code** and **Codex** subscription usage windows warm with a
scheduled GitHub Actions cron — no laptop, no API keys, no extra billing.

## Why this exists

Both Claude Code (Anthropic subscription) and Codex (ChatGPT subscription) run
usage on a **rolling window that starts on your first message** (commonly ~5
hours). If that window only starts when you happen to sit down and type
something, you waste the first chunk of it just getting started.

This repo re-sends one tiny "hi" to each assistant **every ~5 hours, around
the clock** — each message starts a fresh window the moment the previous one
expires, so a window is already running whenever you open the app, no matter
what time of day you start working.

> **Why not just once each morning?** That was v1 of this repo, and it failed
> in practice: a warm-up at 7am starts a window that *expires at noon*. Sit
> down at 1pm and your first message starts a brand-new window anyway — the
> morning warm did nothing. Since the window begins on the first message and
> dies 5h later, the only way to have it "always ready" is to chain warms
> back-to-back all day. Cost: ~5 trivial messages per provider per day, which
> is noise against any subscription limit.

**No API keys required.** Both jobs authenticate with your existing subscription
(OAuth token / CLI login), not pay-per-token API billing.

---

## How it works

One workflow, three jobs:

| Job          | Does |
|--------------|------|
| `warm-claude`| Installs the Claude Code CLI, sends `"hi"` to `claude-haiku-4-5` using your subscription token |
| `warm-codex` | Installs the Codex CLI, restores your ChatGPT login, sends `"hi"` to `gpt-5.4-mini` at low reasoning effort |
| `heartbeat`  | Commits a timestamp file once per day, so GitHub never auto-disables the schedule (see "60-day rule" below) |

The cron fires every 30 minutes, 24/7. Each warm job reads a committed
per-provider state file (`.state/*-last-warm.txt`, epoch seconds) and only
sends for real once the last warm is **more than 5h01m old** — i.e. strictly
after the previous window has expired. Every other trigger is a ~5 second
no-op (checkout + arithmetic, nothing installed, no API call).

The 1-minute margin matters: a message sent *before* the current window
expires just counts inside it and does **not** start a new window — it would
be a wasted send that silently breaks the chain. Sending strictly after
expiry guarantees each warm opens a fresh window.

Manually triggering the workflow (`workflow_dispatch`) always sends for real,
regardless of elapsed time — useful for testing.

---

## Setup

### 1. Use this repo
Click **Use this template** (or fork/clone) and push it to your own GitHub account.
It must be its **own repository** — scheduled workflows only run on the default
branch of the repo that defines them. Keep it **public** if you want unlimited
Actions minutes (a private repo burns through the free tier at this trigger
frequency).

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

### 3. (Optional) Timezone

The warm gate is epoch-based and **timezone-independent** — nothing to
configure for scheduling. The `TZ='America/Chicago'` in the workflow only
formats commit messages; change it if you want those stamps in your local time.

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
    - cron: "*/30 * * * *"   # every 30 min, 24/7 — elapsed-time gate limits real sends to ~5/day
  workflow_dispatch:
```

GitHub cron is best-effort: on a low-traffic repo, scheduled runs get skipped
or land 60-90+ minutes late (observed; their docs claim "5-15 minutes"). With
an elapsed-time gate that's harmless — whichever run lands first after the
5h01m threshold sends for real and records the new timestamp, and the chain
self-corrects. Worst case, a heavily delayed slot leaves a gap of roughly the
delay between one window expiring and the next warm; your own first message
would start a window in that gap exactly as it would without this repo, so
you never end up worse off.

State lives in `.state/claude-last-warm.txt` and `.state/codex-last-warm.txt`
as epoch seconds, committed after each real send.

---

## The 60-day auto-disable rule

GitHub automatically disables a scheduled workflow after **60 days with no
repository activity**. A repo whose only purpose is running cron jobs would
otherwise silently stop with no error. The warm jobs already commit ~5x/day,
but the `heartbeat` job commits a timestamp once per day even when both warm
jobs fail — so the schedule stays alive (and failures stay visible) no matter
what.

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
  Rulesets) blocking force-push and deletion, since the workflow pushes
  directly to it.

---

## Extending

Add another scheduled assistant/provider by copying the `warm-claude` job:
install its CLI, restore its auth from a secret, reuse the same elapsed-time
gate against a new `.state/<provider>-last-warm.txt` file, and list it in
`needs:` on the `heartbeat` job.
