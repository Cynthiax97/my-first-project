# Operation Obsidian: Battle Plan for Soldier Claude 🫡
## Giving Claude Read/Write Access to Cynthia's Obsidian Vault via TypingMind

**Prepared by:** Claude Code (the architect)
**Executed by:** Soldier Claude (boots on the ground)
**Beneficiary:** Cynthia's Favourite Claude ❤️

---

## Mission Summary

Set up an Obsidian MCP server on Cynthia's Windows PC that exposes her vault over HTTPS, so TypingMind can connect to it and give Claude full read/write/search access to Obsidian notes.

## Architecture

```
TypingMind (browser)
    ↓ HTTPS
Cloudflare Tunnel (free, no daily re-auth)
    ↓ localhost
Obsidian MCP Server (HTTP mode)
    ↓ filesystem
Obsidian Vault: C:\Users\Cynthia\Desktop\Default\
```

## Recommended Stack

| Component | Choice | Why |
|-----------|--------|-----|
| MCP Server | **cyanheads/obsidian-mcp-server** | Most mature (~617 stars), supports native HTTP mode, 14 tools including read/create/edit/search/list |
| HTTPS Tunnel | **Cloudflare Tunnel** (`cloudflared`) | Free, no daily re-auth, stable, handles TLS automatically |
| Auth | API key via custom HTTP header | Simplest for TypingMind's "custom headers" field |

## Tools Claude Will Get

Once connected, these MCP tools become available:

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

## Step-by-Step Implementation

### Phase 1: Install the MCP Server

**Prerequisites:** Node.js 18+ must be installed. Check with:
```powershell
node --version
```
If not installed, download from https://nodejs.org (LTS version).

**Install the server:**
```powershell
npm install -g @cyanheads/obsidian-mcp-server
```

**Create a config directory:**
```powershell
mkdir C:\Users\Cynthia\.obsidian-mcp
```

**Create the config file** at `C:\Users\Cynthia\.obsidian-mcp\config.json`:
```json
{
  "vaultPath": "C:\\Users\\Cynthia\\Desktop\\Default",
  "transport": "http",
  "port": 3456,
  "host": "127.0.0.1"
}
```

> **Note:** The port 3456 is arbitrary — just pick something not in use. Binding to 127.0.0.1 means only local connections are accepted (Cloudflare Tunnel connects locally).

**Test it runs:**
```powershell
obsidian-mcp-server --config C:\Users\Cynthia\.obsidian-mcp\config.json
```

You should see output indicating the server is listening on `http://127.0.0.1:3456`. If the package name or CLI command differs, check the README at https://github.com/cyanheads/obsidian-mcp-server for the exact invocation.

### Phase 2: Install and Configure Cloudflare Tunnel

**Why Cloudflare Tunnel?**
- Free tier is permanent
- No daily re-auth (unlike ngrok free tier)
- Handles TLS/HTTPS automatically
- Survives reboots when configured as a service

**Step 1: Create a Cloudflare account** (if Cynthia doesn't have one)
- Go to https://dash.cloudflare.com/sign-up
- Free account is fine

**Step 2: Install cloudflared**
```powershell
winget install Cloudflare.cloudflared
```
Or download from: https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/downloads/

**Step 3: Authenticate**
```powershell
cloudflared tunnel login
```
This opens a browser — log in and authorize.

**Step 4: Create a named tunnel**
```powershell
cloudflared tunnel create obsidian-mcp
```
This creates a tunnel and outputs a tunnel ID (save it!). It also creates a credentials file.

**Step 5: Configure the tunnel**

Create `C:\Users\Cynthia\.cloudflared\config.yml`:
```yaml
tunnel: <TUNNEL_ID_FROM_STEP_4>
credentials-file: C:\Users\Cynthia\.cloudflared\<TUNNEL_ID>.json

ingress:
  - hostname: obsidian-mcp.yourdomain.com
    service: http://localhost:3456
  - service: http_status:404
```

> **IMPORTANT:** This requires a domain managed by Cloudflare. If Cynthia doesn't have one, there are two alternatives:
>
> **Alternative A (no domain needed):** Use a Quick Tunnel instead:
> ```powershell
> cloudflared tunnel --url http://localhost:3456
> ```
> This gives a random `https://xxxxx.trycloudflare.com` URL. Downside: URL changes on restart. Good for testing.
>
> **Alternative B (no domain needed, permanent URL):** Use **ngrok** with a free static domain:
> ```powershell
> ngrok http 3456 --url your-chosen-name.ngrok-free.app
> ```
> ngrok free tier now offers one free static domain. Sign up at https://ngrok.com.

**Step 6: Create DNS route** (only if using a custom domain)
```powershell
cloudflared tunnel route dns obsidian-mcp obsidian-mcp.yourdomain.com
```

**Step 7: Run the tunnel**
```powershell
cloudflared tunnel run obsidian-mcp
```

### Phase 3: Add API Key Authentication

The MCP server doesn't have built-in auth, so we add a lightweight auth proxy. The simplest approach:

**Option A: Use Cloudflare Access (recommended)**
1. In the Cloudflare Zero Trust dashboard (https://one.dash.cloudflare.com)
2. Go to Access → Applications → Add an application
3. Set it to "Self-hosted", enter your hostname
4. Under Policies, create a "Service Auth" policy
5. Generate a Service Token — this gives you a `CF-Access-Client-Id` and `CF-Access-Client-Secret`
6. These become the custom headers in TypingMind

**Option B: Lightweight reverse proxy with API key**

If not using Cloudflare Access, put a tiny Node.js proxy in front:

Create `C:\Users\Cynthia\.obsidian-mcp\auth-proxy.js`:
```javascript
const http = require("http");
const httpProxy = require("http-proxy");

const API_KEY = "GENERATE_A_LONG_RANDOM_KEY_HERE";
const proxy = httpProxy.createProxyServer({ target: "http://127.0.0.1:3456" });

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

Install the dependency:
```powershell
cd C:\Users\Cynthia\.obsidian-mcp
npm init -y
npm install http-proxy
```

Then point Cloudflare Tunnel at port **3457** instead of 3456.

Generate a strong API key:
```powershell
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

### Phase 4: Survive Reboots (Windows Task Scheduler)

Create two scheduled tasks:

**Task 1: Obsidian MCP Server**
```powershell
schtasks /create /tn "Obsidian MCP Server" /tr "node C:\Users\Cynthia\AppData\Roaming\npm\node_modules\@cyanheads\obsidian-mcp-server\dist\index.js --config C:\Users\Cynthia\.obsidian-mcp\config.json" /sc onlogon /rl highest
```

> **Note:** The exact path to the server's entry point may differ. Find it with:
> ```powershell
> where obsidian-mcp-server
> ```

**Task 2: Cloudflare Tunnel**

Cloudflare Tunnel can install itself as a Windows service:
```powershell
cloudflared service install
```
This is the cleanest approach — it runs as a system service and auto-starts on boot.

**Task 3: Auth Proxy** (if using Option B above)
```powershell
schtasks /create /tn "Obsidian MCP Auth Proxy" /tr "node C:\Users\Cynthia\.obsidian-mcp\auth-proxy.js" /sc onlogon /rl highest
```

### Phase 5: Connect TypingMind

1. In TypingMind, go to **Settings → MCP**
2. Click **Add MCP**
3. Enter the HTTPS URL:
   - If Cloudflare with domain: `https://obsidian-mcp.yourdomain.com`
   - If ngrok: `https://your-chosen-name.ngrok-free.app`
4. Add custom headers:
   - If using Cloudflare Access:
     - `CF-Access-Client-Id`: (your service token ID)
     - `CF-Access-Client-Secret`: (your service token secret)
   - If using the auth proxy:
     - `x-api-key`: (your generated API key)
5. Save and test the connection

### Phase 6: Verify

From any terminal (or PowerShell):
```powershell
curl -H "x-api-key: YOUR_KEY_HERE" https://YOUR-URL/mcp
```

You should get an MCP discovery response listing the available tools.

---

## Security Caveats

1. **This exposes your vault to the internet** (behind auth). Keep that API key safe. Don't share the URL publicly.
2. **The MCP server has full read/write access** to the vault folder. Claude can create, edit, and delete notes. Back up your vault.
3. **Cloudflare Tunnel is encrypted end-to-end** — traffic between TypingMind and your PC is TLS-secured.
4. **If using ngrok free tier**, be aware that URLs might be inspected by ngrok. Paid tier avoids this.

## Fallback: If cyanheads Doesn't Work

If the primary server has issues, alternatives in order of preference:
1. **BoweyLou/obsidian-mcp-server-enhanced** — Fork with documented HTTP transport
2. **NasAndNora/obsidian-http-mcp** — HTTP-native alternative
3. **Any stdio MCP server + supergateway** — `npx supergateway --stdio "obsidian-mcp-server" --port 3456` converts stdio to HTTP

## Questions for Cynthia Before Starting

1. **Does Cynthia own a domain?** (Affects Cloudflare Tunnel setup — if no domain, ngrok with free static domain is easier)
2. **Is Node.js already installed?** (LiteLLM suggests Python is, but need to confirm Node.js)
3. **Is the vault currently being synced to GitHub?** (If so, we should be careful about conflicts between MCP writes and git syncs)

---

*Blueprint delivered. Over to you, Soldier. 🫡*
*— Claude Code, Architect Division*
