# Local Kubernetes and GitOps Lab on macOS

A working runbook for the lab built on a MacBook Pro (Apple silicon): Docker image, local registry,
kind cluster, Traefik ingress on port 80, a Helm chart for the app, and Argo CD deploying that chart
from GitHub.

## What the lab does

A browser request reaches the app through this chain:

1. Mac port 80.
2. Docker forwards it to port 30080 on the kind node container.
3. That is Traefik's node port, so Traefik receives the request.
4. Traefik reads the Ingress rule and matches the host and path.
5. The request goes to the `python-app` Service on port 80.
6. The Service forwards it to the pod on port 5000.

Deployment works the other way round. A commit to the GitHub repo is picked up by Argo CD, which runs
the Helm chart and applies the result to the cluster.

## Names and ports used

| Item | Value |
| --- | --- |
| Registry | `localhost:5001` |
| Image | `localhost:5001/python-app:v1` |
| Cluster | `kind` (context `kind-kind`) |
| Node ports | 30080 to host 80, 30443 to host 443 |
| Ingress class | `traefik` |
| App URL | http://python.localhost/api/v1/info |
| Argo CD URL | http://argocd.localhost |
| GitOps repo | https://github.com/ramenika/kind-gitops |
| Chart path in repo | `charts/python-app` |

---

## 1. Prerequisites

1. Install the tools:

```
brew install kind kubectl helm gh
```

2. Check Docker Desktop is running:

```
docker info
```

3. Give Docker Desktop at least 8 GB of memory and 4 CPUs under **Settings → Resources**.

---

## 2. Local Docker registry

1. Create a folder for the registry data:

```
mkdir -p ~/docker-registry
```

2. Start the registry:

```
docker run -d -p 5001:5000 --restart=always --name registry -v ~/docker-registry:/var/lib/registry registry:3
```

3. Check it answers:

```
curl http://localhost:5001/v2/
```

Port 5001 is used rather than 5000 because AirPlay Receiver takes port 5000 on macOS.

---

## 3. Build and push the image

The Dockerfile:

```dockerfile
FROM python:3.12-alpine

COPY requirements.txt /tmp/

RUN pip install -r /tmp/requirements.txt

COPY ./src /src

CMD ["python", "/src/app.py"]
```

1. Build from the folder holding the Dockerfile:

```
docker build -t python-app:v1 .
```

2. Tag it for the registry:

```
docker tag python-app:v1 localhost:5001/python-app:v1
```

3. Push it:

```
docker push localhost:5001/python-app:v1
```

4. List what the registry holds:

```
curl http://localhost:5001/v2/_catalog
```

---

## 4. Create the kind cluster

1. Create the config file:

```
mkdir -p ~/kind && cd ~/kind
cat > kind-port80.yaml <<'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30080
    hostPort: 80
    protocol: TCP
  - containerPort: 30443
    hostPort: 443
    protocol: TCP
EOF
```

2. Create the cluster:

```
kind create cluster --name kind --config kind-port80.yaml
```

3. Check the node is ready:

```
kubectl get nodes
```

Port mappings can only be set when the cluster is created. To change them, delete the cluster and
create it again.

4. Load the image into the cluster. kind nodes cannot reach `localhost:5001` on the Mac, so the image
   is copied in directly:

```
kind load docker-image localhost:5001/python-app:v1 --name kind
```

---

## 5. Install Traefik as the ingress controller

The community `ingress-nginx` project reached end of life in March 2026, so Traefik is used instead.

1. Add the chart repo:

```
helm repo add traefik https://traefik.github.io/charts
helm repo update
```

2. Install it on the node ports the cluster maps:

```
helm install traefik traefik/traefik -n traefik --create-namespace \
  --set service.spec.type=NodePort \
  --set ports.web.nodePort=30080 \
  --set ports.websecure.nodePort=30443
```

3. Let Traefik publish an address into each Ingress status. Argo CD treats an Ingress without an
   address as still progressing:

```
helm upgrade traefik traefik/traefik -n traefik --reuse-values \
  --set providers.kubernetesIngress.publishedService.enabled=false \
  --set providers.kubernetesIngress.ingressEndpoint.ip=127.0.0.1
```

4. Wait for the restart:

```
kubectl -n traefik rollout status deployment/traefik
```

5. Save the settings to a file so they are not lost later. The `tail` removes the
   `USER-SUPPLIED VALUES:` header that Helm prints:

```
helm -n traefik get values traefik | tail -n +2 > ~/kind/traefik-values.yaml
```

6. From then on, upgrade with the file:

```
helm upgrade traefik traefik/traefik -n traefik -f ~/kind/traefik-values.yaml
```

---

## 6. Package the app as a Helm chart

1. Generate the chart scaffold:

```
cd ~/kind
helm create python-app
```

2. Create the values file. `httpGet: null` removes the chart default, otherwise Helm merges the two
   probe handlers together and Kubernetes rejects the pod:

```
cat > ~/kind/my-values.yaml <<'EOF'
image:
  repository: localhost:5001/python-app
  pullPolicy: IfNotPresent
  tag: "v1"

service:
  type: ClusterIP
  port: 5000

ingress:
  enabled: true
  className: traefik
  hosts:
    - host: python.localhost
      paths:
        - path: /
          pathType: Prefix

livenessProbe:
  httpGet:
    path: /api/v1/info
    port: http

readinessProbe:
  httpGet:
    path: /api/v1/info
    port: http
EOF
```

3. Check the chart is valid:

```
helm lint ~/kind/python-app -f ~/kind/my-values.yaml
```

4. See what it will create:

```
helm template python-app ~/kind/python-app -f ~/kind/my-values.yaml | head -60
```

5. Add the host names to `/etc/hosts` so the browser can reach them:

```
echo "127.0.0.1 python.localhost" | sudo tee -a /etc/hosts
echo "127.0.0.1 argocd.localhost" | sudo tee -a /etc/hosts
```

### Direct Helm install (before Argo CD takes over)

```
helm upgrade --install python-app ~/kind/python-app -f ~/kind/my-values.yaml --wait
curl -s -H "Host: python.localhost" http://localhost/api/v1/info
```

`deploy.sh` in `~/kind` does this with checks and a verification step.

---

## 7. Install Argo CD

1. Add the chart repo:

```
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

2. Create the values file. `server.insecure: true` is needed because Traefik talks HTTP to the
   backend; without it the server serves HTTPS and the browser gets a redirect loop:

```
cat > ~/kind/argocd-values.yaml <<'EOF'
global:
  domain: argocd.localhost

configs:
  params:
    server.insecure: true

server:
  ingress:
    enabled: true
    ingressClassName: traefik
    path: /
    pathType: Prefix

dex:
  enabled: false

notifications:
  enabled: false

applicationSet:
  enabled: false
EOF
```

Dex, notifications and ApplicationSets are turned off to keep the footprint small on a laptop.

3. Install it:

```
helm install argocd argo/argo-cd -n argocd --create-namespace -f ~/kind/argocd-values.yaml --wait --timeout 600s
```

4. Check the pods:

```
kubectl -n argocd get pods
```

5. Get the initial admin password:

```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
```

6. Open http://argocd.localhost and log in as `admin`.

7. Change the password and remove the initial secret:

```
brew install argocd
argocd login argocd.localhost --username admin --plaintext
argocd account update-password
kubectl -n argocd delete secret argocd-initial-admin-secret
```

---

## 8. Put the chart in Git and push it

1. Build the repo folder:

```
mkdir -p ~/kind/gitops/charts
cp -R ~/kind/python-app ~/kind/gitops/charts/python-app
cp ~/kind/my-values.yaml ~/kind/gitops/charts/python-app/my-values.yaml
cd ~/kind/gitops
```

The values file sits inside the chart folder because Argo CD resolves value files relative to the
chart path.

2. Initialise the repo:

```
git init -b main
```

3. Set your identity if it is not set globally:

```
git config --get user.email || git config --global user.email "you@example.com"
```

4. Commit:

```
git add -A
git commit -m "Add python-app Helm chart"
```

5. Log in to GitHub:

```
gh auth login
```

Choose GitHub.com, HTTPS, then authenticate in the browser.

6. Create the repo and push in one step:

```
gh repo create kind-gitops --public --source=. --remote=origin --push
```

7. Confirm it is there:

```
gh repo view --web
```

---

## 9. Hand the app over to Argo CD

Argo CD cannot manage objects that a Helm release already owns, so remove the release first.

1. Uninstall the Helm release:

```
helm uninstall python-app
```

2. Check the objects are gone:

```
kubectl get deploy,svc,ingress -l app.kubernetes.io/instance=python-app
```

3. Write the Application manifest, filling in the repo URL automatically:

```
REPO_URL=$(cd ~/kind/gitops && gh repo view --json url -q .url)
cat > ~/kind/argocd-python-app.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: python-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: ${REPO_URL}.git
    targetRevision: main
    path: charts/python-app
    helm:
      valueFiles:
        - my-values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF
```

4. Apply it:

```
kubectl apply -f ~/kind/argocd-python-app.yaml
```

5. Watch it settle. You want `Synced` and `Healthy`:

```
kubectl -n argocd get app python-app -w
```

6. Test the app:

```
curl -s -H "Host: python.localhost" http://localhost/api/v1/info
```

---

## 10. How the continuous delivery works

Argo CD compares three things: what is in Git, what it last applied, and what is running in the
cluster.

**Sync status** answers "does the cluster match Git?" — `Synced` or `OutOfSync`.

**Health status** answers "are those objects working?" — `Healthy`, `Progressing`, `Degraded`.

The `syncPolicy` in the Application controls the behaviour:

| Setting | Effect |
| --- | --- |
| `automated` | Argo CD applies changes itself. Without it, you press Sync in the UI or run `argocd app sync`. |
| `selfHeal: true` | Anything changed directly in the cluster is reverted to what Git says. A manual `kubectl scale` is undone within seconds. |
| `prune: true` | Deleting a file or an object from the chart deletes it from the cluster on the next sync. |
| `CreateNamespace=true` | Creates the destination namespace if it is missing. |

Argo CD polls the repo every three minutes by default. A webhook removes that delay in a real setup;
locally you can force a check:

```
kubectl -n argocd patch app python-app --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"normal"}}}'
```

Use `"hard"` instead of `"normal"` to make it re-render the chart as well as re-read the repo.

The important consequence: once an app is under Argo CD, `helm upgrade` by hand is no longer the way
to change it. Git is the only input. Anything else gets reverted.

---

## 11. Day-to-day: change, commit, deploy

### Changing configuration only

1. Edit the values file in the repo:

```
open -e ~/kind/gitops/charts/python-app/my-values.yaml
```

2. Check the change:

```
cd ~/kind/gitops && git diff
```

3. Commit and push. This push is the deployment:

```
git commit -am "Scale python-app to 3 replicas"
git push
```

4. Force an immediate check rather than waiting three minutes:

```
kubectl -n argocd patch app python-app --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"normal"}}}'
```

5. Watch the result:

```
kubectl get pods -l app.kubernetes.io/instance=python-app -w
```

### Changing the application code

The image is not in Git, so it has to reach the cluster first.

1. Build with a new tag:

```
cd ~/projets/idp/python-app
docker build -t python-app:v2 .
docker tag python-app:v2 localhost:5001/python-app:v2
docker push localhost:5001/python-app:v2
```

2. Load it into the cluster:

```
kind load docker-image localhost:5001/python-app:v2 --name kind
```

3. Change the tag in the values file:

```
sed -i '' 's/tag: "v1"/tag: "v2"/' ~/kind/gitops/charts/python-app/my-values.yaml
```

4. Commit and push:

```
cd ~/kind/gitops && git commit -am "Deploy python-app v2" && git push
```

5. Refresh and watch:

```
kubectl -n argocd patch app python-app --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"normal"}}}'
kubectl rollout status deployment/python-app
```

### Rolling back

Revert the commit and push. Argo CD applies the previous state:

```
cd ~/kind/gitops
git revert HEAD
git push
```

The Argo CD UI also has a History and Rollback button, but reverting in Git keeps the repo and the
cluster telling the same story.

---

## 12. Checks that are worth knowing

1. Application status:

```
kubectl -n argocd get app python-app
```

2. Pods, with restart counts:

```
kubectl get pods -l app.kubernetes.io/instance=python-app
```

3. Service endpoints:

```
kubectl get endpointslices -l kubernetes.io/service-name=python-app
```

4. Ingress, including the published address:

```
kubectl get ingress python-app
```

5. The app itself:

```
curl -s -H "Host: python.localhost" http://localhost/api/v1/info
```

6. Prove self-healing. Scale by hand, then watch Argo CD put it back:

```
kubectl scale deployment python-app --replicas=1
kubectl get pods -l app.kubernetes.io/instance=python-app -w
```

---

## 13. Problems hit while building this, and the fixes

**`docker build` fails with "failed to compute cache key"**
The file named in `COPY` is not in the build context. Check it exists next to the Dockerfile and is
not excluded by `.dockerignore`.

**`kubectl apply` fails with "could not find expected ':'"**
The YAML was pasted with leading spaces, usually from an indented code block, so the `---`
separators are no longer at the left margin. Write the file with a `cat > file <<'EOF'` block
instead, or use `:set paste` in vim.

**Pod restarts with exit code 137**
The liveness probe is failing and the kubelet is killing the container. In this lab the probe hit
`/` and the app returned 404. Point the probes at a path the app serves, or add a `/health` route.

**`may not specify more than 1 handler type`**
Helm merged your probe override into the chart's default rather than replacing it. Set the unwanted
handler to `null` in your values file.

**Argo CD stuck on `Progressing` although the pod is fine**
The Ingress has no address, because a NodePort service never gets one. Set
`providers.kubernetesIngress.ingressEndpoint.ip` in Traefik, as in section 5 step 3.

**`helm upgrade` rejects the values file with "additional properties 'USER-SUPPLIED VALUES' not allowed"**
`helm get values` prints a header line. Strip it with `tail -n +2`, or use `-o json`.

**`deployment "python-app" not found`**
Usually the wrong cluster or namespace. Check with `kubectl config current-context` and
`kubectl get deployments -A`.

---

## 14. Start and stop the lab

Stop for the day, keeping everything:

```
docker stop $(kind get nodes --name kind) registry
```

Start again:

```
docker start registry $(kind get nodes --name kind)
```

Remove the app only:

```
kubectl delete -f ~/kind/argocd-python-app.yaml
```

Remove everything:

```
helm uninstall argocd -n argocd
helm uninstall traefik -n traefik
kind delete cluster --name kind
docker rm -f registry
```

---

## 15. Worth doing next

- Commit `traefik-values.yaml` and `argocd-values.yaml` to the gitops repo, so the whole lab can be
  rebuilt from Git after deleting the cluster.
- Add an Argo CD Application for Traefik, so the ingress controller is managed the same way as the
  app.
- Replace the Flask development server with gunicorn in the Dockerfile. The `Werkzeug` response
  header shows the development server is in use, which is fine for a lab and not for anything else.
- Look at Gateway API as the successor to Ingress. Both Traefik and the maintained NGINX controller
  support it.
