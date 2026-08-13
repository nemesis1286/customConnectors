# Custom Connectors

Custom Microsoft Sentinel data connectors, each packaged as a single self-contained ARM deployment template at the repository root. There is no application code, build system, package manager, test suite, or CI pipeline — the JSON templates are the entire deliverable.

## Connectors

### Anthropic Claude Activities (`ClaudeActivities.json`)

Current version: **3.0.2** (MIT-licensed, Community tier)

Ingests the Anthropic Claude Compliance API Activity Feed (authentication, chat, file, project, administrative, and platform events) into Sentinel using the **Codeless Connector Framework (CCF)** — no Function App or custom code; Sentinel's managed `RestApiPoller` does the polling.

Requires **Claude Enterprise** and a Compliance API key carrying the `read:compliance_activities` scope.

#### Data flow

1. Sentinel's `RestApiPoller` calls `GET https://api.anthropic.com/v1/compliance/activities` on 5-minute windows, authenticating with the customer's Compliance API key in the `x-api-key` header.
2. Time-windowing uses the `created_at.gte` / `created_at.lt` query parameters; pagination follows a `NextPageToken` pattern (`$.last_id` fed back as `after_id`, gated by `$.has_more`, `limit=1000`). Events are extracted from `$.data[*]`.
3. Events flow through a Data Collection Endpoint (DCE) into the DCR `DCR-AnthropicClaudeActivities`, which maps the raw snake_case API fields to PascalCase columns (`TimeGenerated` comes from `created_at`, falling back to `now()` when null).
4. Rows land in the custom table **`ClaudeActivity_CL`** (90-day retention).
5. The KQL parser function **`ClaudeActivity`** flattens the `Actor` dynamic column into nine `Actor*` scalar fields (type, email, user id, IP, user agent, API key ids, service account id, unauthenticated email) at query time. Queries should use the parser, not the raw table.

Delivery is at-least-once — de-duplicate on `Id` (e.g. `summarize arg_min(TimeGenerated, *) by Id`). Polling starts at connect time with no historical backfill, and a late-indexed event can occasionally fall outside its poll window.

#### Deploying

```bash
# Validate JSON syntax
python3 -m json.tool ClaudeActivities.json > /dev/null && echo OK

# Validate against Azure (requires az CLI and a Sentinel-enabled Log Analytics workspace,
# with the resource group in the same region as the workspace)
az deployment group validate -g <rg> --template-file ClaudeActivities.json --parameters workspace=<workspace-name>

# Deploy
az deployment group create -g <rg> --template-file ClaudeActivities.json --parameters workspace=<workspace-name>
```

After deployment, the connector appears in Sentinel under **Data connectors → Anthropic Claude Activities**. Entering a Compliance API key and clicking **Connect** deploys the stored `RestApiPoller` template. Verify by confirming rows arrive in `ClaudeActivity_CL` and that the `ClaudeActivity` function returns flattened `Actor*` columns.

## Adding a new connector

New connectors should follow the same pattern: one self-contained ARM template per connector at the repo root, mixing live resources (deployed immediately) with stored CCF content (deployed later by Sentinel). See `CLAUDE.md` for the detailed conventions this repo follows — duplicated-content sync rules, bracket-escaping in nested templates, versioning scheme, and coupled naming — before editing or adding a template.

## Security notes

Never commit real API keys. Any `sk-ant-...`-style strings in a template are placeholder format hints in UI copy only; the actual key is supplied by the customer in the Sentinel portal and handled as a `securestring` parameter.

## License

MIT — see [LICENSE](LICENSE).
