# Cato Networks for Microsoft Sentinel — schema updates

This repository serves the schema update for the **Cato Networks Events (Push)** connector in
Microsoft Sentinel.

The solution in the Content Hub carries the event schema as it stood when that version was
published, and Cato adds event fields more often than the solution is republished. This template
brings the custom table and Data Collection Rule your connector uses up to the current schema.

**Apply it once after you install the connector, and again whenever you want newly added fields.**
Installing a newer version of the solution does not do this for you: the table and Data Collection
Rule were created by the **Deploy** button on the connector page and belong to your subscription,
so only this template changes them.

Nothing breaks if you skip it. Events keep arriving, and fields not in your table are simply not
stored.

## Apply the update

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fcatonetworks%2Fcato-sentinel-ccf%2Fmain%2FupdateTemplate.json)

1. Select the **subscription** and the **resource group** that hold your Sentinel workspace.
2. Enter the **workspace name**. That is the only value you supply.
3. Select **Review + create**.

The region field is Azure's standard deployment setting and does not affect the connector. Your
Data Collection Rule keeps the region it was created in.

## What changes, and what does not

| | |
|---|---|
| Custom table `CatoNetworksEvents_CL` | New columns added. Existing data and columns untouched. |
| Data Collection Rule | Stream and transformation updated to carry the new fields. |
| Ingestion URL, Data Collection Rule immutable ID, Data Collection Endpoint | **Unchanged** |
| Entra application and client secret | **Unchanged** |
| Table retention | **Unchanged**, whether workspace default or one you set |
| Data already ingested | **Unchanged** |

Nothing needs re-entering in the Cato Management Application, and ingestion continues while the
update is applied.

The schema only ever grows. Columns are added, never removed or retyped, so an update cannot break
a workbook, analytics rule or saved query built on the existing fields.

## After deploying

**Allow up to 30 minutes.** The ingestion endpoint caches the previous schema briefly. During that
window events are still accepted normally and the new fields are simply empty, so check with a
query before concluding anything is wrong:

```kusto
CatoNetworksEvents_CL
| where TimeGenerated > ago(1h)
| take 10
```

If new fields are still empty after an hour, contact Cato support.

## Event field reference

The fields in `CatoNetworksEvents_CL` mirror the Cato events feed. Two field names differ, because
Log Analytics reserves them:

| Cato field | Column in Sentinel |
|---|---|
| `time` | `TimeGenerated` |
| `title` | `activity_title` |

- **Field reference** — [Cato event field definitions](https://api.catonetworks.com/documentation/#definition-EventFieldName)
- **Connector setup and troubleshooting** — [Integrating Cato Events with Microsoft Sentinel](https://knowledge.catonetworks.com/docs/integrating-cato-events-with-microsoft-sentinel)
- **Knowledge Base** — [knowledge.catonetworks.com](https://knowledge.catonetworks.com/)

## Support

Contact [Cato Networks support](https://support.catonetworks.com/) with your account ID and the
name of your Log Analytics workspace.

---

`updateTemplate.json` is generated from the Cato event schema and published automatically. Do not
edit it by hand. Previous versions are kept under [`archive/`](archive) so support can identify
which schema a deployment is on.
