# Fabric-Vigil

One-click deployment of automated health monitoring for Microsoft Fabric Mirrored Databases.

Vigil deploys a notebook, pipeline, schedule, and Data Activator reflex into your workspace. Every 30 minutes (configurable), it checks the mirroring status of every table in every mirrored database. If any table is unhealthy, the pipeline fails and Data Activator sends an email alert with failure details.

## Potential Enhancements

Vigil is designed as a starting point. Some directions you could take it:

- **Historical logging**: Modify the monitoring notebook to write table-level mirroring stats (status, last sync time, rows replicated, etc.) to a Lakehouse or Warehouse on each run. Over time this builds a health timeline you can report on, trend, or use for SLA tracking.
- **Granular alerting**: Extend the failure classification logic to route different failure types (gateway vs. table-level) to different recipients or channels.
- **Multi-workspace monitoring**: Run a single deployment notebook across multiple workspaces by parameterizing the workspace ID list.
- **Teams/Slack integration**: Replace or supplement the email alert with a webhook-based notification to a Teams channel or Slack workspace.

## What Gets Deployed

| Item | Name | Purpose |
|------|------|---------|
| Notebook | `nb_vigil_mirror_monitoring` | Discovers all mirrored databases and checks table health |
| Pipeline | `pl_vigil_mirror_monitoring` | Executes the notebook |
| Schedule | Cron (30 min default) | Triggers the pipeline on a recurring interval |
| Reflex | `da_vigil_mirror_monitoring` | Sends email when the pipeline job fails |

## How It Works

1. The monitoring notebook enumerates all mirrored databases in the workspace via the Fabric REST API.
2. For each database, it calls the `getTablesMirroringStatus` endpoint and classifies failures as table-level or gateway-level.
3. If any table is unhealthy, the notebook raises an exception, which causes the pipeline activity to fail.
4. Data Activator subscribes to `Microsoft.Fabric.JobEvents` for the pipeline. When it detects a failure status, it sends an email alert to the configured recipient.

## Prerequisites

### 1. Workspace Permissions

The user running the deployment notebook must have **Contributor** or **Admin** role on the target workspace.

### 2. Fabric Capacity

The target workspace must be on a Fabric capacity (F-SKU or Trial). Data Activator requires Fabric capacity.

### 3. (Optional) Service Principal Setup

These steps are only required if you switch `AUTH_MODE` to `"spn"`.

**Entra ID App Registration**

1. Go to **Microsoft Entra ID > App registrations > New registration**.
2. Name it (e.g., `spn-fabric-vigil`) and register.
3. Under **Certificates & secrets**, create a client secret. Save the value immediately.
4. Note the **Application (client) ID** and **Directory (tenant) ID**.

**Fabric Tenant Admin Settings**

A Fabric Administrator must enable these settings in the **Admin Portal**:

- **Service principals can use Fabric APIs** (Developer settings)
- **Allow service principals to create and use profiles** (Developer settings)

For each setting, add a security group that contains your service principal.

**Workspace Access for SPN**

The service principal must have **Contributor** or **Admin** role on the target workspace. Add it via **Workspace Settings > Manage Access**.

**Azure Key Vault (Recommended for SPN)**

Store the SPN credentials as Key Vault secrets. The notebook expects these default secret names:

- `fabric-spn-client-id`: The Application (client) ID
- `fabric-spn-client-secret`: The client secret value

If your Key Vault already stores these values under different names, update the corresponding `notebookutils.credentials.getSecret(...)` calls in the `get_auth_headers` function to match your existing secret names. There is no need to create duplicate secrets.

The user running the deployment notebook must have **Key Vault Secrets User** role (or an equivalent access policy with Get permission).

Alternatively, you can paste the credentials directly into the configuration cell (not recommended for production).

## Deployment

1. Download `Deploy_Vigil.ipynb` from this repository.
2. Import it into the Fabric workspace that contains your mirrored databases.
3. Open the notebook and edit the **Configuration** cell:
   - **Choose an auth mode.** `AUTH_MODE` defaults to `"user"`, meaning all deployed items are owned by your personal identity. This is the simplest path: no app registration, no secrets, no admin settings. Switch to `"spn"` if you need items owned by a service principal (e.g., so monitoring survives employee offboarding, or so a shared identity owns the pipeline and reflex across environments).
   - If using SPN mode, set `KEY_VAULT_NAME` to your Key Vault name, or populate `SP_CLIENT_ID` and `SP_CLIENT_SECRET`.
   - Optionally adjust `SCHEDULE_INTERVAL_MINUTES` (default: 30).
   - Optionally set `ALERT_EMAIL_OVERRIDE` to route alerts to a specific address or distribution list. If left blank, alerts go to the user running the notebook.
4. Run all cells.
5. Deployment takes approximately 30 to 60 seconds. The output log shows each item created with its GUID.

## Post-Deployment

- The schedule starts immediately. The first pipeline run will occur within the configured interval.
- To add additional alert recipients, open the reflex in the Fabric UI and add coworker email addresses to the notification rule. The deployment gets you started with a single recipient; from there you can expand the distribution as needed.
- To change the schedule frequency, update the schedule in the pipeline settings or redeploy.
- To uninstall, delete the four items from the workspace (notebook, pipeline, reflex, and the schedule is removed with the pipeline).

## Repository Structure

~~~
Fabric-Vigil/
  README.md
  Deploy_Vigil.ipynb
  files/
    nb_vigil_mirror_monitoring.ipynb
    pipeline_content.json
    pipeline_platform.json
    reflex_entities.json
    reflex_platform.json
~~~

## Configuration Reference

| Parameter | Default | Description |
|-----------|---------|-------------|
| `AUTH_MODE` | `"user"` | `"user"` for interactive identity (default), `"spn"` for service principal |
| `KEY_VAULT_NAME` | `""` | Azure Key Vault name for credential retrieval |
| `SP_CLIENT_ID` | `""` | Hardcoded client ID (fallback if no Key Vault) |
| `SP_CLIENT_SECRET` | `""` | Hardcoded client secret (fallback if no Key Vault) |
| `SCHEDULE_INTERVAL_MINUTES` | `30` | Pipeline execution frequency in minutes |
| `SCHEDULE_TIMEZONE` | `"Central Standard Time"` | Timezone for the cron schedule |
| `ALERT_EMAIL_OVERRIDE` | `""` | Explicit alert recipient (blank = auto-detect from user token) |
| `NOTEBOOK_NAME` | `"nb_vigil_mirror_monitoring"` | Display name for the deployed notebook |
| `PIPELINE_NAME` | `"pl_vigil_mirror_monitoring"` | Display name for the deployed pipeline |
| `REFLEX_NAME` | `"da_vigil_mirror_monitoring"` | Display name for the deployed reflex |

## License

MIT
