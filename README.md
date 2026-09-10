**[English](README.md)** | [Español](README.es.md)

# GitHub Copilot enterprise managed settings example

This repository is a public, non-production example of GitHub Copilot
enterprise managed settings. It demonstrates:

- A baseline `managed-settings.json` policy that configures only MCP server
  allow and deny lists.
- Different MCP policies for enterprise teams.

The MCP examples use servers listed in the
[GitHub MCP Registry](https://github.com/mcp). Other sample values use fictitious
organizations, domains, repositories, and team slugs. Replace every `contoso`
value before using this configuration.

> [!IMPORTANT]
> A server-managed production configuration belongs in the `copilot/` directory
> of an organization-owned `.github-private` repository selected as the
> enterprise source of client governance. This public repository is only a
> reference implementation and does not apply settings to an enterprise.

## Repository structure

```text
copilot/
├── managed-settings.json
├── team-mappings.json
└── teams/
    ├── developers.json
    └── security-engineering.json
```

| File | Purpose |
| --- | --- |
| `copilot/managed-settings.json` | Defines the enterprise-wide MCP allow and deny lists and marks them as overridable by enterprise teams. |
| `copilot/team-mappings.json` | Maps team settings files to enterprise team slugs. |
| `copilot/teams/developers.json` | Gives developer teams access to approved remote and local MCP servers. |
| `copilot/teams/security-engineering.json` | Gives security teams access to approved security MCP servers while blocking local MCP processes. |

## How team-specific settings work

Only server-managed deployments support enterprise team overrides.

The baseline file wraps team-overridable values in an `overridable` object:

```json
{
  "allowedMcpServers": {
    "overridable": [
      { "serverUrl": "https://mcp.context7.com/mcp" },
      { "serverUrl": "https://mcp.deepwiki.com/mcp" },
      { "serverCommand": ["uvx", "markitdown-mcp"] }
    ]
  }
}
```

Any entry in the baseline `overridable` list is a **default**: it is allowed
for every enterprise member unless a mapped team file defines its own
`allowedMcpServers` and omits that entry. Because `developers.json` and
`security-engineering.json` both define `allowedMcpServers`, they replace the
baseline list entirely for their members. Members of any other enterprise
team, or members not in a mapped team, receive the baseline defaults,
including servers that are not on either team's allowlist, such as DeepWiki
and MarkItDown.

A mapped team file uses the normal value syntax:

```json
{
  "allowedMcpServers": [
    { "serverUrl": "https://mcp.context7.com/mcp" },
    { "serverUrl": "https://learn.microsoft.com/api/mcp" }
  ]
}
```

`team-mappings.json` maps each settings filename to one or more enterprise team
slugs:

```json
{
  "developers.json": [
    "application-developers",
    "platform-engineering"
  ],
  "security-engineering.json": [
    "security-engineering"
  ]
}
```

If a user belongs to multiple mapped enterprise teams, GitHub combines the team
files using the least restrictive value for each key. Platform-level decisions
still take precedence.

## MCP allow and deny lists

### `allowedMcpServers`

When this key is present, only MCP servers matching an entry are allowed.
Omitting the key allows all servers, subject to `deniedMcpServers`. An empty
array blocks all servers except built-in default servers.

Each entry must contain exactly one matcher:

| Matcher | Use |
| --- | --- |
| `serverName` | Matches the user-assigned server name exactly. It is convenient but does not strongly identify a server. |
| `serverUrl` | Matches a remote HTTP or SSE MCP URL. It supports wildcards for subdomains or path prefixes. |
| `serverCommand` | Matches a local stdio server command and every argument exactly. Wildcards and shell expansion are not supported. |

Prefer `serverUrl` or `serverCommand` when the policy must identify the actual
server rather than a user-selected label.

### `deniedMcpServers`

A matching deny entry blocks a server even when the server also matches the
allowlist. Deny rules take precedence over allow rules. Built-in first-party
Copilot servers, including the built-in GitHub MCP server, cannot be blocked.

The team files in this repository repeat the baseline deny entries because they
override `deniedMcpServers`. This keeps the baseline restrictions visible and
intact while adding team-specific restrictions.

### Registry servers used in this example

| Server | Policy use |
| --- | --- |
| [Context7](https://github.com/mcp/upstash/context7) | Shared remote documentation server allowed by the baseline and both teams. |
| [Microsoft Learn](https://github.com/mcp/microsoftdocs/mcp) | Remote documentation server allowed for developer teams. |
| [Playwright](https://github.com/mcp/microsoft/playwright-mcp) | Local browser automation server allowed for developer teams and denied for security teams. |
| [Sentry](https://github.com/mcp/getsentry/sentry-mcp) | Remote application diagnostics server allowed for security teams. |
| [DeepWiki](https://github.com/mcp/cognitionai/deepwiki) | Remote repository documentation and Q&A server allowed as a baseline default, and not on either team's allowlist. |
| [MarkItDown](https://github.com/mcp/microsoft/markitdown) | Local document-to-Markdown conversion server allowed as a baseline default, and not on either team's allowlist. |

The policies identify the servers but do not configure credentials. Context7
and Sentry may require users to authenticate when they connect.

### Matching details

- `serverName` is an exact, case-sensitive label match and supports no
  wildcards.
- `serverUrl` applies only to remote HTTP or SSE servers.
- `serverCommand` applies only to local stdio servers and must match the entire
  command array exactly.
- URL matching is canonicalized by the client, including lowercasing the scheme
  and host and removing default ports.
- When multiple managed settings sources define an allowlist, the effective
  allowlist is their intersection.
- When multiple managed settings sources define a denylist, the effective
  denylist is their union.

## Deployment checklist

1. Create an organization-owned `.github-private` repository and select it as
   the enterprise source of client governance.
2. Copy the `copilot/` directory into that repository.
3. Replace all fictitious domains, organization names, repositories, commands,
   and enterprise team slugs.
4. Verify each setting is supported by every Copilot client used in the
   enterprise.
5. Review the selected registry servers and approve them for your environment.
6. Review local MCP commands as exact arrays; package versions and optional
   arguments are part of the identity. Consider pinning package versions instead
   of using `@latest` in production.
7. Protect changes to `copilot/managed-settings.json`,
   `copilot/team-mappings.json`, and `copilot/teams/` with branch protection and
   administrator or AI-manager review.
8. Test with a small enterprise team before broad rollout.
9. Allow up to an hour for supported clients to refresh, or restart the client
   or sign in again to trigger an immediate refresh.

## References

- [Enterprise managed settings reference](https://docs.github.com/en/enterprise-cloud@latest/copilot/reference/enterprise-administrators/enterprise-managed-settings)
- [Getting started with enterprise-managed settings](https://docs.github.com/en/enterprise-cloud@latest/copilot/how-tos/administer-copilot/manage-for-enterprise/use-managed-settings/get-started)
- [Overriding enterprise-managed settings for teams](https://docs.github.com/en/enterprise-cloud@latest/copilot/how-tos/administer-copilot/manage-for-enterprise/use-managed-settings/override-settings-for-teams)
- [GitHub MCP Registry](https://github.com/mcp)

GitHub may add settings or change client support over time. Check the current
reference before deploying this sample.
