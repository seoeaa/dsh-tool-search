# dsh-tool-search

**English** | [Русский](README.ru.md)

Search and load tools on demand in DeepSeek Harness. Keep the full catalog on the server and send only the tools needed for the current work to the agent.

This fork extends [lordqyxz/dsh-tool-search](https://github.com/lordqyxz/dsh-tool-search) with a compact default configuration and compatibility with Harness **0.2.1-alpha.5**.

## How it works

1. The agent starts with `tool_search` and essential tools such as `skill` and `ask_user_question`.
2. Before choosing tools for a new task or capability, the agent searches by action and service name.
3. Search returns up to **3 matching tools**, including their parameter schemas.
4. The agent calls the selected tool by its original name through the normal Harness execution pipeline.
5. Up to **6 discovered tools** remain loaded per agent. Older discoveries yield to newer ones; search again to load them when needed.

The full catalog, including names and descriptions, stays out of the prompt by default. Search runs locally over the tools visible to the calling agent, including MCP tools, and needs no extra API key.

Calls to undiscovered deferred tools are denied with a hint to search first. Repeated use of an already loaded tool does not require another search. Validation, permissions, approvals, guards and timeouts still apply.

## Installation

Requires Harness **0.2.1-alpha.5**, Cordis **4.0.5**, and Node.js **22.19+ on the 22.x line, or 24+**. Other Harness versions have not been verified.

In the Harness plugin manager, install this package URL and enable the bundle:

```text
https://github.com/seoeaa/dsh-tool-search/releases/download/v0.1.1/dsh-tool-search-0.1.1.tgz
```

The release archive contains compiled JavaScript and type declarations. Do not enable another plugin that registers `tool_search` at the same time.

To develop or install from source:

```bash
git clone https://github.com/seoeaa/dsh-tool-search.git
cd dsh-tool-search
pnpm install --frozen-lockfile
pnpm run build
```

Install the resulting local directory through the Harness plugin manager. Hot application depends on the host's HMR support; check the manager's application result.

## Configuration

The bundled `cordis.patch.yml` enables this configuration:

```yaml
- insert:
    - id: tool-search
      name: dsh-tool-search
      config:
        threshold: 1
        maxResults: 3
        maxDiscovered: 6
        residentCatalog: false
        requireDiscovery: true
```

Use the plugin's configuration in Harness to change these values.

| Setting | Bundle value | Meaning |
| --- | --- | --- |
| `toolName` | `tool_search` | Discovery tool name. |
| `alwaysVisible` | `skill`, `ask_user_question`, `report`, `submit_*`, `structured_output*` | Patterns for tools available without discovery. |
| `threshold` | `1` | Minimum number of deferrable tools needed to activate filtering. |
| `maxResults` | `3` | Maximum full schemas returned by one search. |
| `maxDiscovered` | `6` | Maximum discovered tools kept loaded; never lower than `maxResults`. |
| `residentCatalog` | `false` | Include deferred names and short descriptions in the prompt when enabled. |
| `requireDiscovery` | `true` | Deny direct calls to undiscovered deferred tools. |
| `charactersPerToken` | `4` | Characters per token for schema cost estimates. |
| `descriptionChars` | `110` | Description limit in the optional resident catalog. |
| `deferToolGuidance` | `true` | Remove prompt sections dedicated to hidden tools. |

These are the effective **bundle settings**. Direct users of `apply()` retain the upstream fallback values: `threshold: 15`, `maxResults: 5`, `maxDiscovered: 20`, `residentCatalog: true`.

`alwaysVisible` replaces the default list; preserve the default patterns when extending it. The reserved `run_code` transport remains exempt from deferral.

## Search example

Search by capability:

```json
{"query":"read text file first lines","max_results":1}
```

If the result contains the built-in `read` tool, call it using the returned schema:

```json
{"file_path":"README.md","offset":1,"limit":20}
```

Use precise action words, a service name or an exact tool name. English keywords help when descriptions are in English. Search is lexical: rephrase a query when results do not match the intended capability.

## Changes from upstream

- Adapted prompt assembly to `AssembleContext.scope`, with a fallback for the earlier `agent` context shape.
- Added `residentCatalog: false` to keep names and descriptions on the server too.
- Configured bounded discovery by default; repeated search refreshes discovery order.
- Kept discovered schemas in the SDK for programmatic tool calling, without declaring native calls alongside `run_code`.
- Aligned dependencies and overrides with the verified host release.
- Added English and Russian documentation.

Discovery state is isolated per agent and can be restored from successful search results in session events. See [CHANGELOG.md](CHANGELOG.md).

## Verification and limits

The adaptation passed the build, type checking and **17 existing upstream checks**. A live native session completed `tool_search` → `read` → file content. Checks also covered separate state for two agents, bounded declarations, eviction, rediscovery and normal runtime guards after discovery.

In one local verification, the initial tool-schema JSON changed from **674 schemas / 1,230,070 bytes** to **3 schemas / 2,093 bytes**. After one search it contained **4 schemas / 2,711 bytes**. These are observed tool-section sizes, not a guarantee for every installation or a measurement of the whole request.

- A selected tool can still have a large schema: the cap limits tool count, not bytes.
- Earlier results and conversation history remain in existing sessions until ordinary compaction removes them.
- Native execution was verified. SDK assembly was checked for programmatic tool calling; full execution in that mode was not tested.
- A tool may be discoverable while its service, executable or credentials are unavailable.
- Disabling the bundle restores ordinary tool presentation; MCP registrations are preserved.

## Development

Use pnpm **11.19.0**. Keep the dependency release line and `pnpm-workspace.yaml` overrides aligned.

```bash
pnpm install --frozen-lockfile
pnpm run typecheck
pnpm run lint
pnpm run test
pnpm run build
pnpm run bench
```

Existing upstream tests are retained. The benchmark uses synthetic schemas and does not represent every real catalog.

## License and origin

[MIT](LICENSE). Original project: [lordqyxz/dsh-tool-search](https://github.com/lordqyxz/dsh-tool-search), based on commit `2a6362bfbb9f6ac17dede5ee5bba24068bf1f239`. Original copyright and license notices are preserved.
