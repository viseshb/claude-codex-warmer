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

The schedule fires every 15 minutes across a wide early-morning UTC window (to
survive GitHub's cron delay, which can run 60-90+ minutes late in practice, not
the "5-15 min" the docs suggest). Each job checks a committed per-provider
state file and only sends for real on the **first run of the calendar day**;
every other trigger that day is a cheap no-op. Net result: **exactly one
Claude call + one Codex call per day**, as early as GitHub actually gets to
it — never zero, never more than one each.

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

Both `warm-claude` and `warm-codex` jobs stamp their dedupe state with:
```bash
TZ="America/Chicago" date +%F
```
Change `America/Chicago` to your [IANA timezone](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones)
(e.g. `America/New_York`, `Europe/London`, `Asia/Kolkata`) in both jobs, and
shift the `cron:` window (see "Schedule reference" below) so it brackets your
target wake-up hour in UTC.

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
    - cron: "*/15 9-13 * * *"   # every 15 min across a wide early-morning window
  workflow_dispatch:
```

GitHub's own docs say scheduled runs are "5-15 minutes late." In practice, on a
low-traffic repo, we observed **60-90+ minutes** of delay on day one. A single
fixed-time trigger cannot survive that — it either fires too late or, if you
add a guard that checks the exact hour, the guard itself can end up rejecting
every delayed run and nothing gets sent at all (this happened; see commit
history).

The fix used here: fire **every 15 minutes across a wide window**
(`9-13` UTC = ~03:00-08:45 CDT / ~02:00-07:45 CST — comfortably brackets a
6am-ish wake-up under heavy delay), and let each job's committed state file
(`.state/claude-last-warm.txt`, `.state/codex-last-warm.txt`) decide whether
today's real message has already gone out. The first run of the day that
actually executes sends for real and records today's date; every other
trigger that day is a ~5 second no-op (checkout + date compare, nothing
installed, no API call). Net effect: **exactly one real Claude call + one
real Codex call per calendar day**, as early as GitHub actually runs the
workflow — never zero, never more than one each.

To retarget: shift the `9-13` hour range in UTC so it brackets your desired
local wake-up time with a few hours of slack in both directions, and update
the `TZ="America/Chicago"` timezone in both jobs (see step 3 above).

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
