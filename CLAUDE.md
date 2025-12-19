# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an MCP (Model Context Protocol) server that provides LLMs like Claude with tools to interact with the ESPN Fantasy Football API. The server wraps the `espn-api` Python library and exposes its functionality through MCP tools.

## Running the Server

The server is designed to be run by MCP clients like Claude Desktop. To test locally:

```bash
uv run espn_fantasy_server.py
```

## Architecture

### Single-File MCP Server
The entire server implementation is in `espn_fantasy_server.py`. It uses the FastMCP framework to define MCP tools.

### Core Components

1. **ESPNFantasyFootballAPI Class** (lines 29-72)
   - Manages ESPN league instances with caching
   - Handles per-session credential storage (ESPN_S2 and SWID cookies)
   - Uses cache keys that include authentication info to support both public and private leagues

2. **Session Management**
   - Uses a simple `SESSION_ID = "default_session"` approach
   - Credentials are stored in memory per-session in the `api.credentials` dict
   - League instances are cached with keys that include auth info: `{league_id}_{year}_{espn_s2}_{swid}`

3. **Football Year Logic** (lines 23-25)
   - Defaults to current calendar year
   - If before July, uses previous year (since NFL season spans calendar years)

### MCP Tools

The server exposes 8 tools (all async functions decorated with `@mcp.tool()`):

- `authenticate()` - Store ESPN credentials for accessing private leagues
- `get_league_info()` - Basic league information
- `get_team_roster()` - Player roster for a specific team
- `get_team_info()` - Team statistics and transaction info
- `get_player_stats()` - Stats for a specific player by name
- `get_league_standings()` - Current standings sorted by wins/points
- `get_matchup_info()` - Weekly matchup details
- `logout()` - Clear stored credentials

### Error Handling

All tools:
- Use try/except blocks with logging to stderr via `log_error()`
- Check for 401 errors or "Private" in exceptions to detect private league access issues
- Return user-friendly string messages (not JSON) since MCP converts them

### ESPN API Integration

- Uses the `espn_api.football.League` class from the espn-api package
- Team IDs are 1-based in the ESPN API (teams[0] is team_id=1)
- League objects are cached to avoid repeated API calls
- Private leagues require ESPN_S2 and SWID cookies (obtained from browser after logging into ESPN)

## Dependencies

Managed via `pyproject.toml` with uv:
- `espn-api>=0.44.1` - ESPN Fantasy Football API wrapper
- `mcp[cli]>=1.5.0` - Model Context Protocol framework

## Key Implementation Details

### Credential Storage
Credentials are stored per-session (not globally) to support multiple concurrent users in theory, though the current implementation uses a single default session ID.

### Logging
The server logs extensively to stderr using `log_error()` function. This is important for debugging in Claude Desktop, which captures stderr output.

### Error Recovery
The server initialization is wrapped in a try/except that keeps the process running even if initialization fails, allowing logs to be visible (lines 368-377).

## Testing Considerations

When testing tools:
- Use a public league first (doesn't require authentication)
- For private leagues, use the `authenticate()` tool first with valid ESPN_S2 and SWID cookies
- Team IDs start at 1, not 0
- Week numbers are typically 1-17 for most leagues
