# employee-app Helm chart

| Component | Type |
|---|---|
| Frontend (Node.js) | Container - Deployment + Service |
| Backend (Spring Boot) | Container - Deployment + Service |
| PostgreSQL | Container - `StatefulSet` + `PVC` (via `local-path-provisioner`) |
| MongoDB | External Vagrant VM - selector-less Service + Endpoints |

Postgres moved in-cluster; MongoDB stays external on purpose (blast-radius
separation). The backend now genuinely uses both: PostgreSQL for
`/api/employees`, MongoDB for `/api/activity`.

## Prerequisites

- Helm CLI and the `local-path` StorageClass already confirmed working
- Both images pushed to Docker Hub (`savirod211/frontend-employee`,
  `savirod211/backend-employee`)
- The `mongodb-db` VM up and reachable

## Before installing, edit `values.yaml`

Only one thing genuinely needs a real value: `mongodb.externalIp` — your
`mongodb-db` VM's actual IP. Everything else (Postgres username/password,
Docker Hub repos) already has a working default matching what we built
earlier.

## Install

```bash
helm install employee-app . --namespace apps --create-namespace
```

Deliberately **not** using `--atomic` here — if anything goes wrong, you
want the failing pod to stick around long enough to read its logs, not get
auto-deleted by a rollback.

```bash
kubectl get pods -n apps
```

You should see three pods: `frontend-employee`, `backend-employee`, and
`postgres-0` (the `-0` is the StatefulSet's stable ordinal identity).

## Verify all three tiers

```bash
kubectl port-forward -n apps svc/frontend-employee 3000:3000
curl http://localhost:3000/
```

For the database-specific endpoints, port-forward the backend directly:
```bash
kubectl port-forward -n apps svc/backend-employee 8080:8080
curl http://localhost:8080/api/employees      # PostgreSQL
curl http://localhost:8080/api/activity        # MongoDB
```

## What changed from the external-VM Postgres version

- `templates/postgres-external.yaml` (selector-less Service + Endpoints) is
  gone, replaced by `templates/postgres-statefulset.yaml` (headless Service
  + `StatefulSet` + `volumeClaimTemplates`).
- `templates/postgres-secret.yaml` is now the single source of truth for
  the Postgres password — both the Postgres container itself and the
  backend's `SPRING_DATASOURCE_PASSWORD` read from it, so they can never
  drift out of sync.
- `templates/mongodb-external.yaml` is new — same selector-less
  Service + Endpoints pattern Postgres used to use, now pointing at the
  `mongodb-db` VM instead.
- The backend Deployment gained `SPRING_DATA_MONGODB_URI`.

## Worth knowing

- **Postgres data now lives on whichever node the pod's PV was created
  on** — the same node-pinning behavior from the `StatefulSet` lab earlier
  applies here for real. If that node goes down, the pod can't reschedule
  elsewhere until it's back.
- **MongoDB's credentials are still plaintext in `values.yaml`** — fine for
  this learning setup, but a real deployment would pull both databases'
  credentials from a proper secrets manager (Vault, or the Secrets Store
  CSI driver from the earlier addon work) instead.
