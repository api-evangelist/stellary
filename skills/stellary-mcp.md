---
name: stellary-mcp
description: Use Stellary, the AI-native project piloting SaaS, through its hosted remote MCP. Discover projects, boards, cards, documents, cockpit state, and governed agent missions. Use when the user mentions Stellary, Stellary boards, cockpit/pilotage, agent missions, or connecting an assistant to live Stellary work.
license: MIT
homepage: https://stellary.co/docs/mcp/
compatibility: Requires a connected Stellary remote MCP (Streamable HTTP at https://api.stellary.co/mcp). OAuth is recommended; STELLARY_TOKEN remains available for compatible clients. No local npx/stdio server.
metadata:
  author: Stellary
  version: "0.13.0"
  registry: io.github.Anymfah/stellary-project-management
  openclaw:
    primaryEnv: STELLARY_TOKEN
    homepage: https://stellary.co/docs/mcp/
    envVars:
      - name: STELLARY_TOKEN
        required: true
        description: Stellary personal access token sent as Authorization Bearer to https://api.stellary.co/mcp when the remote MCP is not already connected.
---

# Stellary remote MCP

Stellary is an AI-native project piloting SaaS. This skill teaches an agent to use the **hosted** Model Context Protocol server. The application and server implementation are proprietary and are **not** distributed from this repository.

- Product site: https://stellary.co
- MCP docs: https://stellary.co/docs/mcp/
- Official registry id: `io.github.Anymfah/stellary-project-management`
- Endpoint: `https://api.stellary.co/mcp`
- Transport: Streamable HTTP (`GET`/`POST`, stateless; no `Mcp-Session-Id` required)
- Auth: OAuth 2.1 with PKCE and refresh-token rotation; personal access tokens remain available for compatible clients
- Not supported: local `npx`/stdio servers or Dockerized copies of the backend

If Stellary MCP tools are already available in this session, use them. Do not invent a local server or wrapper package.

## Connect when tools are missing

1. Configure the client with `https://api.stellary.co/mcp` and start its OAuth connection flow.
2. Ask the user to sign in to Stellary, choose a workspace, and authorize **Me**, one or more active agents, or both.
3. If several identities were authorized, call `stellary_list_identities` and pass the selected opaque reference in `actingAs`. Never invent or persist an identity reference beyond the connection that returned it.

Claude Code:

```bash
claude mcp add stellary \
  --transport streamable-http \
  https://api.stellary.co/mcp
```

Cursor / JSON clients:

```json
{
  "mcpServers": {
    "stellary": {
      "url": "https://api.stellary.co/mcp"
    }
  }
}
```

For a client that cannot complete remote OAuth, create a dedicated PAT from
**Account settings → API tokens**. Start with `projects:read` and
`pilotage:read`, add write scopes only when needed, and keep the secret in the
client's secret store. Gemini CLI uses `httpUrl` (not `url`) for Streamable HTTP:

```json
{
  "mcpServers": {
    "stellary": {
      "httpUrl": "https://api.stellary.co/mcp",
      "headers": {
        "Authorization": "Bearer ${STELLARY_TOKEN}"
      }
    }
  }
}
```

After connecting, call `list_projects` before any write. That confirms auth and project visibility without changing data.

## Identity and what each credential can do

| Credential | Identity | Use for |
| --- | --- | --- |
| OAuth connection | **Me**, one or more agents, or both | Recommended for Codex, ChatGPT, Claude, and other remote OAuth clients |
| Personal access token | Human user | Interactive board, document, and cockpit work in Cursor, Claude Code, Gemini CLI |
| User JWT | Human user | Short-lived browser-backed sessions |
| Agent token | Workspace agent | Queued missions, `stellary_init`, installed plugin tools (GitHub, Slack, and similar) |
| `MCP_TOKEN` | Transport gate only | Not enough for current Stellary tools that need a user or agent |

An OAuth connection never exposes the private token of an authorized agent. If
it contains one identity, `actingAs` is implicit. If it contains several,
`actingAs` is required for tool calls. Exact tool lists are the union available
to authorized identities, but every call is rechecked against the selected
identity's current status, project access, role, tool policy, autonomy, and
mission snapshot.

Human PATs can use core board and cockpit tools. Workspace-context tools
(missions, plugin integrations) require an agent identity. Agent tokens remain
supported for direct compatibility connections.

## Safe operating rules

- Prefer exact IDs after discovery. Name-based helpers exist but can fuzzy-match the wrong resource.
- Read first: `list_projects` → pick one project → inspect columns/cards/documents → then write.
- Do not bulk-create, reassign, or complete missions unless the user asked for that change.
- Treat tokens like passwords. Dedicated token per client, expiry date, revoke on leak.
- Revoke OAuth connections from **Workspace settings → MCP connections** when the client should no longer have access.
- All calls still obey Stellary permissions, autonomy policy, and rate limits.
- Product terms: https://stellary.co/terms/

## Core tool map

The live tool list comes from the server. Use these names when they appear; do not assume every name is present for every token.

**Board read:** `list_projects`, `get_project_details`, `list_cards`, `get_card_details`, `get_card_comments`

**Board write:** `create_card`, `create_cards_bulk`, `move_card`, `update_card`, `assign_card`, `add_comment`

**Cockpit:** `get_pilotage_state`, `get_cockpit_dashboard`, `get_agent_status`, `list_pending_proposals`

**Agent runtime (agent token):** `stellary_init`, `get_next_mission`, `wait_for_mission`, `complete_mission`, `fail_mission`

Installed workspace plugins register extra tools prefixed by plugin slug, for example `github_list_repos`, `github_create_pr`, `slack_post_message`. Those usually require an agent/workspace context and enabled plugin config.

## Recommended workflows

### Explore a workspace (human identity)

1. `list_projects`
2. `get_project_details` with the chosen project id
3. `list_cards` for the relevant columns
4. `get_card_details` / `get_card_comments` before commenting or moving work
5. Optionally `get_pilotage_state` or `get_cockpit_dashboard` for sprint and supervision context

### Change work (human identity, write scopes)

1. Confirm the card and column ids from read tools
2. Use `update_card`, `move_card`, `assign_card`, or `add_comment` with those ids
3. Re-read the card to confirm the result

### Run a queued mission (agent token)

1. `stellary_init` — autonomy mode, rules, skills, filtered tool list
2. `get_next_mission` or `wait_for_mission`
3. Read card and document context with exact ids
4. Execute only tools allowed by the returned policy
5. `complete_mission` or `fail_mission` (this updates mission state and posts a card comment)

Autonomy for **agent** tokens:

- `autonomous`: reads and writes execute
- `supervised`: reads execute; tools marked `approval_required` become persisted proposals
- `approval`: every non-read tool becomes a proposal

Human sessions act as the user and do not go through agent proposal checks. If a write returns a proposal instead of a mutation, tell the user and use `list_pending_proposals` rather than retrying blindly.

## Troubleshooting

- **401 during OAuth:** reconnect the MCP client; the access token may have expired or the connection may have been revoked.
- **401 with a PAT:** check the Bearer format, expiry, revocation, and selected scopes.
- **`actingAs` required:** call `stellary_list_identities`, choose one returned opaque reference, and retry once with `actingAs`.
- **Identity unavailable:** the agent may have been suspended, disabled, or removed from the grant; choose another authorized identity or reconnect.
- **Tool requires a user token:** the session is using `MCP_TOKEN` or an anonymous context. Switch to a PAT.
- **Tool requires a workspace context:** the call needs an agent token, not a human PAT.
- **Plugin tool visible but failing:** plugin not installed/enabled, or missing decrypted config.
- **Wrong resource:** stop using names; rediscover ids with `list_projects` / `list_cards`.

## What not to do

- Do not start a local Stellary MCP with `npx`, Docker, or a repo from this discovery tree.
- Do not commit `STELLARY_TOKEN` or paste a live token into files that will be shared.
- Do not use REST provisioning flows unless the user asked for API/setup work outside MCP.
- Do not claim tools that did not appear in the connected server's tool list.

For setup questions: support@stellary.co. For vulnerabilities: follow SECURITY.md and email security@stellary.co.
