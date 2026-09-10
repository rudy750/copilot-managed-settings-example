# GitHub Copilot enterprise managed settings example

This repository is a public, non-production example of GitHub Copilot
enterprise managed settings. It demonstrates:

- A baseline `managed-settings.json` policy.
- MCP server allow and deny lists.
- Different MCP policies for enterprise teams.
- Additional settings for permissions, models, plugins, remote control,
  telemetry, and the Copilot CLI sandbox.

The values use fictitious organizations, domains, repositories, and team slugs.
Replace every `contoso` value before using this configuration.

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
| `copilot/managed-settings.json` | Defines the enterprise-wide defaults and marks selected keys as overridable by enterprise teams. |
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
      { "serverUrl": "https://mcp.contoso.example/shared/*" }
    ]
  }
}
```

A mapped team file uses the normal value syntax:

```json
{
  "allowedMcpServers": [
    { "serverUrl": "https://mcp.contoso.example/shared/*" },
    { "serverUrl": "https://mcp.contoso.example/development/*" }
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

## Additional managed settings

The baseline demonstrates several settings beyond MCP governance.

| Setting | What it does |
| --- | --- |
| `model` | Sets the default model for new conversations. `"auto"` enables automatic model selection. Users can still choose another allowed model for an individual conversation. |
| `permissions.disableBypassPermissionsMode` | Setting this to `"disable"` prevents users from enabling allow-all or YOLO-style permission bypass. |
| `permissions.deny` | Blocks matching shell commands, file operations, or domains. Deny has the highest permission-rule precedence. |
| `permissions.ask` | Requires fresh human approval every time a matching operation is requested. |
| `permissions.allow` | Allows matching operations without a prompt. If managed rules or an allowlist exist, unmatched supported operations require approval. |
| `enabledPlugins` | Requires a plugin to be enabled with `true`, or disabled with `false`, using `PLUGIN@MARKETPLACE` keys. |
| `extraKnownMarketplaces` | Adds enterprise-approved plugin marketplaces. `autoUpdate` controls whether clients must refresh and update plugins from that marketplace. |
| `strictKnownMarketplaces` | Restricts plugin installation to listed marketplaces. An empty array locks plugin installation down completely. |
| `telemetry` | Configures OpenTelemetry export. Keep `captureContent` disabled unless the enterprise has explicitly approved collection of prompts and responses. |
| `remoteControl` | Controls whether sessions hosted on a device can be remotely controlled. `requireSSO` limits control to clients authorized for listed GitHub organizations. |
| `sandbox` | Enforces minimum Copilot CLI sandbox restrictions for command execution, local MCP servers, language servers, files, network access, and credentials. |

Permission selectors in the example have these meanings:

| Selector | Matches |
| --- | --- |
| `Shell(...)` | Shell commands. A command followed by ` *` matches that command prefix. |
| `Read(...)` | File reads. `/` means the workspace root, `~/` the home directory, and `//` the filesystem root. |
| `Edit(...)` | File writes, using the same path roots and glob behavior as `Read`. |
| `Domain(...)` | Network origins. `*.example.com` matches the domain and its subdomains. |

Permission precedence is `deny` > `ask` > `allow`.

## Settings intentionally not enabled

The example defines an approved plugin marketplace but does not automatically
enable a plugin. Add entries to `enabledPlugins` only after validating the
plugin name and marketplace.

Telemetry export is disabled. To enable it, provide a real OTLP endpoint and
choose either `http/json` or `http/protobuf`. Do not commit collector
credentials to a repository; distribute sensitive headers through an approved
configuration mechanism.

The sandbox policy does not include custom filesystem grant paths. Managed path
lists compare exact strings with user-configured grants, so copying placeholder
paths can unintentionally break development tools.

## Deployment checklist

1. Create an organization-owned `.github-private` repository and select it as
   the enterprise source of client governance.
2. Copy the `copilot/` directory into that repository.
3. Replace all fictitious domains, organization names, repositories, commands,
   and enterprise team slugs.
4. Verify each setting is supported by every Copilot client used in the
   enterprise.
5. Review local MCP commands as exact arrays; package versions and optional
   arguments are part of the identity.
6. Protect changes to `copilot/managed-settings.json`,
   `copilot/team-mappings.json`, and `copilot/teams/` with branch protection and
   administrator or AI-manager review.
7. Test with a small enterprise team before broad rollout.
8. Allow up to an hour for supported clients to refresh, or restart the client
   or sign in again to trigger an immediate refresh.

## References

- [Enterprise managed settings reference](https://docs.github.com/en/enterprise-cloud@latest/copilot/reference/enterprise-administrators/enterprise-managed-settings)
- [Getting started with enterprise-managed settings](https://docs.github.com/en/enterprise-cloud@latest/copilot/how-tos/administer-copilot/manage-for-enterprise/use-managed-settings/get-started)
- [Overriding enterprise-managed settings for teams](https://docs.github.com/en/enterprise-cloud@latest/copilot/how-tos/administer-copilot/manage-for-enterprise/use-managed-settings/override-settings-for-teams)

GitHub may add settings or change client support over time. Check the current
reference before deploying this sample.
