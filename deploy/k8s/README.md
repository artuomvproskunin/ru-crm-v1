# Kubernetes deploy (h3llo, salesdaily-dev)

```bash
export KUBECONFIG=~/.kube/salesdaily-dev_kubeconfig.yaml
```

## 1. Ingress controller (once)

```bash
helm repo add traefik https://traefik.github.io/charts && helm repo update
helm upgrade --install traefik traefik/traefik \
  -n traefik --create-namespace -f deploy/k8s/infra/traefik-values.yaml
kubectl -n traefik get svc traefik     # EXTERNAL-IP → DNS A-record dev.salesdaily.ru
```

## 2. Secrets (once, by hand — never committed)

```bash
kubectl create namespace salesdaily

# Pull access to the private GHCR image: a GitHub PAT (classic) with
# only `read:packages`.
kubectl -n salesdaily create secret docker-registry ghcr-pull \
  --docker-server=ghcr.io \
  --docker-username=<github-login> \
  --docker-password=<PAT>

# Runtime env: copy app/env.example OUTSIDE the repo, fill it in.
kubectl -n salesdaily create secret generic crm-env --from-env-file=<filled-file>
```

Changing a value later:

```bash
kubectl -n salesdaily create secret generic crm-env --from-env-file=<filled-file> \
  --dry-run=client -o yaml | kubectl apply -f -
kubectl -n salesdaily rollout restart deploy/crm
```

## 3. App

```bash
# pin the image built by the docker-image workflow
(cd deploy/k8s/app && kustomize edit set image \
  ghcr.io/artuomvproskunin/ru-crm-v1=ghcr.io/artuomvproskunin/ru-crm-v1:dev-sha-<short>)

kubectl apply -k deploy/k8s/app
kubectl -n salesdaily rollout status deploy/crm
kubectl -n salesdaily get pods,svc,ingress,cronjob
```

## Checks

```bash
# app from inside the cluster
kubectl -n salesdaily run t --rm -i --restart=Never --image=curlimages/curl -- \
  curl -s http://crm/api/health

# through the ingress before DNS exists
curl -s http://<EXTERNAL-IP>/api/health -H 'Host: dev.salesdaily.ru'

# run the daily pipeline now (instead of waiting for 03:00 UTC)
kubectl -n salesdaily create job --from=cronjob/crm-daily-pipeline manual-$(date +%s)
kubectl -n salesdaily logs -f job/<job-name>
```
