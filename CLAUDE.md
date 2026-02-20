# CLAUDE.md — Fabric-Vigil

## Project Overview

**Fabric-Vigil** is a one-click deployment tool for automated health monitoring of Microsoft Fabric Mirrored Databases. Running the deployment notebook once per workspace provisions:

- A **monitoring notebook** that checks every mirrored table's replication status
- A **data pipeline** that executes the notebook
- A **cron schedule** that triggers the pipeline on a configurable interval (default: every 30 minutes)
- A **Data Activator reflex** that sends an email alert when the pipeline fails

The system works by failing the notebook (raising an exception) on any unhealthy table, which causes the pipeline to fail, which triggers the Data Activator alert.

---

## Repository Structure

```
Fabric-Vigil/
  CLAUDE.md                          # This file
  README.md                          # End-user documentation
  Deploy_Vigil.ipynb                 # Deployment orchestrator (run once in Fabric)
  files/
    nb_vigil_mirror_monitoring.ipynb # Monitoring notebook (deployed to workspace)
    pipeline_content.json            # Pipeline definition template (has placeholders)
    pipeline_platform.json           # Pipeline Fabric Git integration metadata
    reflex_entities.json             # Data Activator reflex template (has placeholders)
    reflex_platform.json             # Reflex Fabric Git integration metadata
```

---

## Technology Stack

| Component | Technology |
|-----------|-----------|
| Runtime | Microsoft Fabric (Spark/Trident notebooks) |
| Language | Python 3 |
| HTTP client | `requests` |
| Auth (SPN mode) | `msal` (ConfidentialClientApplication) |
| Auth (user mode) | `notebookutils.credentials.getToken` |
| Secrets | Azure Key Vault via `notebookutils.credentials.getSecret` |
| File system | OneLake via `notebookutils.fs.ls` |
| REST APIs | Fabric REST API (`api.fabric.microsoft.com/v1`) |
| Event alerting | Microsoft Data Activator (Reflex) |

---

## Key Notebooks

### `Deploy_Vigil.ipynb` — Deployment Orchestrator

Run this notebook **once** in the target Fabric workspace. It:

1. Detects user context (UPN and tenant ID) from the JWT of the running user
2. Reads workspace ID from `notebookutils.runtime.context`
3. Acquires an auth token (user or SPN, based on `AUTH_MODE`)
4. Downloads template files from this GitHub repo
5. Injects GUIDs and configuration into the templates (placeholder substitution)
6. Calls the Fabric REST API to create each item in dependency order

**Deployment order** (dependencies flow downward):
```
Notebook → Pipeline (needs notebook ID) → Schedule → Reflex (needs pipeline ID)
```

### `files/nb_vigil_mirror_monitoring.ipynb` — Monitoring Notebook

This is the notebook **deployed** into the workspace. It runs on the pipeline schedule and:

1. Lists all `MirroredDatabase` items in the workspace via the Fabric items API
2. For each database, resolves the OneLake `Tables/` folder path (tries two candidate paths)
3. Discovers all `schema.table` entries by listing OneLake directories
4. Calls `POST .../mirroredDatabases/{id}/getTablesMirroringStatus`
5. Checks each table's status against `HEALTHY_STATUSES = ["Replicating", "Running", "Succeeded"]`
6. Prints a per-database summary
7. **Raises an exception** if any table is unhealthy — this intentionally fails the notebook/pipeline to trigger the Data Activator alert

---

## Template Placeholder Convention

Template files in `files/` use `__UPPERCASE_NAME__` placeholders that `Deploy_Vigil.ipynb` replaces at deployment time via Python string `.replace()`:

| Placeholder | Replaced with | Used in |
|-------------|--------------|---------|
| `__NOTEBOOK_GUID__` | Deployed notebook's item ID | `pipeline_content.json` |
| `__WORKSPACE_GUID__` | Workspace ID | `pipeline_content.json`, `reflex_entities.json` |
| `__PIPELINE_GUID__` | Deployed pipeline's item ID | `reflex_entities.json` |
| `__TENANT_GUID__` | Entra tenant ID (from JWT) | `reflex_entities.json` |
| `__USER_UPN__` | Alert recipient email | `reflex_entities.json` |

After substitution, the content is base64-encoded and submitted as `InlineBase64` payloads to the Fabric REST API.

---

## Authentication

Two modes controlled by `AUTH_MODE` in the configuration cell of `Deploy_Vigil.ipynb`:

### `AUTH_MODE = "user"` (default)
- Uses `notebookutils.credentials.getToken("https://api.fabric.microsoft.com")`
- Deployed items are owned by the running user's identity
- Simplest setup — no app registration required

### `AUTH_MODE = "spn"`
- Uses MSAL `ConfidentialClientApplication` with client credentials flow
- Credentials resolved in priority order:
  1. Azure Key Vault (if `KEY_VAULT_NAME` is set) — secrets named `fabric-spn-client-id` and `fabric-spn-client-secret`
  2. Hardcoded `SP_CLIENT_ID` / `SP_CLIENT_SECRET` (dev/test only — never commit to source control)
- Deployed items are owned by the service principal

**User context** (UPN and tenant ID) is always detected from the interactive user's token regardless of `AUTH_MODE`. This determines the default alert email recipient.

---

## Configuration Reference

All configuration lives in the **Configuration cell** (cell 4) of `Deploy_Vigil.ipynb`:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `AUTH_MODE` | `"user"` | `"user"` or `"spn"` |
| `KEY_VAULT_NAME` | `""` | Azure Key Vault name for SPN credentials |
| `SP_CLIENT_ID` | `""` | SPN client ID (fallback; never commit real values) |
| `SP_CLIENT_SECRET` | `""` | SPN client secret (fallback; never commit real values) |
| `SCHEDULE_INTERVAL_MINUTES` | `30` | Pipeline run frequency in minutes |
| `SCHEDULE_TIMEZONE` | `"Central Standard Time"` | Windows timezone name for the cron schedule |
| `ALERT_EMAIL_OVERRIDE` | `""` | Override alert recipient (blank = auto-detect from token) |
| `NOTEBOOK_NAME` | `"nb_vigil_mirror_monitoring"` | Display name for the deployed notebook |
| `PIPELINE_NAME` | `"pl_vigil_mirror_monitoring"` | Display name for the deployed pipeline |
| `REFLEX_NAME` | `"da_vigil_mirror_monitoring"` | Display name for the deployed reflex |
| `GITHUB_REPO` | `"imtkain/Fabric-Vigil"` | Source repo for template downloads |
| `GITHUB_BRANCH` | `"main"` | Branch to pull templates from |
| `GITHUB_FILES_PATH` | `"files"` | Path within repo where templates live |

---

## Fabric REST API Patterns

### Long-Running Operations (LRO)

Fabric item creation is often asynchronous. The code handles both sync and async responses:

- `201`/`200` → item created synchronously; ID is in the response body
- `202` → LRO started; poll the `Location` header URL until `status` is `"succeeded"` or `"completed"`, then call `find_item_id()` to look up the item ID by name+type

See `wait_for_lro()` and `create_fabric_item()` in `Deploy_Vigil.ipynb`.

### Key Endpoints Used

| Purpose | Method | URL |
|---------|--------|-----|
| List workspace items | GET | `/v1/workspaces/{wsId}/items` |
| Create notebook | POST | `/v1/workspaces/{wsId}/notebooks` |
| Create pipeline | POST | `/v1/workspaces/{wsId}/dataPipelines` |
| Create schedule | POST | `/v1/workspaces/{wsId}/items/{pipelineId}/jobs/Pipeline/schedules` |
| Create reflex | POST | `/v1/workspaces/{wsId}/reflexes` |
| Get mirroring status | POST | `/v1/workspaces/{wsId}/mirroredDatabases/{dbId}/getTablesMirroringStatus` |

---

## Development Conventions

### Logging
Use the `log()` helper in `Deploy_Vigil.ipynb` for all status output — it prefixes each line with a `[YYYY-MM-DD HH:MM:SS]` timestamp.

### Error Handling
- HTTP errors use `resp.raise_for_status()` after logging the error body
- LRO timeouts raise `Exception` with a descriptive message
- Missing configuration raises `ValueError` with actionable instructions

### Secrets
- **Never commit real credentials.** `SP_CLIENT_ID` and `SP_CLIENT_SECRET` fields exist only as fallback for local development.
- Use `KEY_VAULT_NAME` for all production deployments.

### Template Modifications
When modifying template files in `files/`:
- Maintain the existing `__PLACEHOLDER__` naming convention for any new injection points
- Keep `pipeline_platform.json` and `reflex_platform.json` with `"logicalId": "00000000-0000-0000-0000-000000000000"` — the Fabric API assigns real IDs at creation time
- The `reflex_entities.json` contains inline-JSON strings (JSON-within-JSON); edit carefully to preserve escaping

### Adding a New Monitoring Status
To recognize a new Fabric mirroring status as healthy, add it to `HEALTHY_STATUSES` in `files/nb_vigil_mirror_monitoring.ipynb`:
```python
HEALTHY_STATUSES = ["Replicating", "Running", "Succeeded", "YourNewStatus"]
```

---

## Deployment Workflow

1. Fork or clone this repository if making customizations
2. If forked, update `GITHUB_REPO` and `GITHUB_BRANCH` in the configuration cell
3. Download `Deploy_Vigil.ipynb` and import it into the target Fabric workspace
4. Edit the **Configuration cell** (required: review `AUTH_MODE`; optional: all others)
5. Run all cells — the output log shows each item being created with its GUID
6. Verify: check Fabric workspace for the four deployed items

**The deployment notebook downloads templates from GitHub at runtime.** Changes to `files/` must be committed and pushed to the configured branch before redeploying.

---

## Uninstallation

Delete the four items from the Fabric workspace:
- `nb_vigil_mirror_monitoring` (Notebook)
- `pl_vigil_mirror_monitoring` (Data Pipeline — deleting this also removes the schedule)
- `da_vigil_mirror_monitoring` (Reflex)

---

## Common Issues

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| `Could not determine workspace ID` | Not running in Fabric | Must run inside a Fabric notebook |
| `Could not detect UPN from token claims` | Service account or managed identity | Set `ALERT_EMAIL_OVERRIDE` explicitly |
| `SPN token acquisition failed` | Wrong credentials or missing tenant admin settings | Verify app registration and Fabric admin settings |
| Reflex never fires | Reflex not subscribed to the right pipeline | Redeploy; check `__PIPELINE_GUID__` was injected |
| `Could not resolve Tables path` | Mirrored DB not yet synced or unusual naming | Check OneLake paths manually; the code tries two candidate paths |
| Pipeline 403 on schedule creation | SPN lacks permission | Ensure SPN has Contributor or Admin on workspace |
