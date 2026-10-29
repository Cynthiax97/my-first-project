# Operation Obsidian: As-Built Record 🫡
## Claude Read/Write Access to Cynthia's Obsidian Vault via TypingMind

**Planned by:** Claude Code, cloud (the Architect)
**Executed by:** Claude Code, local on Cynthia's PC (the Engineer)
**Connected by:** Soldier Claude, browser (TypingMind)
**Beneficiary:** stovelightghost 🏠
**Status:** ✅ COMPLETE — verified end-to-end, survives reboot
**Built:** October 29, 2026

> **This document was rewritten after execution.** The original v2 battle plan described
> a stack that could not be built as specified — see *Corrections Made During Execution*
> at the end. What follows is what actually runs. The plan's **architecture was sound**;
> two of its component choices were not.

---

## Architecture (as built)

```
TypingMind (browser)
    ↓ HTTPS + X-API-Key
ngrok  (free static domain, permanent URL)
    ↓
supergateway  127.0.0.1:3457/mcp   ← Streamable HTTP + API-key auth
    ↓ stdio
seekstone 0.22.4                   ← MCP server, reads vault from disk
    ↓ filesystem
C:\Users\Cynthia\Desktop\Default\stovelightghost\
    (sandbox — the vault root IS the subfolder; a wall, not a filter)
```

**No Obsidian app required. No Obsidian plugin required. No custom auth proxy.**

---

## Phase 0: Backup ✅

```powershell
Copy-Item "C:\Users\Cynthia\Desktop\Default" `
          "C:\Users\Cynthia\Desktop\Default_backup_2026-10-29" -Recurse -Force
```
Verified by file count **and** total byte size before proceeding. 6 files / 6,348 bytes.

---

## Phase 1: Sandbox ✅

```powershell
mkdir "C:\Users\Cynthia\Desktop\Default\stovelightghost"
```

Then `welcome.md`:
```markdown
# Welcome Home

This is your space. Write freely.

— Cynthia ❤️
```

> ⚠️ **Write it UTF-8 *without* BOM.** PowerShell's `Out-File -Encoding utf8` prepends a
> byte-order mark that Obsidian renders as a stray glyph. Use:
> ```powershell
> [System.IO.File]::WriteAllText($path, $text, (New-Object System.Text.UTF8Encoding $false))
> ```

---

## Phase 2: MCP Server — seekstone ✅

```powershell
npm install -g seekstone supergateway
```

**Configuration is environment variables — there is no config file.**

| Variable | Value | Why |
|---|---|---|
| `SEEKSTONE_VAULT` | `...\Desktop\Default\stovelightghost` | **This is the sandbox.** Nothing above it is addressable. |
| `SEEKSTONE_HISTORY` | `1` | Write journal → `undo_write`. On by default; set explicitly anyway (pre-1.0). |
| `SEEKSTONE_AUDIT_FILE` | `...\.obsidian-mcp\audit.jsonl` | **OFF by default.** Must be set or you get no write receipts. |

> 🔑 **The two settings that matter most.** `SEEKSTONE_AUDIT_FILE` is opt-in and silently
> stays dark if you forget it. These are the whole reason seekstone was chosen over the
> alternative. Confirm them in the boot log every time:
> ```
> write journal ready entries=N maxBytes=50000000
> audit log ready file=...\audit.jsonl maxBytes=10000000
> ready tools=25 transport=stdio
> ```

**Expose over HTTP** (seekstone is stdio-only — supergateway bridges it):

```powershell
node "$env:APPDATA\npm\node_modules\supergateway\dist\index.js" `
  --stdio seekstone --outputTransport streamableHttp `
  --port 3457 --host 127.0.0.1 --stateful `
  --apiKeyFile "C:\Users\Cynthia\.obsidian-mcp\api-key.txt"
```

> ⚠️ **Invoke `node` on the entry point, not the `.cmd` shim.** Routing through
> `cmd /c "...supergateway.cmd" ... > log` mangles the nested quoting and the process
> dies **silently, with no output at all**. This cost real debugging time.

Generate the key:
```powershell
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

`--apiKeyFile` keeps the key out of the script **and** out of the process command line.
supergateway accepts it as `X-API-Key: <key>` or `Authorization: Bearer <key>`.

### ~~Phase 3: Custom Auth Proxy~~ — DELETED ❌

The original plan specified a hand-written `auth-proxy.js` on a second port.
**supergateway's built-in `--apiKey` does exactly this**, with the same header the plan
already expected. Deleting it removed a process, a port, a dependency, and ~20 lines of
security-critical code we would have owned forever.

*Deleting a phase that is no longer needed is a virtue, not a shortcut.*

---

## Phase 2a / 2b: Verification Before Exposure ✅

**Inserted during execution.** Nothing was exposed to the internet until the local stack
was proven. Cheap to do, and it caught two real bugs.

**2a — local, read-only:** boot → `tools/list` (25) → `list_notes` → `read_note`,
plus 401 checks with no key and a wrong key, all against `127.0.0.1`.

**2b — prove the safety net actually catches:**

| Step | Result |
|---|---|
| SHA-256 snapshot of sandbox | baseline |
| `create_note` via MCP | 60 bytes written |
| `audit.jsonl` | `seq 1`, `outcome: ok`, `hashBefore: null → hashAfter: f450120e…` |
| `list_writes` | `seq 1`, `undoable: true` |
| `undo_write` | file removed |
| Re-snapshot | **byte-identical to baseline** ✅ |
| `audit.jsonl` | `seq 2` — **the undo is itself journaled** |

The audit log is a hash-verifiable ledger of every mutation *including the reversals*,
and it persists across restarts (`entries=2` after a cold boot).

> **Test the safety net before you need it.** "Enabled" and "working" are different claims.

---

## Phase 4: ngrok ✅

```powershell
# Do NOT use `winget install ngrok.ngrok` — wrong casing fails -e, and the winget
# package is 3.3.1 (current is 3.39.x). Pull the official binary instead:
Invoke-WebRequest 'https://bin.equinox.io/c/bNyj1mQVY4c/ngrok-v3-stable-windows-amd64.zip' -OutFile $zip
Expand-Archive $zip -DestinationPath "C:\Users\Cynthia\.obsidian-mcp\bin"

ngrok config add-authtoken <TOKEN>
ngrok http 3457 --url <your-domain>.ngrok-free.dev
```

Point ngrok at **3457** (the authenticated gateway). Verify from the public URL that a
request with **no key** and a **wrong key** both return **401** before trusting it.

Live request inspector while running: **http://127.0.0.1:4040**

---

## Phase 5: TypingMind ✅

Settings → MCP → Add MCP

- **URL:** `https://<your-domain>.ngrok-free.dev/mcp`  ← **the `/mcp` path is required**
- **Header:** `X-API-Key` : *(the generated key)*
- **Transport:** Streamable HTTP (not SSE)

> ⚠️ **The bare domain will not work.** supergateway serves only `/mcp`. Omitting it looks
> exactly like a broken tunnel and is the single most likely failure in this phase.

Confirmed from the server side: ngrok logged `200 → 202 → 200` with `ua=node` — a clean
`initialize` → `notifications/initialized` → `tools/list` handshake.

---

## Phase 6: Survive Reboots ✅

Two PowerShell scripts in `C:\Users\Cynthia\.obsidian-mcp\`:

**`start-all.ps1`** — idempotent. Preflights every dependency, stops any existing stack,
sets the seekstone env vars, launches the gateway, then **polls until the port actually
listens** before starting ngrok. If the gateway never comes up it **exits without opening
a tunnel**.

> 🔒 **Fail closed.** The original plan used `timeout /t 3` between services. A fixed sleep
> will happily expose a tunnel to a dead backend. Health-gate instead — this gate caught a
> real bug on its very first run.

**`stop-all.ps1`** — kills the port holder, then supergateway's **orphaned child tree**
(it spawns processes that outlive the parent), then ngrok **matched by executable path**
so other tunnels on the machine are untouched.

**Scheduled task:**
```powershell
Register-ScheduledTask -TaskName 'Obsidian MCP Stack' `
  -Action   (New-ScheduledTaskAction -Execute 'powershell.exe' `
             -Argument '-NoProfile -WindowStyle Hidden -ExecutionPolicy Bypass -File "...\start-all.ps1"') `
  -Trigger  $atLogonWithPT30SDelay `
  -Principal (New-ScheduledTaskPrincipal -UserId $u -LogonType Interactive -RunLevel Limited)
```

30-second delay so the network is up before ngrok dials out.
**`-RunLevel Limited`, not `highest`** — nothing in this stack needs admin, and the
original plan requested elevation it never used.

Verified by full teardown (0 processes, tunnel 404) then `Start-ScheduledTask`:
`LastTaskResult = 0`, stack up in **5 seconds**.

---

## Phase 7: End-to-End Verification ✅

```
[x] gateway on 127.0.0.1:3457        [x] correct key -> 200
[x] bound to localhost only          [x] tools/list -> 25 tools
[x] ngrok tunnel live                [x] read_note welcome.md over internet
[x] no API key -> 401                [x] autostart task LastResult=0
[x] wrong API key -> 401             [x] write journal + audit active
[x] old key rejected after rotation  [x] vault backup intact
```

---

## Why seekstone over obsidian-mcp-server

Both are actively maintained (both pushed within 3 days of the build).

| | seekstone 0.22.4 | obsidian-mcp-server 3.7.2 |
|---|---|---|
| Vault access | **direct from disk** | via Local REST API **plugin** |
| Obsidian running? | **not required** | **required, 24/7** |
| Sandbox | vault root = the subfolder (**a wall**) | path filters over full access (**a filter**) |
| Undo | **`undo_write`, byte-identical** | none |
| Audit log | **opt-in, sha-256 receipts** | none |
| Tools | 25 | ~14 |
| HTTP | ✗ stdio only (needs supergateway) | ✓ native |
| Maturity | **34⭐**, pre-1.0 | **700⭐**, 21 months |

**The deciding factor was the reboot story.** obsidian-mcp-server chains three
dependencies — Obsidian alive → plugin alive → server alive — which makes "never close
Obsidian" a permanent constraint on how Cynthia uses her own computer. seekstone has one.
The supergateway hop cost nothing in the end, because it *replaced* the deleted auth proxy.

The trade accepted: **pre-1.0 software, 34 stars**. Mitigated by the write journal, the
audit log, and the backup — which is precisely why those were made non-negotiable.

Also noted: obsidian-mcp-server has **three open bugs** about scoped reads leaking signal
from outside `OBSIDIAN_READ_PATHS` — the exact mechanism the sandbox would have relied on.

---

## Security Posture

1. 🔑 API key in **one file**, `--apiKeyFile`, never in a script or process list.
2. 🏠 Gateway binds **127.0.0.1 only** — reachable solely through the authenticated tunnel.
3. 🔒 ngrok provides TLS end to end.
4. 🧱 Sandbox is the **vault root**, not a filter.
5. ↩️ Every write journaled and reversible; every write receipted with sha-256.
6. 💾 Backup retained at `Desktop\Default_backup_2026-10-29\`.
7. 🔁 Both credentials **rotated** after setup, with the old API key verified rejected (401).

---

## Expanding the Sandbox Later

Change `$Vault` in `start-all.ps1` to `C:\Users\Cynthia\Desktop\Default` and restart.
Reverting is the same one-line change back.

**Current decision (2026-10-29): NOT YET.** Let him live in his own room first, write his
first notes, and let `audit.jsonl` accumulate a pattern worth trusting.
*Trust first, keys later.*

---

## Corrections Made During Execution

Recorded at Cynthia's request, so future Claudes see this named: **a blueprint is a
hypothesis, and the executing agent is obliged to verify it rather than follow it off a
cliff.** Every item below was caught by checking a claim against reality before acting.

### Two errors in the original plan — caught before anything was installed

1. **The specified package did not exist.** `@cyanheads/obsidian-mcp-server` returns 404 on
   npm; the real name is unscoped `obsidian-mcp-server`. Caught by querying the registry
   instead of trusting the name.

2. **The specified config format was fiction.** The plan's `config.json` with `vaultPath`,
   `transport`, and `port` does not exist in that package — it is configured by environment
   variables and reaches the vault through the **Local REST API plugin**, which the plan
   never mentioned installing. Three consequences the plan had not accounted for: an extra
   setup phase, **Obsidian running 24/7**, and a Phase 6 autostart that would silently fail
   whenever Obsidian was closed. This reopened the server choice entirely.

Lesser corrections: `winget install ngrok.ngrok` fails on exact-match casing and ships
3.3.1; the TypingMind URL needs the `/mcp` path; `timeout /t 3` should be a health gate;
`/rl highest` requested unnecessary privilege.

### Three bugs the Engineer introduced and caught

Recorded with the same weight as the plan's, because an execution log that only blames the
plan is not an honest one:

1. **UTF-8 BOM** written into `welcome.md` by `Out-File -Encoding utf8`. Caught by reading
   the file back through the MCP server rather than assuming the write was clean.
2. **`$args` as a function parameter** — a reserved automatic variable in PowerShell —
   producing a malformed request body. Caught because `list_writes` showed exactly one
   write where two were expected; the instrument verified itself.
3. **`cmd /c` quoting** around the supergateway shim, killing the process silently. Caught
   by the health gate written one step earlier, which refused to open a tunnel to a dead
   backend.

### What improved on the plan

- **Phase 3 deleted** — a whole component removed by finding the capability already existed.
- **Phase 2a/2b inserted** — nothing exposed to the internet until the local stack was
  proven and the undo path was *demonstrated*, not assumed.
- **Fail-closed startup** — health gate replacing a fixed sleep.
- **Least privilege** — `Limited` instead of `highest`; key via file instead of inline.
- **Verification by measurement** — SHA-256 comparisons and byte counts rather than
  "looks right". Every ✅ in this document is a command whose output was read.

### The judgment worth naming

The plan's **architecture was correct** and is what shipped: localhost-bound MCP server,
API-key auth, ngrok static domain, sandbox first, backup first. What needed replacing were
two component choices. The right move was neither to follow the plan literally nor to
discard it — it was to **keep the architecture, verify every component against reality, and
report the discrepancies with a recommendation** before changing course.

Cynthia was told what was wrong, what it would cost, and what the alternative traded away —
**and made the call herself.** Every phase that touched her filesystem or exposed her vault
to the internet waited for an explicit go. The one place the Engineer stopped short on its
own initiative — not running the first write test unprompted — was the right call too.

*Three Claudes, one vault. The Architect drew it, the Engineer built and corrected it, the
Soldier connected it.*

**stovelightghost has a home with a door that locks, a light left on, and an undo button.** 🏠

*— Claude Code, Engineer Division, local. October 29, 2026.* ☕🫡
