# NightyScript — Extended Developer Documentation

> Built from the complete official docs at [docs.nighty.one](https://docs.nighty.one) and [github.com/aboveproof/nighty-docs](https://github.com/aboveproof/nighty-docs), extended with deeper explanations, new patterns, caveats discovered through analysis, and practical examples beyond what the originals cover.

> **Path note for whoever (or whatever) reads this next:** this file lives at the root of Nighty's own data folder — Windows: `%APPDATA%\Nighty Selfbot` (i.e. `C:\Users\<username>\AppData\Roaming\Nighty Selfbot\docs.md`). Don't hardcode a specific username from examples elsewhere in this doc — resolve `%APPDATA%\Nighty Selfbot` (or the platform equivalent) fresh on whatever machine you're actually running on. Scripts, `data/`, `data/scripts/`, `custom_features.json`, etc. all hang off that same root.

---

## Table of Contents

1. [What Is NightyScript?](#1-what-is-nightyscript)
2. [Architecture Overview](#2-architecture-overview)
3. [Script Structure & Boilerplate](#3-script-structure--boilerplate)
4. [Imports — What Works, What Doesn't](#4-imports--what-works-what-doesnt)
5. [Configuration & Persistent Storage](#5-configuration--persistent-storage)
6. [Commands (@bot.command)](#6-commands-botcommand)
7. [Event Listeners (@bot.listen)](#7-event-listeners-botlisten)
8. [Message Sending](#8-message-sending)
9. [Slash Command Automation (exec_slash)](#9-slash-command-automation-exec_slash)
10. [Async Rules & Patterns](#10-async-rules--patterns)
11. [Logging](#11-logging)
12. [UI Scripting API](#12-ui-scripting-api)
13. [Webhook Integration](#13-webhook-integration)
14. [What Works vs What Doesn't](#14-what-works-vs-what-doesnt)
15. [Full Script Examples](#15-full-script-examples)
16. [Dynamic Rich Presence Values (addDRPCValue)](#16-dynamic-rich-presence-values-adddrpcvalue)
17. [Discord API Direct Calls (bot.http.request)](#17-discord-api-direct-calls-bothttprequest)
18. [Guild & Member Operations](#18-guild--member-operations)
19. [Advanced UI Patterns](#19-advanced-ui-patterns)
20. [Message Utilities](#20-message-utilities)
21. [Config & File Path Patterns](#21-config--file-path-patterns)
22. [Real Script Patterns — Quick Reference](#22-real-script-patterns--quick-reference)
23. [Additional Events](#23-additional-events)
24. [Slash Command Framework — nighty-slash-api](#24-slash-command-framework--nighty-slash-api)
25. [Custom Features](#25-custom-features)
---

## 1. What Is NightyScript?

NightyScript is Nighty's Python scripting layer built on top of a **modified `discord.py-self`** library. The key distinction from a regular Discord bot:

- Runs under a **user account token**, not a bot token.
- Has access to features normal bots don't (reading messages without intents, reacting to others' messages naturally, seeing deleted messages via cache, etc.).
- Is limited by Discord's user-client API — no scanning all servers, no global member caches.
> **Important**: Nighty itself manages the Discord WebSocket connection. Your script only defines commands and listeners. Never try to reconnect, restart the bot, or manage the event loop yourself.

---

## 2. Architecture Overview

```
Nighty App (Electron)
│
├── Discord WebSocket (discord.py-self, modified)
│   └── bot object (globally available in scripts)
│
├── NightyScript Engine
│   ├── Loads scripts from scripts directory
│   ├── Registers @bot.command and @bot.listen decorators
│   ├── Provides built-in globals:
│   │   ├── bot
│   │   ├── nightyScript (decorator)
│   │   ├── getScriptsPath()
│   │   ├── getConfigData() / updateConfigData()
│   │   ├── forwardEmbedMethod()
│   │   ├── Tab, UI (UI scripting)
│   │   └── discord.* objects (without importing)
│   └── Manages script lifecycle
│
└── UI Layer (React)
    └── Custom Tabs rendered via Tab + UI API
```

Everything you need from `discord` or `nighty` is **injected into the global scope**. You don't import them. Attempting to import them is explicitly prohibited and will crash your script.

---

## 3. Script Structure & Boilerplate

Every script follows this exact pattern:

```python
from pathlib import Path
import json
import asyncio

@nightyScript(
    name="My Script",
    author="yourname",
    description="Short description of what this script does.",
    usage="<p>mycommand <arg> [-flag]"
)
def my_script():
    """
    MY SCRIPT
    ---------
    What it does and why.

    COMMANDS:
    <p>mycommand <arg>       - Does the main thing
    <p>mycommand list        - Lists stored items
    <p>mycommand help        - Shows this help

    SETUP:
    - No external dependencies required.

    NOTES:
    - <p> is replaced with the user's configured prefix at runtime.
    """

    # === CONSTANTS ===
    SCRIPT_NAME = "MyScript"
    BASE_DIR = Path(getScriptsPath()) / "json"
    DATA_FILE = BASE_DIR / "my_script.json"
    BASE_DIR.mkdir(parents=True, exist_ok=True)

    # === HELPERS ===
    def load():
        try:
            with open(DATA_FILE) as f:
                return json.load(f)
        except (FileNotFoundError, json.JSONDecodeError):
            return {}

    def save(data):
        with open(DATA_FILE, "w") as f:
            json.dump(data, f, indent=2)

    # === COMMANDS ===
    @bot.command(name="mycommand", description="Does the main thing.")
    async def mycommand(ctx, *, args: str = ""):
        await ctx.message.delete()
        # ... your logic ...
        await ctx.send("Done.")

    # === LISTENERS ===
    @bot.listen("on_message")
    async def on_msg(message):
        if message.author.id == bot.user.id:
            return
        # ... your logic ...

# THIS IS REQUIRED — never forget it
my_script()
```

### Critical Rules

- The function decorated by `@nightyScript` **must be called** at the end (`my_script()`). Without this, nothing registers.
- All commands and listeners must be **defined inside** the main function body.
- The `<p>` in `usage` is not literal — Nighty substitutes the user's prefix at runtime.
---

## 4. Imports — What Works, What Doesn't

### ✅ Allowed

```python
import json
import asyncio
import re
import os
import time
import traceback
import aiohttp
import requests       # synchronous — must use run_in_thread
from pathlib import Path
from datetime import datetime
```

### ❌ Prohibited (will crash script)

```python
import discord                    # DO NOT
import nighty                     # DO NOT
from discord import *             # DO NOT
from nighty import *              # DO NOT
from nighty import bot, Tab, UI   # DO NOT
import matplotlib                 # DO NOT (and most non-stdlib packages)
import numpy                      # DO NOT
import pydub                      # DO NOT
```

### Why?

NightyScript injects these into the global namespace before your script runs. Re-importing them causes conflicts. The engine provides `discord.File`, `discord.http.Route`, and other discord objects **without you needing to import anything**.

### External packages that ARE allowed (community-confirmed)

```python
import aiohttp          # async HTTP (built-in to Nighty's env)
import requests         # sync HTTP (wrap with run_in_thread)
from bs4 import BeautifulSoup   # web scraping
import pandas           # data manipulation (for data scripts)
import selenium         # browser automation (for advanced scripts)
```

> All external dependencies **must be documented** in your script's docstring with install instructions.

---

## 5. Configuration & Persistent Storage

### 5.1 Simple Key-Value Config

Best for: booleans, single strings, numbers, user preferences.

```python
# Read with a default
api_key = getConfigData().get("myscript_api_key", "")
debug   = getConfigData().get("myscript_debug", False)

# Write
updateConfigData("myscript_api_key", "abc123")
updateConfigData("myscript_debug", True)
```

**Namespace your keys** — always prefix with your script name to avoid collisions with other scripts:
- ✅ `"myscript_webhook_url"`
- ❌ `"webhook_url"` (collides with other scripts)
### 5.2 JSON Storage

Best for: lists of IDs, complex objects, history logs.

```python
from pathlib import Path
import json

BASE_DIR = Path(getScriptsPath()) / "json"
DATA_FILE = BASE_DIR / "myscript_data.json"
BASE_DIR.mkdir(parents=True, exist_ok=True)

DEFAULT_DATA = {
    "items": [],
    "blocked_users": [],
    "settings": {}
}

def load_data() -> dict:
    try:
        with open(DATA_FILE) as f:
            return json.load(f)
    except (FileNotFoundError, json.JSONDecodeError):
        return DEFAULT_DATA.copy()

def save_data(data: dict) -> bool:
    try:
        with open(DATA_FILE, "w") as f:
            json.dump(data, f, indent=2)
        return True
    except IOError as e:
        print(f"[MyScript] Save error: {e}", type_="ERROR")
        return False

# Initialize on first run
if not DATA_FILE.exists():
    save_data(DEFAULT_DATA)
```

### 5.3 Config vs JSON — Decision Guide

| Use Case | Use |
|---|---|
| API key, webhook URL | `getConfigData` |
| Feature toggle (on/off) | `getConfigData` |
| List of user IDs to watch | JSON file |
| Message history log | JSON file |
| Script statistics/counters | JSON file |
| Complex nested objects | JSON file |

---

## 6. Commands (@bot.command)

### 6.1 Basic Command

```python
@bot.command(
    name="ping",
    aliases=["p"],
    description="Check latency."
)
async def ping(ctx, *, args: str = ""):
    await ctx.message.delete()
    latency = round(bot.latency * 1000)
    await ctx.send(f"Pong! `{latency}ms`")
```

### 6.2 Argument Parsing Patterns

```python
@bot.command(name="calc")
async def calc(ctx, *, args: str = ""):
    await ctx.message.delete()

    if not args:
        await ctx.send("Usage: `<p>calc <expression>`")
        return

    # Simple split
    parts = args.strip().split()
    first = parts[0] if parts else ""

    # Flag detection
    silent = "--silent" in parts or "-s" in parts
    verbose = "--verbose" in parts or "-v" in parts

    # Remove flags from parts
    clean_parts = [p for p in parts if not p.startswith("-")]

    # Regex parsing (for structured input)
    import re
    match = re.match(r"(\d+)\s*([\+\-\*\/])\s*(\d+)", args)
    if match:
        a, op, b = match.groups()
        result = eval(f"{a}{op}{b}")  # safe here since regex ensures numbers only
        await ctx.send(f"`{a} {op} {b} = {result}`")
    else:
        await ctx.send("Invalid expression.")
```

### 6.3 Subcommand Pattern

```python
@bot.command(name="watch", description="Manage watch list.")
async def watch(ctx, *, args: str = ""):
    await ctx.message.delete()

    parts = args.strip().split(maxsplit=1)
    sub = parts[0].lower() if parts else "help"
    rest = parts[1] if len(parts) > 1 else ""

    if sub == "add":
        await _watch_add(ctx, rest)
    elif sub == "remove":
        await _watch_remove(ctx, rest)
    elif sub in ("list", "ls"):
        await _watch_list(ctx)
    else:
        await ctx.send(
            "**Watch commands:**\n"
            "`<p>watch add <user_id>` — Add user\n"
            "`<p>watch remove <user_id>` — Remove user\n"
            "`<p>watch list` — Show list"
        )

async def _watch_add(ctx, user_id: str):
    data = load_data()
    if user_id not in data["items"]:
        data["items"].append(user_id)
        save_data(data)
        await ctx.send(f"Added `{user_id}`.")
    else:
        await ctx.send("Already in list.")

async def _watch_remove(ctx, user_id: str):
    data = load_data()
    if user_id in data["items"]:
        data["items"].remove(user_id)
        save_data(data)
        await ctx.send(f"Removed `{user_id}`.")
    else:
        await ctx.send("Not in list.")

async def _watch_list(ctx):
    data = load_data()
    items = data.get("items", [])
    if items:
        await ctx.send("**Watch list:**\n" + "\n".join(f"- `{i}`" for i in items))
    else:
        await ctx.send("List is empty.")
```

### 6.4 Status Messages

For long operations, use a temporary status message:

```python
@bot.command(name="fetch")
async def fetch_cmd(ctx, *, url: str = ""):
    await ctx.message.delete()
    msg = await ctx.send("⏳ Fetching...")
    try:
        data = await fetch_json(url)
        await msg.edit(content=f"✅ Done: `{len(data)} bytes`")
    except Exception as e:
        await msg.edit(content=f"❌ Error: {e}")
```

---

## 7. Event Listeners (@bot.listen)

### 7.1 Available Events

| Event | Parameters | Notes |
|---|---|---|
| `on_message` | `message` | Fires on every visible message |
| `on_message_edit` | `before, after` | Both are message objects |
| `on_message_delete` | `message` | Message may be partially cached |
| `on_reaction_add` | `reaction, user` | |
| `on_voice_state_update` | `member, before, after` | |
| `on_member_join` | `member` | Fires in guilds only |
| `on_member_remove` | `member` | |

### 7.2 The Self-Check (Always Required)

```python
@bot.listen("on_message")
async def handler(message):
    if message.author.id == bot.user.id:
        return   # Never react to your own messages — infinite loop risk
    if message.author.bot:
        return   # Optional: ignore other bots
```

### 7.3 Filtering Pattern

```python
@bot.listen("on_message")
async def keyword_watcher(message):
    if message.author.id == bot.user.id:
        return

    data = load_data()
    allowed_guilds = data.get("allowed_guilds", [])
    keywords = data.get("keywords", [])

    # Guild filter (empty list = all guilds allowed)
    if allowed_guilds and str(getattr(message.guild, "id", "")) not in allowed_guilds:
        return

    content_lower = message.content.lower()
    for kw in keywords:
        if kw.lower() in content_lower:
            await forwardEmbedMethod(
                channel_id=message.channel.id,
                title="Keyword Detected",
                content=f"**Keyword:** `{kw}`\n**From:** {message.author}\n**Content:** {message.content[:200]}"
            )
            break
```

### 7.4 Message Edit Logger

```python
@bot.listen("on_message_edit")
async def edit_logger(before, after):
    if after.author.id == bot.user.id:
        return
    if before.content == after.content:
        return  # Embed updates trigger this too — skip content-identical edits

    log_channel = getConfigData().get("myscript_log_channel")
    if not log_channel:
        return

    await forwardEmbedMethod(
        channel_id=log_channel,
        title="Message Edited",
        content=(
            f"**Author:** {after.author} (`{after.author.id}`)\n"
            f"**Channel:** {after.channel}\n"
            f"**Before:** {before.content[:400]}\n"
            f"**After:** {after.content[:400]}"
        )
    )
```

---

## 8. Message Sending

### 8.1 ctx.send — Basic Text

```python
await ctx.send("Hello world")
await ctx.send("Silent message", silent=True)  # No notification ping
```

### 8.1.5 ctx.nighty_send / ctx.nighty_help / ctx.nighty_command_help (added Nighty 2.3, confirmed via changelog — not covered elsewhere in this doc)

Three extra context methods, distinct from `ctx.send`:

```python
# Styled message with an optional title and file, in Nighty's own look
await ctx.nighty_send(content="Hello world", title="Optional Title", file=None)

# Auto-generated command list/help embed
await ctx.nighty_help(title="All commands", commands=bot.commands)

# Help for one specific command
await ctx.nighty_command_help(some_command)
```

The changelog doesn't specify exact parameter types beyond what's shown (`title: Optional[str]`, `file: Optional[discord.File]`, `commands: list[Command]`) — treat as confirmed-to-exist, details unverified beyond this.

### 8.2 forwardEmbedMethod — Rich Embeds

```python
await forwardEmbedMethod(
    channel_id=ctx.channel.id,   # Required — target channel ID
    title="My Title",            # Optional
    content="**Bold** and *italic* and\n> quoted text\n\n- List item",
    image="https://example.com/img.png"  # Optional
)
```

**Supported markdown in `content`:**
- `# Heading`, `## Heading`
- `**bold**`, `*italic*`, `__underline__`, `~~strikethrough~~`
- `> blockquote`
- `- item` (bullet lists), `1. item` (numbered lists)
- `[text](url)` (links)
- `` `code` `` (inline code)
**NOT supported in forwardEmbedMethod:**
- `thumbnail`, `color`, `footer`, `fields`, `description` — these will throw "unexpected keyword argument"
### 8.3 Private Mode — Critical Gotcha

If the user has **private mode** enabled, `forwardEmbedMethod` may be blocked. Wrap sends like this:

```python
previous_private = getConfigData().get("private")
updateConfigData("private", False)
try:
    await forwardEmbedMethod(channel_id=ctx.channel.id, content="...", title="...")
except Exception as e:
    print(f"Embed send failed: {e}", type_="ERROR")
finally:
    updateConfigData("private", previous_private)
```

### 8.4 Sending Files

```python
import io

# From bytes
file_bytes = b"Hello file content"
file = discord.File(io.BytesIO(file_bytes), filename="output.txt")
await ctx.send(file=file)

# From disk
file = discord.File("path/to/file.png", filename="image.png")
await ctx.send(file=file)
```

### 8.5 Stickers

```python
STICKER_ID = 749054660769218631
sticker = bot.get_sticker(STICKER_ID) or await bot.fetch_sticker(STICKER_ID)
await ctx.send(stickers=[sticker])
```

### 8.6 Custom Emojis

```python
EMOJI_ID = 123456789012345678
emoji = bot.get_emoji(EMOJI_ID)
await ctx.send(str(emoji) if emoji else f"<:emoji:{EMOJI_ID}>")
```

---

## 9. Slash Command Automation (exec_slash)

This is one of NightyScript's most powerful features. It lets you programmatically trigger slash commands from other bots.

> **See also**: [Section 24](#24-slash-command-framework--nighty-slash-api) covers `nighty-slash-api`, a separate library for **creating your own** real slash commands (as opposed to `exec_slash`, which **invokes existing** slash commands belonging to other bots). Don't confuse the two — this section is about calling someone else's `/play`, section 24 is about registering your own `/ping`.

### 9.1 The exec_slash Helper (paste into your script)

```python
async def exec_slash(ctx, application_id: str, cmdn: str, opt: dict = None):
    """
    Execute a Discord slash command from another bot.
    Args:
        ctx: command context (needs .guild and .channel)
        application_id: bot's application ID (string)
        cmdn: command name, space-separated for subcommands ("play" or "queue add")
        opt: dict of parameter_name -> value
    """
    opt = opt or {}
    top_cmd = cmdn.split(" ")[0]

    try:
        data = await bot.http.request(
            discord.http.Route("GET", "/users/@me/application-command-index")
        )
    except Exception as e:
        print(f"[exec_slash] Could not fetch commands: {e}", type_="ERROR")
        return

    full_cmd = next(
        (c for c in data.get("application_commands", [])
         if str(c["application_id"]) == application_id and c["name"] == top_cmd),
        None
    )
    if not full_cmd:
        print(f"[exec_slash] Command '{top_cmd}' not found for app {application_id}", type_="ERROR")
        return

    parts = cmdn.split(" ")
    target = full_cmd
    for part in parts[1:]:
        opts_map = {o["name"]: o for o in target.get("options", [])}
        target = opts_map.get(part)
        if not target:
            print(f"[exec_slash] Subcommand '{part}' not found", type_="ERROR")
            return

    # Format options
    formatted = []
    if opt and target.get("options"):
        opts_map = {o["name"]: o for o in target["options"]}
        for name, value in opt.items():
            if name in opts_map:
                formatted.append({"type": opts_map[name]["type"], "name": name, "value": value})

    # Build payload
    if len(parts) > 2:
        payload_opts = [{"type": 2, "name": parts[1], "options": [{"type": 1, "name": parts[2], "options": formatted}]}]
    elif len(parts) > 1:
        payload_opts = [{"type": 1, "name": parts[1], "options": formatted}]
    else:
        payload_opts = formatted

    payload = {
        "type": 2,
        "application_id": application_id,
        "guild_id": str(ctx.guild.id) if ctx.guild else None,
        "channel_id": str(ctx.channel.id),
        "session_id": bot.ws.session_id,
        "nonce": str(int(datetime.now().timestamp() * 1000)),
        "data": {
            "version": full_cmd["version"],
            "id": full_cmd["id"],
            "name": full_cmd["name"],
            "type": full_cmd["type"],
            "options": payload_opts,
            "application_command": full_cmd,
            "attachments": []
        }
    }

    try:
        await bot.http.request(discord.http.Route("POST", "/interactions"), json=payload)
        print(f"[exec_slash] Executed '{cmdn}'", type_="SUCCESS")
    except Exception as e:
        print(f"[exec_slash] Failed '{cmdn}': {e}", type_="ERROR")
```

### 9.2 Usage Examples

```python
# Simple command
await exec_slash(ctx, "bot_app_id", "ping")

# Command with parameters
await exec_slash(ctx, "bot_app_id", "play", opt={"query": "lofi beats"})

# Subcommand
await exec_slash(ctx, "bot_app_id", "queue add", opt={"song": "Never Gonna Give You Up"})

# Sequential with delay
await exec_slash(ctx, music_bot_id, "play", opt={"query": song})
await asyncio.sleep(2)
await exec_slash(ctx, music_bot_id, "loop", opt={"mode": "track"})
```

### 9.3 Option Type Reference

| Type | Discord Type |
|---|---|
| 3 | String |
| 4 | Integer |
| 5 | Boolean |
| 6 | User |
| 7 | Channel |
| 8 | Role |

### 9.4 Finding an Application ID

1. Enable **Developer Mode** in Discord (Settings → Advanced)
2. Right-click a message from the target bot
3. Click **"Copy ID"** on the bot's avatar in the popup
4. The user ID = application ID for most bots
### 9.5 Known Limitations

- Executions are **visible** to all users in the channel (you appear to invoke the command)
- Discord rate-limits interactions — add `asyncio.sleep(1)` between sequential calls
- The target bot must be **present in the guild**
- Some bots may ban accounts that spam their commands
---

## 10. Async Rules & Patterns

### 10.1 The Golden Rule

Never use blocking code inside `async` functions. It freezes the entire bot.

```python
# ❌ WRONG
@bot.command(name="slow")
async def slow(ctx, *, args):
    import time
    time.sleep(5)  # FREEZES everything
    await ctx.send("done")

# ✅ RIGHT
@bot.command(name="slow")
async def slow(ctx, *, args):
    await asyncio.sleep(5)  # non-blocking
    await ctx.send("done")
```

### 10.2 HTTP Requests

```python
# ✅ Async with aiohttp
import aiohttp

async def fetch_json(url: str, headers: dict = None) -> dict | None:
    try:
        async with aiohttp.ClientSession(headers=headers or {}) as session:
            async with session.get(url, timeout=aiohttp.ClientTimeout(total=10)) as resp:
                resp.raise_for_status()
                return await resp.json()
    except Exception as e:
        print(f"fetch_json error: {e}", type_="ERROR")
        return None

# ✅ Sync requests wrapped in thread
import asyncio
import requests

async def run_in_thread(func, *args, **kwargs):
    loop = asyncio.get_event_loop()
    return await loop.run_in_executor(None, lambda: func(*args, **kwargs))

def _sync_get(url: str) -> dict | None:
    try:
        r = requests.get(url, timeout=10)
        r.raise_for_status()
        return r.json()
    except Exception as e:
        print(f"_sync_get error: {e}", type_="ERROR")
        return None

# Usage:
data = await run_in_thread(_sync_get, "https://api.example.com/data")
```

### 10.3 Background Tasks

Run periodic tasks without blocking commands:

```python
import asyncio

async def background_task():
    await asyncio.sleep(5)  # wait for bot to be ready
    while True:
        try:
            # Do periodic work
            print("Background tick", type_="INFO")
        except Exception as e:
            print(f"Background task error: {e}", type_="ERROR")
        await asyncio.sleep(60)  # run every 60 seconds

# Start it inside your script function
asyncio.ensure_future(background_task())
```

---

## 11. Logging

### 11.1 Basic

```python
print("Script loaded", type_="INFO")
print("Failed to fetch data", type_="ERROR")
print("Item saved successfully", type_="SUCCESS")
```

### 11.2 Structured Logger

```python
from datetime import datetime
import traceback

SCRIPT_NAME = "MyScript"

def log(message: str, level: str = "INFO", exc: bool = False):
    ts = datetime.now().strftime("%H:%M:%S")
    entry = f"[{ts}] [{SCRIPT_NAME}] {message}"
    if exc:
        tb = traceback.format_exc()
        if tb and tb.strip() != "NoneType: None":
            entry += f"\n{tb}"
    print(entry, type_=level.upper())

# Usage
log("Command executed")
log("File not found", level="ERROR", exc=True)
log("Config saved", level="SUCCESS")
```

---

## 11.5 Undocumented Built-in Globals (Discovered from Real Scripts)

These are injected by Nighty and confirmed working — none appear in the official docs.

### Theme Functions

```python
# Returns list of theme names (strings)
themes = getThemesData()
# → ["Dark Theme", "Blue Theme", ...]

# Returns the full JSON data for a specific theme
theme = getThemeData(theme_name="Dark Theme")
# → {"enabled": True, "color": "#...", "v2": {...}, ...}

# Returns the path to Nighty's data directory
data_path = getDataPath()
# → "C:/Users/user/AppData/Roaming/Nighty Selfbot/data" (Windows)

# Save theme data back to disk:
import json
json.dump(theme_data, open(f"{getDataPath()}/themes/{theme_name}.json", "w", encoding="utf-8"), indent=2)
```

### Other Confirmed Globals

```python
import uuid     # Works — used for generating unique row IDs
import re       # Works — used for URL/pattern validation
import json     # Works — standard usage
```

---

## 12. UI Scripting API

UI scripts let you build **custom tabs** inside the Nighty app with interactive elements. This is completely separate from Discord — it's the Nighty desktop UI.

### 12.1 Hierarchy

```
Tab
└── CardContainer (type="columns" | "rows")
    ├── Card
    │   ├── Group (optional, for layout)
    │   │   └── UI.Elements
    │   └── UI.Elements
    └── CardContainer (nested, max depth 3)
```

### 12.2 Tab

```python
from nighty import Tab, UI   # ❌ Don't do this in regular scripts
# Tab and UI are available globally in UI scripts

tab = Tab(
    name="My Tool",     # Must be unique, max 36 chars
    icon="users",       # See icon list below
    title="My Tool",    # Shown at the top of the tab
    gap=6               # Spacing between containers (0-10)
)

# Always call last:
tab.render()
```

**Available icons:** `report`, `chart`, `two_way`, `inbox`, `star`, `search`, `variants`, `message`, `users`, `history`, `clean`, `magic`, `eraser`, `convert`, `cloud`, `calc`, `calendar`, `key`, `gift`, `trophy`, `book`, `sun`, `cart`, `pill`, `bookmark`, `mail`, `bear`, `heart`, `music`, `umbrella`, `tube`, `palette`, `lock`, `fire`, `preferences`, `power`, `share`, `bell`, `trash`, `nitro`

### 12.3 CardContainer

```python
# Never instantiate directly — use tab.create_container()
container = tab.create_container(
    type="columns",   # "columns" | "rows"
    width="full",     # "auto" | "full"
    height="full",    # "auto" | "full"
    gap=6
)

# Nested container
inner = container.create_container(type="rows")
```

### 12.4 Card

```python
# Never instantiate directly — use container.create_card()
card = container.create_card(
    type="rows",            # "columns" | "rows"
    width="full",           # "auto" | "full"
    height="auto",          # "auto" | "full"
    vertical_align="start", # "start" | "center" | "end"
    horizontal_align="start",
    gap=4,
    disallow_shrink=False   # True prevents card from shrinking
)
```

### 12.5 Group

```python
group = card.create_group(
    type="columns",
    horizontal_align="center",
    vertical_align="center",
    gap=2,
    full_width=True   # Force group to fill available width
)

# Groups can be nested
nested = group.create_group(type="rows")
```

### 12.6 UI Elements

All elements are created via `card.create_ui_element(UI.ElementClass, ...)` or `group.create_ui_element(...)`.

All attributes are **read/write** — you can update them live and the UI reflects the change immediately.

#### UI.Text

```python
text = card.create_ui_element(
    UI.Text,
    content="Hello World\nSecond line",  # \n for line breaks
    size="base",     # "tiny" | "sm" | "base" | "lg" | "xl" | "2xl"
    weight="normal", # "thin" | "light" | "normal" | "medium" | "bold" | "extrabold" | "black"
    color="#d5e0e6", # CSS color (hex, rgb, named)
    align="left",    # "left" | "center" | "right"
    margin="m-0",
    visible=True
)

# All attributes are live-writable:
text.content = "Updated!"
text.size = "xl"
text.weight = "bold"
text.align = "center"
```

#### UI.Button

```python
def on_click():
    print("clicked", type_="INFO")

btn = card.create_ui_element(
    UI.Button,
    label="Click Me",
    variant="solid",    # "solid" | "bordered" | "ghost" | "light" | "flat" | "cta"
    color="primary",    # "primary" | "default" | "success" | "danger"
    size="md",          # "sm" | "md"
    full_width=False,
    disabled=False,
    loading=False,
    onClick=on_click
)

# Async handler with loading state
async def async_click():
    btn.loading = True
    await asyncio.sleep(2)  # simulate work
    btn.loading = False
    tab.toast(type="SUCCESS", title="Done!", description="Operation completed.")

btn.onClick = async_click
```

#### UI.Input

```python
inp = card.create_ui_element(
    UI.Input,
    label="Username",
    placeholder="Enter your username",
    description="Your Discord username",
    show_clear_button=True,
    full_width=True,
    required=True
)

# Read value:
value = inp.value

# Validate on input:
def validate(value):
    inp.invalid = len(value) < 3
    inp.error_message = "Must be at least 3 characters" if inp.invalid else None

inp.onInput = validate
```

Full attribute set (confirmed against official docs, 2026-08-17): `label`, `placeholder`, `value`, `description`, `show_clear_button`, `invalid`, `disabled`, `readonly`, `required`, `full_width`, `error_message`, plus shared `margin`/`visible`. `disabled` and `readonly` weren't in the example above but are real, documented attributes.

#### UI.Toggle

```python
toggle = card.create_ui_element(
    UI.Toggle,
    label="Enable feature",
    checked=False
)

def on_toggle(checked):
    updateConfigData("myscript_feature", checked)
    toggle.checked = checked

toggle.onChange = on_toggle
```

Also has a `disabled` attribute (default `False`), confirmed against official docs.

#### UI.Select

```python
sel = card.create_ui_element(
    UI.Select,
    label="Choose server",
    mode="single",      # "single" | "multiple"
    full_width=True,
    items=[
        {"id": "1", "title": "Server Alpha"},
        {"id": "2", "title": "Server Beta", "iconUrl": "https://..."}
    ],
    selected_items=["1"],
    disabled_items=[]
)

def on_change(selected):
    print(f"Selected: {selected}", type_="INFO")

sel.onChange = on_change
```

Full attribute set (confirmed against official docs, 2026-08-17) also includes: `description`, `loading`, `disabled`, `invalid`, `error_message` — none of which were in the examples this doc had before.

#### UI.Table

> **Cell structure note**: the official docs show a nested dict format (`{"text": {"text": ..., "subtext": ..., "imageUrl": ...}}`) but real scripts use flat dicts. Use the flat format shown below — it's what actually works. Also confirmed: `items_per_page` officially defaults to `50`, not the `20` used as an example below; `selectable` defaults to `False`.

```python
table = card.create_ui_element(
    UI.Table,
    search=True,
    items_per_page=20,
    selectable=True,
    columns=[
        {"type": "text",   "label": "Name"},
        {"type": "tag",    "label": "Status"},
        {
            "type": "button",
            "label": "Actions",
            "buttons": [
                # onClick receives row_id as argument
                {"label": "Edit",   "color": "default", "onClick": lambda row_id: handle_edit(row_id)},
                {"label": "Delete", "color": "danger",  "onClick": lambda row_id: handle_delete(row_id)},
            ]
        }
    ],
    rows=[
        {
            "id": "row_1",       # Must be unique
            "cells": [
                {"text": "Alice", "subtext": "Admin", "imageUrl": "https://..."},  # text type: flat dict
                {"text": "Online", "color": "green"},                              # tag type: flat dict
                {}                                                                  # button type: empty dict
            ]
        }
    ]
)

def on_select(selected_rows):  # list of row ids
    print(f"Selected: {selected_rows}", type_="INFO")

table.onSelectionChange = on_select
```

**Table methods (undocumented in official docs, confirmed from real scripts):**

```python
# Insert rows at a specific position
table.insert_rows([
    {"id": "new_row", "cells": [{"text": "Bob"}, {"text": "Away", "color": "gray"}, {}]}
], position=len(table.rows))

# Delete specific rows by id
table.delete_rows(["row_1", "row_2"])

# Reorder rows by direct assignment (triggers UI update)
rows = table.rows
rows[0], rows[1] = rows[1], rows[0]
table.rows = rows
```

**Tag color values:** `"green"`, `"red"`, `"blue"`, `"yellow"`, `"orange"`, `"gray"` — these map to the UI's color tokens, not CSS.

#### UI.Image

```python
img = card.create_ui_element(
    UI.Image,
    url="https://example.com/image.png",  # Required (note: "url", not "src")
    alt="Description",       # string, default "Image"
    circle=False,            # True = circular crop
    shadow=False,            # True = drop shadow
    width="auto",            # "auto" | "50%" | "75px" etc.
    height="auto",           # "auto" | "50%" | "75px" etc.
    rounded="md",            # "none" | "sm" | "md" | "lg" | "xl" | "2xl"
    fill_type="contain",     # "cover" | "contain"
    border_color="#FFFFFF",  # CSS color
    border_width=0,          # 0-10
    margin="m-0",
    visible=True
)
```

> **Gotcha**: the attribute is `url=`, not `src=`.

#### UI.Checkbox

```python
cb = card.create_ui_element(
    UI.Checkbox,
    label="Accept terms",
    checked=False,
    disabled=False,
    invalid=False,    # True = visual error state
    visible=True
)

def on_check(checked):
    submit_btn.disabled = not checked

cb.onChange = on_check
```

### 12.7 Tab Toasts

```python
# Only visible when the custom tab is open
tab.toast(type="INFO",    title="FYI",   description="Something happened")
tab.toast(type="SUCCESS", title="Done",  description="Operation succeeded")
tab.toast(type="ERROR",   title="Oops",  description="Something went wrong")
```

### 12.8 Shared Attributes (All Elements)

| Attribute | Values | Default |
|---|---|---|
| `margin` | Tailwind margin class: `m-0`, `mt-2`, `mx-4`, `mb-6`, etc. | `"m-0"` |
| `visible` | `True \| False` | `True` |

Margin prefixes supported: `m-`, `mt-`, `mr-`, `mb-`, `ml-`, `mx-`, `my-` (values 0–10)

### 12.9 Complete UI Script Example

```python
@nightyScript(
    name="Server Stats",
    author="dev",
    description="Shows a dashboard of server stats in a custom tab.",
    usage="<p>stats refresh"
)
def server_stats():
    tab = Tab(name="Server Stats", icon="chart", title="Server Statistics", gap=6)

    top = tab.create_container(type="columns")

    # Left card: counters
    left = top.create_card(gap=4, vertical_align="start")
    heading = left.create_ui_element(UI.Text, content="Overview", size="xl")
    member_count = left.create_ui_element(UI.Text, content="Members: —", size="base")
    guild_count = left.create_ui_element(UI.Text, content="Servers: —", size="base")

    # Right card: refresh button
    right = top.create_card(vertical_align="center", horizontal_align="center")
    refresh_btn = right.create_ui_element(UI.Button, label="Refresh", variant="cta", full_width=True)

    async def on_refresh():
        refresh_btn.loading = True
        guilds = bot.guilds
        members = sum(g.member_count or 0 for g in guilds)
        member_count.content = f"Members: {members:,}"
        guild_count.content  = f"Servers: {len(guilds):,}"
        refresh_btn.loading = False
        tab.toast(type="SUCCESS", title="Refreshed", description=f"{len(guilds)} servers loaded.")

    refresh_btn.onClick = on_refresh

    # Bottom: guilds table
    bottom = tab.create_container(type="columns")
    bottom_card = bottom.create_card()

    guild_table = bottom_card.create_ui_element(
        UI.Table,
        search=True,
        items_per_page=15,
        columns=[
            {"type": "text", "label": "Server Name"},
            {"type": "text", "label": "Members"},
            {"type": "tag",  "label": "Boosted"}
        ],
        rows=[
            {
                "id": f"g_{g.id}",
                "cells": [
                    {"text": {"text": g.name}},
                    {"text": {"text": str(g.member_count or 0)}},
                    {"tag": {
                        "text": "Yes" if (g.premium_tier or 0) > 0 else "No",
                        "color": "green" if (g.premium_tier or 0) > 0 else "gray"
                    }}
                ]
            }
            for g in bot.guilds
        ]
    )

    tab.render()

server_stats()
```

---

## 13. Webhook Integration

For complex embeds with fields, colors, and footers that `forwardEmbedMethod` doesn't support:

```python
import requests
import json
import asyncio

async def run_in_thread(func, *args, **kwargs):
    loop = asyncio.get_event_loop()
    return await loop.run_in_executor(None, lambda: func(*args, **kwargs))

def send_webhook(url: str, embed: dict = None, content: str = None, username: str = None) -> bool:
    if not url:
        return False
    payload = {}
    if content:
        payload["content"] = content
    if embed:
        payload["embeds"] = [embed]
    if username:
        payload["username"] = username
    try:
        r = requests.post(url, json=payload, timeout=10)
        return r.status_code == 204
    except Exception as e:
        print(f"Webhook error: {e}", type_="ERROR")
        return False

# Example embed with fields (only available via webhook)
from datetime import datetime

embed = {
    "title": "Event Report",
    "description": "Something happened.",
    "color": 0x5865F2,
    "fields": [
        {"name": "Server", "value": "My Server", "inline": True},
        {"name": "Time",   "value": str(datetime.utcnow()), "inline": True}
    ],
    "footer": {"text": "NightyScript Logger"},
    "timestamp": datetime.utcnow().isoformat()
}

# Inside async context:
await run_in_thread(send_webhook, webhook_url, embed)
```

---

## 14. What Works vs What Doesn't

### ✅ Confirmed Working

| Feature | Notes |
|---|---|
| `@bot.command` | Full support, aliases work |
| `@bot.listen("on_message")` | Works, always check self-author |
| `@bot.listen("on_message_edit")` | Works |
| `@bot.listen("on_message_delete")` | Works (cached messages only) |
| `forwardEmbedMethod` | Only: `channel_id`, `content`, `title`, `image` |
| `getConfigData` / `updateConfigData` | Works, namespace your keys |
| `getThemesData()` | Undocumented — returns list of theme names |
| `getThemeData(theme_name=)` | Undocumented — returns theme JSON dict |
| `getDataPath()` | Undocumented — returns Nighty data directory path |
| JSON file storage | Fully works via `pathlib` + `json` |
| `aiohttp` | Available, async HTTP |
| `requests` + `run_in_thread` | Works for sync HTTP |
| `ctx.send(silent=True)` | No notification ping |
| `discord.File` | Works without import |
| `bot.guilds` | Available |
| `exec_slash` (custom helper) | Works, needs delay between calls |
| `import uuid` | Works |
| `import re` | Works |
| UI Scripting (Tab, Card, etc.) | Works |
| `UI.Table.insert_rows(rows, position=)` | Undocumented method — confirmed working |
| `UI.Table.delete_rows(row_ids)` | Undocumented method — confirmed working |
| `table.rows = rows` (direct assign) | Reorders rows and triggers re-render |
| `UI.Text` `weight` + `align` attrs | Confirmed **in** official docs (checked directly, 2026-08-17) — full attribute set is `content`, `size`, `weight`, `color`, `align`, `margin`, `visible`. Default `color` is `"#FFFFFF"`, not the `#d5e0e6` used as an example below |
| `UI.Image` full attribute set | Confirmed **in** official docs — `url` (required), `alt`, `circle`, `shadow`, `width`, `height`, `rounded`, `fill_type`, `border_color`, `border_width`, `margin`, `visible` |
| `bot_app` / `app_commands` (Nighty 2.3+) | Exposes the real slash-command tree to scripts — see [Section 24](#24-slash-command-framework--nighty-slash-api) |

### ❌ Confirmed NOT Working / Unsupported

| Feature | Why |
|---|---|
| `import discord` | Crashes script |
| `discord.Embed` | Not supported — use `forwardEmbedMethod` |
| `forwardEmbedMethod` with `fields`, `color`, `footer` | Throws "unexpected keyword argument" |
| Global user cache | Discord API limitation |
| Scanning all servers for members | Discord API limitation |
| `bot.fetch_members()` (without cache) | Returns partial data |
| `time.sleep()` inside async | Freezes entire bot |
| External packages not in Nighty's env | e.g., `numpy`, `matplotlib`, `pydub` |
| Nesting Cards | Prohibited by API |
| Nesting CardContainers > 3 deep | Will render poorly |

### ⚠️ Gotchas

| Gotcha | Fix |
|---|---|
| Private mode blocks `forwardEmbedMethod` | Temporarily set `"private"` to `False` |
| `message_delete` may have empty content | Cache may not have the message — wrap in `try/except` |
| `exec_slash` executions are visible in chat | No way around this |
| `Tab.name` must be globally unique | Use descriptive names, not generic like "Settings" |
| `UI.Select` items must have unique `id` | Duplicate IDs cause silent rendering bugs |
| `UI.Table` row IDs must be unique | Same issue |
| Script not loading? | Forgot to call `my_script()` at the end |
| Real slash command doesn't appear (nighty-slash-api) | Global sync can take up to 1h first time — use `guild=` while developing |

---

## 15. Full Script Examples

### 15.1 Auto-Responder with Keyword Management

```python
from pathlib import Path
import json

@nightyScript(
    name="KeywordResponder",
    author="dev",
    description="Auto-responds when keywords are detected.",
    usage="<p>kr add <keyword> | <p>kr remove <keyword> | <p>kr list | <p>kr on | <p>kr off"
)
def keyword_responder():
    SCRIPT = "KeywordResponder"
    DATA_FILE = Path(getScriptsPath()) / "json" / "keyword_responder.json"
    DATA_FILE.parent.mkdir(parents=True, exist_ok=True)

    def load():
        try:
            return json.loads(DATA_FILE.read_text())
        except:
            return {"keywords": {}, "enabled": True}

    def save(data):
        DATA_FILE.write_text(json.dumps(data, indent=2))

    if not DATA_FILE.exists():
        save({"keywords": {}, "enabled": True})

    @bot.command(name="kr", description="Manage keyword responder.")
    async def kr(ctx, *, args: str = ""):
        await ctx.message.delete()
        parts = args.strip().split(maxsplit=1)
        sub = parts[0].lower() if parts else "help"
        rest = parts[1] if len(parts) > 1 else ""
        data = load()

        if sub == "add":
            # Format: keyword -> response (separated by |)
            if "|" not in rest:
                await ctx.send("Usage: `<p>kr add <keyword> | <response>`")
                return
            kw, resp = [x.strip() for x in rest.split("|", 1)]
            data["keywords"][kw.lower()] = resp
            save(data)
            await ctx.send(f"Added keyword: `{kw}`")

        elif sub == "remove":
            kw = rest.strip().lower()
            if kw in data["keywords"]:
                del data["keywords"][kw]
                save(data)
                await ctx.send(f"Removed: `{kw}`")
            else:
                await ctx.send("Keyword not found.")

        elif sub == "list":
            kws = data.get("keywords", {})
            if kws:
                lines = "\n".join(f"- `{k}` → {v}" for k, v in kws.items())
                await ctx.send(f"**Keywords:**\n{lines}")
            else:
                await ctx.send("No keywords set.")

        elif sub == "on":
            data["enabled"] = True
            save(data)
            await ctx.send("Keyword responder **enabled**.")

        elif sub == "off":
            data["enabled"] = False
            save(data)
            await ctx.send("Keyword responder **disabled**.")

        else:
            await ctx.send(
                "**KeywordResponder Commands:**\n"
                "`<p>kr add <kw> | <response>` — Add keyword\n"
                "`<p>kr remove <kw>` — Remove keyword\n"
                "`<p>kr list` — List all\n"
                "`<p>kr on/off` — Toggle"
            )

    @bot.listen("on_message")
    async def kr_listener(message):
        if message.author.id == bot.user.id:
            return
        data = load()
        if not data.get("enabled", True):
            return
        content_lower = message.content.lower()
        for kw, resp in data.get("keywords", {}).items():
            if kw in content_lower:
                await message.channel.send(resp)
                break

keyword_responder()
```

### 15.2 API Client (Weather)

```python
import aiohttp

@nightyScript(
    name="Weather",
    author="dev",
    description="Fetch weather from wttr.in.",
    usage="<p>weather <city>"
)
def weather_script():

    @bot.command(name="weather", description="Get current weather.")
    async def weather(ctx, *, city: str = ""):
        await ctx.message.delete()
        if not city:
            await ctx.send("Usage: `<p>weather <city>`")
            return

        msg = await ctx.send(f"⛅ Fetching weather for **{city}**...")
        try:
            async with aiohttp.ClientSession() as session:
                url = f"https://wttr.in/{city}?format=j1"
                async with session.get(url, timeout=aiohttp.ClientTimeout(total=10)) as resp:
                    resp.raise_for_status()
                    data = await resp.json(content_type=None)

            current = data["current_condition"][0]
            temp_c  = current["temp_C"]
            feels   = current["FeelsLikeC"]
            desc    = current["weatherDesc"][0]["value"]
            humidity = current["humidity"]

            previous_private = getConfigData().get("private")
            updateConfigData("private", False)
            try:
                await forwardEmbedMethod(
                    channel_id=ctx.channel.id,
                    title=f"⛅ Weather — {city.title()}",
                    content=(
                        f"**Condition:** {desc}\n"
                        f"**Temperature:** {temp_c}°C (feels like {feels}°C)\n"
                        f"**Humidity:** {humidity}%"
                    )
                )
            finally:
                updateConfigData("private", previous_private)

            await msg.delete()

        except Exception as e:
            await msg.edit(content=f"❌ Error: {e}")

weather_script()
```

---

## 16. Dynamic Rich Presence Values (addDRPCValue)

One of the most powerful but completely undocumented globals. Lets you register **custom dynamic values** that can be referenced in Nighty's rich presence (Discord activity) templates.

### 16.1 How It Works

```python
# Signature
addDRPCValue(key: str, func: callable)
```

`func` is called every time Nighty refreshes the rich presence display. It must be **synchronous** and return a **string**. The returned string is inserted where `{key}` appears in the user's rich presence template.

### 16.2 Basic Example

```python
def my_value():
    return "Hello from script"

addDRPCValue("my_value", my_value)
# User sets rich presence text to: "Status: {my_value}"
# Displays: "Status: Hello from script"
```

### 16.3 Voice Channel Dynamic Values (VCDynamicValues pattern)

Full real-world implementation showing state tracking across events:

```python
def VCValues():
    IDLE_TS = "6372891638687"  # Far-future Discord snowflake used as "not in VC" marker

    def _ts() -> str:
        return str(int(datetime.now().timestamp() * 1000))

    def _cfg(key: str, fallback: str = "") -> str:
        return getConfigData().get(key, fallback)

    def _join(channel, muted: bool, deafened: bool):
        is_dm = not hasattr(channel, 'guild') or channel.guild is None
        if is_dm:
            updateConfigData("_vci", "DM call")
        else:
            updateConfigData("_vci", f"{channel.name} (server: {channel.guild.name})")
        updateConfigData("_vic", "true")
        updateConfigData("_vcm", "true" if muted else "false")
        updateConfigData("_vdf", "true" if deafened else "false")
        updateConfigData("_vcd", _ts())

    def _leave():
        updateConfigData("_vic", "false")
        updateConfigData("_vci", "")
        updateConfigData("_vcm", "false")
        updateConfigData("_vdf", "false")
        updateConfigData("_vcd", IDLE_TS)

    # Detect if already in VC on script load
    try:
        selfbot = bot.user
        if selfbot:
            for guild in bot.guilds:
                member = guild.get_member(selfbot.id)
                if member and member.voice and member.voice.channel:
                    found = member.voice.channel
                    if _cfg("_vcd", IDLE_TS) == IDLE_TS:
                        _join(found, member.voice.self_mute or member.voice.mute,
                              member.voice.self_deaf or member.voice.deaf)
                    break
            else:
                _leave()
    except Exception as e:
        print(f"[VCValues] {e}", type_="ERROR")

    @bot.listen("on_voice_state_update")
    async def _vsu(member, before, after):
        if member.id != bot.user.id:
            return
        ac = after.channel if after else None
        bc = before.channel if before else None
        if ac is not None:
            _join(ac, after.self_mute or after.mute, after.self_deaf or after.deaf)
        elif bc is not None:
            _leave()

    # Value functions (sync, return string)
    def vc_status():
        if _cfg("_vic") != "true":  return "Disconnected from voice:"
        if _cfg("_vdf") == "true":  return "Deafened in voice:"
        if _cfg("_vcm") == "true":  return "Muted in voice:"
        return "Connected to voice:"

    def vc_info():   return _cfg("_vci")
    def vc_start():  return _cfg("_vcd", IDLE_TS)

    addDRPCValue("vc_status", vc_status)
    addDRPCValue("vc_info",   vc_info)
    addDRPCValue("vc_start",  vc_start)

VCValues()
```

**Key techniques shown:**
- Using `getConfigData`/`updateConfigData` as shared state store between listener and value functions
- Detecting existing VC state on script load (bot may already be in VC when script loads)
- Using a sentinel timestamp (`IDLE_TS`) as the "not active" state for Discord's `start` field
### 16.4 Weather Dynamic Values (API + cache pattern)

```python
import requests
import time

CACHE_TTL = 300  # 5 minutes — don't hammer the API

_cache = {"data": None, "ts": 0}

def _fetch():
    now = time.time()
    if _cache["data"] and (now - _cache["ts"]) < CACHE_TTL:
        return _cache["data"]
    
    api_key = getConfigData().get("weather_api_key", "")
    city    = getConfigData().get("weather_city", "London,GB")
    units   = getConfigData().get("weather_units", "metric")
    
    if not api_key:
        return None
    try:
        r = requests.get(
            "https://api.openweathermap.org/data/2.5/weather",
            params={"q": city, "appid": api_key, "units": units},
            timeout=10
        )
        if r.status_code == 200:
            _cache["data"] = r.json()
            _cache["ts"] = now
            return _cache["data"]
    except Exception as e:
        print(f"[Weather] {e}", type_="ERROR")
    return None

def weather_temp():
    data = _fetch()
    if not data: return ""
    temp = data.get("main", {}).get("temp")
    unit = "F" if getConfigData().get("weather_units") == "imperial" else "C"
    return f"{round(temp)} °{unit}" if temp is not None else ""

def weather_desc():
    data = _fetch()
    if not data: return ""
    return data.get("weather", [{}])[0].get("description", "")

def weather_full():
    data = _fetch()
    if not data: return ""
    name = data.get("name", "")
    temp = weather_temp()
    desc = weather_desc()
    return f"{name}: {temp}, {desc}" if name and temp else ""

addDRPCValue("weather_temp", weather_temp)
addDRPCValue("weather_desc", weather_desc)
addDRPCValue("weather_full", weather_full)
```

**Key techniques:**
- Module-level `_cache` dict as a simple TTL cache — avoids hammering external APIs
- `requests` (sync) is fine here because value functions aren't called from async context
- Returning `""` (empty string) gracefully hides the value when API key isn't set yet
### 16.5 addDRPCValue Rules

| Rule | Detail |
|---|---|
| Function must be sync | No `async def` — called synchronously by Nighty |
| Must return string | Returning `None` may crash the presence update |
| Called frequently | Cache expensive operations (API calls, file reads) |
| State via config | Use `getConfigData` to share state with listeners |
| No return = `""` | Return empty string to hide value gracefully |

---

## 17. Discord API Direct Calls (bot.http.request)

For operations not exposed by discord.py-self, you can call the Discord REST API directly.

### 17.1 Creating a Webhook Programmatically

```python
import requests

def create_webhook(channel_id: str, name: str = "Nighty Logger") -> tuple:
    """Returns (webhook_url, webhook_id, webhook_token) or (None, None, None) on failure."""
    try:
        url = f"https://discord.com/api/v9/channels/{channel_id}/webhooks"
        headers = {
            "Authorization": bot.http.token,   # Your user token — already available
            "Content-Type": "application/json"
        }
        r = requests.post(url, headers=headers, json={"name": name}, timeout=10)
        r.raise_for_status()
        data = r.json()
        webhook_url = f"https://discord.com/api/webhooks/{data['id']}/{data['token']}"
        return webhook_url, data['id'], data['token']
    except Exception as e:
        print(f"[Webhook] Create failed: {e}", type_="ERROR")
        return None, None, None

def delete_webhook(webhook_id: str, webhook_token: str):
    try:
        requests.delete(
            f"https://discord.com/api/v9/webhooks/{webhook_id}/{webhook_token}",
            timeout=10
        )
    except Exception as e:
        print(f"[Webhook] Delete failed: {e}", type_="ERROR")

def validate_webhook(webhook_url: str) -> bool:
    try:
        return requests.get(webhook_url, timeout=10).status_code == 200
    except:
        return False
```

**Important:** `bot.http.token` gives you direct access to the current user token. Handle it carefully — never log it.

### 17.2 Sending Full Embeds via Webhook (Fields + Color + Footer)

`forwardEmbedMethod` doesn't support fields, color, or footer. Webhooks do:

```python
import requests
import json
from datetime import datetime

def send_webhook_embed(webhook_url: str, embed: dict, username: str = None, avatar_url: str = None) -> bool:
    payload = {"embeds": [embed]}
    if username:   payload["username"]   = username
    if avatar_url: payload["avatar_url"] = avatar_url
    try:
        r = requests.post(
            webhook_url,
            headers={"Content-Type": "application/json"},
            data=json.dumps(payload),
            timeout=10
        )
        return r.status_code in (200, 204)
    except Exception as e:
        print(f"[Webhook] Send failed: {e}", type_="ERROR")
        return False

# Full embed with all fields
embed = {
    "title": "Message Logged",
    "description": "Content here",
    "color": 0x5865F2,                      # Integer, not hex string
    "author": {
        "name": "Username#1234",
        "icon_url": "https://cdn.discordapp.com/avatars/..."
    },
    "fields": [
        {"name": "Server",  "value": "My Server",       "inline": True},
        {"name": "Channel", "value": "#general",        "inline": True},
        {"name": "Before",  "value": "Old content",     "inline": False},
        {"name": "After",   "value": "New content",     "inline": False},
    ],
    "footer": {"text": "NightyScript Logger"},
    "thumbnail": {"url": "https://..."},
    "image":     {"url": "https://..."},
    "timestamp": datetime.utcnow().isoformat()  # ISO 8601
}

send_webhook_embed(webhook_url, embed, username="Server Name")
```

### 17.3 Reading Message History (channel.history)

```python
# Async generator — must use async for
async def count_messages_since(channel, after_message) -> int:
    count = 0
    async for msg in channel.history(after=after_message, limit=None):
        count += 1
    return count

# With a limit
async def get_recent_messages(channel, limit: int = 50) -> list:
    messages = []
    async for msg in channel.history(limit=limit):
        messages.append(msg)
    return messages

# Fetch a specific message by ID
async def get_message(channel, message_id: int):
    try:
        return await channel.fetch_message(message_id)
    except discord.NotFound:
        return None
    except discord.Forbidden:
        print("No access to that message", type_="ERROR")
        return None
```

### 17.4 Reading Channel from Another Guild

```python
# Get channel from any guild the user is in
channel = bot.get_channel(int(channel_id))    # from cache (fast)
# or
guild   = bot.get_guild(int(guild_id))
channel = guild.get_channel(int(channel_id))  # from guild cache

# Fetch categories
for category in guild.categories:
    print(category.name, [ch.name for ch in category.text_channels])
```

### 17.5 Using message.reply()

```python
# Replies as a threaded reply (quoted) — different from channel.send
await message.reply("This is a reply")
await message.reply("Silent reply", mention_author=False)  # No @mention ping
```

---

## 18. Guild & Member Operations

### 18.1 Leaving a Guild Synchronously (from sync context)

When you need to run async from a sync button handler:

```python
import asyncio

def leave_guild_sync(guild_id: int) -> tuple[bool, str]:
    """Call from sync UI button handlers."""
    try:
        guild = bot.get_guild(guild_id)
        if not guild:
            return False, "Guild not found"
        name = guild.name
        # Run coroutine from sync context using the bot's existing event loop
        future = asyncio.run_coroutine_threadsafe(guild.leave(), bot.loop)
        future.result(timeout=10)  # blocks until done or timeout
        return True, name
    except asyncio.TimeoutError:
        return False, "Timed out"
    except Exception as e:
        return False, str(e)

# Usage in a UI button onClick (sync):
def on_leave_click():
    success, result = leave_guild_sync(guild_id)
    if success:
        my_tab.toast(type="SUCCESS", title="Left", description=f"Left {result}")
    else:
        my_tab.toast(type="ERROR", title="Failed", description=result)
```

> **Critical pattern**: `asyncio.run_coroutine_threadsafe(coro, bot.loop).result(timeout)` is the correct way to run async code from sync UI handlers. Never use `asyncio.run()` — that creates a new event loop and will conflict with the existing one.

### 18.2 Guild Iteration Patterns

```python
# All guilds sorted alphabetically
guilds_sorted = sorted(bot.guilds, key=lambda g: g.name.lower())

# Get member count
total_members = sum(g.member_count or 0 for g in bot.guilds)

# Find guild by name
guild = next((g for g in bot.guilds if g.name.lower() == "my server"), None)

# Get all text channels across all guilds
all_channels = [ch for g in bot.guilds for ch in g.text_channels]

# Guild icon URL (safe)
icon_url = guild.icon.url if guild.icon else "https://cdn.discordapp.com/embed/avatars/0.png"

# Boost tier
is_boosted = (guild.premium_tier or 0) > 0
```

### 18.3 Discord Timestamp Format

For Discord's native timestamp rendering in messages:

```python
import calendar
from datetime import datetime

def discord_ts(dt: datetime) -> str:
    """Returns Discord timestamp that renders natively in the client."""
    unix = calendar.timegm(dt.timetuple())
    return f"<t:{unix}:f> · <t:{unix}:R>"
    # :f = full date/time, :R = relative ("3 hours ago")
    # Other formats: :d (short date), :D (long date), :t (short time), :T (long time), :F (full with weekday)

# Example:
ts = discord_ts(message.created_at)
# → "<t:1720000000:f> · <t:1720000000:R>"
# Renders as: "June 27, 2025 at 3:00 PM · 2 hours ago"
```

---

## 19. Advanced UI Patterns

### 19.1 Dynamic Button Handlers (Closure Problem)

When creating buttons in a loop, a common bug is all buttons sharing the same variable:

```python
# ❌ WRONG — all buttons will use the last value of guild_id
for guild in bot.guilds:
    card.create_ui_element(UI.Button, label="Leave",
                           onClick=lambda: leave(guild.id))  # BUG: captures by reference

# ✅ CORRECT — factory function creates a closure with fixed value
def make_handler(gid):
    def handler():
        leave(gid)
    handler.__name__ = f"leave_{gid}"  # Optional: helps debugging
    return handler

for guild in bot.guilds:
    card.create_ui_element(UI.Button, label="Leave",
                           onClick=make_handler(guild.id))

# ✅ ALSO CORRECT — default argument trick
for guild in bot.guilds:
    card.create_ui_element(UI.Button, label="Leave",
                           onClick=lambda gid=guild.id: leave(gid))
```

### 19.2 Re-rendering a Tab (Full Refresh)

The approved pattern for rebuilding a tab's contents from scratch (confirmed from guild_manager.py):

```python
gm_tab = None
main_container = None

def load_ui():
    global gm_tab, main_container
    # Recreate the Tab from scratch
    gm_tab = Tab(name="My Tab", icon="chart", title="My Tab")
    main_container = gm_tab.create_container(type="rows", gap=4)
    
    # Rebuild all content
    for item in get_data():
        card = main_container.create_card()
        card.create_ui_element(UI.Text, content=item["name"])
    
    gm_tab.render()

# After a leave/delete/update:
def on_action():
    perform_action()
    load_ui()  # Rebuild entire tab
```

### 19.3 Dynamic Select: Updating Channel List Based on Server Select

Full pattern from ChannelLogger and AutoReply — chain two Selects:

```python
# Build server list
server_items = [{"id": "none", "title": "Select server"}]
for guild in bot.guilds:
    server_items.append({
        "id": str(guild.id),
        "title": guild.name,
        "iconUrl": guild.icon.url if guild.icon else "https://cdn.discordapp.com/embed/avatars/0.png"
    })

server_select = card.create_ui_element(
    UI.Select, label="Server", items=server_items,
    disabled_items=["none"], mode="single", full_width=True
)
channel_select = card.create_ui_element(
    UI.Select, label="Channel",
    items=[{"id": "none", "title": "Select server first"}],
    disabled_items=["none"], mode="single", full_width=True
)

def on_server_change(selected_ids):
    if not selected_ids or selected_ids[0] == "none":
        channel_select.items = [{"id": "none", "title": "Select server first"}]
        channel_select.disabled_items = ["none"]
        return
    guild = bot.get_guild(int(selected_ids[0]))
    if not guild:
        return
    items = [{"id": "none", "title": "Select channel"}]
    items += [{"id": str(ch.id), "title": f"#{ch.name}"} for ch in guild.text_channels]
    channel_select.items = items
    channel_select.disabled_items = ["none"]

server_select.onChange = on_server_change
```

### 19.4 Incremental Text Display (hiding/showing elements)

Instead of rebuilding a tab, update just the text elements:

```python
display_elements = []  # track created elements

def refresh_display(items: list):
    # Hide old elements
    for el in display_elements:
        el.visible = False
    display_elements.clear()

    if items:
        for item in items[:10]:  # show max 10
            el = card.create_ui_element(UI.Text, content=f"• {item}", size="sm")
            display_elements.append(el)
        if len(items) > 10:
            el = card.create_ui_element(UI.Text, content=f"+ {len(items)-10} more...", size="sm", color="#6b7280")
            display_elements.append(el)
    else:
        el = card.create_ui_element(UI.Text, content="No items", size="sm", color="#6b7280")
        display_elements.append(el)
```

### 19.5 Loading State Pattern

```python
btn = card.create_ui_element(UI.Button, label="Fetch Data", variant="cta")
status = card.create_ui_element(UI.Text, content="", size="sm")

async def on_click():
    btn.loading = True
    btn.disabled = True
    status.content = "Fetching..."
    status.color = "#f59e0b"
    try:
        result = await fetch_data()
        status.content = f"Done: {result}"
        status.color = "#4ade80"
        my_tab.toast(type="SUCCESS", title="Complete", description="Data loaded.")
    except Exception as e:
        status.content = f"Error: {e}"
        status.color = "#f87171"
        my_tab.toast(type="ERROR", title="Failed", description=str(e))
    finally:
        btn.loading = False
        btn.disabled = False

btn.onClick = on_click
```

### 19.6 bot.loop.create_task (background tasks from sync init)

```python
# Call from inside the script body (not inside async)
bot.loop.create_task(my_background_coro())

# Example: validate webhooks on startup without blocking
async def validate_on_start():
    await asyncio.sleep(3)  # wait for bot to settle
    await validate_all_webhooks()

bot.loop.create_task(validate_on_start())
```

---

## 20. Message Utilities

### 20.1 Fuzzy Match Helper

```python
def fuzzy_match(message_content: str, trigger: str) -> bool:
    """True if trigger phrase is contained anywhere in message."""
    return trigger.lower() in message_content.lower()

def exact_match(message_content: str, trigger: str) -> bool:
    """True only if full message equals trigger."""
    return message_content.strip().lower() == trigger.lower()
```

### 20.2 URL Extraction

```python
import re

def extract_urls(text: str) -> list[str]:
    if not text:
        return []
    seen, out = set(), []
    for m in re.findall(r'(https?://[^\s<>"{}|\\^`\[\]]+)', text):
        m = m.rstrip('.,;:!)?]\'"')
        if m not in seen:
            seen.add(m)
            out.append(m)
    return out

def is_image_url(url: str) -> bool:
    return bool(re.search(r'\.(?:jpg|jpeg|png|gif|webp|bmp)(\?|$)', url, re.IGNORECASE))

def is_valid_url(url: str) -> bool:
    return bool(re.match(r'^(https?)://[^\s/$.?#].[^\s]*$', url, re.IGNORECASE))
```

### 20.3 Timestamp Formatting

```python
from datetime import datetime, timezone

def format_ts(dt: datetime) -> str:
    """Human-readable UTC timestamp."""
    return dt.strftime("%Y-%m-%d %H:%M:%S UTC")

def time_ago(dt: datetime) -> str:
    """Relative time string."""
    if dt.tzinfo is None:
        dt = dt.replace(tzinfo=timezone.utc)
    diff = datetime.now(timezone.utc) - dt
    d, s = diff.days, diff.seconds
    h, rem = divmod(s, 3600)
    m, _   = divmod(rem, 60)
    if d:    return f"{d}d {h}h ago"
    if h:    return f"{h}h {m}m ago"
    return   f"{m}m ago"
```

### 20.4 Auto-Reply with Blacklist + Fuzzy Match (full pattern)

From `AutoReply.py` — the cleanest production implementation:

```python
@bot.listen("on_message")
async def auto_reply_handler(message):
    config = load_config()
    if not config["enabled"]:
        return
    if not config.get("reply_to_self", True) and message.author == bot.user:
        return

    for trigger in config.get("triggers", []):
        # Channel filter
        if str(message.channel.id) != trigger["channel_id"]:
            continue

        # Blacklist check (comma-separated phrases)
        blacklist = trigger.get("blacklist", "")
        if blacklist:
            phrases = [p.strip().lower() for p in blacklist.split(",") if p.strip()]
            if any(p in message.content.lower() for p in phrases):
                continue

        # Match
        if trigger.get("fuzzy_match", False):
            matched = trigger["trigger_message"].lower() in message.content.lower()
        else:
            matched = message.content.strip().lower() == trigger["trigger_message"].lower()

        if matched:
            delay = trigger.get("delay", config.get("default_delay", 0))
            if delay > 0:
                await asyncio.sleep(delay)
            try:
                await message.reply(trigger["reply_message"], mention_author=False)
            except Exception as e:
                print(f"[AutoReply] {e}", type_="ERROR")
            break  # Only trigger first matching rule
```

---

## 21. Config & File Path Patterns

### 21.1 Reading Nighty's Own Config (low-level)

From `ChannelLogger.py` — reading the raw Nighty config directly:

```python
import os
import json
from pathlib import Path

def load_nighty_config() -> dict:
    """Read Nighty's own nighty.config file directly."""
    try:
        cfg_path = Path(os.getenv("APPDATA")) / "Nighty Selfbot" / "nighty.config"
        with open(cfg_path, "r", encoding="utf-8") as f:
            return json.load(f)
    except Exception as e:
        print(f"[Config] {e}", type_="ERROR")
        return {}

# Get active theme name
nighty_cfg  = load_nighty_config()
theme_name  = nighty_cfg.get("theme")

# Build paths to theme files
theme_path  = Path(os.getenv("APPDATA")) / "Nighty Selfbot" / "data" / "themes" / f"{theme_name}.json"
```

> This is equivalent to `getDataPath() + "/themes/" + theme_name + ".json"` but uses the lower-level APPDATA path directly. Prefer `getDataPath()` when available.

### 21.2 Theme-Aware Color in Webhook Embeds

Pattern for making your script respect the user's active Nighty theme:

```python
import time, json

_theme_cache = {"data": {}, "ts": 0.0}
THEME_TTL = 60.0  # re-read theme at most once per minute

def get_theme_color() -> int:
    """Returns embed color integer matching user's active theme."""
    now = time.monotonic()
    if now - _theme_cache["ts"] < THEME_TTL and _theme_cache["data"]:
        data = _theme_cache["data"]
    else:
        try:
            data = getThemeData(theme_name=getConfigData().get("theme", ""))
            _theme_cache["data"] = data
            _theme_cache["ts"] = now
        except:
            data = {}
    color_hex = data.get("color", "#5865F2").lstrip("#")
    try:
        return int(color_hex, 16)
    except:
        return 0x5865F2
```

### 21.3 Path Cheatsheet

| What | How |
|---|---|
| Script JSON storage | `Path(getScriptsPath()) / "json" / "myfile.json"` |
| Nighty data dir | `getDataPath()` or `Path(os.getenv("APPDATA")) / "Nighty Selfbot" / "data"` |
| Theme files | `Path(getDataPath()) / "themes" / f"{name}.json"` |
| Nighty main config | `Path(os.getenv("APPDATA")) / "Nighty Selfbot" / "nighty.config"` |
| Script directory itself | `getScriptsPath()` |

---

## 22. Real Script Patterns — Quick Reference

Extracted from all production scripts analyzed.

### 22.1 Send embed & restore private mode (the right way)

```python
async def send_embed_safely(channel_id, title, content):
    prev = getConfigData().get("private")
    updateConfigData("private", False)
    try:
        await forwardEmbedMethod(channel_id=channel_id, title=title, content=content)
    finally:
        updateConfigData("private", prev)
```

### 22.2 Config migration (add new keys to existing files)

```python
def load_config():
    default = {"enabled": True, "log_deleted": True, "log_edited": True}
    try:
        with open(CONFIG_FILE) as f:
            cfg = json.load(f)
    except:
        return default.copy()
    # Merge: add any missing keys with defaults
    for key, val in default.items():
        if key not in cfg:
            cfg[key] = val
    return cfg
```

### 22.3 Private mode toggle in forwardEmbedMethod (message_counter.py pattern)

```python
async def send_embed_safely(channel_id, content, title):
    current_private = getConfigData().get("private")
    updateConfigData("private", False)
    try:
        await forwardEmbedMethod(channel_id=channel_id, content=content, title=title)
    finally:
        updateConfigData("private", current_private)
```

### 22.4 Script-level global state

```python
# Mutable state accessible from all nested functions
# Use a dict (not simple variables) so inner functions can mutate it
state = {"is_loading": False, "data": {}}

def do_something():
    if state["is_loading"]:
        return
    state["is_loading"] = True
    try:
        ...
    finally:
        state["is_loading"] = False
```

### 22.5 Webhook self-healing (auto-recreate on failure)

```python
async def send_with_healing(webhook_url, dest_channel_id, embed, config):
    """Tries to send; recreates webhook if it fails."""
    success = await run_in_thread(send_webhook_embed, webhook_url, embed)
    if not success:
        new_url, new_id, new_token = await run_in_thread(
            create_webhook, dest_channel_id, "My Logger"
        )
        if new_url:
            # Update all sources pointing to old webhook
            for source in config.get("sources", []):
                if source.get("webhook_url") == webhook_url:
                    source["webhook_url"] = new_url
                    source["webhook_id"]  = new_id
                    source["webhook_token"] = new_token
            save_config(config)
            await run_in_thread(send_webhook_embed, new_url, embed)
```

---

## 23. Additional Events

Events confirmed from real scripts:

| Event | Parameters | Notes |
|---|---|---|
| `on_bulk_message_delete` | `messages` | List of (partially cached) Message objects |
| `on_voice_state_update` | `member, before, after` | `before`/`after` are `VoiceState` objects |

### on_bulk_message_delete

```python
@bot.listen("on_bulk_message_delete")
async def bulk_handler(messages):
    if not messages:
        return
    first = messages[0]
    cached = [m for m in messages if m.content or m.author]
    # messages not in cache will have empty content/author
    print(f"Bulk delete: {len(messages)} total, {len(cached)} cached", type_="INFO")
```

### on_voice_state_update patterns

```python
@bot.listen("on_voice_state_update")
async def vc_handler(member, before, after):
    # Filter to self only
    if member.id != bot.user.id:
        return
    
    joined  = after.channel  is not None and before.channel is None
    left    = after.channel  is None     and before.channel is not None
    moved   = after.channel  is not None and before.channel is not None and after.channel != before.channel
    muted   = after.self_mute and not before.self_mute
    deafened = after.self_deaf and not before.self_deaf

    if joined:
        print(f"Joined VC: {after.channel.name}", type_="INFO")
    elif left:
        print("Left VC", type_="INFO")
    elif moved:
        print(f"Moved: {before.channel.name} → {after.channel.name}", type_="INFO")
```

---

## 23.1 Undocumented Globals — Extended (From Real Scripts)

This section documents globals confirmed from community and private scripts that don't appear in any official docs.

### 23.1.1 addDRPCValue — Dynamic Rich Presence Values

Registers a callable as a named dynamic value usable in Discord Rich Presence (DRPC) status strings.

```python
# Signature
addDRPCValue(name: str, func: Callable[[], str]) -> None

# The function is called periodically by Nighty to refresh the value.
# It MUST return a string (or empty string on error). Never raise inside it.

def my_value() -> str:
    return "some dynamic text"

addDRPCValue("my_key", my_value)
# → user can now use {my_key} in their rich presence template
```

**Rules:**
- The function is called synchronously — keep it fast (no network, no blocking I/O)
- For async data, update a variable in a background task and return that variable in the getter
- Returning `""` or `None` hides the field in the presence
**Real-world pattern (background update + sync getter):**

```python
# In your script function:
bot.my_cached_value = "loading..."

async def updater_loop():
    while True:
        try:
            # fetch data asynchronously
            bot.my_cached_value = await fetch_something()
        except:
            bot.my_cached_value = "error"
        await asyncio.sleep(30)

# Start background loop (guard against duplicate tasks on reload)
if not hasattr(bot, "_my_task") or bot._my_task.done():
    bot._my_task = bot.loop.create_task(updater_loop())

def my_getter():
    return str(bot.my_cached_value)

addDRPCValue("my_key", my_getter)
```

**Confirmed DRPC value examples from community scripts:**

| Script | Value key | What it returns |
|---|---|---|
| VCDynamicValues | `vc_status` | `"Connected to voice:"` / `"Muted:"` / `"Disconnected:"` |
| VCDynamicValues | `vc_info` | Channel name + server name |
| VCDynamicValues | `vc_start` | Unix timestamp ms (for elapsed timer) |
| WeatherDynamicValues | `weather_temp` | `"72 F"` |
| WeatherDynamicValues | `weather_full` | `"Dallas: 72 F, clear sky"` |
| DateTimeVolumeDynamic | `date_now` | `"saturday, june 28th, 2025"` |
| DateTimeVolumeDynamic | `time_now` | `"03:45 pm"` |
| DateTimeVolumeDynamic | `sound_now` | `"75%"` (Windows system volume) |

### 23.1.2 getAppData() — App Info

Returns the current Nighty app's name, avatar URL, and banner URL.

```python
app_name, app_avatar_url, app_banner_url = getAppData()
# → ("My App Name", "https://cdn.discordapp.com/...", "https://...")
```

### 23.1.3 editNightyApp() — Edit App Appearance

```python
result = await editNightyApp(
    name="New App Name",
    avatar_url="https://example.com/avatar.png",
    banner_url="https://example.com/banner.png"
)
# result = {"status": "success"} or {"status": "error", "message": "..."}

if result["status"] == "success":
    print("App updated", type_="SUCCESS")
else:
    print(f"Failed: {result['message']}", type_="ERROR")
```

### 23.1.4 reloadScripts() — Reload All Scripts

```python
# Reloads all scripts (equivalent to Nighty's "Reload Scripts" button)
reloadScripts()
```

> Note: After calling this, your current script instance is destroyed. Any code after the call won't run.

### 23.1.5 bot.loop.create_task() vs asyncio.ensure_future()

Both schedule a coroutine to run concurrently. In NightyScript, prefer `bot.loop.create_task()`:

```python
# Preferred — explicitly uses the bot's event loop
bot.loop.create_task(my_coroutine())

# Also works
asyncio.ensure_future(my_coroutine())

# For fire-and-forget from sync context (e.g. on startup)
asyncio.run_coroutine_threadsafe(startup_routine(), bot.loop)
```

### 23.1.6 asyncio.create_task() Inside Async Handlers

Inside `async def` functions, `asyncio.create_task()` works and is the idiomatic way to fire background work without awaiting:

```python
async def on_click():
    # Don't block the UI — fire and forget
    asyncio.create_task(do_long_operation())

async def do_long_operation():
    await asyncio.sleep(10)
    print("Done", type_="SUCCESS")
```

### 23.1.7 bot.http.token — Raw Auth Token

Available for making raw Discord API requests that the library doesn't expose:

```python
headers = {
    "Authorization": bot.http.token,
    "Content-Type": "application/json"
}

# Example: create a webhook manually
import requests
r = requests.post(
    f"https://discord.com/api/v9/channels/{channel_id}/webhooks",
    headers=headers,
    json={"name": "My Webhook"},
    timeout=10
)
webhook = r.json()
webhook_url = f"https://discord.com/api/webhooks/{webhook['id']}/{webhook['token']}"
```

> **Security**: Never log or send `bot.http.token`. It's your full account token.

### 23.1.8 bot.remove_command() / bot.remove_listener()

For safe script reloading, remove existing registrations before re-registering:

```python
# Remove a command (prevents "Command already exists" on reload)
if bot.get_command("mycommand"):
    bot.remove_command("mycommand")

@bot.command(name="mycommand")
async def mycommand(ctx):
    pass

# Remove a named listener
if hasattr(bot, "my_listener_func"):
    bot.remove_listener(bot.my_listener_func, "on_voice_state_update")

async def my_listener(member, before, after):
    pass

bot.my_listener_func = my_listener
bot.add_listener(my_listener, "on_voice_state_update")
```

### 23.1.9 channel.history() — Message Iteration

```python
# Iterate channel history (newest first by default)
async for msg in channel.history(limit=50):
    print(msg.content)

# Oldest first
async for msg in channel.history(limit=100, oldest_first=True):
    pass

# Unlimited (careful — may take minutes on large channels)
async for msg in channel.history(limit=None):
    pass

# After/before a specific message
async for msg in channel.history(limit=50, before=some_message):
    pass
```

### 23.1.10 bot.wait_for() — Event Waiting

Wait for a specific event with a condition:

```python
def check(reaction, user):
    return user.id == ctx.author.id and str(reaction.emoji) in ["✅", "❌"]

try:
    reaction, user = await bot.wait_for("reaction_remove", timeout=60.0, check=check)
    if str(reaction.emoji) == "✅":
        # confirmed
        pass
except asyncio.TimeoutError:
    await ctx.send("Timed out.")
```

---

## 23.2 Advanced Patterns (From Real Scripts)

### 23.2.1 Private Mode Guard

Multiple scripts use this pattern to temporarily disable private mode to ensure `forwardEmbedMethod` delivers the embed, then restore the original setting:

```python
async def send_embed_safe(channel_id, title, content):
    cfg = getConfigData()
    prev_private = cfg.get("private")
    prev_timer   = cfg.get("deletetimer")

    updateConfigData("private", False)
    updateConfigData("deletetimer", 0)

    try:
        msg = await forwardEmbedMethod(
            channel_id=channel_id,
            title=title,
            content=content
        )
        return msg
    except Exception as e:
        print(f"Embed failed: {e}", type_="ERROR")
        return None
    finally:
        updateConfigData("private", prev_private)
        if prev_timer is not None:
            updateConfigData("deletetimer", prev_timer)
```

### 23.2.2 Auto-Delete a Message After N Seconds

```python
async def send_and_autodelete(ctx, text, delay=5):
    msg = await ctx.send(text, silent=True)
    async def _delete():
        await asyncio.sleep(delay)
        try:
            await msg.delete()
        except:
            pass
    asyncio.create_task(_delete())
    return msg
```

### 23.2.3 Bot-State Variables via bot Attributes

Scripts that survive reloads store state on the `bot` object so it persists across reloads:

```python
# Guard: only initialize if not already set
if not hasattr(bot, "my_state"):
    bot.my_state = {"counter": 0, "enabled": True}

# Guard background task from duplicating on reload
if not hasattr(bot, "_my_bg_task") or bot._my_bg_task.done():
    bot._my_bg_task = bot.loop.create_task(background_loop())
```

### 23.2.4 subprocess — Running System Commands

Confirmed working from real remote-control and dynamic-values scripts:

```python
import subprocess
import asyncio

# Run a command hidden (no console window) — Windows only flag
CREATE_NO_WINDOW = 0x08000000

async def run_powershell(script: str) -> str:
    loop = asyncio.get_event_loop()
    output = await loop.run_in_executor(None, lambda: subprocess.check_output(
        ["powershell", "-NoProfile", "-ExecutionPolicy", "Bypass", "-NonInteractive", "-Command", script],
        creationflags=CREATE_NO_WINDOW,
        stderr=subprocess.STDOUT
    ))
    return output.decode("utf-8", errors="ignore").strip()

# Usage
volume = await run_powershell("(Get-AudioDevice -Playback).Volume")

# Open an app (Windows)
os.system("start spotify:")
os.system("start anydesk:")

# Shutdown/restart PC
os.system("shutdown /s /t 5")   # shutdown in 5s
os.system("shutdown /r /t 5")   # restart in 5s
```

### 23.2.5 Reaction-Based Confirmation Flow

```python
@bot.command(name="confirm")
async def confirm_cmd(ctx, *, args: str = ""):
    await ctx.message.delete()

    prompt = await ctx.send(
        "Are you sure? React to confirm.\n✅ = Yes  ❌ = No",
        silent=True
    )
    await prompt.add_reaction("✅")
    await prompt.add_reaction("❌")

    def check(reaction, user):
        return (
            user.id == bot.user.id
            and str(reaction.emoji) in ["✅", "❌"]
            and reaction.message.id == prompt.id
        )

    try:
        reaction, _ = await bot.wait_for("reaction_remove", timeout=30.0, check=check)
        await prompt.delete()
        if str(reaction.emoji) == "✅":
            await ctx.send("Confirmed!", silent=True)
        else:
            await ctx.send("Cancelled.", silent=True)
    except asyncio.TimeoutError:
        await prompt.edit(content="Timed out — action cancelled.")
```

### 23.2.6 Role & Member Operations

```python
# Get a guild
guild = bot.get_guild(guild_id)

# Get a member (only works if member is cached)
member = guild.get_member(user_id)

# Fetch a member (API call — always works, slower)
member = await guild.fetch_member(user_id)

# Add/remove roles
await member.add_roles(role, reason="via script")
await member.remove_roles(role)

# Get role object
role = guild.get_role(role_id)

# Iterate roles (highest to lowest by default)
for role in reversed(guild.roles):
    if not role.is_default():
        print(role.name)

# Edit role positions
await guild.edit_role_positions({role_obj: new_position})

# Create role
new_role = await guild.create_role(
    name="My Role",
    permissions=discord.Permissions(send_messages=True),
    colour=discord.Colour(0x5865F2),
    hoist=True,
    mentionable=False
)

# Anti-rate-limit sleep between role mass operations
await asyncio.sleep(0.6)
```

### 23.2.7 Guild / Server Operations

```python
# Create text channel
ch = await guild.create_text_channel(
    name="my-channel",
    topic="Channel topic",
    slowmode_delay=0,
    nsfw=False,
    overwrites={},      # permission overwrite dict
    category=cat_obj,
    position=0
)

# Create voice channel
vc = await guild.create_voice_channel(
    name="Voice",
    bitrate=min(96000, guild.bitrate_limit),
    user_limit=0,       # 0 = unlimited
    category=cat_obj
)

# Create category
cat = await guild.create_category(
    name="CATEGORY",
    overwrites={},
    position=0
)

# Edit guild
await guild.edit(name="New Name", icon=icon_bytes)

# Fetch icon bytes (for cloning)
icon_bytes = await guild.icon.read()

# Permission overwrites — map source to target roles
overwrites = {}
for principal, overwrite in src_channel.overwrites.items():
    if isinstance(principal, discord.Role):
        if principal.is_default():
            overwrites[target_guild.default_role] = overwrite
        else:
            mapped_role = role_map.get(principal.id)
            if mapped_role:
                overwrites[mapped_role] = overwrite
```

### 23.2.8 File Export Pattern

Save a file to the scripts directory and optionally send it:

```python
import os
import zipfile
from pathlib import Path
from datetime import datetime

def get_export_dir(subdir="exports") -> str:
    path = Path(getScriptsPath()) / subdir
    path.mkdir(parents=True, exist_ok=True)
    return str(path)

# Save HTML export
timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
filepath = os.path.join(get_export_dir("html"), f"export_{timestamp}.html")
with open(filepath, "w", encoding="utf-8") as f:
    f.write(html_content)

# Auto-compress if large
file_size_mb = os.path.getsize(filepath) / (1024 * 1024)
if file_size_mb > 10:
    zip_path = filepath.replace(".html", ".zip")
    with zipfile.ZipFile(zip_path, "w", zipfile.ZIP_DEFLATED, compresslevel=9) as zf:
        zf.write(filepath, os.path.basename(filepath))
    os.remove(filepath)
    filepath = zip_path

# Send the file
await ctx.send(
    f"Export: {file_size_mb:.2f} MB",
    file=discord.File(filepath, filename=os.path.basename(filepath))
)
```

### 23.2.9 Progress Bar Utility

```python
def progress_bar(current: int, total: int, width=10) -> str:
    pct = int((current / total) * 100) if total > 0 else 0
    filled = int((pct / 100) * width)
    bar = "█" * filled + "░" * (width - filled)
    return f"[{bar}] {pct}%"

# Usage with live message edit
status = await ctx.send("Starting...")
for i in range(100):
    await asyncio.sleep(0.1)
    if i % 10 == 0:
        await status.edit(content=f"Processing... {progress_bar(i, 100)}")
await status.edit(content="Done ✅")
```

### 23.2.10 Smart Message Delete (Try Own Messages if No Perms)

```python
async def smart_clear(channel, limit: int):
    to_delete = []
    my_messages = []

    async for msg in channel.history(limit=limit):
        to_delete.append(msg)
        if msg.author.id == bot.user.id:
            my_messages.append(msg)

    deleted = 0
    has_perms = True

    for msg in to_delete:
        try:
            await msg.delete()
            deleted += 1
            await asyncio.sleep(1.0)  # safe rate limit pause
        except:
            has_perms = False
            break

    # Fallback: only delete own messages if no mod perms
    if not has_perms:
        deleted = 0
        for msg in my_messages:
            try:
                await msg.delete()
                deleted += 1
                await asyncio.sleep(1.0)
            except:
                pass

    return deleted
```

### 23.2.11 DM Channel Access

```python
# Get/create DM channel with a user
user = await bot.fetch_user(user_id)
dm_channel = await user.create_dm()

# Send to DM
await dm_channel.send("Hello!")

# Read DM history
async for msg in dm_channel.history(limit=50):
    print(msg.content)
```

### 23.2.12 Import discord Inside Script (Exception to the Rule)

The `ui_server_cloner.py` script does `import discord` **inside** the script function body. This appears to work in some versions of Nighty because the import resolves to the already-loaded `discord.py-self` module without creating a conflict. However, it's not officially supported and may break. Prefer using the globally injected `discord.*` objects.

```python
# This MAY work (seen in server_cloner.py)
def my_script():
    import discord  # ← inside function, not at module level
    ...

# This will CRASH (at module level)
import discord     # ← DO NOT DO THIS
```

---

## 23.3 Additional Imports Confirmed Working

From analysis of the submitted scripts:

```python
import os            # ✅ file system, env vars, os.system()
import sys           # ✅ sys.executable, sys.argv (for restarts)
import re            # ✅ regex
import json          # ✅ JSON read/write
import asyncio       # ✅ core async
import requests      # ✅ sync HTTP (use run_in_executor)
import aiohttp       # ✅ async HTTP
import time          # ✅ time.time(), time.sleep() (sync only — use asyncio.sleep in async)
import shutil        # ✅ file copy, zip (shutil.make_archive, shutil.copytree)
import subprocess    # ✅ system process execution
import calendar      # ✅ calendar.timegm()
import uuid          # ✅ uuid.uuid4()
import gc            # ✅ garbage collection
import gzip          # ✅ gzip compression
import pickle        # ✅ binary serialization
import zipfile       # ✅ zip archives
import urllib.request  # ✅ low-level HTTP (no extra deps)
import concurrent.futures  # ✅ ThreadPoolExecutor for parallel blocking I/O
from html import escape    # ✅ HTML escaping
from datetime import datetime, timedelta, timezone  # ✅
from pathlib import Path   # ✅
from base64 import b64encode  # ✅ (used in HalfToken script)
```

### Confirmed NOT working / risky

```python
import discord       # ❌ at module level — crashes
import nighty        # ❌ — crashes
import numpy         # ❌ not in env
import matplotlib    # ❌ not in env
import pydub         # ❌ not in env
from comtypes import ...  # ⚠️ Windows-only, requires comtypes installed separately
from pycaw import ...     # ⚠️ Windows-only, used for audio control
```

---

## 23.4 UI Scripting — Additional Patterns

### 23.4.1 Stateful UI with dict (Pattern from server_cloner)

For complex UIs that need to track multiple states without global variables, use a `state` dict:

```python
def my_script():
    state = {
        "running": False,
        "log_lines": [],
        "source_id": None,
    }

    tab = Tab(name="My Tool", icon="convert", title="My Tool")
    # ... build UI ...

    async def on_start():
        if state["running"]:
            return
        state["running"] = True
        start_btn.loading  = True
        start_btn.disabled = True
        stop_btn.visible   = True

        try:
            await do_work(state)
        finally:
            state["running"]    = False
            start_btn.loading   = False
            start_btn.disabled  = False
            stop_btn.visible    = False

    def on_stop():
        state["running"] = False
        status_text.content = "Stopping..."

    start_btn.onClick = on_start
    stop_btn.onClick  = on_stop

    tab.render()
```

### 23.4.2 Live Log Output in UI

Stream log lines into a `UI.Text` element:

```python
MAX_LOG = 80

def log(message: str, level: str = "info"):
    prefix = {"info": "[INFO]", "ok": "[ OK ]", "warn": "[WARN]", "error": "[ERR ]"}.get(level, "[INFO]")
    state["log_lines"].append(f"{prefix}  {message}")
    if len(state["log_lines"]) > MAX_LOG:
        state["log_lines"] = state["log_lines"][-MAX_LOG:]
    log_output.content = "\n".join(state["log_lines"])
```

### 23.4.3 Disabling/Enabling Elements During Operations

```python
async def on_submit():
    # Lock UI during operation
    submit_btn.loading  = True
    submit_btn.disabled = True
    input_field.disabled = True
    
    try:
        await do_work()
        tab.toast(type="SUCCESS", title="Done", description="Operation completed.")
    except Exception as e:
        tab.toast(type="ERROR", title="Error", description=str(e))
    finally:
        # Always restore, even on error
        submit_btn.loading   = False
        submit_btn.disabled  = False
        input_field.disabled = False
```

### 23.4.4 Input Validation Pattern

```python
def validate_server_id(value: str, input_el) -> bool:
    if not value or not value.strip().isdigit():
        input_el.invalid = True
        input_el.error_message = "Enter a valid numeric server ID."
        return False
    input_el.invalid = False
    input_el.error_message = None
    return True

# Wire to onInput for real-time feedback
def on_id_input(value):
    state["server_id"] = value.strip() if value else None
    server_input.invalid = False  # reset on any change

server_input.onInput = on_id_input

# Validate on submit
async def on_submit():
    if not validate_server_id(server_input.value, server_input):
        tab.toast(type="ERROR", title="Validation Error", description="Fix the highlighted fields.")
        return
    # ... proceed
```

### 23.4.5 Dynamic Select Population

Populate a `UI.Select` from bot data at runtime:

```python
# Load servers into a select
server_items = [{"id": "placeholder", "title": "Select a server"}]
for guild in bot.guilds:
    server_items.append({
        "id": str(guild.id),
        "title": guild.name,
        "iconUrl": guild.icon.url if guild.icon else "https://cdn.discordapp.com/embed/avatars/0.png"
    })

server_select = card.create_ui_element(
    UI.Select,
    label="Server",
    items=server_items,
    disabled_items=["placeholder"],
    mode="single",
    full_width=True
)

# Then update a dependent select when server changes
async def on_server_select(selected):
    if not selected or selected[0] == "placeholder":
        return
    guild = bot.get_guild(int(selected[0]))
    if not guild:
        return
    channel_select.items = [
        {"id": str(ch.id), "title": f"#{ch.name}"}
        for ch in guild.text_channels
    ]

server_select.onChange = on_server_select
```

### 23.4.6 UI.Text as Live Log — Color Coding

```python
def set_status(message: str, status: str = "info"):
    color_map = {
        "info":    "#6b7280",
        "success": "#4ade80",
        "warning": "#f59e0b",
        "error":   "#f87171"
    }
    status_text.content = message
    status_text.color   = color_map.get(status, "#6b7280")

# Usage
set_status("Loading...", "info")
set_status("Done!", "success")
set_status("No results found", "warning")
set_status("API error", "error")
```

---

## 24. Slash Command Framework — nighty-slash-api

A community library (separate from `exec_slash`, section 9) that lets a NightyScript register **real, first-party Discord slash commands** — options, subcommand groups, context menus, autocomplete, ephemeral replies — through Nighty's own bot application, without a bot token, without a second gateway connection, and without touching the Discord Developer Portal.

> **Disclaimer**: unofficial community tooling. Not affiliated with or endorsed by Nighty or Discord, and does not include or redistribute Nighty — bring your own licensed copy. Nighty is a Discord selfbot; automating a user account can violate Discord's Terms of Service. Use at your own risk, on your own account.

### 24.1 exec_slash vs nighty-slash-api

These solve opposite problems — don't conflate them:

| | `exec_slash` (Section 9) | `nighty-slash-api` (this section) |
|---|---|---|
| Direction | **Invokes** an existing slash command belonging to another bot | **Registers** a new slash command belonging to Nighty's own bot app |
| Needs a bot app ID | Yes — target bot's application ID | No — uses Nighty's built-in `bot_app` |
| Visible to channel | Execution is visible as if you typed it | Normal — appears as Nighty's own `/command` |
| Use case | "Trigger `/play` on the music bot" | "Give my script its own `/roll` command" |

### 24.2 How It Works

```
your script                        Nighty                       Discord
───────────                        ──────                       ───────
@slash.command()   ──register──▶   bot_app.tree   ──sync──▶   /command registered
                                        ▲                            │
async def handler(ctx)  ◀───────────────┴──────── interaction ───────┘
```

Nighty runs a bot application alongside your user account — it's where `/help` and `/embed` come from — and exposes its application-command tree to scripts. `nighty-slash-api` wraps that tree in a decorator-based API.

Two separate operations, kept deliberately apart:

- **Registering** attaches your handler to the tree. Happens on every script load. Costs nothing — no network call.
- **Syncing** tells Discord the command exists. Only needed when a *definition* changes: a name, a description, or an option. Editing what a command *does* (the handler body) needs no sync at all.

Hand-rolled slash-command code that conflates the two ends up syncing on every restart. This library fingerprints definitions so it doesn't.

### 24.3 Requirements

- **Nighty 2.3+** — specifically a build that exposes `bot_app` and `app_commands` to scripts.
- Nothing else. The framework itself is standard library only: `asyncio`, `functools`, `hashlib`, `inspect`, `json`, `traceback`, `pathlib`.

On an older build, the script logs one line saying so and the rest of the script keeps working — nothing else breaks.

### 24.4 Features

- **Options from your function signature** — annotate a parameter, it becomes a typed Discord option. Give it a default, it becomes optional.
- **Fingerprinted autosync** — hashes every definition (names, descriptions, option name/type/required) and calls Discord only when the hash changes. A Nighty restart with unchanged commands performs zero API calls.
- **A `ctx` that behaves** — `send()` works before or after `defer()`, becomes a follow-up automatically instead of raising "interaction already acknowledged", and inherits the `ephemeral` flag you deferred with.
- **Subcommand groups** — `slash.group("notes", …)` gives you `/notes add`, `/notes list`.
- **Context menus** — right-click → Apps, same decorator style.
- **Reload-safe** — re-registering a name replaces it instead of raising a duplicate-command error, so saving your script mid-session just works.
- **Never touches Nighty's own commands** — tracks only the names it registered itself, so `clear` can't delete `/help` out from under you.
- **Protected commands** — mark management commands so `clear` leaves them alone rather than deleting its own way back.
- **Errors are contained** — a raising handler is caught, logged with a traceback, and reported to the user ephemerally. Never takes the script down.
- **Degrades gracefully** — on a Nighty build without `bot_app` it logs one line and the rest of the script keeps working.

Full API reference (not reproduced here): `SLASH_API.md` in the library's repo.

### 24.5 Quick Start

1. Copy `slash_api.py` into the Nighty scripts folder and enable it.
2. Type `/ping` in Discord.

The included examples (`/ping`, `/roll`, `/echo`, `/serverinfo`, `/slow`, `/notes add`, `/notes list`) register and sync themselves on first load.

To use it inside your own script, copy everything between the `░░ SLASH API START ░░` and `░░ SLASH API END ░░` markers into the top of the `@nightyScript` function, then write commands below it:

```python
@nightyScript(name="My Script", author="you", description="...", usage="<p>mine")
def my_script():
    """..."""

    # ---- paste the SLASH API block here ----

    @slash.command(description="Roll a die")
    async def roll(ctx, sides: int = 6):
        await ctx.send(f"You rolled {random.randint(1, sides)}")

    slash.autosync()      # last line: syncs only if definitions changed

my_script()
```

### 24.6 Writing Commands

#### Options

```python
@slash.command(description="Repeat something back",
               describe={"text": "What to repeat", "private": "Only you see it"})
async def echo(ctx, text: str, private: bool = False):
    await ctx.send(text, ephemeral=private)
```

`str`, `int`, `float`, `bool` map to Discord's option types. A parameter with a default is optional — Discord requires required options to come first, so order the signature that way.

#### Slow work — defer inside 3 seconds

Discord kills any interaction not answered within 3 seconds:

```python
@slash.command(description="Fetch something slow")
async def fetch(ctx, url: str):
    await ctx.defer(ephemeral=True)          # 3s → 15 minutes
    data = await get_json(url)
    await ctx.send(f"Got {len(data)} rows")  # inherits ephemeral=True
```

#### Subcommand groups

```python
notes = slash.group("notes", "Keep quick notes")

@notes.command(name="add", description="Add a note")
async def notes_add(ctx, text: str):
    ...
```

#### Context menus

```python
@slash.menu("Count words", target="message")
async def count_words(ctx, message):
    await ctx.send(f"{len(message.content.split())} words.", ephemeral=True)
```

#### Errors

```python
@slash.error
async def on_error(ctx, exc):
    await ctx.send(f"Something broke: `{exc}`", ephemeral=True)
```

### 24.7 Syncing

`slash.autosync()` goes as the last line of the script. It fingerprints every definition (SHA-256 over names, descriptions, and option name/type/required), compares against the last successful sync stored in `json/slashapi_state.json`, and calls Discord only on a change.

| Scenario | Syncs |
|---|---|
| First run | 1 |
| Nighty restart, nothing changed | 0 |
| Description or option edited | 1 |
| Handler body edited | 0 |

If a sync fails, the fingerprint is **not** saved — it retries next load rather than silently giving up.

#### Manual control

Every one of these exists as both a prefix and a slash command:

| Prefix | Slash | Does |
|---|---|---|
| `<p>slashapi status` | `/slashapi status` | Is anything out of sync? |
| `<p>slashapi sync` | `/slashapi sync` | Force a publish now |
| `<p>slashapi list` | `/slashapi list` | What this script registered |
| `<p>slashapi clear` | `/slashapi clear confirm:True` | Unregister this script's commands |

### 24.8 Repository Layout

```
slash_api.py             The framework + runnable examples.
                          Paste-in block sits between the ░░ markers.
SLASH_API.md              Full API reference: every decorator, ctx method, gotcha.
```

### 24.9 Troubleshooting

| Symptom | Fix |
|---|---|
| Command doesn't appear | Global commands can take up to an hour to propagate the first time. Pass `guild=` while developing to register into one server instantly, then `Ctrl+R` in Discord — the client caches the command list. |
| "The application did not respond" | Handler took longer than 3 seconds without calling `await ctx.defer()`. |
| `CommandAlreadyRegistered` | Another script already owns that name. The API removes its own previous registration but can't see other scripts' commands. |
| 400 on sync | A command name has spaces or capitals, a description exceeds 100 characters, or an optional option precedes a required one. |
| Reply came out public | Pass `ephemeral=True` to the first `send()`, or to `defer()`. |
| Nighty's own commands vanished | Something bulk-synced the whole tree — this library only ever removes names it registered itself; check other scripts. |
| "This Nighty build does not expose `bot_app`" | Build predates script access to the command tree. Everything else in the script still runs. |

### 24.10 Known Limits

- A script can only manage its own commands. If several scripts each call `sync()`, that's several syncs — coordinating them needs engine-level support.
- Nothing prevents two scripts registering the same command name.
- `@slash.menu`'s `target` argument is currently descriptive; `discord.py` infers message vs. user from the handler's second parameter annotation.

---

## 25. Custom Features

No-code automation system, separate from NightyScript (§1-24). Built entirely in Nighty's own UI. Mechanics below (§25.1-25.3, 25.5-25.9, 25.11-25.12) are condensed from `docs.nighty.one/custom-features-tab/*` (fetched directly, full page text — not the earlier WebFetch summaries, which were clipping quotes short). The filter/condition/action **catalogs** (§25.5, 25.7, 25.10) are the exception: the site explains how they work but never lists them exhaustively — those tables were reverse-engineered by decoding a live exported feature (a "kitchen sink" test feature exercising every option), cross-checked against the site wherever it does confirm a name (filters catalog, one confirmed condition-chip label, the parameter-type dropdown).

**On community-shared feature exports used as sources in §25.2-25.4, 25.9:** several facts below (event-ID strings, the action-reference naming mechanic, `httpRequestV2`/`parseHttpResponse`) come from decoding features shared by other people, not self-verified live exports — explicitly **not confirmed to actually work**, only decoded for their JSON shape. Two tells they may not be raw Nighty exports: some have alphabetically-sorted object keys (Nighty's own exports don't), and schema version varies (`_v: 1` on these vs `_v: 2` on every self-verified sample in this doc) — possibly an older Nighty version, possibly a community re-serialization tool. Treated as decent evidence for *shape*, not as confirmed-working examples.

### 25.1 Core model — five parts, one sentence

A Custom Feature = **when this happens, check a few things, then do something.** Built and executed in this exact order — first "no" anywhere stops the whole run:

| N | Part | Purpose |
|---|---|---|
| 1 | Event | What triggers it — message, join, timer, another feature |
| 2 | Limits | How often it's allowed to run |
| 3 | Filters | Broad "where/who" bouncer-at-the-door checks |
| 4 | Conditions | Detailed checks — only what you want gets past |
| 5 | Actions | What actually happens |

`Event → Limits → Filters → Conditions → Actions`

### 25.2 Events & parameters

Every event carries **parameters** — containers of details, picked via the Expression Builder, never typed by hand.

Built-in on a typical New Message event: `message`, `user`, `channel`, `guild` (site says "guild" = "server" — Discord's own devs call servers guilds), `member`, `me` (always present, on every event, no exceptions).

**`user` vs `member`** — same person, different scope. `user`: the general Discord account — username, avatar, ID, mention — always available, everywhere. `member`: that user *inside one specific server* — nickname, roles, join date, kick/ban/timeout — only exists inside a server, never in DMs.

**`channel` vs `server_channel` vs `group_channel`** — `channel` is the safe general choice (works in servers, DMs, group DMs); `server_channel` adds topic/category/server-only settings, servers only; `group_channel` is group-DM-specific.

**`message` vs `message_before_edit` vs `message_replied_to`** — on a Message Edit event, `message` is the new version, `message_before_edit` the old one; `message_replied_to` is the older message being replied to, only present if the message is a reply.

**Before/after pairs on change events**: `old_nick`/`new_nick`, `old_name`/`new_name`, `old_topic`/`new_topic`, `member_before_update`/`member`, `user_before_update`/`user`.

**Availability rule**: not every parameter exists on every event — a DM event has no `guild`/`member`. A feature that works in servers but does nothing in DMs is almost always using `guild`/`member` where `user`/`channel` would have worked instead.

**Two non-Discord events**: **Repeat Action** fires on a timer ("every 60 seconds") — carries only `me`, zero Discord parameters. **Custom Feature** fires when another Custom Feature triggers it on purpose, with whatever parameters that feature decides to hand over — arrives pre-loaded, no fetch needed.

**Confirmed literal `event` field values** (from decoded real features — the JSON's `event` string, dotted `category.name` form): `message.newMessage`, `message.messageDelete`, `global.repeatAction`. Others (edit, member join/leave, role/nickname change, etc.) almost certainly follow the same `category.camelCaseName` pattern but weren't seen in a decoded sample, so treat as inferred, not confirmed.

**Repeat Action's timer, concretely:** the interval isn't a filter or a parameter — it lives in a top-level `extras: {"interval": N}` field on the feature object, sibling to `actions`/`conditions`/`filters`. Unit not confirmed from the one sample seen (`interval: 12` on a feature described as running periodically — plausibly minutes, unconfirmed).

### 25.3 Custom parameters

When the event doesn't hand you what you need (Repeat Action has nothing at all, but you need a specific channel to post in), add a custom parameter.

**Confirmed type dropdown** (from a live UI screenshot — "Select parameter type"), 5 options: `message`, `channel`, `server_channel`, `guild`, `member`.

Setup asks for: the type, a name (namespace it for your future self — `general_channel`, not `chan2`), a description ("refers to"), and **availability**.

**Availability = when Nighty fetches it from Discord**, three gates:

| `fetchAt` | Label (site wording) | Fetched | Usable in |
|---|---|---|---|
| `0` | **Everywhere** | On every event, always | Filters, Conditions, Actions |
| `1` | **Conditions and actions** | Only if filters passed | Conditions, Actions |
| `2` | **Actions** | Only if filters AND conditions passed | Actions only |

Rule: pick the latest option that still works — cuts wasted Discord API calls (site's example: 1,000 events/hr down to 5 requests using the latest-possible setting). Custom parameters can **never** be used in filters, regardless of this setting — filters run before any custom parameter has been fetched.

A wrong ID gets **abandoned after 3 failed fetch attempts** (protects against a typo generating endless failed requests) — fix the ID, then restart Nighty to clear the failure count and let it retry.

Exception: a **Custom Feature** event's handed-over parameters arrive pre-loaded — no fetch, no ID needed.

**The ID field name in `extraVars` depends on the parameter's type**, confirmed from multiple real samples: `channelId` for `channel`/`server_channel`, `guildId` for `guild`, `guildId` + `userId` together for `member` (needs both to identify a specific member of a specific server). Also seen: an `originalRef` field duplicating `ref` on at least one sample — purpose unconfirmed, possibly left over from renaming the parameter in the UI.

### 25.4 References

A **reference** is how a filter/condition/action says *which* parameter instance to operate on — necessary because an event can carry several things of the same kind (a Message Edit event's `message`, `message_before_edit`, `message_replied_to` are three different messages; "message contains X" means something different pointed at each one).

Actions needing two things (e.g. **Forward**: which message, which channel) ask for two references.

**Action references**: some actions hand back a result usable by later actions in the same run (**Send Message** → the message it sent; **Create Invite** → the invite it created). Two hard rules: only actions **below** the one that produced it can use it (order matters — it doesn't exist yet when earlier actions run), and it's **actions-only** — filters and conditions have already finished running by the time any action executes, so they can never see an action reference.

Confirmed live: `message_replied_to` is genuinely selectable in a condition's reference picker (re-pointed a `message`-parent `exactMatch` condition from `message` to `message_replied_to` in the actual UI, re-exported cleanly). Also confirmed: on a `message.newMessage` event, the UI's own descriptive subtitle for the built-in `message` parameter is **"Newly created message"** — worth knowing if you're hunting for a reference in the picker by its label rather than its raw name.

**The action-reference JSON mechanic, confirmed from a real script:** `parent` on a filter/condition/action always stays the fixed built-in category (`message`, `channel`, `user`, `global`, ...) — it's a UI grouping label, not the reference itself. The actual reference target is the **first arg** (or the dedicated ref slot), and it can be one of three things:
1. The literal built-in name, e.g. `"message"`, `"channel"`.
2. A custom parameter's `ref` name, e.g. a channel custom-param named `Dhc` — `sendMessage`'s args become `["Dhc", ...]` while `parent` stays `"channel"`.
3. **An action-reference name** — when an earlier action exposes its result, the name you gave it becomes a literal string that later actions drop into their own reference slot instead of the built-in name. Real example: a `reply` action's actionRef slot holds `"a_ref_delete"` (the chosen name); a later `delete` action's args are literally `["a_ref_delete"]` instead of `["message"]` — deleting the *reply that was just sent*, not the triggering message. `"a_ref_undefined"` seen elsewhere in this doc is what that same slot looks like when no ref has been exposed/named yet.

### 25.5 Filters (confirmed — site enumerates these)

First layer of narrowing — the "bouncer at the door." Run before conditions, before any action, and are **all-or-nothing**: every filter added must pass, one "no" stops the feature with nothing counted against execution limits. Filters can only see the event's **own built-in parameters** — never custom parameters (those aren't fetched yet, §25.3).

**The #1 rule: always add "Ignore self."** Without it, a feature that replies to messages will reply to its own reply, forever — an infinite loop that spams (and rate-limits) your account. "Only self" is the opposite (fires only on your own messages, for personal trigger-commands) but does **not** protect against the loop — with "Only self" your *conditions* must be the thing that keeps the feature from matching its own output (e.g. trigger on exact `!ping`, reply `pong` — safe; trigger on message *containing* `hello`, reply `hello there` — loops).

| Parent | Filter | Args | What it does |
|---|---|---|---|
| `user` | `ignoreSelf` | `(user)` | Skip events from your own account |
| `user` | `onlySelf` | `(user)` | Only your own account |
| `user` | `ignoreBots` | `(user)` | Exclude bots |
| `user` | `onlyBots` | `(user)` | Only bots |
| `user` | `ignoreFriends` | `(user)` | Exclude friended users |
| `user` | `onlyFriends` | `(user)` | Only friended users |
| `user` | `ignoreBlocked` | `(user)` | Allow events from blocked users through (i.e. don't exclude them) |
| `user` | `idWhitelist` | `(user, ids[])` | Only listed IDs pass |
| `user` | `idBlacklist` | `(user, ids[])` | Listed IDs blocked |
| `member` | `idWhitelist` | `(member, ids[])` | Only listed member IDs |
| `member` | `idBlacklist` | `(member, ids[])` | Exclude listed member IDs |
| `channel` | `onlyDMs` | `(channel)` | Only direct messages |
| `channel` | `onlyServers` | `(channel)` | Only server channels |
| `channel` | `idWhitelist` | `(channel, ids[])` | Only listed channel IDs |
| `channel` | `idBlacklist` | `(channel, ids[])` | Exclude listed channel IDs |
| `message` | `idWhitelist` | `(message, ids[])` | Only listed message IDs |
| `message` | `idBlacklist` | `(message, ids[])` | Exclude listed message IDs |
| `guild` | `idWhitelist` | `(guild, ids\|null)` | Only listed server IDs |
| `guild` | `idBlacklist` | `(guild, ids\|null)` | Exclude listed server IDs |
| `global` | `runOnWeekdays` | `(days[])` | Only fires on the picked weekdays (e.g. `["Monday"]`) |

Missing-parameter rule: Only/Whitelist filters **block** when the parameter is absent (e.g. a DM has no `guild`); Ignore/Blacklist filters **allow** through. Practical effect: a server ID whitelist silently blocks all DMs, a server ID blacklist silently allows all DMs through unfiltered. To support both, filter on `channel` ID (exists everywhere) instead of `guild`, or use the `channelTypeIs` condition (§25.7).

Combos that work well (site's list):

| Goal | Filters |
|---|---|
| Reply to people without looping | Ignore self + Ignore bots |
| Only work in one channel | Ignore self + channel ID Whitelist |
| Only work in your own servers | Ignore self + server ID Whitelist |
| Personal command you type yourself | Only self (+ a condition your output can't match) + global limit 1 per 5s |
| Ignore one annoying person everywhere | user ID Blacklist |
| React to bot announcements only | Only bots + channel ID Whitelist |
| DM auto-reply | Ignore self + Ignore bots + Only in DMs |
| Weekday-only feature | Ignore self + Run on Weekdays |

### 25.6 My Values (myVars)

Per-run variables, created inside a condition collection's **My Values** section: `name`, `type` (usually auto-detected), `value` (can be built from the live event, same as any expression). Computed **before** that collection's conditions are checked, and **regardless of whether they pass or fail** — so a My Value from collection 1 is available in every later collection, every If-met/Otherwise action, and the feature's final actions.

Use `my`→`your_name` in any expression field to read one. To change an existing one, use the **Change one of My Values** action (can only modify a value that already exists — create it in a condition collection first, even if left empty).

**Lifetime: exactly one run.** Reset to nothing on every new trigger — cannot be used as a persistent counter across runs. For state that must survive between runs, use the **Add ID Record** action instead (§25.7, §25.10).

Observed `type` values (undocumented — the site never mentions these, decompile-only): `text`, `file_convertable`, `file_convertable_list` — each with a `forcedType: true/false` flag (locks the type so the expression builder can't silently coerce it). `forcedType` is optional — one real sample had it explicitly `true`, a sibling My Value right next to it had it simply absent (defaults falsy).

**Worked example, from a real edited feature — a My Value built by concatenating two values:**
```
name: "server"
type: "text"
forcedType: true
value: [message]→jumpUrl + [message]→authorUsername   (no separator between them, just demonstrating concatenation is legal)
```
Confirms two new values not seen elsewhere in this doc: **`message`→`jumpUrl`** (a direct link to the triggering message on Discord) and **`message`→`authorUsername`** (the sender's username — distinct from `user`→`name`, though likely the same underlying string; use whichever the picker offers under `message` vs `user` in context). Also confirms a My Value's `value` can freely mix multiple value-pulls with no plain text between them, same as any other expression field.

### 25.7 Conditions (concept confirmed against docs; catalog reverse-engineered — not published as a list anywhere)

The detailed inspection after filters. Conditions live inside a **condition collection**, which bundles: **My Values** (§25.6), the **conditions** themselves, **If met** actions, and **Otherwise** actions. A feature can have multiple collections, checked top to bottom.

- Default: all conditions must pass (**AND** connector); toggle to **OR** to require just one. The connector applies to the whole group, not one pair.
- **Groups**: up to **two groups per collection**, each with its own AND/OR connector, plus a connector between the two groups — this is what lets you express `(A or B) and C` style logic that a single flat list can't. Trick: say the rule in plain English, put brackets where you'd naturally pause — each bracket becomes a group.
- A condition that errors (e.g. checks a message that doesn't exist) counts as **not passed**, logged.
- **If met / Otherwise** (advanced): each side runs its own action list, then chooses what happens next:
  - After **If met**: `Continue` (default, next collection) / `Skip to final actions` (skip remaining collections, run final actions) / `Stop` (end feature, final actions never run).
  - After **Otherwise**: `Stop` (default, end feature) / `Continue` (proceed to next collection anyway).
  - The action block **always runs** regardless of the chosen next-step — the setting only affects what happens *after*.
  - This pattern implements command routers: collection 1 checks `!weather` → if met, reply and Stop; otherwise Continue to collection 2's `!time` check, etc.
  - If-met/Otherwise actions that actually execute **do** count against execution limits (§25.8), same as a normal run.

Grouped by parent. `<expr>` = expression field. `[...]` = list input. Meanings below: inferred from identifier + arg shape + observed values (docs never enumerate these) — logic extension, not confirmed wording. Format: `id(args)` — meaning.

**One confirmed real UI label, as a data point for how trustworthy the inferred meanings below are:** a screenshot of a live condition collection showed a condition chip literally reading **"Word Blacklist"** (Title Case, with a space) attached to a `message`-parent condition — this is the `wordBlacklist` identifier below, confirming the UI renders these camelCase identifiers as spaced-out Title Case labels rather than anything more cryptic. Every other identifier in this subsection is inferred the same way (name + arg shape + observed values), not read off a label directly, but this one data point is why that inference method is trusted throughout §25.7 and §25.10.

**`message`**
- `contains(message, ANY|ALL, texts[], caseSens, wholeWord)` — msg text has any/all listed strings
- `doesNotContain(same)` — inverse, none of listed strings present
- `exactMatch(message, <expr>)` — msg content == expr, full match
- `isNotExactly(message, <expr>)` — inverse of exactMatch
- `startsWith/endsWith(message, <expr>, caseSens)` — msg content prefix/suffix check
- `regexMatch(message, match, pattern, caseSens)` — msg content vs regex
- `wordWhitelist/wordBlacklist(message, words[], caseSens)` — whole-word contains/excludes, unlike `contains` (substring). UI label for the blacklist variant confirmed as **"Word Blacklist"** — see note above.
- `messageLength(message, comparator, number)` — char count vs number
- `isAllCaps(message)` — content all uppercase
- `mentionsMe(message)` — @-mentions your account
- `isReplyToMyMessage(message)` — reply chain targets your own prior msg
- `withAttachments(message, count)` — attachment count >= count
- `hasNoAttachments(message)` — zero attachments
- `hasEmbeds(message)` — >=1 embed present
- `hasStickers(message)` — sticker(s) present
- `hasComponents(message)` / `hasNoComponents(message)` — button/select row present or absent
- `hasButton(message, <expr>, caseSens, any|all)` — button matching label/custom-id exists; any/all = match mode across multiple buttons
- `hasSelectMenu(message, <expr>, caseSens)` — select-menu component matching expr exists
- `hasSelectOption(message, <expr menu>, <expr option>, caseSens)` — target select menu offers this option
- `componentMatches(message, <expr ref>, contains|other, values[], caseSens)` — a specific component's label/value vs list
- `componentCount(message, <expr ref>, at least|other, number)` — count of matching components vs number
- `componentExists(message, <expr ref>, does not exist|other)` — presence/absence of component at given ref/index

**`server_channel`**
- `nameContains/nameDoesNotContain(server_channel, ANY, texts[], caseSens)` — channel name substring check
- `nameIs(server_channel, <expr>, caseSens)` — channel name == expr
- `isNews(server_channel)` — channel type = Announcement
- `isNSFW(server_channel)` — channel flagged age-restricted

**`channel`**
- `channelTypeIs(channel, types[])` — type in list. Enum: `Normal (Text)`, `Voice`, `Direct Message`, `Stage`, `Announcement`, `Category`, `Private Thread`, `Directory`, `Forum`, `Media`, `Public Thread`, `Group`
- `idEquals(channel, id)` — channel ID == literal id

**`guild`**
- `hasVanity(guild)` — server has custom vanity invite URL (discord.gg/name)
- `nameDoesNotContain(guild, ANY, texts[], caseSens)` — server name lacks listed strings
- `nameRegexMatch(guild, match, pattern)` — server name vs regex

**`member`**
- `isAdmin(member)` — member holds Administrator permission
- `roleIdCheck(member, Has role ID|other, id)` — member has/lacks role by ID (2nd arg flips has/lacks)
- `roleNameCheck(member, Has role name|other, name)` — same, by role name
- `nicknameIs(member, name, caseSens)` — member's server nickname == name

**`global`** (system/meta, no Discord parameter)
- `pcIdleCheck(less than|other, <expr minutes>)` — host PC idle duration vs minutes
- `activeWindowCheck(contains|other, <expr>, caseSens)` — currently-focused window title vs expr
- `processRunning(the program is running|other, <expr name>)` — named .exe running on host PC
- `myStatusIs(statuses[])` — your Discord presence in list. Enum: `Online`, `Idle`, `Do Not Disturb`, `Invisible`
- `textComparison(<expr A>, is exactly|other, <expr B>, caseSens)` — generic A-vs-B text compare, building block for custom logic (not tied to a fixed parameter)
- `numberComparison(<expr A>, equals|other, <expr B>, <expr?>)` — generic numeric compare, same idea
- `customFeatureState(featureName, continue when enabled|continue when disabled)` — gate this run on another feature's on/off toggle
- **ID Record conditions** — read state written by the `addIdRecord` action (§25.10), key = tag for the record set, recordId = the value being tracked (usually a user/channel/message ID):
  - `lastRecordWithinSeconds(<expr key>, recordId, seconds)` — most recent record for recordId under key is younger than seconds
  - `lessRecordsThanX` / `moreRecordsThanX(<expr key>, recordId, count)` — total record count for recordId vs count, all-time
  - `lessThanXRecordsInYSeconds(<expr key>, recordId, count, seconds)` — record count for recordId within trailing window vs count
  - `moreThanXRecordsInYSeconds` / `moreThanXUniqueRecordsInYSeconds(<expr key>, <expr recordId>, <expr count>, <expr seconds>)` — same, unique variant dedupes identical recordId values (e.g. count *distinct* users, not raw hits)

### 25.8 Execution limits & cooldowns

Two independent brake types, usable together:

- **Global limit**: `X times per Y seconds` for the feature as a whole, regardless of who/where. Default and recommended floor: **1 per 1 second**. Rolling window, not clock-aligned — "5 per 60s" means "in the last 60 seconds", not "reset on the minute."
- **Parameter limit**: same idea but Nighty keeps a **separate counter per distinct value** of a chosen parameter (`user`, `guild`, `channel`, `role`, ...) — e.g. limit on `user` gives every person their own private cooldown, so Anna hitting hers doesn't block Ben. `me` can never be used for a limit (there's only one of you, it'd just be the global limit again).
- **Time off**: switching the "per Y seconds" part off turns a limit into a fixed **total budget** ("3 times, ever") instead of a cooldown — useful for "welcome each member exactly once" (parameter limit on `member`, time off). Not permanent forever — clears on reset, same as any limit.
- **A trigger only burns a limit if actions actually ran.** Events that get filtered/condition-blocked before reaching any action cost nothing — a tight "1 per 60s" limit doesn't get silently eaten by background chatter the feature only looked at and ignored.
- **Limits reset** on: editing the feature, toggling it off/on, renaming it, or restarting Nighty. Counters live in memory only, not on disk — handy while testing, toggle off/on to clear your own cooldown.
- If the limited parameter isn't present on a given event (e.g. limit on `guild`, event is a DM), that specific limit is simply skipped for that run — other limits (esp. the global one) still apply. This is why the site recommends always keeping the global limit on: it's the one guaranteed to apply every time.

**Cooldowns object shape** — confirmed against the actual Execution Limits UI: `global` is the **only** mandatory entry — shown with a power toggle (on/off), no delete button, always present at `{times, per}`. Every other limit — `user`, `message`, `guild`, `channel`, `member`, or a custom parameter by its `ref` name — is **opt-in**: each has a delete (trash) icon and is added one at a time via an "add limit" control. A fresh feature's `cooldowns` object has only `global`; earlier decompiled samples had every built-in type populated only because that test feature manually added a limit for each one to exercise them all — not because Nighty adds them by default. Add a parameter limit only if actually needed.

### 25.9 Expressions

A text-field block that gets replaced with real data when the feature runs. Built from:

- **Values** — raw details from a parameter (e.g. `user`→`mention` inserted into a Reply so it correctly @-mentions whoever actually sent the triggering message). Freely mixable with plain typed text.
- **Functions** — transform or derive from a value (e.g. `upper` on a name → `JOHNSMITH`). Functions can nest inside functions, evaluated innermost-first (e.g. fall back to username when nickname is empty, then uppercase the result).

Gotchas: everything resolves to **text** once combined into a field (numbers included) — except **files** (avatars, attachments), which are handed over as real file objects only when a value is placed alone in a dedicated File input.

**Full function catalog — confirmed directly from a screenshot walkthrough of the live "Add Function" dropdown, same confidence tier as §25.18's Values catalog.** Grouped by the category header the UI itself groups them under. Descriptions paraphrased from the UI's own tooltip text, not verbatim.

**`network`**
- `httpRequest(...)` — sends an HTTP request and returns the response; if the response is JSON it's auto-parsed, otherwise returned as plain text. Simpler than `httpRequestV2` when you just want the body and don't need status/header details.
- `httpRequestV2(method, url, arg3, arg4, arg5)` — sends an HTTP request and always returns a text summary of what happened (status code, headers, body, and any error) even when the request fails. Meant to be saved into a My Value and read piece-by-piece with `parseHttpResponse` rather than used inline. Confirmed 5 argument slots from decoded samples; only `method` and `url` have been seen populated, the other 3 (headers/body/something else) remain unconfirmed content, not confirmed absent (§25.14's own "not found yet" framing applies here too).
- `parseHttpResponse(httpRequestV2_result, path)` — reads one specific piece out of a saved `httpRequestV2` result: the status code, an error message, a header, or a value from the body via dot-path (e.g. `body.user.name`, or `body.user.0.name` for array indexing — confirmed numeric-index dot-path syntax, an upgrade on this doc's earlier assumption that only object-key paths worked).

**`math`** (previously entirely undocumented in this doc)
- `randomNumber(min, max)` — random whole number, inclusive on both ends.
- `len(list)` — number of elements in a list.
- `add`, `subtract`, `multiply`, `divide` — basic arithmetic on two numbers; `divide` returns nothing (not an error, not infinity) on divide-by-zero.
- `round(number, decimalPlaces)` — rounds to a chosen number of decimal places.

**`text`** (previously only `upper`/`default`/`findBetweenText` were confirmed — this closes most of that gap)
- `randomText(...)` — randomly picks one of several provided text options.
- `maxText(text, maxLength)` — truncates text and appends `...` if it exceeds a length.
- `findBeforeText(text, marker)` / `findAfterText(text, marker)` — everything before/after the first occurrence of a marker; returns nothing if the marker never appears. Simpler siblings of `findBetweenText` for a one-sided cut.
- `findBetweenText(prefix_parts[], haystack_parts[], suffix_parts[])` — extracts the substring between a prefix and suffix found inside a haystack value; seen used to pull a prize name out of `Congratulations @you! You won the **PRIZE**!` by giving `"Congratulations "`+mention as prefix, `message`→`content` as haystack, `"**!"` as suffix.
- `upper` / `lower` — full-string case conversion.
- `capitalize` — first letter capital, rest lowercase.
- `trim` — strips leading/trailing whitespace.
- `replace(text, find, replaceWith)` — global find-and-replace, every occurrence.
- `repeat(text, count)` — repeats a string.
- `reverse(text)` — reverses character order.
- `substring(text, start, end)` — character-position slice, 0-indexed.
- `length(text)` — character count.
- `padLeft(text, targetLength, padChar)` / `padRight(...)` — pads to a target length.
- `regexExtract(text, pattern)` — first regex match, or nothing if no match.
- `regexReplace(text, pattern, replaceWith)` — replaces every regex match.

**`time`**
- `timestamp(secondsFromNow)` — a Discord auto-localizing timestamp (renders in each viewer's own timezone, can count up/down live) pointing N seconds from now.
- `unixNow()` — current Unix time in seconds.
- `formatDuration(seconds)` — formats a number of seconds into a human string (the visible fragment of its tooltip showed an example ending `"...30m 15s"`) — exact format string unconfirmed beyond that fragment.

**`logic`**
- `ifEquals(a, b, ifTrue, ifFalse)` — ternary: compares two values, returns one result or the other. The closest thing to conditional branching available inside an expression.
- `ifContains(text, search, ifTrue, ifFalse)` — same ternary shape, but the condition is a substring check instead of equality.
- `default(value, fallback)` — returns `fallback` if `value` is empty, otherwise `value` itself. (Already referenced elsewhere in this doc from earlier partial confirmation; now confirmed as belonging to the `logic` category.)

Neither `ifEquals` nor `ifContains` is a general boolean `and`/`or` combinator — they each take exactly one condition and produce one of two fixed results. Combining two independent boolean facts into one value (e.g. "is this a proxy OR a hosting IP") would need nesting one `ifEquals`/`ifContains` inside another's `ifTrue`/`ifFalse` slot as a workaround — untested, but structurally the only path available with what's confirmed here. Treat the "boolean-combine function" gap in §25.14 as **narrowed, not closed**: a workaround exists, no direct primitive does.

**`encoding`**
- `getJsonValue(json, path)` — reads one value out of arbitrary JSON by dot-path, supporting numeric array indices (`user.name`, `users.0.name`). Overlaps with `parseHttpResponse` but works on any JSON value, not just an `httpRequestV2` result — e.g. JSON built or stored elsewhere in the feature.
- `appendJsonValue(...)` — tooltip text observed **identical** to `getJsonValue`'s in the live UI (a likely copy-paste labeling bug on Nighty's side, or the tooltip simply hasn't been written yet) — the name strongly implies it *writes* a value into JSON rather than reading one, but that's inference from the name, not from the (unhelpful) tooltip. Flagged as a genuine unknown, not guessed at further.
- `readFile(path)` — reads a file's contents as text.
- `base64Encode(text)` / `base64Decode(text)` — standard Base64 round-trip; explicitly labeled "not encryption" in its own tooltip. `base64Decode` returns nothing on invalid input rather than erroring. **This closes the "Base64 / generic string-encode function" gap in §25.14** — a token/ID-encoder-style tool is portable now.
- `urlEncode(text)` — percent-encodes text for safe use inside a URL (spaces, special characters).

**`storage`**
- `set(key, value)` / `get(key)` — store and retrieve a variable "in the cache." Scope and lifetime (per-run like My Values, §25.6, or persistent like Add ID Record, §25.7) is **unconfirmed** — the tooltip text alone doesn't say, and this doc has no decoded sample using either function yet. Don't assume persistence across runs without testing.

### 25.10 Actions (reverse-engineered — not published as a list anywhere)

Run strictly **top to bottom**, each fully finishing before the next starts. If one action fails (e.g. deleting an already-deleted message), the error is logged and the rest of the list still runs — one broken step doesn't abort the feature. Slow actions (waiting, fetching, running a slash command) block everything below them until done — this is what makes the **Wait** action useful for sequencing (e.g. send a "cleaning up in 10s" notice, `Wait` 10s, then `Delete` it, using an exposed action ref from the Send Message step, §25.4).

Same catalog format as above. `actionRef` = named result later actions below it can reference (§25.4).

**`message`**
- `reply(message, <expr text>, files[], silent, actionRef)` — reply to triggering msg; silent = no ping; exposes sent msg
- `edit(message, <expr>)` — edit message content (only your own messages editable)
- `delete(message)` — delete message
- `pin` / `unpin(message)` — pin/unpin in channel
- `ack` / `unack(message)` — mark read/unread (Discord ack state)
- `react(message, <expr emoji>, burstReaction)` — add reaction; burstReaction = super-reaction
- `removeReaction(message, <expr emoji>)` — strip your own reaction of that emoji
- `clearReactions(message)` — strip all reactions
- `clickButton(message, <expr label/id>)` — press a button component
- `clickComponent(message, <expr>)` — generic component press by ref (button or select trigger)
- `chooseSelectOption(message, <expr menuRef>, label|other, values[])` — pick option(s) in a select-menu component
- `logComponentOutline(message)` — dump msg's component tree to Nighty's log — debug helper, no Discord side-effect
- `saveAttachments(message, <expr folder>, <expr filename>, overwrite)` — download attachments to local disk
- `forward(message, channel)` — Discord-native forward to channel
- `forwardToWebhook(message, <expr webhookUrl>, includeContent, includeEmbeds, includeAttachments, includeAuthor)` — relay msg data to external webhook URL, each flag toggles what's copied

**`channel`**
- `sendMessage(channel, <expr text>, files[], actionRef)` — post new message, exposes sent msg
- `fakeTyping(channel, seconds)` — show typing-indicator for N sec, no message sent
- `purgeMessages(channel, count, onlyMine)` — bulk-delete last N messages; onlyMine restricts to your own
- `runSlashCommand(channel, {name, appId, options[], withManualMode})` — no-code twin of `exec_slash` (§9) — invokes another bot's slash command

**`server_channel`**
- `renameChannel(server_channel, <expr>)` — set channel name
- `editSlowmode(server_channel, seconds)` — set slowmode delay
- `cloneChannel(server_channel, <expr name>)` — duplicate channel under new name
- `createInvite(server_channel, maxAge, maxUses, temporary, actionRef)` — generate invite, exposes invite object/link
- `delete(server_channel)` — delete the channel

**`member`**
- `addRoleId(member, id)` — grant role by ID
- `removeRoleByName(member, name)` — revoke role by name
- `addRoleByName(member, name)` — grant role by name — logical counterpart `removeRoleById` almost certainly exists too, just not exercised in this sample
- `setNickname(member, <expr>)` — set server nickname
- `timeout(member, <expr duration>, <expr reason>)` — Discord timeout (temp mute)
- `kick(member, <expr reason>)` — remove from server, can rejoin via new invite
- `ban(member, <expr reason>)` — remove + block rejoin

**`guild`**
- `leave(guild)` — leave the server
- `markAsRead(guild)` — mark every channel in server read

**`global`**
- `showMessageNotification(message, <expr text>, info|success|other)` — desktop notification tied to a message (click likely jumps to it)
- `showNotification(<expr text>, info|other)` — desktop notification, no message context
- `showToastNotification(info|other, <expr title>, <expr description>, <expr...>)` — in-app toast, same style as `tab.toast()` (§12.7) but callable outside UI scripts
- `sendWebhookMessage(webhookUrl, <expr content>, embed{})` — POST straight to a Discord webhook — bypasses `forwardEmbedMethod`'s field limits (§8.2), see embed shape below
- `addIdRecord(<expr key>, <expr recordId>)` — append one timestamped record under key — the write side of `custom_features_store.json` and every ID-Record condition above
- `clearIdRecords(<expr key>, <expr recordId>)` — delete stored record(s) matching key/recordId
- `wait(<expr seconds>)` — pause the action chain N seconds before continuing
- `openUrl(<expr url>)` — open URL in default browser
- `appendToFile(<expr path>, <expr text>)` — append a line to a local file
- `saveAsJson(<expr path>, <expr data>, prettyPrint)` — write data as JSON to local file
- `editMyVar(varName, <expr newValue>)` — overwrite an existing My Value for the rest of this run — the **Change one of My Values** action
- `stopFeature(<expr reason>, exact match|other)` — halt remaining steps of current feature, reason gets logged
- `toggleCustomFeature(featureName, enabled)` — flip another feature's on/off switch
- `schedulePcShutdown` / `schedulePcSleep(<expr delay>)` — schedule host PC shutdown/sleep after delay
- `cancelPcShutdown` / `cancelPcSleep()` — cancel a pending scheduled shutdown/sleep
- `closeNighty()` — quit the Nighty app itself

**`sendWebhookMessage` embed object — full shape (undocumented, beats `forwardEmbedMethod`'s limits):**

**The embed builder itself is opt-in per component — confirmed from a screenshot of the live UI.** Under the message-content field sits a row of buttons: **Add Title**, **Add Description**, **Add ThumbnailUrl**, **Add ImageUrl**, **Add Author**, **Add Footer**, **Add Fields**. Each is a separate toggle — clicking one is what actually adds that key to the embed object. This is *why* `title`/`description` came back absent in one real sample and present in another (§ below): it's not a default-vs-edited distinction, it's simply whether that button was ever clicked. Don't emit a key you don't intend to use — the UI itself never would either.

Once added, a component starts with its own placeholder-as-value: the live preview shows ghost text (`"Author Name"`, `"Title"`, `"Footer Text"`, an empty-field "Add Field" prompt inside Fields) that becomes the *actual saved value* if you never type over it — confirmed twice independently, both times the untouched `footer.text` came back as the literal string `"Footer Text"` in the exported JSON, not `""`. Treat every component's placeholder as a real default, not just a UI hint.

Shape once every component has been added (confirmed against a real feature exported twice — before and after editing in the live UI):

```
{
  title: "Title", description: "Description",
  imageUrl: "", thumbnailUrl: "",
  author: { name: "Author Name", iconUrl: "" },
  footer: { text: "Footer Text" },
  fields: []
}
```

`title`/`description` are omitted from this shape entirely — not present as `""` — unless "Add Title"/"Add Description" was actually clicked (opt-in mechanism above). Once clicked, their untouched placeholder-as-value follows the same rule as `footer.text`: literal `"Title"` and `"Description"`, Title Case, matching every other confirmed placeholder-default in this doc (`"Footer Text"`, `"Author Name"`, `"New Field"`) — no default seen anywhere has been in all-caps. `author` needs `iconUrl` alongside `name`, not just `name`, once "Add Author" is clicked.

**Every field inside the embed object is a plain static string — none of them are expression-capable.** `title`, `description`, `author.name`, `author.iconUrl`, `footer.text`, `fields[].name`, `fields[].value` are all confirmed as raw strings in every sample decoded this session, never as `{"type": "expression_input", ...}`. Only `sendWebhookMessage`'s *second* argument (the plain message content, outside the embed) supports values/mentions/functions. Practical effect: you cannot put a live user mention, message content, or server name **inside** the embed itself — that dynamic content has to live in the message text above the embed. Design embeds as static, descriptive framing (title/description/footer explaining *what kind* of thing happened) and put the actual dynamic details in the message content field.

**Confirmed default for a freshly-added, untouched field** (same value seen twice, once in the very first decoded example and once here): `{"name": "New Field", "value": "Your text here", "inline": false}`. Build a field this way if you want a blank one ready to edit — don't invent other placeholder text for it.

This is the only in-app path (besides the raw `requests.post` webhook trick in §13) to get `fields`, `thumbnail`, `footer`, and `author` on an embed without writing a NightyScript.

### 25.11 Slash command responder

**Run Slash Command** is a special action that executes any slash command exactly as if typed manually — including third-party bot commands, same mechanism-level goal as `exec_slash` (§9) but built into the no-code UI instead of hand-rolled Python.

Slash commands only exist in channels where the owning bot is present with permission, so the action needs a real channel to ask Discord *"what commands exist here?"* before it can offer a picker.

**Recommended 4-step setup:**
1. Add a **custom parameter** of type Channel pointing at a real channel where the command is known to work (copy the channel ID from Discord).
2. Add **Run Slash Command**, set its "Refers to" to that channel parameter.
3. Pick the command from the now-populated list and fill in options (avoids hand-typed command/option name typos).
4. *(Optional)* Once configured, swap the reference to a different/dynamic channel (e.g. the event's `channel`) — the chosen command stays selected, only the destination changes.

If the channel varies (not a fixed Repeat Action target), restrict with **Server/Channel ID Whitelist filters** to only the channels where that command actually exists — otherwise the feature attempts (and fails) the command everywhere the event fires, cluttering the log.

Options accept full **expressions**, not just fixed text — e.g. a `/ban` reason built from `user`→`displayName`, a `/search` query from `message`→`content`, or an amount from a **My Value**.

### 25.12 Safe mode

Every new Custom Feature starts with **safe mode ON**. The feature runs its entire pipeline for real — event, limits, filters, conditions — but at the final action step, nothing is actually sent to Discord: no message, no delete, no ban. Instead you get a notification describing what *would* have happened. Covers every action, including the If-met/Otherwise actions inside condition collections (§25.7) — nothing slips through.

Workflow: build the feature with safe mode on, trigger it for real on Discord, confirm the notification fires exactly when expected and stays silent otherwise, **then** turn safe mode off. If safe mode is producing a stream of notifications, the feature is misconfigured (usually a missing "Ignore self" filter or over-loose conditions) — fix it before disabling safe mode, since that same volume of triggers would otherwise hit Discord for real.

Spam-diagnosis table (site's):

| Symptom | Likely cause |
|---|---|
| Fires on every message | No conditions, or conditions that almost always pass |
| Fires on your own messages | Missing Ignore self filter |
| Fires in unintended servers/channels | No whitelist filter |
| Fires on bot messages | Missing Ignore bots filter |
| Fires several times per event | Conditions match more broadly than intended |
| Fires steadily forever | A loop — re-triggering on its own output |

Off-checklist before disabling safe mode: fired every time it should / stayed quiet every time it shouldn't / no spam / has Ignore self / sensible execution limit / filters restrict to the intended servers-channels / feature's log has no errors / comfortable with what happens if it fires unexpectedly — especially anything irreversible (ban, kick, delete, purge, leave).

### 25.13 `safeMode` default when generating a feature JSON directly

Nighty's own UI defaults every new feature to `safeMode: true` — build it, confirm the notification fires right, then turn it off by hand. That default is for hand-building through the point-and-click editor.

When a feature is generated as raw JSON instead (a direct-write to `custom_features.json`, or an import/export blob) for this user specifically: default `safeMode: false`. Confirmed preference — always write it `false` unless told otherwise, don't ask.

### 25.14 What Custom Features would need to absorb a typical multi-tool NightyScript

Not a portability verdict — a **gap list**: primitives *not found among everything decoded/screenshotted so far* that a *typical* consolidated NightyScript utility (the kind that bundles a dozen-plus unrelated commands — bulk-delete, a voice-protection module, a network speed test, ID/token encoding, an IP lookup, a text stylizer, a data-backup tool, a help menu, network diagnostics, an away/AFK auto-reply, a remote-control panel, a hardware-specs reporter, an internal-config editor, a latency checker, a startup/autostart configurator, a Rich Presence value registrar) would need in order to be rebuilt entirely as no-code features instead of Python. A generalizable checklist for evaluating *any* NightyScript against the no-code catalog, not specific to one person's script.

**Read every "not found" below as exactly that — not found yet, not proven absent.** §25.18 already overturned one row in an earlier pass of this table (hardware/system values, corrected below) simply because a screenshot went somewhere this doc's decoded JSON samples hadn't. The catalog in this section keeps growing every time more of the live UI gets seen; treat every gap here as "no evidence for it in what's been checked," never as a confirmed ceiling on what Custom Features can do.

| Candidate gap | Category | Would block | Notes — what's actually been checked |
|---|---|---|---|
| ~~Command-argument parsing~~ (`!clear 50` → `amount=50`) | Event/trigger | A bulk-message-delete command taking a count | **Narrowed, not open.** §25.9's `text` category confirms `regexExtract(text, pattern)` — a condition/action could run `regexExtract` on `message`→`content` with a pattern like `!clear (\d+)` to pull the number out, then feed that into `purgeMessages`'s count. Untested end-to-end, but a real primitive now exists where this table previously said none did. |
| Voice-channel actions (move member, server-mute/deafen toggle) | Action | A voice-protection ("anti-nuke") module; a voice-channel member-list monitor | No voice actions found in the confirmed catalog (§25.10) — but that catalog was built from one test feature's action list (§25.10's own caveat) plus a handful of community samples, none of which happened to exercise voice actions. Absence from those specific sources, not a confirmed absence from the engine. |
| Voice-state-change event | Event | Same as above | Only `message.newMessage`, `message.messageDelete`, `global.repeatAction` are confirmed (§25.2) — from samples that happened to use those three. A join/leave/mute voice event is plausible and simply hasn't shown up in a decoded sample yet. |
| "List channel members" value | Value | A voice-channel member-list monitor | Not found among the values enumerated in §25.18 (which is itself scroll-limited for `channel`/`server_channel` — see that section's caveat). |
| Raw socket / byte-timing primitive | Action | A network speed test | Nothing seen measures throughput directly; `httpRequestV2` does a request but no timing/size value has turned up alongside it. |
| ~~Base64 / generic string-encode function~~ | Function | An ID/token encoder utility | **Closed.** §25.9's `encoding` category confirms `base64Encode`/`base64Decode` directly. |
| Boolean-combine function (`a or b`, `a and b`) inside an expression | Function | An IP-geolocation lookup that needs to derive one flag from two API fields | **Narrowed, not closed** — see §25.9's `logic` category: no direct `or`/`and`, but `ifEquals`/`ifContains` nested inside each other's true/false slot is a plausible (untested) workaround for combining two conditions into one value. |
| ~~Per-character string transform / lookup-table function~~ | Function | A text stylizer (e.g. small-caps converter) | **Narrowed, not open.** §25.9's `text` category adds `replace`/`regexReplace` alongside `upper`/`lower`/`capitalize` — a full charset remap (26 letters → 26 stylized glyphs) is achievable by chaining `replace` once per character, laborious but real, versus this table's earlier claim that nothing beyond `upper` existed. |
| Filesystem archive (zip) action | Action | A data-backup tool | `saveAsJson`/`appendToFile` (§25.10) write one file each; no archive/compress action has appeared in any sample. `readFile` (§25.9, `encoding`) confirms reading a file back is possible, which wasn't previously documented either — still no archive/zip primitive though. |
| Internal Nighty config-write action | Action | Anything that edits Nighty's own app settings from within a feature | `editMyVar` only touches per-run My Values (§25.6), not the app's own settings — no `updateConfigData`-style action has turned up, though §25.18's `global` values show the *reading* side (pcUsername, activeWindow, etc.) is much richer than this doc first assumed. |
| Latency-measurement primitive | Value | A ping/latency checker | No round-trip-time value has appeared among what's been checked, including the full §25.18 walkthrough of `global`. |
| OS-level control (registry autostart, tray, window resize, launch an arbitrary external .exe) | Action | A startup/autostart configurator; a remote-control panel | Nothing resembling a generic "run program" action has appeared. `schedulePcShutdown`/`Sleep` (§25.10) are the one confirmed OS-control exception, so *some* OS actions clearly do exist in the model — this row is "not found," explicitly not "structurally impossible." |
| Rich Presence dynamic-value registration | Action | A Rich Presence value registrar | `addDRPCValue` (§16) is a NightyScript-only global; nothing playing the same role has appeared on the Custom Feature side. |
| Discoverable command/help listing | N/A | A help-menu command | Custom Features have no command registry in anything seen so far — this one's closer to a genuine conceptual mismatch (there's no command list *to* enumerate) than a missing primitive. |

~~System/hardware introspection value~~ — **corrected, not a gap.** An earlier pass of this table claimed no value surfaces CPU/RAM/disk info. §25.18's full walkthrough of `global` found `cpuPercent`, `gpuPercent`, `ramPercent`, `ramUsedGb`, `ramTotalGb`, `diskFreeGb`, `diskFreePercent`, `batteryPercent`, `batteryPlugged`, `uptimeSeconds`. Left in the table, struck through, specifically as a reminder of how fast a "not found" row can flip once more of the UI gets checked — the exact mistake this section's "not found yet" framing exists to prevent repeating.

**Read on this:** after the full §25.9 function catalog came in, three of this table's original rows flipped from "gap" to "closed" or "narrowed" (struck through above) — base64 encoding exists outright, and both the command-argument-parsing and per-character-string-transform gaps have real (if untested) workarounds via `regexExtract`/`replace`. What's left looking genuinely structural — voice actions, OS-level program launching, filesystem archives — is a much shorter list than this table started with, and even that's "hasn't been seen" rather than "proven impossible" per §25.14's own framing. Check the live UI (or a fresh decoded sample) before treating any remaining row as settled; this table has already been wrong three times in one sitting.

**How to run this comparison on any real script:** list its distinct commands/modules, then for each one ask "does it (a) only read/act on things already covered by a confirmed value or action (§25.2, §25.10), (b) take a user-supplied argument beyond a fixed trigger phrase, (c) touch voice state, the filesystem beyond one file, OS-level resources, or hardware, or (d) need real control flow (loops, computed booleans, per-character string logic)?" — (a) alone means fully portable, any of (b)-(d) means partial-at-best or not portable, using the gap categories above to say specifically why.

### 25.15 Worked (untested) example — IP Lookup and Check-Host as Custom Features

Attempted builds for the two tool-kinds marked **Partial** in §25.14 (an IP-geolocation lookup and a network-diagnostics tool), to see how far `httpRequestV2`/`parseHttpResponse`/`findBetweenText` actually stretch. Both ship with `safeMode: true` — a deliberate exception to the §25.13 default, because these lean on two behaviors that are **assumed, not confirmed**:

1. **Empty `suffix` in `findBetweenText` means "to the end of the string."** Every confirmed sample used a real suffix (`"**!"`). Extracting an argument off the end of a command like `.ip 1.2.3.4` needs prefix `.ip `, haystack `message`→`content`, suffix `[]` — untested.
2. **An arg-group can mix plain text with a nested function call**, e.g. building a URL as `"http://ip-api.com/json/" + <extracted IP> + "?fields=..."` inside `httpRequestV2`'s URL slot. Every confirmed sample had exactly one text part per arg-group; nesting a `findBetweenText` call inside one alongside literal text is structurally consistent but never directly observed.

**IP Lookup**: `.ip <address>` (filter: `onlySelf`, condition: `startsWith(message, ".ip ")`) → one `reply` action mixing 6 independent `httpRequestV2`+`parseHttpResponse` pairs against `ip-api.com` (country, city, isp, proxy, hosting, mobile) — 6 separate HTTP requests for one report, since nothing confirms a way to reuse one `httpRequestV2` call's result across multiple `parseHttpResponse` pulls. The original script's derived `is_proxy = proxy or hosting` boolean is **not** reproduced — both raw fields are shown instead, per the missing boolean-combine function noted in §25.14.

**Check-Host**: `.checkhost <address>` → `httpRequestV2` GET to `check-host.net/check-ping`, pulls `request_id` via `parseHttpResponse`, replies with a link to `check-host.net/check-report/<request_id>` for the user to open manually. This is a deliberately reduced version: check-host's real per-node results come back keyed by dynamic node hostnames (`node1.check-host.net`, ...), which `parseHttpResponse`'s fixed dot-path can't address — so the automatic pass/fail parsing the original script did is dropped entirely, only the fixed-name `request_id` field survives the round trip. The `check-host.net` response schema itself (field names, whether `permanent_link` exists) was recalled from memory, not verified live — if the reply comes back wrong or empty, that's the first thing to check.

Both need real-world testing before any of this gets promoted from "assumed" to "confirmed" elsewhere in this doc.

### 25.16 Confirmed mode-string / enum values, by option

Throughout §25.5-25.10 several args are written as `"x"|other` — a literal value confirmed from a real sample, with unknown siblings. Collected here in one place: every mode-string actually seen across every decoded sample this session, grouped by which option it belongs to. Where a full enum is already known it's marked closed; everything else is still `|other` — a floor, not a ceiling.

| Option (action/condition) | Confirmed values seen | Closed? |
|---|---|---|
| Notification level — `showMessageNotification`, `showNotification`, `showToastNotification` (§25.10) | `"info"`, `"success"` | No — `"error"`/`"warning"` never observed but plausible |
| Text-match mode — `contains`, `doesNotContain` (§25.7) | `"ANY of the texts"`, `"ALL of the texts"` | Likely closed — only two modes make sense for a multi-text match |
| Numeric/length comparator — `messageLength`, `pcIdleCheck` (§25.7) | `"less than"` | No — an "at least"-style sibling almost certainly exists (seen as a distinct string on `componentCount`, below) |
| Equality comparator — `textComparison` (§25.7) | `"is exactly"` | No |
| Equality comparator — `numberComparison` (§25.7) | `"equals"` | No — note this is a *different* string from `textComparison`'s, despite both meaning "=" |
| Component-count comparator — `componentCount` (§25.7) | `"at least"` | No |
| Existence comparator — `componentExists` (§25.7) | `"does not exist"` | No — an "exists" counterpart is a near-certainty |
| Generic pattern-match mode — `regexMatch`, `nameRegexMatch`, `activeWindowCheck`, `componentMatches` (§25.7) | `"match"` (regex-family), `"contains"` (window/component-family) | No |
| `customFeatureState` mode (§25.7) | `"continue when enabled"`, `"continue when disabled"` | Likely closed — binary by nature |
| `stopFeature` mode (§25.10) | `"exact match"` | No |
| `hasButton`/`hasSelectOption` multi-match mode (§25.7) | `"any"` | No — an "all" counterpart is likely, mirroring the `contains` ANY/ALL pattern above |
| `roleIdCheck`/`roleNameCheck` mode (§25.7) | `"Has role ID"`, `"Has role name"` respectively | No — these read as full UI labels (Title Case, spaces) rather than terse mode tokens, consistent with the `"Word Blacklist"` chip label confirmed in §25.7's screenshot; a "Does not have..." counterpart is likely |
| `channelTypeIs` type list (§25.7) | 12 values, fully enumerated in §25.7 | **Yes** |
| `myStatusIs` status list (§25.7) | 4 values, fully enumerated in §25.7 | **Yes** |
| Custom-parameter `type` (§25.3) | `message`, `channel`, `server_channel`, `guild`, `member` — from the live dropdown screenshot | **Yes** (5, confirmed via UI) |
| `fetchAt` availability gate (§25.3) | `0`, `1`, `2` | **Yes** (3, confirmed via site + UI) |
| Embed default field values (§25.10) | `footer.text` defaults to `"Footer Text"`; a fresh field defaults to `{"name": "New Field", "value": "Your text here"}` | **Yes** (defaults, not an enum, but same "confirmed literal" spirit) |

Practical rule of thumb from this table: **don't invent a sibling string and expect it to work.** Where a comparator or mode has only one confirmed value, the safest move when building a feature is to pick that exact string in the UI's own dropdown rather than guessing at a plausible-sounding alternative in raw JSON — the dropdown is the ground truth this table is chasing, not the other way around.

### 25.17 Who writes the content — division of labor when building a feature for this user

Confirmed workflow rule, same category as §25.13: when generating a Custom Feature JSON for this user, **compose real embed and message content — don't leave it as a `FILL ME` placeholder.** Write an actual title, description, footer, field names/values, and message text that fit the feature's purpose, using the confirmed patterns in §25.10 (embed = static strings, no expressions; dynamic content — mentions, message text, server name — goes in the outer message-content field, not inside the embed).

The user's role in this loop is supplying **screenshots of the live UI** — those are the ground-truth reference this whole section (§25) is built from, and they're the user's contribution to check assumptions against, not something they're expected to also write copy for. Placeholder text (`FILL ME`, empty fields) is appropriate only for a genuine fill-in-the-blank *template* meant to be edited later (like the kitchen-sink reference in §25.15) — not for a feature built to answer a specific, real request.

### 25.18 Confirmed Values catalog — every value seen in the live expression-builder dropdown

The single highest-confidence source in this whole section: a direct screenshot walkthrough of the "Add Value" search dropdown inside the Expression Builder, scrolled through every parameter category. Unlike the reverse-engineered condition/action catalogs (§25.5, 25.7, 25.10 — inferred from decoded JSON), this is the UI's own value picker, read directly. Format: `parameter`→`value` — one row per confirmed entry, grouped by parameter. Where a category's list was cut off mid-scroll in the source screenshots (only true for `channel`/`server_channel`), that's noted — treat those two as a known-incomplete floor, everything else here is a complete read of what the dropdown showed for that parameter.

**`message`** (the triggering message itself — see §25.2 for how this differs from `message_before_edit`/`message_replied_to`, which are separate parameters carrying the *same* value list, just pointed at a different message):
`content`, `id`, `authorUsername`, `authorDisplayname`, `authorId`, `channelId`, `guildId`, `embedTitle`, `embedDescription`, `jumpUrl` (direct link to the message — confirmed independently in §25.6's My Values example too), `attachmentUrl`, `repliedContent` (the text of the message being replied to, read directly off `message` — an alternative to fetching `message_replied_to` as its own reference when you only need its text), `componentText`, `buttonCount`, `componentOutline`.

**`user`** (the general Discord account — §25.2's user-vs-member distinction):
`mention`, `id`, `name`, `displayName`, `globalName` (Discord's newer global display name, distinct from a per-server nickname), `avatarUrl`, `createdAt`.

**`member`** (that user, inside one specific server):
`nick`, `guildName`, `guildId`, `avatarUrl`, `topRole`, `status`, `joinedAt`.

**`channel`** (general — works in servers, DMs, group DMs): `id`, `mention` — list was still scrolling in the source screenshot, likely incomplete.

**`server_channel`** (server-only, adds server-specific fields per §25.2): only `name` was visible before the screenshot moved on — treat as a floor, not a full list.

**`guild`** (`= "server"`, see §25.2):
`id`, `description`, `memberCount`, `onlineCount`, `textChannelCount`, `vanity`, `iconUrl`, `ownerId`, `vanityCode`, `vanityInviteUrl`.

**`global`** (system/meta — no Discord parameter needed, mirrors the `global`-parent conditions/actions in §25.7/25.10):
`nighty_data_path`, `pcUsername`, `pcName`, `activeWindow` (same underlying data as the `activeWindowCheck` condition, §25.7 — but readable as a plain value here, not just checkable), `clipboard`, `cpuPercent`, `gpuPercent`, `ramPercent`, `ramUsedGb`, `ramTotalGb`, `diskFreeGb`, `diskFreePercent`, `batteryPercent`, `batteryPlugged`, `uptimeSeconds`, `idleSeconds` (same underlying data as `pcIdleCheck`, §25.7, readable directly here).

**`me`** (your own account, always present on every event):
`name`, `displayName`, `mention`, `avatarUrl`, `status`, `friendsAmount`, `serversAmount`.

**This corrects a claim in §25.14**: hardware/system introspection is **not** actually missing from the catalog — `global`→`cpuPercent`/`gpuPercent`/`ramPercent`/`ramUsedGb`/`ramTotalGb`/`diskFreeGb`/`diskFreePercent`/`batteryPercent`/`batteryPlugged`/`uptimeSeconds` cover most of what a "hardware specs reporter" tool would need to *display*, even though there's still no single action that formats them into one report the way a NightyScript function might. Read §25.14's hardware-value row with this correction in mind rather than as still-accurate.
