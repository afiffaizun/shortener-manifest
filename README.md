# Shortener Manifests

Kubernetes manifests for the URL shortener infrastructure (`shortener` namespace):
Postgres + API (placeholder `nginx:alpine`), Traefik Ingress, HPA.

## Prerequisites

- A running Kubernetes cluster with `kubectl` configured
- Traefik ingress controller and the `local-path` storage class (defaults on k3s)

## Setup

Create the local secret (git-ignored, never committed):

```bash
cp 02-secret.example.yaml 02-secret.yaml
# edit 02-secret.yaml and set a real DB_PASSWORD
```

## Run

Apply in order:

```bash
kubectl apply -f 00-namespace.yaml -f 01-config.yaml -f 02-secret.yaml \
  -f 03-postgres.yaml -f 04-rbac.yaml -f 05-api.yaml -f 06-ingress.yaml -f 07-hpa.yaml
```

## Verify

```bash
kubectl get pods,svc,ingress,hpa -n shortener
kubectl run tmp-curl --rm -i --restart=Never --image=curlimages/curl:8.5.0 \
  -n shortener -- curl -s -o /dev/null -w "svc: %{http_code}\n" http://shortener-svc/
kubectl exec -n shortener deploy/postgres -- \
  sh -c 'pg_isready -U "$POSTGRES_USER" -d "$POSTGRES_DB"'
```

Ingress: map the host to your cluster IP, then open it in a browser or curl:

```bash
# example (replace IP with your Traefik external IP)
echo "192.168.123.240 shortener.latihan.local" | sudo tee -a /etc/hosts
curl -H "Host: shortener.latihan.local" http://192.168.123.240/
```

## Cleanup

```bash
kubectl delete namespace shortener
```

## Notes

- The API currently uses placeholder `nginx:alpine` to prove the infra works.
  When the real app image is ready (listens on `PORT=3000`), replace the image
  in `05-api.yaml` and switch `containerPort`, probes, and Service `targetPort` to `3000`.
- HPA (`07-hpa.yaml`) needs a working metrics-server; without it pods stay at `minReplicas: 2`.
