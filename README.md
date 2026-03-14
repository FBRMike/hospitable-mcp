# Hospitable MCP Server - Breezy Beach Stays

MCP server connecting Claude to the Hospitable API v2. Supports dual transport:
- **STDIO** for Claude Desktop (local)
- **Streamable HTTP** for claude.ai custom connector (remote)

## Available Tools

| Tool | Description |
|------|------------|
| `list_properties` | List all 50+ properties with listings |
| `get_property` | Get details for a specific property |
| `search_properties` | Find available properties in a date range |
| `list_reservations` | List reservations with filters (requires property UUIDs) |
| `get_reservation` | Full reservation details with financials |
| `get_upcoming_checkins` | Check-ins in the next N days across all properties |
| `get_upcoming_checkouts` | Check-outs in the next N days across all properties |
| `get_calendar` | Pricing and availability calendar for a property |
| `update_calendar` | Update pricing/availability/min stay (prices in cents) |
| `list_messages` | Get guest conversation thread for a reservation |
| `send_message` | Send a message to a guest via booking platform |
| `list_reviews` | Get reviews for a property |
| `respond_to_review` | Post a public review response |
| `get_account_info` | Verify API connection / get account details |

## Quick Setup

### 1. Get Your Hospitable PAT

1. Log in to my.hospitable.com
2. Go to Apps > API access > Access tokens
3. Click + Add new, select Read and Write permissions
4. Copy the token

### 2. Install

```bash
cd hospitable-mcp
python -m venv venv
source venv/bin/activate   # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

### 3a. Claude Desktop (STDIO)

Edit `~/Library/Application Support/Claude/claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "hospitable": {
      "command": "/path/to/hospitable-mcp/venv/bin/python",
      "args": ["/path/to/hospitable-mcp/server.py"],
      "env": {
        "HOSPITABLE_PAT": "your-token-here"
      }
    }
  }
}
```

Restart Claude Desktop. You should see "Hospitable" in the MCP tools list.

### 3b. Claude.ai Custom Connector (Streamable HTTP)

Start the server in HTTP mode:

```bash
export HOSPITABLE_PAT="your-token-here"
python server.py --http --port 8000
```

Then in claude.ai:
1. Go to Settings > Integrations (or the connectors menu)
2. Add custom connector
3. Enter your server URL: `http://your-server-ip:8000/mcp`

For production, run behind a reverse proxy (nginx/caddy) with HTTPS.

### 3c. Run as a Background Service (macOS)

Create `~/Library/LaunchAgents/com.breezy.hospitable-mcp.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.breezy.hospitable-mcp</string>
    <key>ProgramArguments</key>
    <array>
        <string>/path/to/hospitable-mcp/venv/bin/python</string>
        <string>/path/to/hospitable-mcp/server.py</string>
        <string>--http</string>
        <string>--port</string>
        <string>8000</string>
    </array>
    <key>EnvironmentVariables</key>
    <dict>
        <key>HOSPITABLE_PAT</key>
        <string>your-token-here</string>
    </dict>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/tmp/hospitable-mcp.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/hospitable-mcp.err</string>
</dict>
</plist>
```

Load it:
```bash
launchctl load ~/Library/LaunchAgents/com.breezy.hospitable-mcp.plist
```

## Important API Notes

- `/reservations` endpoint REQUIRES `properties[]` as a parameter. Always fetch properties first.
- Calendar prices are in CENTS. $150/night = 15000.
- Message rate limits: 2/min per reservation, 50/5min global.
- Calendar update rate limit: 1000 req/min.
- Messages with no sender field are AI-generated (Hospitable automation).
- PAT tokens expire after ~1 year. Rotate at my.hospitable.com.

## Combining with Other MCP Servers

Claude Desktop supports multiple MCP servers. Example config:

```json
{
  "mcpServers": {
    "hospitable": {
      "command": "/path/to/hospitable-mcp/venv/bin/python",
      "args": ["/path/to/hospitable-mcp/server.py"],
      "env": { "HOSPITABLE_PAT": "your-token" }
    },
    "pricelabs": {
      "command": "python",
      "args": ["/path/to/pricelabs-mcp/server.py"],
      "env": { "PRICELABS_API_KEY": "your-key" }
    }
  }
}
```

## Troubleshooting

**"HOSPITABLE_PAT environment variable is not set"**
- Make sure the env var is set in the Claude Desktop config or your shell

**401 Unauthorized**
- Your PAT may have expired. Generate a new one from my.hospitable.com

**Rate limit errors (429)**
- Wait and retry. The server returns rate limit details in the error.

**STDIO mode works but HTTP doesn't connect from claude.ai**
- Make sure your firewall allows inbound on the port
- If behind a router, set up port forwarding or use a tunnel (ngrok, Cloudflare Tunnel)
- claude.ai custom connector needs HTTPS in production
