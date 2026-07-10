# claude-codex-warmer

**Keep your Claude Code and Codex usage windows always warm - from a free GitHub
Actions cron. No laptop, no API keys, no extra billing.**

Both Claude (Anthropic Pro/Max subscription) and Codex (ChatGPT subscription)
meter usage in a **rolling ~5-hour window that starts on your first message**.
If the window only starts when you sit down and type, you burn the first part
of every session just getting the clock going - and the reset lands at an
awkward time.

claude-codex-warmer sends one tiny `"hi"` to each assistant **every ~5 hours, around the
clock**. Each message starts a fresh window the moment the previous one
expires, so whenever you open Claude or Codex, a window is already running.

Cost: ~5 messages per provider per day, each about 4,500 tokens (almost all of
it the CLI's own system prompt) - a fraction of a percent of any window's
budget. Runs entirely on GitHub's servers; your machine can be off.

> **Why not one warm-up each morning?** That was v1, and it failed in practice:
> a 7 AM warm-up starts a window that dies at noon. Sit down at 1 PM and your
> first message starts a brand-new window anyway. Because a window can't be
> extended or pre-booked - it starts on the first message *after* the previous
> one expires - the only way to have it "always ready" is to chain warms
> back-to-back all day.

---

## How it works

One workflow ([`.github/workflows/warm.yml`](.github/workflows/warm.yml)),
four jobs:

| Job          | What it does |
|--------------|--------------|
| `warm-claude`| Installs the Claude Code CLI, sends `"hi"` to `claude-haiku-4-5` using your subscription OAuth token |
| `warm-codex` | Installs the Codex CLI, restores your ChatGPT login, sends `"hi"` to `gpt-5.6-sol` at low reasoning effort (NOT the mini model - see FAQ) |
| `notify`     | If a warm job genuinely fails, opens one GitHub issue with the exact fix commands (GitHub emails you); auto-closes it once runs succeed again |
| `heartbeat`  | Commits a timestamp once per day so GitHub never auto-disables the schedule (see FAQ) |

The cron triggers every 30 minutes, 24/7 (at :07 and :37, deliberately off
the congested :00/:30 marks where GitHub delays cron jobs the most). Each
warm job then asks the **account's own usage meter** (the same API your apps
read) whether a 5-hour window is already running:

- Window active: skip. This includes windows **you** started yourself from
  any device - the meter sees them, so the warm-up never wastes a send
  inside them.
- No window: send one `"hi"`, which starts a fresh window.

If the usage endpoint is ever unreachable, the job falls back to a committed
state file (`.state/<provider>-last-warm.txt`, epoch seconds) and only sends
once its own previous warm is more than 5 h 01 m old. Either way, a skipped
trigger is a cheap no-op.

Why never send into an active window: a message sent *before* the current
window expires just counts inside it and does **not** start a new window -
it would be a wasted send.

Manually triggering the workflow (**Actions → warm-usage-windows → Run
workflow**) always sends for real regardless of timing - that's your
end-to-end test button.

### Built to be left alone

- **Warm-up calls retry 3x** with a short backoff before counting as failed,
  so a transient network or API blip never surfaces as a failure.
- **CLI versions are pinned** (`CLAUDE_CLI_VERSION` / `CODEX_CLI_VERSION` in
  `warm.yml`) to the last verified-working releases, so an upstream update
  can't silently break a flag the workflow depends on.
- **Real failures notify you.** If a job still fails after retries - in
  practice, a revoked token - the `notify` job opens a single GitHub issue
  containing the copy-paste recovery commands. GitHub emails repo owners
  about new issues by default, so the failure finds you; you never have to
  poll the Actions tab. The issue auto-closes when a later run succeeds.

---

## Setup (10 minutes)

### 0. Prerequisites

- A GitHub account
- A Claude **Pro or Max** subscription (for `warm-claude`)
- A ChatGPT subscription with **Codex** access (for `warm-codex`)
- The `claude` and/or `codex` CLIs installed locally, plus the
  [GitHub CLI](https://cli.github.com) (`gh`) for setting secrets

Only want one of the two providers? Delete the other job from `warm.yml`
(and drop it from the `needs:` lists of the `notify` and `heartbeat` jobs) -
they're fully independent.

### 1. Get your own copy of this repo

Click **Use this template** (or fork it). It must be its **own repository** -
GitHub only runs scheduled workflows from the default branch of the repo that
defines them. Keep the repo **public** if you want unlimited free Actions
minutes; on a private repo this trigger frequency will eat the free tier.

### 2. Add your secrets

> ⚠️ **The single most common failure:** minting a token while logged into a
> different account than the one you actually use. The warm-up then happily
> heats *someone else's* window every day while yours stays cold, and every
> run still shows green. Before running the commands below, make sure the CLI
> is logged into the **same account** your Claude/Codex apps use.

```bash
# Claude - run in YOUR terminal (opens a browser; approve with the right account)
claude setup-token
gh secret set CLAUDE_CODE_OAUTH_TOKEN --repo <you>/<repo>
# paste the sk-ant-oat... token when prompted

# Codex - run in YOUR terminal (opens a browser; log in with the right account)
codex login
gh secret set CODEX_AUTH_JSON --repo <you>/<repo> < ~/.codex/auth.json
```

PowerShell equivalent for the Codex line (no `<` redirection):

```powershell
Get-Content -Raw "$env:USERPROFILE\.codex\auth.json" | gh secret set CODEX_AUTH_JSON --repo <you>/<repo>
```

Both commands need a real interactive terminal - they open a browser and wait
for an OAuth callback, so they hang inside CI shells or agent sessions.

| Secret                    | Required | Notes |
|---------------------------|----------|-------|
| `CLAUDE_CODE_OAUTH_TOKEN` | for Claude | From `claude setup-token`. Valid ~1 year. |
| `CODEX_AUTH_JSON`         | for Codex  | Contents of `~/.codex/auth.json` after `codex login`. Can expire - see FAQ. |
| `GH_PAT`                  | optional   | Fine-grained PAT with **Secrets: read/write** on this repo; lets the Codex job write its refreshed token back so the login survives rotation. |

### 3. Test it

```bash
gh workflow run warm.yml --repo <you>/<repo>
gh run watch --repo <you>/<repo>
```

Manual runs always send for real. Green `warm-claude` + `warm-codex` means
both accounts just received a message and their windows are running.

### 4. Verify it's hitting *your* account

The next morning, open Claude **before typing anything**. You should see a
session already in progress at 0% used, with a reset time that is *not* five
hours from now. If you still get "send a message to start" - see the FAQ
entry on wrong-account tokens.

---

## Configuration

- **Warm interval** - `WARM_INTERVAL_SECONDS` in `warm.yml` (default `18060`
  = 5 h 01 m). Don't set it below your provider's window length: a send that
  lands inside a still-active window is wasted and breaks the chain.
- **Quiet hours** - to skip overnight warms, narrow the cron (e.g.
  `7,37 12-23,0-4 * * *` ≈ 7 AM–11 PM US Central; GitHub cron is UTC). The
  elapsed-time gate self-corrects whenever triggers resume. The trade-off:
  your first window of the day starts at the first morning trigger instead of
  already running from overnight.
- **Models** - Claude warms with `claude-haiku-4-5` (cheapest, and it counts
  toward the meter). Codex warms with `gpt-5.6-sol` at low reasoning effort:
  NOT the cheapest, on purpose. The warm-up model must be one that registers
  on the account's 5-hour meter, otherwise the send never starts a window
  (see FAQ).
- **CLI versions** - pinned via `CLAUDE_CLI_VERSION` and `CODEX_CLI_VERSION`
  in `warm.yml`. To upgrade, bump the pin, run a manual test
  (`gh workflow run warm.yml`), and confirm it's green before walking away.
- **Timezone** - scheduling is epoch-based and timezone-independent. The
  `TZ='America/Chicago'` in the workflow only formats commit-message
  timestamps; change it for prettier logs, nothing else.

---

## FAQ / Troubleshooting

**Every run is green but my window is never warm.**
Two known causes:

1. *Wrong account.* Your token was minted from a different account than the
   one your apps use (multiple Google logins are the classic cause). The runs
   really are sending - to the wrong account, which no log can show you.
   Re-mint the secret from a terminal where the CLI is logged into the
   account you actually use (step 2), then re-verify (step 4).
2. *A model that does not touch the meter.* This project originally warmed
   Codex with `gpt-5.4-mini` to save tokens. The sends succeeded, returned
   replies, and reported token usage - and never started a single window,
   because mini-model usage bills a separate bucket that does not register
   on the account's 5-hour meter. If you change the warm-up model, verify a
   send actually pins your reset time before trusting it.

**How do I know this isn't happening to me right now?**
The gate itself is the canary: it reads your live meter every trigger and
logs either "window active (resets in Ns)" or "no active window - warming".
If runs keep logging "no active window" every 30 minutes, sends are not
registering and the safety floor is limiting the damage - check the model
and the account.

**Why is there sometimes a short gap right after a window expires?**
GitHub's cron is best-effort - on low-traffic repos, triggers regularly land
30–90 minutes late or get skipped. Whichever run lands first after the 5 h 01 m
threshold re-warms and the chain self-corrects. If you message inside a gap,
you start the window yourself - exactly what would happen without this repo,
so you're never worse off.

**How will I know if something breaks?**
You'll get an email. A real failure (all retries exhausted) makes the
`notify` job open a GitHub issue titled "Warm-up needs attention" with the
exact recovery commands for whichever provider broke, and GitHub notifies
repo owners about new issues by default. Fix the token, and the issue closes
itself on the next successful run. No dashboard-watching required.

**Codex suddenly stopped warming (401 errors).**
OpenAI has no official CI token for Codex, so this repo restores
`~/.codex/auth.json` from a secret. That file holds a refresh token which the
OAuth server may rotate or revoke; on an ephemeral runner a rotated token is
lost unless written back. Set the optional `GH_PAT` secret to enable
automatic write-back - without it, this failure is a matter of time (ours
died after ~30 hours). When it does break: re-run `codex login` locally and
update `CODEX_AUTH_JSON` (the failure issue contains these exact commands).
The Claude path doesn't have this problem - `claude setup-token` exists
precisely for headless use.

**"hi" cost 4,577 tokens?!**
Normal. Agent CLIs send their full system prompt, tool definitions, and
environment context with every request; your one-word prompt rides on top.
It's the floor cost of any real message, and it's under 1% of a window.

**Will this stop working if I don't touch the repo for two months?**
No - that's what `heartbeat` is for. GitHub disables scheduled workflows
after 60 days without repository activity; the daily heartbeat commit counts
as activity and also keeps job failures visible.

**Does this work for all my devices?**
Yes. Windows live on the *account*, not the device. Every app logged into the
warmed account (desktop, phone, web, CLI) sees the same running window.

---

## Security notes

- No API keys and no plaintext credentials in the repo - auth lives only in
  encrypted GitHub Actions secrets, and GitHub masks secret values in logs.
- [`.gitignore`](.gitignore) blocks local `auth.json` / `.env` / credential
  files from ever being committed.
- The workflow pushes state commits to `main`; consider a ruleset
  (Settings → Rules → Rulesets) blocking force-push and branch deletion.

---

## Extending to other providers

Copy the `warm-claude` job: install the provider's CLI (pinned version),
restore its auth from a secret, adapt the gate (query the provider's usage
endpoint if it has one; otherwise use the elapsed-time fallback against a new
`.state/<provider>-last-warm.txt`), and add the job to the `needs:` lists of
`notify` and `heartbeat`. Verify a warm send actually pins the account's
reset time - a green run alone proves nothing (see FAQ). Any assistant with a headless CLI and subscription
auth fits the pattern.

---

## License

[MIT](LICENSE) - use it, fork it, ship it. Attribution appreciated, not
required.
