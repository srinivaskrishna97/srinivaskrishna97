# Plan: Self-Host github-readme-stats on Kubernetes (Detailed)

**TL;DR**: Fork and prepare the `github-readme-stats` Node.js app, containerize it, push to your private cluster registry, deploy it to a dedicated `github-stats` namespace with a `Secret` for your GitHub PAT, expose it via `NodePort`, wire it through Nginx Proxy Manager with TLS, then update your README to point all widget URLs at your own domain. This gives the app full access to your private GitHub data using your own PAT.

---

## Phase 1 — GitHub PAT

**1.1 Generate a Personal Access Token**
- Go to [github.com/settings/tokens](https://github.com/settings/tokens) → **"Generate new token (classic)"**
- Name it something like `k8s-readme-stats`
- Set expiration (recommend 1 year or no expiry)
- Check the following scopes:
  - `repo` (full control of private repositories — needed to read private commit data)
  - `read:user` (read-only user profile)
- Click **Generate token** and copy it immediately — GitHub only shows it once
- Store it safely (password manager) — you'll use it in Phase 3

---

## Phase 2 — Prepare the Source Code

**2.1 Fork the repository**
- Go to [github.com/anuraghazra/github-readme-stats](https://github.com/anuraghazra/github-readme-stats)
- Click **Fork** → fork to your own GitHub account

**2.2 Clone your fork locally**
```bash
git clone https://github.com/<your-username>/github-readme-stats.git
cd github-readme-stats
```

**2.3 Move `express` to `dependencies`**
- Open `package.json`
- Find `"express"` under `devDependencies` and move it to `dependencies`
- This is mandatory — without it, `npm ci --omit=dev` won't include Express and the server won't start

**2.4 Create a `Dockerfile`** in the project root:
```dockerfile
FROM node:22-alpine

WORKDIR /app

# Copy package files first for better layer caching
COPY package*.json ./

# Install only production dependencies
RUN npm ci --omit=dev

# Copy the rest of the source
COPY . .

EXPOSE 9000

CMD ["node", "express.js"]
```

**2.5 Create a `.dockerignore`** in the project root:
```
node_modules
.git
.github
*.md
tests
```

**2.6 Commit the Dockerfile to your fork**
```bash
git add Dockerfile .dockerignore package.json
git commit -m "Add Dockerfile and move express to dependencies"
git push
```

---

## Phase 3 — Build & Push Docker Image

**3.1 Confirm your private registry address**
- Your cluster registry will have an address like `registry.yourdomain.com` or an IP like `192.168.x.x:5000`
- Confirm it's reachable from your local machine: `curl http://<registry-address>/v2/`

**3.2 Build the image**
```bash
docker build -t <your-registry-address>/github-readme-stats:latest .
```

**3.3 If your registry requires login**
```bash
docker login <your-registry-address>
```

**3.4 Push the image**
```bash
docker push <your-registry-address>/github-readme-stats:latest
```

**3.5 Verify the image is in the registry**
```bash
curl http://<your-registry-address>/v2/github-readme-stats/tags/list
# Should return: {"name":"github-readme-stats","tags":["latest"]}
```

---

## Phase 4 — Kubernetes Manifests

Create a folder `k8s/` in your local working directory with the following files:

**4.1 `k8s/namespace.yaml`**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: github-stats
```

**4.2 `k8s/secret.yaml`** — stores your GitHub PAT
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: github-stats-secret
  namespace: github-stats
type: Opaque
stringData:
  PAT_1: "<paste-your-github-pat-here>"
```
> Note: `stringData` lets you paste the raw token — Kubernetes base64-encodes it automatically.

**4.3 `k8s/deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: github-readme-stats
  namespace: github-stats
  labels:
    app: github-readme-stats
spec:
  replicas: 1
  selector:
    matchLabels:
      app: github-readme-stats
  template:
    metadata:
      labels:
        app: github-readme-stats
    spec:
      containers:
        - name: github-readme-stats
          image: <your-registry-address>/github-readme-stats:latest
          ports:
            - containerPort: 9000
          env:
            - name: PAT_1
              valueFrom:
                secretKeyRef:
                  name: github-stats-secret
                  key: PAT_1
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
          livenessProbe:
            httpGet:
              path: /api/status/up
              port: 9000
            initialDelaySeconds: 10
            periodSeconds: 30
          readinessProbe:
            httpGet:
              path: /api/status/up
              port: 9000
            initialDelaySeconds: 5
            periodSeconds: 10
      imagePullSecrets:
        - name: registry-credentials   # only needed if your registry requires auth from the cluster
```

**4.4 `k8s/service.yaml`** — NodePort so Nginx Proxy Manager can reach it
```yaml
apiVersion: v1
kind: Service
metadata:
  name: github-readme-stats
  namespace: github-stats
spec:
  type: NodePort
  selector:
    app: github-readme-stats
  ports:
    - port: 9000
      targetPort: 9000
      nodePort: 32099   # pick any free port in range 30000–32767
```

> If your registry requires authentication **from within the cluster**, run:
> ```bash
> kubectl create secret docker-registry registry-credentials \
>   --docker-server=<your-registry-address> \
>   --docker-username=<user> \
>   --docker-password=<pass> \
>   -n github-stats
> ```

---

## Phase 5 — Deploy to Kubernetes

```bash
# Apply in order
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/secret.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
```

**Verify everything is running:**
```bash
# Check pod is Running
kubectl get pods -n github-stats

# Check service and note the NodePort
kubectl get svc -n github-stats

# Tail logs to confirm the app started
kubectl logs -f deployment/github-readme-stats -n github-stats
# Should see: "Listening on port 9000"
```

**Quick smoke test from your local machine:**
```bash
curl http://<any-node-IP>:32099/api/status/up
# Expected: {"status":"UP"}

curl http://<any-node-IP>:32099/api/status/pat-info
# Should show your PAT is valid and not rate-limited
```

---

## Phase 6 — Nginx Proxy Manager Setup

**6.1 Add a new Proxy Host in NPM UI**
- **Domain Name**: your chosen subdomain (e.g. `stats.yourdomain.com`)
- **Scheme**: `http`
- **Forward Hostname/IP**: IP of any cluster node
- **Forward Port**: `32099` (the NodePort you set)
- **Cache Assets**: optional (app sends its own `Cache-Control` headers)
- **Block Common Exploits**: enable

**6.2 Enable SSL**
- Go to the **SSL** tab of the proxy host
- Select **"Request a new SSL certificate"** via Let's Encrypt
- Enable **"Force SSL"** and **"HTTP/2 Support"**
- Enter your email and agree to the ToS

**6.3 DNS**
- Point your subdomain's A record to your cluster's public IP (wherever Nginx Proxy Manager is exposed)
- Wait for propagation: `dig stats.yourdomain.com` should resolve to your IP

---

## Phase 7 — Update README

Replace both existing widget URLs in README.md:

| Widget | New URL |
|---|---|
| Stats card | `https://stats.yourdomain.com/api?username=srinivaskrishna97&show_icons=true&count_private=true&include_all_commits=true&hide_border=true&title_color=02D9F7FF&text_color=02D9F7FF&bg_color=0d1117&icon_color=02D9F7FF` |
| Top languages | `https://stats.yourdomain.com/api/top-langs/?username=srinivaskrishna97&layout=compact&count_private=true&hide_border=true&title_color=02D9F7FF&text_color=02D9F7FF&bg_color=0d1117` |

Then commit and push:
```bash
git add README.md
git commit -m "Switch stats widgets to self-hosted instance with private contributions"
git push
```

---

## Verification Checklist

- [ ] `curl https://stats.yourdomain.com/api/status/up` → `{"status":"UP"}`
- [ ] `curl https://stats.yourdomain.com/api/status/pat-info` → PAT valid, not rate-limited
- [ ] GitHub profile stats card shows full contribution count (e.g. 443)
- [ ] Top languages card reflects private repo languages

---

## Notes

- NodePort `32099` + Nginx Proxy Manager — matches your existing infrastructure, no in-cluster Ingress controller needed
- Namespace `github-stats` — isolates the deployment cleanly
- PAT stored as a Kubernetes `Secret` — never hardcoded or exposed in the image
- 1 replica is sufficient; each PAT gives 5,000 API req/hr with built-in HTTP-level caching
- To add more PATs later (for higher throughput), add `PAT_2`, `PAT_3`, etc. to the Secret and Deployment env — the app rotates through them automatically on rate-limit
