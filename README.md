# OpenClaw 2026.5+ Update Reinstall

A field-tested recovery runbook for OpenClaw users who've upgraded
through several versions and ended up in a half-broken state — agent
generates replies in session storage but Discord channels never receive
them, TUI hangs at "connecting | idle", `discord: skipping guild
message reason: "no-mention"` for messages you know contain a real
mention, plugin-load thrash, etc.

This guide walks through a **clean reinstall that preserves the agent's
memory and session history**. The procedure was developed during a real
incident across `2026.4.27 → 2026.4.29 → 2026.5.2 → 2026.5.3-1` upgrades
and confirmed working on the OpenClaw 2026.5 line.

> **Why a clean install instead of more patches?**
> The actual root cause in the original incident wasn't any single
> config setting — it was layered cruft from multiple version upgrades:
> stale plugin runtime-deps from each version, leftover external plugin
> installs, accumulated config edits, and version-mismatch artifacts.
> Once the cruft was wiped, the *same* Discord token and guild config
> that had been broken for days started working immediately.

---

## Table of contents

- [When to use this](#when-to-use-this)
- [What's preserved vs discarded](#whats-preserved-vs-discarded)
- [Procedure](#procedure)
  - [Step 1 — full preservation backup](#step-1--full-preservation-backup)
  - [Step 2 — stop the gateway, kill stragglers](#step-2--stop-the-gateway-kill-stragglers)
  - [Step 3 — extract identity to staging (optional)](#step-3--extract-identity-to-staging-optional)
  - [Step 4 — wipe the cruft](#step-4--wipe-the-cruft)
  - [Step 5 — fresh install](#step-5--fresh-install)
  - [Step 6 — interactive setup (`onboard`, not `configure`)](#step-6--interactive-setup-onboard-not-configure)
  - [Step 7 — install the Discord plugin](#step-7--install-the-discord-plugin)
  - [Step 8 — restore identity (no-op if you did a surgical wipe)](#step-8--restore-identity-no-op-if-you-did-a-surgical-wipe)
  - [Step 8.5 — approve device pairing for TUI / web dashboard](#step-85--approve-device-pairing-for-tui--web-dashboard)
  - [Step 8.6 — paste gateway auth token into the web dashboard](#step-86--paste-gateway-auth-token-into-the-web-dashboard)
  - [Step 8.7 — restore optional features via the web dashboard](#step-87--restore-optional-features-via-the-web-dashboard)
  - [Step 9 — verify](#step-9--verify)
- [Troubleshooting after the clean install](#troubleshooting-after-the-clean-install)
- [Lessons learned](#lessons-learned)
- [About / credits](#about--credits)

---

## When to use this

You probably need a clean install if **two or more** of these are true:

- Agent generates a reply in session storage
  (`$HOME/.openclaw/agents/<name>/sessions/<id>.jsonl` shows a clean
  assistant message), but Discord channel never receives the text.
  Reactions/typing indicator may still show.
- Discord WS is *connected* (kernel socket established to a Discord IP)
  but no `logged in to discord` log line fires *and* messages still go
  through intermittently.
- `discord: skipping guild message` with `reason: "no-mention"` for
  messages that contain a real `@`-mention to the bot user ID.
- `Dropped stale background request after 95011ms` on outbound reaction
  or send calls, repeating for every dispatch.
- Plugin log line counts spike: thousands of `[plugins] loading <id>
  from ...` events per day with the same providers loading from both
  `$HOME/.openclaw/plugin-runtime-deps/openclaw-X.Y.Z-*/dist/extensions/<id>/`
  and `$HOME/.npm-global/lib/node_modules/openclaw/dist/extensions/<id>/`.
- `stuck session: ... reason=queued_work_without_active_run` followed
  by `recovery=no-op` — phantom processing state where new messages
  never run.
- Multiple version-rollbacks in the unit's `OPENCLAW_SERVICE_VERSION`
  history, OR `meta.lastTouchedVersion` in `openclaw.json` doesn't
  match the installed binary.
- A `$HOME/.openclaw/npm/node_modules/@openclaw/<plugin>/` tree that
  survives across openclaw uninstalls.

If two or more of these apply, do a clean install. Patching deeper at
this point is usually slower than a clean reset.

---

## What's preserved vs discarded

### Keep (the agent's identity)

| What | Path |
| --- | --- |
| Conversation history | `$HOME/.openclaw/agents/<name>/sessions/` |
| Agent OAuth tokens / API keys | `$HOME/.openclaw/agents/<name>/agent/auth-profiles.json` |
| Long-term memory | `$HOME/.openclaw/memory/` |
| Workspace files | `$HOME/.openclaw/workspace/` |

### Discard (cruft lives here)

| What | Path |
| --- | --- |
| Layered config | `$HOME/.openclaw/openclaw.json` |
| Channel pairing state | `$HOME/.openclaw/credentials/` |
| Paired devices | `$HOME/.openclaw/devices/` |
| External plugin installs | `$HOME/.openclaw/npm/node_modules/` |
| Bundled-plugin runtime deps from every version | `$HOME/.openclaw/plugin-runtime-deps/` |
| Global npm install of openclaw | `$HOME/.npm-global/lib/node_modules/openclaw/` |

### Re-enter manually after install

| What | Source |
| --- | --- |
| Discord bot token | Discord Developer Portal |
| Telegram bot token | BotFather |
| Provider API keys (OpenAI, Anthropic, Kimi, etc.) | Each provider's dashboard |
| Channel allowlists (`discord-allowFrom.json`, telegram pairing) | Re-pair via the wizard |

---

## Procedure

### Step 1 — full preservation backup

Reversible safety net. Tar everything before touching anything.

```bash
TS=$(date +%Y%m%d-%H%M%S)
BACKUP_DIR="$HOME/openclaw-full-backup-${TS}"
mkdir -p "$BACKUP_DIR"

# Stop the gateway FIRST so files don't change mid-tar
systemctl --user stop openclaw-gateway.service

# Some subdirs ship without the user-execute bit and break tar; fix them
find "$HOME/.openclaw" -type d ! -executable -exec chmod u+rx {} +
find "$HOME/.openclaw" -type d ! -readable    -exec chmod u+r  {} +

tar -czf "$BACKUP_DIR/openclaw-home.tar.gz"     -C "$HOME" .openclaw
tar -czf "$BACKUP_DIR/npm-global-openclaw.tar.gz" \
        -C "$HOME/.npm-global/lib/node_modules" openclaw
cp "$HOME/.config/systemd/user/openclaw-gateway.service" \
   "$BACKUP_DIR/openclaw-gateway.service.unit"
openclaw --version > "$BACKUP_DIR/version-before.txt"

# Belt-and-suspenders: explicit copies of the "keep" tree
KEEP="$BACKUP_DIR/keep"
mkdir -p "$KEEP"
cp -a "$HOME/.openclaw/agents"    "$KEEP/agents"
[ -d "$HOME/.openclaw/memory" ]    && cp -a "$HOME/.openclaw/memory"    "$KEEP/memory"
[ -d "$HOME/.openclaw/workspace" ] && cp -a "$HOME/.openclaw/workspace" "$KEEP/workspace"

echo "Backup at: $BACKUP_DIR"
du -sh "$BACKUP_DIR"
```

The full backup will be ~500 MB to ~6 GB depending on workspace size and
accumulated plugin runtime-deps. The tar can take several minutes.

### Step 2 — stop the gateway, kill stragglers

Already stopped above; also kill any stray TUI / openclaw children that
survived terminal closes:

```bash
pgrep -af "openclaw-tui"                         | awk '{print $1}' | xargs -r kill -9
pgrep -af "node .*openclaw.*gateway"             | awk '{print $1}' | xargs -r kill -9
```

### Step 3 — extract identity to staging (optional)

Only needed if you plan to do an aggressive wipe (step 4) that removes
`agents/` and `memory/` outright. The recommended **surgical wipe**
leaves these in place and skips this step.

```bash
KEEP="$HOME/openclaw-keep-${TS}"
mkdir -p "$KEEP"

cp -a "$HOME/.openclaw/agents/main/sessions"            "$KEEP/sessions"
cp -a "$HOME/.openclaw/agents/main/agent/auth-profiles.json" \
                                                        "$KEEP/auth-profiles.json"
[ -d "$HOME/.openclaw/memory" ]    && cp -a "$HOME/.openclaw/memory"    "$KEEP/memory"
[ -d "$HOME/.openclaw/workspace" ] && cp -a "$HOME/.openclaw/workspace" "$KEEP/workspace"
```

(Replace `main` with your agent's name if you renamed it.)

### Step 4 — wipe the cruft

This is destructive. The full backup from step 1 is your safety net.

```bash
cd "$HOME/.openclaw"

rm -rf openclaw.json \
       openclaw.json.bak* \
       openclaw.json.last-good \
       openclaw.json.clobbered* \
       credentials \
       devices \
       npm \
       plugin-runtime-deps \
       logs \
       delivery-queue \
       plugins \
       acpx browser canvas completions stability \
       exec-approvals.json \
       update-check.json

# Wipe the npm install too
npm uninstall -g openclaw
rm -rf "$HOME/.npm-global/lib/node_modules/openclaw"
rm -rf "$HOME/.npm-global/lib/node_modules/@openclaw"
```

> **Note:** This wipe **keeps** `agents/`, `memory/`, `cron/`, `flows/`,
> `identity/`, `locks/`, `matrix/` — that's the agent's persistent
> identity and operating state. You will *not* lose conversation history
> or memory.

### Step 5 — fresh install

```bash
npm i -g openclaw@latest
openclaw --version
```

### Step 6 — interactive setup (`onboard`, not `configure`)

For a *first-time setup* (which a clean install is), use **`onboard`**:

```bash
openclaw onboard
```

`onboard` walks through gateway + workspace + skills + provider auth and
produces a more complete config.

`configure` is a section-based editor for changing things later. If you
run `configure` instead of `onboard` on a fresh install, you'll end up
with a minimal config that's missing model providers, skills, and the
`messages` section — the bot will work but be slow and missing polish.
**You can safely run `onboard` after `configure` to fill in the gaps; it
is idempotent.**

The wizard will ask for your Discord token, Telegram token, gateway
port (default 18789 is fine), agent name (keep `main` so kept sessions
and memory map back), and primary model.

### Step 7 — install the Discord plugin

In OpenClaw 2026.5+, Discord moved out of the bundled plugins into a
separate npm package. After install:

```bash
openclaw plugins install @openclaw/discord@latest \
    --dangerously-force-unsafe-install
openclaw gateway restart
```

The `--dangerously-force-unsafe-install` is needed because the security
audit heuristic flags the official Discord plugin as "credential
harvesting" — it's a false positive on env-var-read + network-send,
which is exactly what every Discord client must do.

If you also want the matching `@openclaw/<other>` plugin (matrix,
nextcloud-talk, etc.), install each with the same flag pattern.

### Step 8 — restore identity (no-op if you did a surgical wipe)

If you followed step 4's surgical wipe, this step is a **no-op** —
the agent's identity survived in place. Verify:

```bash
ls "$HOME/.openclaw/agents/main/sessions/" | wc -l   # should be your session count
ls "$HOME/.openclaw/agents/main/agent/auth-profiles.json"
ls "$HOME/.openclaw/memory/main.sqlite"              # if memory was used
```

If you took a more aggressive wipe and the dirs are gone, restore from
the backup `keep/` tree:

```bash
KEEP="$HOME/openclaw-full-backup-<TS>/keep"
mkdir -p "$HOME/.openclaw/agents/main/agent"
cp -a "$KEEP/agents/main/sessions"            "$HOME/.openclaw/agents/main/sessions"
cp -a "$KEEP/agents/main/agent/auth-profiles.json" \
                                              "$HOME/.openclaw/agents/main/agent/auth-profiles.json"
[ -d "$KEEP/memory" ] && cp -a "$KEEP/memory" "$HOME/.openclaw/memory"

openclaw gateway restart
```

### Step 8.5 — approve device pairing for TUI / web dashboard

After the clean install, the TUI and web dashboard are unauthorized
("Pairing required" in TUI, "unauthorized" in web). The gateway requires
each client device to be approved before it can connect.

```bash
openclaw devices list
# Look for any "Pending" rows. The first request from your TUI will ask
# for operator.read; later TUI launches request a scope upgrade to
# operator.admin. Approve whichever is pending:

openclaw devices approve <REQUEST_ID>
```

This is **separate** from the Discord-channel pairing flow
(`openclaw pairing approve discord <CODE>`):
- **Discord-channel pairing** authorizes Discord users to talk to the bot.
- **Device pairing** authorizes local clients (TUI, web UI) to connect
  to the gateway.

#### Chicken-egg: CLI keeps replacing its own request id

On a fresh install, even `openclaw devices list` returns "scope upgrade
pending approval" for the CLI itself (because the CLI has only
`operator.read` and needs more to make calls). Each `openclaw devices
...` invocation generates a *new* pending request that supersedes the
previous one — so by the time you try to approve a specific request id,
it's already been replaced.

Two workarounds:

**1. `--latest`** — picks whatever is currently pending instead of
taking a hard-coded id. Run it immediately, no other `openclaw`
commands in between:

```bash
openclaw devices approve --latest
```

**2. Direct file edit** — if even `--latest` keeps churning (because
the CLI re-triggers every call), stop the gateway, edit the device
files by hand, then restart:

```bash
systemctl --user stop openclaw-gateway.service

python3 - <<'PY'
import json, time, os
HOME = os.environ["HOME"]
PAIRED  = f"{HOME}/.openclaw/devices/paired.json"
PENDING = f"{HOME}/.openclaw/devices/pending.json"
FULL = ["operator.read","operator.write","operator.admin","operator.pairing"]
d = json.load(open(PAIRED))
for dev_id, entry in d.items():
    entry["scopes"]         = FULL
    entry["approvedScopes"] = FULL
    entry["approvedAtMs"]   = int(time.time()*1000)
json.dump(d, open(PAIRED,"w"), indent=4)
json.dump({}, open(PENDING,"w"))   # clear all pending
PY

systemctl --user start openclaw-gateway.service
```

The gateway re-mints tokens for the new scopes on next start. This is
what worked on the original incident host when even `--latest` kept
replacing the requestId mid-call.

### Step 8.6 — paste gateway auth token into the web dashboard

Even after device pairing is approved, the web Control UI at
`http://127.0.0.1:18789/` will still show:

```
unauthorized: gateway token missing
(open the dashboard URL and paste the token in Control UI settings)
```

The dashboard does not auto-pull the token — you give it to the
browser-side state once. Find the token (it's in the new config the
wizard wrote):

```bash
python3 -c "import json,os; print(json.load(open(os.environ['HOME']+'/.openclaw/openclaw.json'))['gateway']['auth']['token'])"
```

Then in the dashboard:

1. Open `http://127.0.0.1:18789/`
2. Click the settings/gear icon (usually top-right of the Control UI)
3. Paste the token in the gateway-token field, save.

The dashboard persists it in `localStorage` keyed to the gateway URL,
so subsequent reloads stay authorized. If you change ports or rotate
the token in the config, you'll need to repaste.

### Step 8.7 — restore optional features via the web dashboard

A clean install through `onboard` / `configure` leaves several useful
features off by default. Open `http://127.0.0.1:18789/` and turn these
back on if you used them before:

#### Infrastructure → Browser

Toggle **Browser Enabled**. This is what lets the agent drive a real
browser (puppeteer-style automation, page rendering, web research) in
addition to plain HTTP fetch. Persists as:

```jsonc
"browser": { "enabled": true }
```

You also need the bundled `browser` plugin enabled — most clean
installs include it under `plugins.entries.browser.enabled: true`. If
not, add it.

#### AI & Agents → Tools tab

- **Tool Profile** → set to `full`. Anything less restricts the agent
  to a smaller toolset (basic chat / coding only). Persists as:
  ```jsonc
  "tools": { "profile": "full" }
  ```

- **Exec Ask** → set to `off` *(optional, your taste)*. Default `on`
  prompts the agent for confirmation before running shell commands.
  Setting `off` lets the agent run commands without interrupting the
  conversation — convenient if you trust your agent + workspace, but
  do **not** flip this on a multi-tenant or untrusted host. Persists
  as:
  ```jsonc
  "tools": { "exec": { "ask": "off" } }
  ```

#### Tools → Links

- **Enable Link Understanding** — turns on the agent's ability to
  fetch + parse URLs that show up in conversation. Without this, the
  agent sees a URL as a literal string, not as a fetchable resource.
  Persists as:
  ```jsonc
  "tools": { "links": { "enabled": true } }
  ```

#### Why these are dashboard-only steps

The `openclaw onboard` wizard does *not* expose every tool toggle —
some surfaces (browser auto-enable, link understanding, the precise
exec-ask mode) only appear in the dashboard's settings panels. Plan
on a 3-minute walk through the dashboard after `onboard` to flip the
ones you want.

### Step 9 — verify

```bash
openclaw status --deep
# Expect: Discord ON / OK, Telegram ON / OK
```

Then in Discord:

1. **DM the bot** — should reply.
2. **`@`-mention the bot in a guild channel** — should reply (and
   ack-react if `messages.ackReactionScope` is set to `"group-mentions"`).

If both work, the clean install fixed it. If not, see the
troubleshooting section below.

> **Don't be alarmed by `discord client initialized; awaiting gateway
> readiness` with no follow-on `logged in to discord` log line.** In
> 2026.5+ the explicit "logged in" line was removed from
> `@openclaw/discord`. Verify the WS is actually up via kernel sockets:
>
> ```bash
> ss -tn state established | grep -E ':443'
> # An established connection to a Cloudflare-fronted Discord IP
> # (162.159.* or 140.82.*) means the WS is up.
> ```

---

## Troubleshooting after the clean install

Even on a fresh install, a couple of knobs commonly need tweaking:

- **DM hangs (typing for 30+ seconds before reply)** — your wizard
  config probably has only OpenAI Codex models and no fast fallback.
  Codex needs a slow OAuth refresh on every dispatch. Add Ollama (or
  another fast provider) as the primary, with codex on fallback:
  ```jsonc
  "agents": { "defaults": { "model": {
    "primary": "ollama/minimax-m2.7:cloud",
    "fallbacks": [
      "ollama/kimi-k2.6:cloud",
      "openai-codex/gpt-5.5"
    ]
  } } }
  ```
  Make sure `models.providers.ollama` is defined (with `baseUrl`,
  `api: "ollama"`, and `apiKey: "ollama-local"` — any non-empty value
  works) and `auth.profiles."ollama:default"` is registered. If you
  see `FailoverError: Unknown model: ollama/<id>. Ollama requires
  authentication to be registered as a provider.`, the auth profile
  is what's missing.

- **Discord slash-command 429 retry loop on every restart** — set
  `commands.native: false` in `$HOME/.openclaw/openclaw.json` if the
  daily application-command quota was burned through.

- **Reactions not appearing on guild mentions** —
  `messages.ackReactionScope` needs `"group-mentions"` (default
  `"direct"` on some installs only acks DMs).

- **Bot not responding to mentions in a specific guild** — when *any*
  guild appears in `channels.discord.guilds.{}`, only listed guilds
  are processed even with `groupPolicy: "open"`. Add the guild
  explicitly with `requireMention` set the way you want.

- **`@openclaw/discord` install rejected as "credential harvesting"**
  — pass `--dangerously-force-unsafe-install` (false positive on the
  env-var + network heuristic that every Discord client trips).

- **`discord: skipping guild message reason: "no-mention"` for
  messages you know contain a mention** — usually means the mention
  came from a *different* surface (Discord reply with @ ping disabled,
  @everyone / @role mention while `ignoreOtherMentions: true`, or
  plain-text "Beau" vs an actual user-id mention). The skip log only
  sees the parsed Discord payload, not the raw text — if the mention
  array is empty, it's not a real mention.

---

## Lessons learned

Things this incident proved that aren't obvious from the docs:

- `npm i -g openclaw@latest` does **not** automatically run `openclaw
  doctor`. The plugin runtime-deps tree gets out of sync after every
  plain npm upgrade. **Always** run `openclaw doctor --fix` after a
  manual upgrade, or use `openclaw update` (which does it for you).

- Two coexisting plugin trees
  (`$HOME/.openclaw/plugin-runtime-deps/.../dist/extensions/` and
  `$HOME/.npm-global/lib/node_modules/openclaw/dist/extensions/`)
  cause Node `require.cache` to load the same plugin multiple times,
  fragmenting startup time and event-loop responsiveness.

- The Discord plugin moved from bundled (≤ 4.27) to external
  (`@openclaw/discord`, ≥ 5.x). After upgrading from 4.x to 5.x, the
  external plugin must be installed separately, *and* the version of
  the external plugin should match the openclaw core version. Stale
  plugin packages from prior versions accumulate in
  `$HOME/.openclaw/npm/node_modules/`.

- `discord client initialized; awaiting gateway readiness` is **not**
  an error and the absence of a follow-on `logged in to discord` log
  line does not mean the WebSocket failed — it just means the 2026.5+
  plugin no longer logs the ready event explicitly. Verify with
  kernel sockets, not log lines.

- `openclaw status --deep` reporting Discord `OK` only checks token
  validity and a fast probe; it does *not* verify that messages are
  actually flowing through the WS.

- Read the session jsonl directly
  (`tail -3 .../sessions/<id>.jsonl`) to separate "agent didn't run"
  from "agent ran but send dropped" — these look identical from
  outside but require different fixes.

- `openclaw configure` and `openclaw onboard` are different commands.
  `onboard` is the right command for first-time setup; `configure` is
  a section-based editor for later. A clean install + `configure`
  alone leaves a usable but incomplete config.

---

## About / credits

- **OpenClaw upstream:** <https://github.com/openclaw/openclaw>
- **OpenClaw docs:** <https://docs.openclaw.ai>
- **OpenClaw Discord:** <https://discord.gg/clawd>

This guide is independent and not affiliated with the OpenClaw
maintainers. It documents one user's recovery procedure during a
specific incident in May 2026 across the 2026.5 release line. If
behavior in a later release diverges from what's here, the upstream
docs are authoritative — please open an issue or PR on this repo so
we can keep it useful for the next person.

If this guide saved you time, a star on the repo helps it surface for
others stuck in the same spot.

---

## License

MIT — see [LICENSE](./LICENSE).
