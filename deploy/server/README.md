# Happy-Improved — Server (self-hosted relay)

Deploys `happy-server` (the Happy relay backend) to the cluster, so the app +
CLI talk to **our** server instead of the public relay `api.cluster-fluster.com`.
Mirrors the web/benjamins/quietplan pattern on the same cluster + edge.

## How traffic flows (no ingress controller — edge nginx + NodePort)

```
mobile / CLI / browser
  │  api  ──TLS──▶ edge nginx (api.happy.ahposten.com, certbot)
  │                  └─▶ nodeIP:30083  ─▶ happy-server Service ─▶ happy-server pods (:3005)
  │
  └ files ──TLS──▶ edge nginx (files.happy.ahposten.com, certbot)
                     └─▶ nodeIP:30903  ─▶ happy-minio Service   ─▶ MinIO (:9000)
```

- **Attachments** upload/download directly to MinIO via **presigned URLs** that
  happy-server signs for `files.happy.ahposten.com` (so that host must resolve +
  serve MinIO before the server boots — `loadFiles()` calls `bucketExists()`).
- **WebSocket**: the app/CLI open a socket.io connection at `/v1/updates`; the
  `api` vhost is WebSocket-aware via `conf.d/00-websocket-map.conf`.
- **Images**: Docker Hub `ahmadposten/happy-improved-server` (public, no pull
  secret), built from `Dockerfile.server`.
- **NodePorts**: `30083` api (after 30082 happy-web), `30903` MinIO S3
  (after 30902 benjamins instrument-logos MinIO).

## Components (namespace `happy`)

| Manifest | What |
|---|---|
| `k8s/postgres.yaml` | Postgres 16 StatefulSet + headless Service `happy-postgres:5432`, 10Gi local-path PVC, pinned to `server-guru-eu` |
| `k8s/redis.yaml` | Redis 7 (AOF) Deployment + Service `happy-redis:6379`, 1Gi PVC — socket.io/eventbus fan-out for multi-replica |
| `k8s/minio.yaml` | MinIO Deployment + 20Gi PVC + ClusterIP + NodePort `30903`; CORS for the web origin |
| `k8s/minio-setup-job.yaml` | Idempotent Job creating the private `happy` bucket |
| `k8s/deployment.yaml` | `happy-server` (2 replicas), init: wait-postgres → prisma `migrate deploy`; probes `/health` |
| `k8s/service.yaml` | NodePort `30083` |
| `k8s/secret.example.yaml` | **template** — the real `happy-server-secret` is created out-of-band |
| `nginx/happy-server-upstreams.conf` | edge upstreams `happy_api` (30083) + `happy_files` (30903) |
| `nginx/happy-api-sites.conf` | edge vhost `api.happy.ahposten.com` (WebSocket-aware) |
| `nginx/happy-files-sites.conf` | edge vhost `files.happy.ahposten.com` (MinIO, large uploads) |

## One-time setup

1. **DNS** — A records to the edge VM:
   ```
   api.happy.ahposten.com   → 35.179.90.95
   files.happy.ahposten.com → 35.179.90.95
   ```

2. **Secret** (cluster admin, once — the Jenkins deployer Role can't touch
   Secrets):
   ```sh
   PGPW=$(openssl rand -hex 24)
   kubectl -n happy create secret generic happy-server-secret \
     --from-literal=HANDY_MASTER_SECRET="$(openssl rand -hex 48)" \
     --from-literal=POSTGRES_PASSWORD="$PGPW" \
     --from-literal=DATABASE_URL="postgresql://happy:$PGPW@happy-postgres:5432/happy?schema=public" \
     --from-literal=S3_ACCESS_KEY="happy-$(openssl rand -hex 4)" \
     --from-literal=S3_SECRET_KEY="$(openssl rand -hex 32)"
   ```
   Rotating `HANDY_MASTER_SECRET` invalidates every device pairing — treat it as
   permanent.

3. **Edge nginx + TLS** — on the edge VM (as `jenkins-deploy`, which has scoped
   NOPASSWD sudo). MinIO must be up first (see Deploy) so its NodePort answers:
   ```sh
   sudo cp happy-server-upstreams.conf /etc/nginx/conf.d/
   sudo certbot certonly --nginx -d files.happy.ahposten.com
   sudo certbot certonly --nginx -d api.happy.ahposten.com
   sudo cp happy-files-sites.conf /etc/nginx/sites-enabled/happy-files
   sudo cp happy-api-sites.conf   /etc/nginx/sites-enabled/happy-api
   sudo nginx -t && sudo systemctl reload nginx
   ```

4. **Jenkins job `happy-server`** — New Item → Pipeline → *Pipeline script from
   SCM* → Git `https://github.com/Posten-Lab/happy-that-works.git` → Script Path
   `deploy/server/Jenkinsfile` → branch `main` (add a webhook to auto-build).

## Deploy

Trigger the `happy-server` job (or push to `main`). It kaniko-builds
`Dockerfile.server` → pushes `ahmadposten/happy-improved-server:{latest,git-<sha>}`
→ applies `deploy/server/k8s/*` → rolls out `deployment/happy-server` in `happy`.

## Verify

```sh
kubectl -n happy rollout status deploy/happy-server
curl -s https://api.happy.ahposten.com/health          # {"status":"ok",...}
curl -sI https://files.happy.ahposten.com/minio/health/live   # 200
```

## Client wiring

The app + CLI default to `https://api.happy.ahposten.com`
(`packages/happy-app/sources/sync/serverConfig.ts`,
`packages/happy-cli/src/configuration.ts`). The web build bakes it via the
`HAPPY_SERVER_URL` build-arg in `deploy/web/Jenkinsfile`. Override per-run with
`HAPPY_SERVER_URL` (CLI) or `EXPO_PUBLIC_HAPPY_SERVER_URL` (web build).
