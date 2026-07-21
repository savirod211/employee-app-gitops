### YOU MOTHER FUCKING GOT THIS!!! ###
# employee-app Helm chart

Packages `frontend-employee` + `backend-employee` as Kubernetes Deployments,
wired to the external Postgres VM via the same selector-less Service +
Endpoints pattern from before - now managed by the chart instead of a
standalone `kubectl apply`.

## 0. Prerequisites (from earlier)

```bash
helm version           # confirm Helm CLI is installed
kubectl get storageclass   # confirm local-path shows up as (default)
```
Not required for *this* chart specifically - neither app uses a PVC - but
worth confirming before we go further, since ArgoCD's own install and any
future StatefulSet work will assume both are already in place.

## 1. Push both images to Docker Hub

Since GitHub Actions doesn't exist yet, this is manual for now:
```bash
docker login

cd backend-employee
docker build -t your-dockerhub-username/backend-employee:latest .
docker push your-dockerhub-username/backend-employee:latest

cd ../frontend-employee
docker build -t your-dockerhub-username/frontend-employee:latest .
docker push your-dockerhub-username/frontend-employee:latest
```

## 2. Edit values.yaml

Three things need real values before this works:
- `frontend.image.repository` / `backend.image.repository` → your Docker Hub username
- `postgres.externalIp` → your `postgres-db` VM's actual IP
- `backend.env.datasourcePassword` → whatever you set in `provision-postgres.sh`

## 3. Clean up the standalone Postgres wiring from before

If you already ran `kubectl apply -f k8s-postgres-external-service.yaml`
earlier, remove it first - the chart now owns this Service/Endpoints pair,
and Helm will refuse to install if a same-named resource already exists
that it doesn't own:
```bash
kubectl delete -f k8s-postgres-external-service.yaml
```

## 4. Install

```bash
helm install employee-app ./employee-app --namespace apps --create-namespace
kubectl get pods -n apps
```
Both `frontend-employee` and `backend-employee` should reach `Running`.

## 5. Test it

```bash
kubectl port-forward -n apps svc/frontend-employee 3000:3000
curl http://localhost:3000/
```
Expected: the frontend's response showing it successfully called the
backend, which successfully queried Postgres - the full chain, now running
as real Kubernetes workloads instead of Docker Compose.

## What's deliberately not in this chart yet

- **No Postgres StatefulSet** - it's external on purpose, per your call
  earlier on blast-radius separation.
- **No MongoDB wiring** - the backend doesn't consume it yet; add a
  `mongodb-external.yaml` template (identical pattern to `postgres-external.yaml`)
  once there's actual code calling it.
- **No ArgoCD `Application` yet** - that's next: push this chart to a git
  repo, install ArgoCD, then point it here so it takes over from this
  manual `helm install`.
