# Install and connect Awa Rails

This guide is for the agent receiving the onboarding prompt. Source: https://github.com/awafinance/awa-rails-skill.

## Install the skill

1. Identify the current agent/client and project. Inspect its existing skill and MCP configuration without displaying stored credentials. Use the client's supported project skill directory by default; use global installation only if the user requests it.
2. Retrieve `skills/awa-rails/` from the public repository, including `SKILL.md` and the entire `references/` directory. You can use a temporary Git checkout or download the files from GitHub. Inspect the instructions before installing; there is no installer script to execute. Preserve an existing installation and merge/update it deliberately rather than overwriting unrelated files.
3. Install the complete directory as `awa-rails` in the agent's supported skill location. Claude Code uses `.claude/skills/awa-rails/`; Codex uses `.agents/skills/awa-rails/`. For another client, verify its current skill support and location in that client's official docs. If it cannot install skills, load the instructions as context and state that limitation.
4. Read `SKILL.md`. Resolve references relative to the installed directory. Reload the agent if its skill discovery requires it, and verify the skill is discoverable; merely downloading one Markdown file is not a complete installation.

Optional, if the user already uses the skills CLI: `npx skills add awafinance/awa-rails-skill --skill awa-rails`. Select the actual agent; project installation is the default. Do not install a package manager just for this route.

## Obtain the API key

Open https://rails.awafinance.com/app/ (or the user's selected service), finish company onboarding, then use **Chaves de API**. Beta organizations may need manual approval. Store the once-shown key as `AWA_RAILS_API_KEY` in the client's secret settings or a local secret store. Ensure it is available to the actual MCP host process, including a desktop app launched outside the shell. Do not paste it into the prompt, source files, screenshots or terminal output.

## Configure the actual MCP client

Default endpoint: `https://rails.awafinance.com/mcp`. Use the same API key as REST. This server uses a fixed bearer token, not an OAuth login flow. Preserve unrelated servers and existing user configuration. Do not label one client's JSON format as universal.

### Codex

The CLI supports a named environment variable for bearer authentication:

```sh
codex mcp add awa-rails --url https://rails.awafinance.com/mcp --bearer-token-env-var AWA_RAILS_API_KEY
```

Equivalent TOML entry (merge into the appropriate existing config):

```toml
[mcp_servers.awa-rails]
url = "https://rails.awafinance.com/mcp"
bearer_token_env_var = "AWA_RAILS_API_KEY"
```

The variable must be available to the MCP host, not just a child terminal. Refer to https://developers.openai.com/codex/mcp for the current client behavior.

### Claude Code

Merge a project `.mcp.json` entry. The literal environment reference is expanded by Claude Code; the key itself stays out of the file:

```json
{
  "mcpServers": {
    "awa-rails": {
      "type": "http",
      "url": "https://rails.awafinance.com/mcp",
      "headers": { "Authorization": "Bearer ${AWA_RAILS_API_KEY}" }
    }
  }
}
```

Refer to https://code.claude.com/docs/en/mcp for environment expansion, scope and project trust requirements. Do not run shell expansion that writes the key into committed config.

### Cursor

Merge `.cursor/mcp.json` using Cursor's environment interpolation (different from Claude Code):

```json
{
  "mcpServers": {
    "awa-rails": {
      "url": "https://rails.awafinance.com/mcp",
      "headers": { "Authorization": "Bearer ${env:AWA_RAILS_API_KEY}" }
    }
  }
}
```

### Other clients

Use the client's remote HTTP MCP connection and supported secret/environment reference for the Authorization header. Verify its exact syntax in its official documentation (Cursor: https://cursor.com/docs/context/mcp). Some desktop clients require a supported connector or bridge; do not promise native remote HTTP support without checking. If the client lacks a secure credential route, describe that limitation and offer secret-backed REST access instead of inventing a config.

## Verify the connection without fiscal side effects

1. Initialize the MCP session with the host and discover tools via `tools/list`.
2. Call `get_transport_capabilities`, then `list_companies`. Resolve the company by CNPJ and report `nfse_emission`, `nfe_distribution`, `nfe_events` and `cert_check` separately.
3. `list_notes` and `list_emissions` can show existing documents. Do not emit a test invoice, send a manifestation, run `sync_notes` or call `test_certificate` solely to prove installation.
4. Inspect `isError` in a tool result even if HTTP succeeded. If tools are absent after installation, guide a reload/reconnect and report the actual state.

Then help with the user's authorized workflow using [operations.md](operations.md) and [webhooks.md](webhooks.md). Ask only for missing business facts, secret entry in the proper secure surface, or actions needing authorization that the user has not already supplied.
