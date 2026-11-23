# Oxynet skills

Teaches AI agents to analyse cardiopulmonary exercise tests (CPET) with
[Oxynet](https://www.oxynet.net): the agent hands Oxynet the vendor file, receives a handle,
and asks for the analyses by name. Oxynet's validated models compute the physiology. It also
documents Oxynet's MCP server, which GitHub Copilot in VS Code and any other MCP client can
connect to directly ([below](#the-oxynet-mcp-server)).

| Tool | What to install | How it reaches Oxynet |
|---|---|---|
| Claude Code, Claude chat, Cowork | The plugin in [`claude/`](claude/) | The Oxynet MCP server; you sign in once with your key |
| Gemini CLI | The extension (`gemini-extension.json`) | REST with curl, key in `OXYNET_API_KEY` |
| OpenAI Codex, Cursor, GitHub Copilot and others | The skill in [`skills/oxynet-cpet/`](skills/oxynet-cpet/SKILL.md) | REST with curl, key in `OXYNET_API_KEY` |
| VS Code with GitHub Copilot, or any MCP client | Nothing: [the MCP server](#the-oxynet-mcp-server) | Your key in the `X-API-Key` header, or sign in once |

You need an Oxynet API key. Get a free one at
[app.oxynet.net/signup](https://app.oxynet.net/signup): the page also gives you, for your
system, the one command that saves it as `OXYNET_API_KEY`.

## The Oxynet MCP server

Oxynet runs a remote MCP server, listed in the official MCP Registry as `net.oxynet/cpet`:

```
https://app.oxynet.net/oxynet-mcp        (streamable HTTP)
```

The server is a hosted service: its code is not in this repository, which holds what teaches
agents to use it. Any MCP client can connect, including VS Code with GitHub Copilot, Claude,
Gemini CLI and Cursor.

**Configure it.** Send your key in the `X-API-Key` header. In VS Code, add this to
`.vscode/mcp.json` (or run **MCP: Add Server**). VS Code asks for the key once and keeps it out
of the file:

```json
{
  "inputs": [
    {"type": "promptString", "id": "oxynet-key", "description": "Oxynet API key", "password": true}
  ],
  "servers": {
    "oxynet": {
      "type": "http",
      "url": "https://app.oxynet.net/oxynet-mcp",
      "headers": {"X-API-Key": "${input:oxynet-key}"}
    }
  }
}
```

A client that supports MCP sign-in can leave the header out: a browser opens and Oxynet's page
asks for the key once.

**Use it.** Ask in plain words, for example "what are the ventilatory thresholds in
`tests/athlete_03.xlsx`?" The agent uploads the file, receives a handle, and asks for the
analyses by name:

| Tool | What it does |
|---|---|
| `get_capabilities` | What this key may do. Call it first |
| `get_sample_cpet` | A handle to a synthetic recording, to try Oxynet without data of your own |
| `create_upload`, `upload_cpet` | Send a file: one-shot upload URLs, or the contents directly |
| `get_cpet` | Format, channels, sampling, load and quality flags of an uploaded recording |
| `assess_cpet` | Everything that applies to the recording, in one call |
| `analyze_vt` | Ventilatory thresholds VT1 and VT2 |
| `analyze_eov` | Exercise oscillatory ventilation (beta) |
| `analyze_substrate` | Fat and carbohydrate use, FATMAX |
| `list_metrics`, `compute_metrics` | Derived quantities (phases, gas quality, peaks) and what each needs |
| `get_cpet_series` | A downsampled slice of the signals, for plotting; not for analysis |
| `delete_cpet` | Delete a recording now, instead of at its 24-hour expiry |

## Install

**Claude Code**

```
/plugin marketplace add oxynet-cpet/oxynet-skills
/plugin install oxynet@oxynet
```

Then run `/mcp`, choose `oxynet` and sign in: Oxynet's page asks for your key once. No
environment variable is needed. If you already added Oxynet as a connector on claude.ai,
Claude Code uses that connector instead and lists no separate `oxynet` server: sign in
there (claude.ai, Settings, Connectors) and the plugin's skill uses its tools.

**Gemini CLI**

```
gemini extensions install https://github.com/oxynet-cpet/oxynet-skills
```

**Any tool that reads Agent Skills:** copy `skills/oxynet-cpet/` into that tool's skills
directory.

Then hand the agent a CPET export and ask for what you need: "what are the ventilatory
thresholds in `tests/athlete_03.xlsx`?" The agent asks you before each call that sends a file
to Oxynet. That is expected: it is the moment you decide that a file leaves your computer.

## What Oxynet does

Oxynet reads the raw export of a metabolic cart, detects the format, and returns
ventilatory thresholds (VT1 and VT2), substrate use (fat and carbohydrate oxidation,
FATMAX), exercise oscillatory ventilation and signal-quality checks as structured,
versioned JSON. The assistant routes the question and explains the answer; Oxynet's
validated models decide the physiology.

Research software, not a diagnostic device. Results carry no numeric confidence, because
nothing is calibrated against clinical outcomes; an answer that looks unusual is marked
low confidence, with the stated reason. Oscillatory ventilation analysis is beta:
it was developed on one heart failure cohort and its transportability is untested.

The agent sends the file you name and nothing else. Oxynet parses the bytes and discards
them, and deletes the parsed record after 24 hours. It needs no patient identifiers, so
anonymise a name or a date of birth before uploading.

## License

Apache License 2.0 ([LICENSE](LICENSE), [NOTICE](NOTICE)). Use, copy and adapt the skill
freely. If you redistribute it, modified or not, keep the attribution in [NOTICE](NOTICE):
section 4(d) of the license requires it. The license grants no right to the Oxynet name,
and using the Oxynet API requires a key and is subject to its own
[terms](https://www.oxynet.net/terms).

## Links

- Agent guide: https://app.oxynet.net/llms.txt
- Full agent guide: https://app.oxynet.net/llms-full.txt
- OpenAPI schema: https://app.oxynet.net/v1/openapi.json
- Developer docs: https://www.oxynet.net/developers
- MCP server, for assistants that prefer a connector: https://app.oxynet.net/oxynet-mcp
  (in claude.ai and Claude Desktop, Oxynet is in Claude's connector directory:
  https://claude.ai/directory/oxynet)
