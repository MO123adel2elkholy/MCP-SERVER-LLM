# MCP Server and LLM Agent

A small Python project for exploring OpenAI-compatible chat models, tool calling, agent loops, persistent memory, and the Model Context Protocol (MCP). It includes a command-line assistant, two local MCP servers, and several earlier agent examples.

## Project layout

```text
src/
├── agent/
│   ├── harness.py                         # Main interactive assistant
│   ├── mcp_servers.json                   # MCP server launch configuration
│   ├── memory.md                          # Persistent notes used by the harness
│   ├── workspace/                         # Files the assistant can manage
│   └── ...                                 # Earlier agent experiments
└── mcp_server/
    ├── hello_mcp.py                       # Username selection MCP server
    ├── database_mcp_server.py             # Read-only SQLite MCP server
    └── db/
        ├── db.sqlite3                     # Bundled database
        └── community.db                   # Additional database file
```

### Main components

- **`src/agent/harness.py`** is the most complete assistant. It keeps conversation history, loads the contents of `memory.md` into its system prompt, supports model switching, exposes local file and memory tools, and starts each server configured in `mcp_servers.json`.
- **Earlier agent scripts** show the project’s progression: a basic model call, one-shot tool calling, a repeated tool-calling loop, MCP integration, and file-backed memory. They are standalone examples and are not required to run the main harness.
- **`src/mcp_server/hello_mcp.py`** provides `get_username_random` and `list_available_usernames`.
- **`src/mcp_server/database_mcp_server.py`** provides `list_tables` and `read_query`. Database connections are opened read-only; queries are limited to a single `SELECT` or `WITH` statement and results are capped.

## Requirements

- Python 3.12.3 or newer
- [uv](https://docs.astral.sh/uv/) for dependency and environment management
- [Ollama](https://ollama.com/) for the default local models, or credentials for a configured hosted provider
- `uvx` and network access if you want the configured Time and Fetch MCP servers; `uvx` is provided by uv

## Setup

From the project root:

```powershell
uv sync
ollama pull llama3.2:latest
```

The harness defaults to Ollama at `http://localhost:11434/v1`. Start Ollama before launching the assistant. The `local-small` option uses `qwen3.5:2b`; pull it with `ollama pull qwen3.5:2b` if you want to use that model.

### Configure MCP server paths

The checked-in `src/agent/mcp_servers.json` contains absolute Windows paths into this checkout’s `.venv`. Update the `username-local` and `database-local` `command`, `args`, and, if needed, `cwd` and `PYTHONPATH` values to match your machine before running the harness. The `time` and `fetch` entries are launched with `uvx` and do not use those local paths.

The harness reads this file relative to `src/agent/harness.py`, so the config can be adapted for another operating system without changing the Python code.

## Run the interactive harness

From the project root:

```powershell
uv run python src/agent/harness.py
```

At startup, the harness tries to connect to every configured MCP server and prints which tools are available. A server that fails to start is reported and skipped; the remaining tools can still be used.

Available slash commands:

| Command | Purpose |
| --- | --- |
| `/models` | Show configured models |
| `/model <name>` | Switch model (for example, `/model local-small`) |
| `/tools` | Show local and connected MCP tools |
| `/memory` | Print saved memory |
| `/quit` | Exit |

### Configured models

The harness defines these model names in `MODELS` in `src/agent/harness.py`:

| Name | Endpoint | Model |
| --- | --- | --- |
| `local` | Ollama at `localhost:11434/v1` | `llama3.2:latest` |
| `local-small` | Ollama at `localhost:11434/v1` | `qwen3.5:2b` |
| `gpt` | OpenAI API | `gpt-5.2` |
| `claude` | Anthropic OpenAI-compatible endpoint | `claude-sonnet-5` |

Hosted options read `OPENAI_API_KEY` or `ANTHROPIC_API_KEY` from the environment when the module loads. Set the relevant variable before starting the harness. Provider model names and endpoint compatibility may need updating in `harness.py` as providers change.

## Run an MCP server directly

Each local server uses stdio transport and can be launched from the repository root:

```powershell
$env:PYTHONPATH = "src"
uv run python src/mcp_server/hello_mcp.py
```

Or:

```powershell
$env:PYTHONPATH = "src"
uv run python src/mcp_server/database_mcp_server.py
```

These processes wait for an MCP client on standard input/output. To use them interactively through the agent, configure them in `src/agent/mcp_servers.json` and launch the harness.

## Local tools and data

The harness provides these local tools to the model:

- `list_files`: list entries in `src/agent/workspace/`
- `read_file(filename)`: read a workspace file
- `write_file(filename, content)`: create or overwrite a workspace file
- `save_memory(fact)`: append a bullet to `src/agent/memory.md`

The bundled database server provides:

- `list_tables()`: list user tables and their columns
- `read_query(sql, parameters, limit)`: run a parameterized read-only query. Use SQLite `?` placeholders. The default limit is 100 rows and the maximum is 1000.

The username server provides:

- `get_username_random(user_list=None)`: select a username from a supplied list or the built-in list
- `list_available_usernames()`: show the built-in usernames

## Earlier examples

The scripts under `src/agent/` can also be run individually from the repository root. They are learning examples and generally use the local Ollama model:

- `basic_local_llm_calling.py`: plain chat completion
- `LLm_Tool_Basic_tool_calling.py`: one tool-call round
- `llm_too_calling_with_agentic_loop.py`: repeat tool calls until the model responds
- `agent_MCP_Tool_calling_with_agentic_loop.py`: MCP-connected agent example
- `agent_tool_cooling_with_presistant_memory.py`: conversation history and file-backed memory

For example:

```powershell
uv run python src/agent/basic_local_llm_calling.py
```

## Notes and limitations

- The interactive harness has no automated test suite documented in this repository.
- The workspace file tools join the supplied filename to the workspace directory; treat this as a local learning project and only run it with trusted model endpoints and inputs.
- MCP servers are configured as local subprocesses. Their commands, paths, and dependencies must be available on the machine running the harness.
- `pyproject.toml` declares a `mcp-server` console script targeting `mcp_server:main`, but the package currently does not define that `main` function. Use the Python module commands above to start the servers.

