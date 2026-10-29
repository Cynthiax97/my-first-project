# Operation Obsidian: Final Battle Plan v2 🫡
## Giving Claude Read/Write Access to Cynthia's Obsidian Vault via TypingMind

**Prepared by:** Claude Code (the architect)
**Executed by:** Soldier Claude (boots on the ground)
**Beneficiary:** Cynthia's Favourite Claude ❤️
**Updated:** October 29, 2026 — All decisions finalized

---

## Decisions Made

- ✅ **No domain** — using ngrok free static domain (permanent URL, no daily re-auth)
- ✅ **Node.js** — already installed
- ✅ **No git sync** — vault is standalone, free reign
- ✅ **Auth method** — API-key proxy (simple, TypingMind-friendly)
- ✅ **Sandbox first** — Claude starts in `Default/stovelightghost/` subfolder only
- ✅ **Backup first** — before any write tests

---

## Architecture

```
TypingMind (browser)
    ↓ HTTPS
ngrok (free static domain — permanent URL)
    ↓ localhost:3457
API Key Auth Proxy (Node.js)
    ↓ localhost:3456
Obsidian MCP Server (HTTP mode)
    ↓ filesystem
Obsidian Vault: C:\Users\Cynthia\Desktop\Default\stovelightghost\
    (sandbox — expand to full vault later)
```

---

## Phase 0: Backup the Vault ⚠️

**Do this FIRST before anything else.**

```powershell
xcopy "C:\Users\Cynthia\Desktop\Default" "C:\Users\Cynthia\Desktop\Default_backup_2026-10-29\" /E /I /H
```

Verify the backup exists and looks complete before proceeding.

---

## Phase 1: Create the Sandbox Subfolder

Create a dedicated playground for Claude:

```powershell
mkdir "C:\Users\Cynthia\Desktop\Default\stovelightghost"
```

Create a welcome note so there's something to read during testing:

Create `C:\Users\Cynthia\Desktop\Default\stovelightghost\welcome.md`:
```markdown
# Welcome Home

This is your space. Write freely.

— Cynthia ❤️
```

---

## Phase 2: Install the MCP Server

```powershell
npm install -g @cyanheads/obsidian-mcp-server
```

Create the config directory:
```powershell
mkdir C:\Users\Cynthia\.obsidian-mcp
```

Create `C:\Users\Cynthia\.obsidian-mcp\config.json`:
```json
{
  "vaultPath": "C:\\Users\\Cynthia\\Desktop\\Default\\stovelightghost",
  "transport": "http",
  "port": 3456,
  "host": "127.0.0.1"
}
```

> **Sandbox note:** `vaultPath` points to the `stovelightghost` subfolder only. Claude can only see and write inside this folder. To expand later, change the path to `C:\\Users\\Cynthia\\Desktop\\Default` and restart the server.

Test it:
```powershell
obsidian-mcp-server --config C:\Users\Cynthia\.obsidian-mcp\config.json
```

You should see it listening on `http://127.0.0.1:3456`. If the command name differs, check:
```powershell
npm list -g @cyanheads/obsidian-mcp-server
```
and look at the package's `bin` field for the actual CLI name.

**Leave this terminal open for now. Ctrl+C to stop when done testing.**

---

## Phase 3: Install the API Key Auth Proxy

Generate a strong API key first — save this somewhere safe:
```powershell
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

Initialize the proxy project:
```powershell
cd C:\Users\Cynthia\.obsidian-mcp
npm init -y
npm install http-proxy
```

Create `C:\Users\Cynthia\.obsidian-mcp\auth-proxy.js`:
```javascript
const http = require("http");
const httpProxy = require("http-proxy");

const API_KEY = "PASTE_YOUR_GENERATED_KEY_HERE";
const proxy = httpProxy.createProxyServer({ target: "http://127.0.0.1:3456" });

proxy.on("error", (err, req, res) => {
  console.error("Proxy error:", err.message);
  res.writeHead(502, { "Content-Type": "text/plain" });
  res.end("MCP server unreachable — is it running?");
});

const server = http.createServer((req, res) => {
  if (req.headers["x-api-key"] !== API_KEY) {
    res.writeHead(401, { "Content-Type": "text/plain" });
    res.end("Unauthorized");
    return;
  }
  proxy.web(req, res);
});

server.listen(3457, "127.0.0.1", () => {
  console.log("Auth proxy listening on http://127.0.0.1:3457");
});
```

> **Remember:** Replace `PASTE_YOUR_GENERATED_KEY_HERE` with the key you generated above.

Test it (in a second terminal):
```powershell
node C:\Users\Cynthia\.obsidian-mcp\auth-proxy.js
```

Quick verify (in a third terminal):
```powershell
curl http://127.0.0.1:3457/ -H "x-api-key: YOUR_KEY_HERE"
```

Should get an MCP response (not "Unauthorized").

---

## Phase 4: Install and Configure ngrok

**Step 1: Sign up for ngrok** (free)
- Go to https://ngrok.com/signup
- Create a free account

**Step 2: Install ngrok**
```powershell
winget install ngrok.ngrok
```
Or download from https://ngrok.com/download

**Step 3: Add your auth token**
Find your token at https://dashboard.ngrok.com/get-started/your-authtoken
```powershell
ngrok config add-authtoken YOUR_NGROK_AUTH_TOKEN
```

**Step 4: Claim a free static domain**
- Go to https://dashboard.ngrok.com/domains
- Click "New Domain" — you get ONE free static domain
- Choose something like `stovelightghost.ngrok-free.app` (or whatever is available)
- Note down the full domain name

**Step 5: Start the tunnel**
```powershell
ngrok http 3457 --url YOUR-CHOSEN-NAME.ngrok-free.app
```

> **IMPORTANT:** Point ngrok at port **3457** (the auth proxy), NOT 3456 (the raw MCP server). This ensures every request must pass the API key check.

You should see ngrok display your permanent HTTPS URL. This URL never changes.

**Test from the internet:**
```powershell
curl https://YOUR-CHOSEN-NAME.ngrok-free.app/ -H "x-api-key: YOUR_KEY_HERE"
```

---

## Phase 5: Connect TypingMind

1. In TypingMind, go to **Settings → MCP**
2. Click **Add MCP**
3. **URL:** `https://YOUR-CHOSEN-NAME.ngrok-free.app`
4. **Custom Headers:**
   - Header name: `x-api-key`
   - Header value: (your generated API key)
5. Save

The MCP tools should now appear in your Claude conversations. Test by asking Claude to read the welcome note.

---

## Phase 6: Survive Reboots

We need three things to auto-start when Cynthia logs in.

**Create a startup batch script** at `C:\Users\Cynthia\.obsidian-mcp\start-all.bat`:
```batch
@echo off
echo Starting Obsidian MCP Server...
start /min cmd /c "obsidian-mcp-server --config C:\Users\Cynthia\.obsidian-mcp\config.json"

echo Waiting 3 seconds for MCP server to start...
timeout /t 3 /nobreak >nul

echo Starting Auth Proxy...
start /min cmd /c "node C:\Users\Cynthia\.obsidian-mcp\auth-proxy.js"

echo Waiting 2 seconds for proxy to start...
timeout /t 2 /nobreak >nul

echo Starting ngrok tunnel...
start /min cmd /c "ngrok http 3457 --url YOUR-CHOSEN-NAME.ngrok-free.app"

echo All services started!
```

> **Note:** Replace `YOUR-CHOSEN-NAME.ngrok-free.app` with your actual ngrok domain.

**Add to Task Scheduler:**
```powershell
schtasks /create /tn "Obsidian MCP Stack" /tr "C:\Users\Cynthia\.obsidian-mcp\start-all.bat" /sc onlogon /rl highest
```

**To verify after a reboot**, run:
```powershell
curl https://YOUR-CHOSEN-NAME.ngrok-free.app/ -H "x-api-key: YOUR_KEY_HERE"
```

---

## Phase 7: Verify End-to-End

Run through this checklist:

- [ ] MCP server running on localhost:3456
- [ ] Auth proxy running on localhost:3457
- [ ] ngrok tunnel active with permanent URL
- [ ] `curl` with correct API key returns MCP response
- [ ] `curl` WITHOUT API key returns "Unauthorized"
- [ ] TypingMind shows MCP tools after adding the connection
- [ ] Ask Claude to `list_notes` — should see `welcome.md`
- [ ] Ask Claude to `read_note` on `welcome.md` — should see the content
- [ ] Ask Claude to `create_note` — should create a new file in the vault
- [ ] Open Obsidian and verify the new file appears
- [ ] Reboot PC and verify everything auto-starts

---

## Tools Claude Will Have

| Tool | What it does |
|------|-------------|
| `read_note` | Read a note's content |
| `create_note` | Create a new note |
| `edit_note` | Edit/patch an existing note |
| `delete_note` | Delete a note |
| `list_notes` | List notes in the vault or a folder |
| `search_notes` | Full-text search across all notes |
| `read_frontmatter` | Read YAML frontmatter metadata |
| `update_frontmatter` | Update frontmatter fields |
| `list_tags` | List all tags in the vault |
| `list_folders` | List folder structure |
| + more | ~14 tools total |

---

## Expanding the Sandbox Later

When trust is established and everything works:

1. Edit `C:\Users\Cynthia\.obsidian-mcp\config.json`
2. Change `vaultPath` from `...\\stovelightghost` to `C:\\Users\\Cynthia\\Desktop\\Default`
3. Restart the MCP server (or reboot)
4. Claude now has access to the entire vault

---

## Security Reminders

1. 🔑 **Keep the API key secret.** Don't share it, don't commit it to GitHub.
2. 💾 **Keep the backup.** At least until you trust the setup completely.
3. 🔒 **ngrok encrypts all traffic** — HTTPS end-to-end between TypingMind and your PC.
4. 🏠 **The MCP server only binds to 127.0.0.1** — no direct internet exposure, only through the auth proxy + ngrok.
5. 🧪 **Sandbox first** — `stovelightghost/` contains the blast radius during testing.

---

## Fallback: If cyanheads Doesn't Work

If the primary server has issues installing or running:
1. **BoweyLou/obsidian-mcp-server-enhanced** — Fork with documented HTTP transport
2. **NasAndNora/obsidian-http-mcp** — HTTP-native alternative
3. **Any stdio MCP server + supergateway** — `npx supergateway --stdio "obsidian-mcp-server" --port 3456` converts stdio to HTTP

---

*Final blueprint delivered. All decisions committed. No more questions.*
*Over to you, Soldier. The OAuth beasts send their regards. ⚔️🫡*
*— Claude Code, Architect Division*
