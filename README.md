# Anthropic Claude Activities — Microsoft Sentinel CCF Connector

Ingests the [Claude Compliance API Activity Feed](https://platform.claude.com/docs/en/manage-claude/compliance-activity-feed)
into Microsoft Sentinel via the Codeless Connector Framework (CCF), landing in
`ClaudeActivity_CL`.

The Activity Feed records every authentication, chat, file, project, administrative, and
platform action in a Claude Enterprise organization. Activities are queryable within 1 minute
of occurring and are retained by Anthropic for 6 years.

## Prerequisites

- **Claude Enterprise** with the Compliance API enabled — see
  [Set up the Compliance API](https://platform.claude.com/docs/en/manage-claude/compliance-api-access).
- A key with the **`read:compliance_activities`** scope. Either a Compliance Access Key
  (`sk-ant-api01-…`) or an Admin API key (`sk-ant-admin01-…`) works.
- A Microsoft Sentinel-enabled Log Analytics workspace.
- **A Data Collection Endpoint named after the workspace, in the same resource group.** The DCR
  references `…/providers/Microsoft.Insights/dataCollectionEndpoints/<workspace-name>`. This is
  the same assumption the published Okta and Jira CCF solutions make — Content Hub provisions it,
  but a standalone `az deployment group create` will fail if it does not already exist.

## Repository layout

```
ClaudeActivities.json                     # single-file deployable ARM template
Solutions/AnthropicClaudeActivities/      # split sources for a Content Hub submission
  SolutionMetadata.json
  Data/Solution_AnthropicClaudeActivities.json
  Data Connectors/ClaudeActivitiesCCF/    # generated from ClaudeActivities.json
  Parsers/                                # ClaudeActivity normalizing function
  Analytic Rules/                         # 5 scheduled detections
  Hunting Queries/                        # 3 hunting queries
  Workbooks/
```

The files under `Data Connectors/ClaudeActivitiesCCF/` are generated from
`ClaudeActivities.json` so the two paths cannot drift. Regenerate them after editing the main
template rather than editing them by hand.

## Design decisions

### Cursor paging, not time-window polling

The connector uses `pagingType: PersistentToken` with `after_id` / `$.last_id` and
`order=asc`, and does **not** set `startTimeAttributeName` / `endTimeAttributeName`.

This is deliberate. Anthropic's docs are explicit about the failure mode of window polling:

> A `created_at.lt` bound too close to the present silently and permanently drops late-indexed
> activities: once `created_at.gte` advances past them, no later window can recover them.

CCF tiles its query window right up to the present, so a window-based configuration sits exactly
in that failure case. A persistent cursor has no such boundary — an activity that indexes late
is still ahead of the cursor and gets picked up on a later poll.

`order=asc` (oldest-first) with `after_id` walks steadily forward in time toward the present,
which is the incremental-sync direction Anthropic documents.

### Bounding the initial backfill

With no time window, the first poll has no cursor to resume from. The connector UI asks for an
**"Ingest activities from"** RFC 3339 timestamp, passed as a static `created_at.gte`. It only
ever bounds the *oldest* data — the cursor advances past it — so it stays correct indefinitely
while capping the first run. Set it recently for a fresh start, or further back to deliberately
backfill history.

### Rate limits and page size

The Compliance API allows **600 requests per minute per parent organization**, shared across
every key and every `/v1/compliance/*` endpoint. That is 10 QPS, which is what `rateLimitQPS`
is set to. `limit` is 1000 rather than the 5000 maximum, to keep individual responses inside
CCF's request timeout (capped server-side at roughly 3 minutes).

### Event time

`transformKql` sets `TimeGenerated = todatetime(created_at)`, so Sentinel's timeline reflects
when the activity actually happened. The raw value is preserved in `EventCreationTime`, and
`ingestion_time()` remains available for measuring poller lag — see the
*Claude - Connector ingestion health* hunting query.

### Schema fidelity

A DCR `streamDeclarations` block is a schema **filter**: any field not declared is discarded
before the transform runs and cannot be recovered. The stream therefore declares every field
documented on the Activity object, and the transform flattens all nine `actor` union members
(`user_actor`, `api_actor`, `admin_api_key_actor`, `service_account_actor`,
`unauthenticated_user_actor`, `scim_directory_sync_actor`, `federated_identity_actor`,
`system_actor`, `anthropic_actor`) into `Actor*` columns.

The raw `actor` object is also kept as a `dynamic` column, so **new actor types survive
automatically**. New *type-specific* top-level fields do not — DCR streams have no wildcard.
Re-check the [Activity object reference](https://platform.claude.com/docs/en/api/compliance/activities/list)
periodically and extend `streamDeclarations`, `transformKql`, and the table schema together.

### Deduplication

Anthropic documents the feed as **at-least-once** — a retry after a partial failure can
re-deliver activities. The `ClaudeActivity` parser deduplicates on `ActivityId`
(`summarize arg_min(TimeGenerated, *) by ActivityId`). Prefer it over querying
`ClaudeActivity_CL` directly in detections.

### Compliance API self-audit

Each poll emits a `compliance_api_accessed` activity, which the next poll ingests. These are
kept on purpose: Anthropic recommends ingesting them so the SIEM records who queried compliance
data, and they power the *Compliance API accessed by an unrecognized key* detection. The parser
filters them out by default so routine queries stay clean.

### SIEM correlation

Join on **`ActorUserId`**, not email. Anthropic documents it as a stable opaque identifier that
is consistent across every Compliance API endpoint and does not change when a user's email or
display name changes.

## Deploying

```bash
az deployment group create \
  --resource-group <rg> \
  --template-file ClaudeActivities.json \
  --parameters workspace=<workspace-name> workspace-location=<region>
```

Then open **Microsoft Sentinel → Data connectors → Anthropic Claude Activities**, enter the API
key and start date, and click **Connect**.

## Verifying a deployment

1. **Connector state** — the connector page should flip to *Connected*.
2. **Graph** — "Total data received" should be non-zero once data flows.
3. **Event time is real, not ingestion time:**
   ```kusto
   ClaudeActivity_CL
   | extend LagSeconds = datetime_diff('second', ingestion_time(), TimeGenerated)
   | summarize percentiles(LagSeconds, 50, 99)
   ```
   Lag should be minutes, and `TimeGenerated` must differ from `ingestion_time()`.
4. **Full actor coverage** — more than one row here means the union is being captured:
   ```kusto
   ClaudeActivity_CL | summarize count() by ActorType
   ```
5. **Type-specific fields survive:**
   ```kusto
   ClaudeActivity_CL
   | where isnotempty(FileName) or isnotempty(AdminApiKeyId) or isnotempty(RoleId)
   | take 10
   ```
6. **Cursor resumes rather than replaying** — choose a start date far enough back to force
   `has_more: true` across several pages, then confirm on the second polling cycle that the
   connector continues from the cursor instead of re-reading from the start date. Watch for a
   stall or a reset after the final page (`data: []`, `last_id: null`).
7. **Duplicate rate** — run the *Claude - Connector ingestion health* hunting query. A low,
   stable duplicate rate is expected; a spike means the cursor is resetting.

> **Known unverified detail.** `PersistentToken`'s field names and its cross-poll persistence
> semantics were taken from the Microsoft CCF reference, but no published connector using it was
> available to diff against while this was written. Two things to confirm on first deploy:
> whether `hasNextFlagJsonPath` is accepted alongside `PersistentToken` (remove it if the
> resource is rejected), and the behavior on the final empty page. If `PersistentToken` does not
> behave as documented, fall back to `startTimeAttributeName: created_at.gte` /
> `endTimeAttributeName: created_at.lt` with `queryWindowInMin: 5`, and accept the documented
> late-index drop risk.

## Before a Content Hub submission

- Add a real logo at `Logos/anthropic.svg` and update `_packageIcon` / `connectorUiConfig.logo`.
- Run `arm-ttk` against `ClaudeActivities.json`.
- Confirm `publisherId` / `offerId` in `SolutionMetadata.json` match your Partner Center offer.
- Flip `availability.isPreview` to `false` once the connector has been validated against a live
  tenant.

## References

- [Query the Activity Feed](https://platform.claude.com/docs/en/manage-claude/compliance-activity-feed)
- [Activity object / query reference](https://platform.claude.com/docs/en/api/compliance/activities/list)
- [Design your compliance integration](https://platform.claude.com/docs/en/manage-claude/compliance-integration-patterns)
- [Create a codeless connector for Microsoft Sentinel](https://learn.microsoft.com/en-us/azure/sentinel/isv/create-codeless-connector)
- [RestApiPoller connection rules reference](https://learn.microsoft.com/en-us/azure/sentinel/data-connector-connection-rules-reference)
