# Happy-Improved — Deployment

How the web app ships to `happy.ahposten.com` and how the iOS app ships to
TestFlight, both driven by Jenkins. Mirrors the benjamins / quietplan setup
on the same cluster + edge.

## How traffic flows (no ingress controller, no cert-manager)

```
browser ──TLS──▶ edge nginx VM (35.179.90.95, certbot/Let's Encrypt)
                   │  proxy_pass http://happy_frontend
                   ▼
        nodeIP:30082  (NodePort, 3 public nodes: 81.99.40.60, 89.125.50.58, 89.125.255.36)
                   ▼
        happy-web Service (NodePort) ─▶ happy-web pods (nginx serving the Expo web export)
```

- **Images**: Docker Hub, `ahmadposten/happy-improved-web`. Public repo, no pull secret (same as `ahmadposten/benjamins-web`).
- **CI**: Jenkins in-cluster. `kaniko` agent builds+pushes images; `eas` agent builds the mobile app.
- **NodePort `30082`**: next free in the `300xx` frontend convention (30080 quietplan, 30081 benjamins, 30088 jenkins taken).
- **Backend**: the web app is a pure client; it defaults to the relay `https://api.cluster-fluster.com`. No backend to deploy. Override at build time with `--build-arg HAPPY_SERVER_URL=…` in `deploy/web/Jenkinsfile`.

---

## Web app — `happy.ahposten.com`

Artifacts: `deploy/web/k8s/{namespace,deployment,service}.yaml`, `deploy/web/nginx/{happy-upstreams,happy-sites}.conf`, `deploy/web/Jenkinsfile`.

### One-time setup

1. **DNS** — add an A record: `happy.ahposten.com → 35.179.90.95` (the edge VM).

2. **Edge nginx + TLS** — on the edge VM `35.179.90.95` (as the admin user, or the `jenkins-deploy` user):
   ```sh
   # 1. Drop happy's two configs into the active nginx config dir.
   #    (append — do NOT rsync --delete; that would wipe the other sites)
   sudo cp happy-upstreams.conf happy-sites.conf /etc/nginx/staged/   # or wherever active configs live

   # 2. Issue the cert (DNS from step 1 must already resolve to this VM).
   #    certbot uses the already-running nginx for the HTTP-01 challenge.
   sudo certbot certonly --nginx -d happy.ahposten.com

   # 3. Validate + reload.
   sudo nginx -t && sudo systemctl reload nginx
   ```
   `happy-sites.conf` references `/etc/letsencrypt/live/happy.ahposten.com/…`, so run certbot **before** the reload (or deploy an HTTP-only stub first — `certbot certonly --nginx` handles the challenge either way as long as nginx is running).

3. **Jenkins job `happy-web`** — New Item → Pipeline:
   - Pipeline: *Pipeline script from SCM* → Git → `git@github.com:Ahmadposten/happy-that-works.git`
   - Script Path: `deploy/web/Jenkinsfile`
   - Branch: `main` (add a GitHub webhook or poll SCM to auto-build on push).

### Deploy
Trigger the `happy-web` job (or push to `main`). It builds `Dockerfile.webapp` with kaniko, pushes `ahmadposten/happy-improved-web:{latest,git-<sha>}`, applies `deploy/web/k8s`, and rolls out `deployment/happy-web` in namespace `happy`.

### Verify
```sh
kubectl -n happy rollout status deploy/happy-web
curl -I http://89.125.50.58:30082        # NodePort direct (expect 200)
curl -I https://happy.ahposten.com       # through the edge (expect 200)
```

---

## Mobile app — iOS → TestFlight

Artifacts: `deploy/mobile/Jenkinsfile`, plus edits to `packages/happy-app/{app.config.js,eas.json}` (see below).

### Values needed to wire it to YOUR identity
The repo currently ships the upstream author's identity (Expo owner `bulkacorp`, bundle `com.ex3ndr.happy`, Steve's ASC app). Provide these and they get substituted into `app.config.js` + `eas.json`:

| Value | Where it comes from |
|---|---|
| iOS **bundle identifier** | the bundle ID you registered for your existing App Store Connect app |
| **ascAppId** (numeric) | App Store Connect → your app → App Information → “Apple ID” |
| Expo **owner** (account/org) | your Expo account username |
| EAS **projectId** | output of `eas init` run once inside `packages/happy-app` under your Expo account |

Apple **team** is already `H2XR8XWZXW` (individual) in `deploy/mobile/Jenkinsfile`.

### Jenkins credentials (reused from benjamins-mobile — verify they exist)
App Store Connect API keys are **team-wide**, so the existing key already covers a second app under team `H2XR8XWZXW`. Confirm these Jenkins credentials exist (Manage Jenkins → Credentials):

| ID | Type | What |
|---|---|---|
| `expo-token` | Secret text | Expo access token for the account that owns the happy EAS project |
| `asc-key-id` | Secret text | ASC API key **Key ID** |
| `asc-issuer-id` | Secret text | ASC API key **Issuer ID** |
| `asc-key-p8` | Secret text | base64 of the ASC API `.p8` file (`base64 -w0 AuthKey_XXXX.p8`) |

> If your happy EAS project lives under a **different** Expo account than benjamins, mint a new `expo-token` for it (or point both at one Expo account).

### Jenkins job `happy-mobile`
New Item → Pipeline → *Pipeline script from SCM* → same repo → Script Path `deploy/mobile/Jenkinsfile`. It installs the workspace, typechecks, runs `eas build -p ios --profile production --wait`, then `eas submit -p ios --profile production --latest` (TestFlight).

---

## Apple / App Store Connect — crisp step-by-step (one-time)

You said you already have an app record. To let Jenkins build + submit unattended you need a **team API key** (not your Apple ID password). If benjamins already submits via CI, this key exists and you can skip to step 4.

1. **App Store Connect → Users and Access → Integrations → App Store Connect API → Team Keys → “+”.**
   - Name: `ci-eas`. Access: **App Manager**. Generate.
2. **Download the `.p8`** (one chance only). Note the **Key ID** and the **Issuer ID** shown on that page.
3. Base64-encode it: `base64 -w0 AuthKey_<KEYID>.p8` (macOS: `base64 -i AuthKey_<KEYID>.p8`).
4. In Jenkins, create/confirm the three secrets: `asc-key-id` = Key ID, `asc-issuer-id` = Issuer ID, `asc-key-p8` = the base64 string.
5. Confirm your app record’s **bundle identifier** matches what we put in `app.config.js`, and grab its numeric **Apple ID** (ascAppId) from App Information.
6. Expo: `cd packages/happy-app && eas login && eas init` (creates the project under your account) → copy the printed **projectId** and your **owner** to me.

Once you send **bundle id · ascAppId · Expo owner · projectId**, I substitute them into `app.config.js` + `eas.json` and the `happy-mobile` job is ready.
