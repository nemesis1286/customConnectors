# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

Custom Microsoft Sentinel data connectors, each packaged as a single self-contained ARM deployment template at the repository root. There is no application code, build system, package manager, test suite, or CI pipeline — the JSON templates are the entire deliverable.

The only connector today is **`ClaudeActivities.json`**: the "Anthropic Claude Activities" Sentinel solution (currently v3.0.2, MIT-licensed, Community tier). It ingests the Anthropic Claude Compliance API Activity Feed (authentication, chat, file, project, administrative, and platform events) into Sentinel using the **Codeless Connector Framework (CCF)** — no Function App or custom code; Sentinel's managed `RestApiPoller` does the polling.

New connectors should follow the same pattern: one self-contained ARM template per connector at the repo root.

## How the connector works (data flow)

1. Sentinel's `RestApiPoller` calls `GET https://api.anthropic.com/v1/compliance/activities` on 5-minute windows (`queryWindowInMin: 5`), authenticating with the customer's Compliance API key in the `x-api-key` header.
2. Time-windowing uses the `created_at.gte` / `created_at.lt` query parameters; pagination is `NextPageToken` style (`$.last_id` fed back as `after_id`, gated by `$.has_more`, `limit=1000`). Events are extracted from `$.data[*]`.
3. Events flow through the Data Collection Endpoint (DCE) into the DCR `DCR-AnthropicClaudeActivities`, whose `transformKql` maps the raw snake_case API fields to PascalCase columns (`TimeGenerated` comes from `created_at`, falling back to `now()` when null).
4. Rows land in the custom table **`ClaudeActivity_CL`** (90-day retention).
5. The KQL parser function **`ClaudeActivity`** flattens the `Actor` dynamic column into nine `Actor*` scalar fields (type, email, user id, IP, user agent, API key ids, service account id, unauthenticated email) at query time. Queries should use the parser, not the raw table.

Operational characteristics documented in the connector UI: delivery is at-least-once (de-duplicate on `Id`, e.g. `summarize arg_min(TimeGenerated, *) by Id`), polling starts at connect time with no historical backfill, and a late-indexed event can occasionally fall outside its poll window.

## Anatomy of the template

`ClaudeActivities.json` is one ARM template whose `resources` array mixes **live resources** (deployed immediately) with **stored content** (CCF `contentTemplates` whose nested `mainTemplate`s Sentinel deploys later). In file order:

| # | Resource | Role |
|---|----------|------|
| 1 | `Microsoft.Insights/dataCollectionEndpoints` (named after the workspace) | Live DCE so the connector stands up in a fresh tenant; idempotent where one already exists |
| 2 | `workspaces/tables` → `ClaudeActivity_CL` | Live table created at deploy time — prevents the `InvalidOutputTable` race when Connect instantiates the DCR |
| 3 | `workspaces/savedSearches` → `ClaudeActivity` | Live parser function (direct-deploy path) |
| 4 | `contentTemplates` (kind `DataConnector`) | Stored template: connector-definition UI + DCR + a second copy of the table + metadata |
| 5 | `dataConnectorDefinitions` (kind `Customizable`) | Live copy of the connector UI (direct-deploy path) |
| 6 | `metadata` (DataConnector) | Live metadata linking the definition to the solution |
| 7 | `contentTemplates` (kind `ResourcesDataConnector`) | Stored template: the `RestApiPoller` dataConnector, deployed when the user clicks **Connect** |
| 8 | `contentTemplates` (kind `Parser`) | Stored copy of the parser for Content Hub |
| 9 | `contentPackages` (Solution) | Content Hub package metadata; every `contentTemplates` resource `dependsOn` it |

Deployment parameters: `workspace` (Log Analytics workspace with Sentinel enabled) and `location` (defaults to the resource group's location, which must match the workspace region).

## Critical conventions — read before editing

### 1. Duplicated content must stay in sync

The same content deliberately exists twice — once as a live resource for direct deployment, once as stored Content Hub/Connect content. When you change one copy, make the identical change in the other:

- **Table schema** — top-level `ClaudeActivity_CL` resource ⟷ the table copy inside the DataConnector `contentTemplate`'s `mainTemplate` (a comment in the file mandates they stay identical)
- **Connector UI config** (`connectorUiConfig`: description, sample queries, instruction steps, permissions) — live `dataConnectorDefinitions` resource ⟷ the copy inside the DataConnector `contentTemplate`
- **Parser query** — live `savedSearches` resource ⟷ the Parser `contentTemplate` copy
- **DataConnector metadata** — live `metadata` resource ⟷ the copy inside the DataConnector `contentTemplate`

### 2. Bracket escaping inside nested templates

Inside a nested `mainTemplate`, a single-bracket expression (`"[parameters('workspace')]"`) evaluates when *this repo's template* deploys, baking the value into the stored content. A double-bracket expression (`"[[parameters('apiKey')]"`) is escaped and evaluates later, when Sentinel deploys the stored template (at Connect time). Both are used intentionally: the workspace name is baked in with `[`, while the API key and the wizard-supplied `dcrConfig` use `[[`. Getting this wrong either bakes in a wrong/empty value or breaks the inner deployment.

### 3. Versioning

- `_solutionVersion` (`3.0.2`) — semver for the Content Hub package; bump on any released change.
- `dataConnectorVersionConnectorDefinition` / `dataConnectorVersionConnections` — date-stamped strings in the form `1.0.<yyyyMMdd><seq>` (e.g. `1.0.20260812000001`). Bump the one whose `contentTemplate` changed; the version is concatenated into the contentTemplate resource name, so a bump produces a new template version in the workspace.
- `parserVersion1` (`1.0.0`) — bump when the parser query changes.
- Keep the dependency `criteria` blocks (Solution → DataConnector + Parser; connector definition → Connections) pointing at the matching `contentId`/`version` pairs — a transposed contentId/version here has broken deployment before.

### 4. Coupled names

Renames fan out; find every occurrence before changing any of these:

- Stream `Custom-ClaudeActivity_CL` — DCR `streamDeclarations`, DCR `dataFlows` (input stream and `outputStream`), and the poller's `dcrConfig.streamName`
- Table `ClaudeActivity_CL` — both table resources, the parser query, `graphQueriesTableName`, `dataTypes`, and `lastDataReceivedQuery`
- DCE name = workspace name — the DCR's `dataCollectionEndpointId` is built by string concatenation from the workspace parameter and assumes the DCE from resource #1

### 5. Adding or changing an ingested field

Touch all of these together: DCR `streamDeclarations` (the raw snake_case shape returned by the API), DCR `transformKql` (mapping to the PascalCase column), **both** table schemas, and the parser if the field needs flattening or belongs in `project-reorder`. Invalid schema properties or wrong apiVersions on the table/DCR have caused `InvalidOutputTable` failures before — keep `workspaces/tables` at apiVersion `2022-10-01` and the DCR/DCE at `2022-06-01` unless deliberately upgrading.

## Development workflow

**Validate JSON syntax after every edit** (there is no other automated check):

```bash
python3 -m json.tool ClaudeActivities.json > /dev/null && echo OK
```

**Validate or deploy against Azure** (requires the az CLI and a Sentinel-enabled Log Analytics workspace; the resource group must be in the same region as the workspace):

```bash
az deployment group validate -g <rg> --template-file ClaudeActivities.json --parameters workspace=<workspace-name>
az deployment group create   -g <rg> --template-file ClaudeActivities.json --parameters workspace=<workspace-name>
```

Do not run `create` against any real Azure environment without explicit human approval — `validate` is safe, deployment changes infrastructure.

After deployment the connector appears in Sentinel under **Data connectors → Anthropic Claude Activities**. Entering a Compliance API key and clicking **Connect** deploys the stored `RestApiPoller` template. End-to-end verification is manual: confirm rows arrive in `ClaudeActivity_CL` and that the `ClaudeActivity` function returns flattened `Actor*` columns.

## Git conventions

- Default branch: `main`. Feature work happens on `claude/<topic>-<suffix>` branches merged via pull request.
- Commit messages are imperative one-liners describing the specific fix (e.g. "Fix RestApiPoller pagination and time-window checkpoint config").

## Security notes

- Never commit real API keys. The `sk-ant-api01-...` / `sk-ant-admin01-...` strings in the template are placeholder format hints in UI copy only; the actual key is supplied by the customer in the portal and handled as a `securestring` parameter.
- The Compliance API requires Claude Enterprise and a key carrying the `read:compliance_activities` scope.
