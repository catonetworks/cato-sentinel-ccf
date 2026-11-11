# Cato ASIM parsers for Microsoft Sentinel — preview

Deploy the preview parsers as workspace saved searches to normalize Cato events to Microsoft's Advanced Security Information Model (ASIM). The preview includes **6 schemas and 12 functions** (Catalog version 2.0.1; Microsoft schema version 1.0.0). Mapping review is ongoing; this is not a certification or a Content Hub release.

Choose the template for your table:

- **Existing custom table with a supported name:** [![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fcatonetworks%2Fcato-sentinel-ccf%2F7aa153c0ef56f28a39dba54c290a8a2925abb31f%2Fasim%2FarmTemplate.json)
- **Standard `CatoNetworksEvents_CL` table:** [![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fcatonetworks%2Fcato-sentinel-ccf%2F7aa153c0ef56f28a39dba54c290a8a2925abb31f%2Fasim%2FccfTemplate.json)

## Deploy in Azure portal

1. Select the subscription and resource group containing your existing Log Analytics workspace.
2. Enter `workspaceName`. The custom-table ARM template also asks for `tableName`; use the exact existing Cato custom-table name. The table must already exist and match ASCII `[A-Za-z][A-Za-z0-9_]*_CL` (first character an ASCII letter, remaining characters ASCII letters, digits, or underscores, ending `_CL`). Invalid names fail deployment; names are never sanitized. The CCF template takes only `workspaceName` and uses the fixed `CatoNetworksEvents_CL` table.
3. Review and create the deployment.

Both templates create workspace saved searches only. Your deploying identity needs normal ARM deployment permissions and `Microsoft.OperationalInsights/workspaces/savedSearches/write`; inspection and querying additionally require the corresponding read/query permissions. The CCF template uses aliases suffixed for `CatoNetworksEvents_CL`; the custom-table template uses aliases suffixed for your selected table. Both are built from the same canonical mappings.

## Deploy with Azure CLI

Set these shell variables to your own values; for the standard CCF template, set `TABLE_NAME` to `CatoNetworksEvents_CL` (the template itself has no table parameter):

```bash
SUBSCRIPTION_ID='<subscription-id>'
RESOURCE_GROUP='<resource-group>'
WORKSPACE_NAME='<workspace-name>'
TABLE_NAME='<existing-table-name>'
TEMPLATE_URI='https://raw.githubusercontent.com/catonetworks/cato-sentinel-ccf/7aa153c0ef56f28a39dba54c290a8a2925abb31f/asim/armTemplate.json'
```

Preview with what-if, then deploy incrementally:

```bash
az deployment group what-if --subscription "$SUBSCRIPTION_ID" --resource-group "$RESOURCE_GROUP" --template-uri "$TEMPLATE_URI" --parameters workspaceName="$WORKSPACE_NAME" tableName="$TABLE_NAME"
az deployment group create --subscription "$SUBSCRIPTION_ID" --resource-group "$RESOURCE_GROUP" --mode Incremental --template-uri "$TEMPLATE_URI" --parameters workspaceName="$WORKSPACE_NAME" tableName="$TABLE_NAME"
```

For `ccfTemplate.json`, set `TEMPLATE_URI` to its pinned raw URL and omit `tableName="$TABLE_NAME"` from both commands. Use **Incremental** mode only, never Complete.

## Query and integrate

Functions are saved searches, not ingestion changes. For a custom table named `MyCatoEvents_CL`, invoke a source function directly, for example:

```kusto
vimNetworkSessionCato_MyCatoEvents_CL(starttime=ago(1d))
```

For the standard table and CCF template:

```kusto
vimNetworkSessionCato_CatoNetworksEvents_CL(starttime=ago(1d))
```

To include Cato in a shared ASIM unifier, add its source-function invocation to the applicable unifier using the documented extension pattern. Merge without replacing existing sources; do not overwrite a built-in or other source's definition. A query can return zero rows when there are no matching events, the selected time range is empty, or fields/values do not match the parser. Zero rows alone do not indicate an installation failure.

Dashboards need events in the chosen table, required fields populated by those events, and a time range containing matching data. Empty results or panels can be valid when those conditions are not met.

## Catalog 2.0.1 mapping scope

| Schema | Cato EVENTS selected |
| --- | --- |
| NetworkSession | Security / Internet Firewall, WAN Firewall, LAN Firewall; IDS projection for IPS, Suspicious Activity, Dynamic Prevention, DNS Protection |
| Dns | Security with exact `DNS` array membership in `app_stack`; a nonempty DNS query is required |
| WebSession | Security / Apps Security, or Security with exact `HTTP(S)` array membership in `app_stack`; a usable URL is required |
| Authentication | Connectivity / Authenticated |
| AlertEvent | Security / Anti Malware, SaaS Security API Anti Malware, SaaS Security API Data Protection, EPP Malware, Endpoint Protection, Data Security Alert, Data Discovery, Email Security Alert; an alert/event ID is required |
| AuditEvent | System / Audit with an operation and object |

Optional evidence does not further restrict these classifiers. Missing HTTP status is not Success; absent DNS replies do not invent requests. Generic `file_hash` is retained as raw Cato context, not assigned an algorithm from its length. FLOWS/appstats are outside this package.

Source epoch-millisecond timestamps are already mapped to native `TimeGenerated` by the Cato connector's DCR. Parsers use a retained original `time` when available, otherwise that mapped source clock, with `time_str` as an alternative when the primary clock is unavailable/invalid. Timestamp normalization and parser time filters use the same selected clock. Missing/null event counts become 1; malformed-present or invalid counts are excluded.

Version 2.0.1 corrects AuditEvent user-type normalization for native Azure Boolean columns: `is_admin` maps to `Admin`/`Regular` regardless of Boolean text casing. Classification and function aliases are unchanged. Existing customers apply the patch manually using the pinned installer.

### Upgrading the earlier eight-schema preview

This breaking preview revision removes the speculative Entra API routes. FileEvent is on hold; UserManagement is outside the reviewed baseline. Earlier coverage measurements are not representative of this catalog, and independent schemas may overlap.

Incremental deployment updates the twelve active functions but does **not delete** the four retired saved searches. Back up customizations and review dependent queries/workbooks before removing them. Remove only their Cato source registrations from custom unifier unions, preserving all other providers, then explicitly remove the retired aliases if no longer needed:

- `vimFileEventCato_<table>` and `ASimFileEventCato_<table>`
- `vimUserManagementCato_<table>` and `ASimUserManagementCato_<table>`

Replace `<table>` with the exact existing table name, including `_CL`. The installer never edits unifiers or deletes these functions; until explicit cleanup, an old function remaining visible does not mean it is supported by catalog 2.0.1.

## Scope, permissions, and rollback

These templates deploy saved searches only. They do **not** change ingestion, the Data Collection Rule, RBAC, credentials, retention, or the connector, and do not recreate or modify the table. The root [`updateTemplate.json`](../updateTemplate.json) is separate: it updates the ingestion table schema and Data Collection Rule; it is not required to deploy these parsers.

Existing ARM/CCF connector customers can deploy these templates manually; no Content Hub update is needed. This installer does not enable automatic updates or change existing update settings. Back up any customized managed functions before deployment. Keep local forks outside Cato's aliases to avoid collisions. Roll back by redeploying the prior approved commit's template with Incremental mode. Incremental deployment does not delete newer functions absent from that template; remove them only after reviewing dependent queries and unifier registrations. A later Content Hub parser release must document migration from manual installations; this preview does not publish or validate that solution upgrade.

Parser upgrades are manual for existing ARM and CCF installations: deploy the approved version from this public repository when you choose to update. The daily ingestion-schema reconciliation does not update parser functions, and its auto-update setting does not opt a connector into parser updates. Cato's runtime application does not need new saved-search permissions for this manual workflow; the authorized customer/admin deployment identity performs the update.

Customer deployment requires no private GitHub access or Python. When an approved parser release is updated, both artifacts are regenerated from one canonical source and the pinned buttons are updated; do not hand-edit either template.
