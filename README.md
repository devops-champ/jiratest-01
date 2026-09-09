
# mysql-mcp — Deployment & Usage Notes
> Everything you need to know about the `mysql-mcp` server deployed on GKE:
> the env vars that configure it, how it was found/fixed, and how to query
> the `lake` DB (e.g. finding the `inventory-batch` apps).
---
## 1. Environment variables
The Secret is read by the `mysql-mcp` container image
(`@benborla29/mcp-server-mysql`), which reads env vars **by these exact names**
(confirmed from the image source at `/app/dist/src/config/index.js`). The
Secret is injected into the pod via the Deployment's
`envFrom -> secretRef: mysql-mcp-secret`.
### The full Secret
```yaml
MYSQL_HOST: "devlake-mysql.devlake.svc.cluster.local"
MYSQL_PORT: "3306"
MYSQL_USER: "merico"
MYSQL_PASS: "merico"
MYSQL_DB: "lake"
PORT: "8000"
IS_REMOTE_MCP: "true"
REMOTE_SECRET_KEY: "ddccb5306c2b27e6f43c4a112d1d143e"
```
### Var-by-var
**`MYSQL_HOST` = `devlake-mysql.devlake.svc.cluster.local`**
- **What:** The hostname the MCP server connects to for MySQL.
- **Where from:** The MySQL StatefulSet lives in the `devlake` namespace, but
  the MCP pod runs in the `mcp` namespace — so `localhost`/`127.0.0.1` (the
  image default) would never work.
- Confirmed with `kubectl get svc -n devlake | grep -i mysql` →
  `devlake-mysql ClusterIP 3306/TCP`.
- `devlake-mysql.devlake.svc.cluster.local` is the standard K8s DNS FQDN for a
  service in another namespace (`<service>.<namespace>.svc.cluster.local`).
**`MYSQL_PORT` = `3306`**
- **What:** MySQL's listening port.
- **Where from:** Service discovery above (`... 3306/TCP`) + the standard
  MySQL port.
**`MYSQL_USER` = `merico`**
- **What:** DB user the MCP server authenticates as.
- **Where from:** The playbook (Step 3) — the default DevLake MySQL user.
  Cross-checked against the pre-existing `mysql-mcp-secret`.
- ⚠️ **Name nuance:** this app reads **`MYSQL_PASS`**, not `MYSQL_PASSWORD`.
  The original playbook used `MYSQL_PASSWORD: merico`, which the image never
  reads → empty password → auth failed → CrashLoopBackOff. Fixed by adding the
  correctly-named `MYSQL_PASS` key (same value).
**`MYSQL_DB` = `lake`**
- **What:** The specific database to query (schema `lake` — where
  `_tool_argocd_applications` etc. live).
- ⚠️ **Name nuance:** the app reads **`MYSQL_DB`**, not `MYSQL_DATABASE`. The
  image default was `db_name`, which pointed at a non-existent schema.
**`PORT` = `8000`**
- **What:** The HTTP port the MCP server listens on.
- **Where from:** Deliberate choice (not provided) — the app reads
  `process.env.PORT || 3000`. Set to `8000` to align everywhere:
  - Deployment `containerPort: 8000`
  - Service `targetPort: 8000` (80 → 8000)
  - `HealthCheckPolicy` TCP health check on `8000`
  - `tcpSocket` liveness/readiness probes on `8000`
**`IS_REMOTE_MCP` = `true`**
- **What:** A mode switch in the app's code.
- **From source:** `const IS_REMOTE_MCP = process.env.IS_REMOTE_MCP === "true";`
- Must be `"true"` (combined with a non-empty `REMOTE_SECRET_KEY`) or the
  server runs **stdio** and never listens on a port — breaking the
  gateway/HTTP setup.
**`REMOTE_SECRET_KEY` = `ddccb5306c2b27e6f43c4a112d1d143e`**
- **What:** The bearer token required on every HTTP request
  (`Authorization: Bearer <key>`).
- **Where from:** Generated with `openssl rand -hex 16` (needed for HTTP mode).
- You'll need this exact value in your MCP client config — it's the
  `Authorization: Bearer ddccb5...` header.
### Provenance summary
| Env var | Source |
|---|---|
| `MYSQL_HOST` | Playbook + confirmed via `kubectl` (devlake service) |
| `MYSQL_PORT` | Standard + confirmed via `kubectl` |
| `MYSQL_USER` / `MYSQL_PASS` | Playbook + confirmed from existing secret; renamed `PASSWORD`→`PASS` for the app |
| `MYSQL_DB` | Playbook; renamed `DATABASE`→`DB` for the app |
| `PORT` | Chosen = `8000`, aligned with k8s container/service/health-check ports |
| `IS_REMOTE_MCP` | Required by app source (`true` enables HTTP) |
| `REMOTE_SECRET_KEY` | Generated (`openssl rand -hex 16`) |
---
## 2. Query the `lake` DB via mysql-mcp
The `inventory-batch` app names live in the **`lake._tool_argocd_applications`**
table (ArgoCD applications ingested by DevLake):
| name | namespace | project | repo_url | sync_status | health_status |
|---|---|---|---|---|---|
| inventory-batch-dev | argocd | default | …/inventory-batch.git | Synced | Healthy |
| inventory-batch-qa | argocd | default | …/inventory-batch.git | Synced | Healthy |
### Exact curl command (`tools/call` → `mysql_query`)
```bash
curl -i \
  -X POST \
  http://136.68.103.66/mysql-mcp/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -H 'Authorization: Bearer ddccb5306c2b27e6f43c4a112d1d143e' \
  -d '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"tools/call",
    "params":{
      "name":"mysql_query",
      "arguments":{
        "sql":"SELECT name, namespace, project, repo_url, sync_status, health_status, updated_at FROM lake._tool_argocd_applications WHERE name LIKE \"%inventory-batch%\""
      }
    }
  }'
```
### Variations
- **Broader** (inventory-service etc.): replace the `WHERE` with:
  ```sql
  WHERE name LIKE "%inventory%" OR name LIKE "%batch%"
  ```
- **Just the names:**
  ```sql
  SELECT name FROM lake._tool_argocd_applications WHERE name LIKE "%inventory-batch%"
  ```
### How to adapt the queries
- `_tool_argocd_applications` has **no `id` column** (hits an error). Right
  columns: `name`, `namespace`, `project`, `repo_url`, `dest_namespace`,
  `sync_status`, `health_status`, `updated_at`, etc. (confirmed via
  `information_schema.columns`).
- The raw table `_raw_argocd_api_applications` stores JSON in a `payload`
  column — use the **tool-level** `_tool_argocd_applications` table instead.
> ⚠️ **Read-only note:** the `mysql_query` tool allows INSERT/UPDATE (write
> ops are on). The commands above are `SELECT`-only (safe). If you run write
> queries later, be intentional about which table you target.
---
## 3. Quick reference — key ports & resources
| Resource | Value |
|---|---|
| MCP endpoint URL | `http://136.68.103.66/mysql-mcp/mcp` |
| Gateway | `backstage-gateway` (namespace `gateway-system`, IP `136.68.103.66`) |
| Bearer key | `ddccb5306c2b27e6f43c4a112d1d143e` |
| MySQL host | `devlake-mysql.devlake.svc.cluster.local:3306` |
| Database | `lake` (user `merico`) |
