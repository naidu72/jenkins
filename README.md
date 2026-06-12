# Jenkins on k3s — ArgoCD GitOps Setup
## naidu72 homelab | DFW Pi cluster

---

## Directory Structure

```
pi-k8s/
└── apps/
    └── jenkins/
        ├── base/
        │   ├── 00-namespace.yaml       # jenkins namespace
        │   ├── 01-pvc.yaml             # 10Gi local-path PVC
        │   ├── 02-rbac.yaml            # SA + ClusterRole for agent pod creation
        │   ├── 03-deployment.yaml      # Jenkins controller deployment
        │   ├── 04-service.yaml         # ClusterIP (HTTP) + ClusterIP (JNLP)
        │   ├── 05-ingress.yaml         # jenkins.naidu72.info via ingress-nginx
        │   ├── 06-certificate.yaml     # TLS cert via cert-manager
        │   ├── 07-casc-configmap.yaml  # JCasC config (clouds, agent templates)
        │   ├── plugins.txt             # plugin list
        │   └── Dockerfile              # custom image with plugins baked in
        └── argocd/
            └── jenkins-app.yaml        # ArgoCD Application
```

---

## Step 1 — Push to your GitOps repo

```bash
# On sourceone (WSL2)
cd ~/git/pi-k8s
mkdir -p apps/jenkins/base apps/jenkins/argocd

# Copy all files from this package into the above paths
# Then commit and push
git add apps/jenkins/
git commit -m "feat: add Jenkins deployment on k3s"
git push origin main
```

---

## Step 2 — (Optional but Recommended) Build Custom Jenkins Image

This bakes all plugins into the image — no manual install on first boot.

```bash
# On sourceone (WSL2) — must have Docker running
startdocker    # your Fish function

cd ~/git/pi-k8s/apps/jenkins/base

# Multi-arch build for amd64 + arm64 (Pi is arm64)
docker buildx build \
  --platform linux/arm64,linux/amd64 \
  -t ghcr.io/naidu72/jenkins:lts \
  --push .
```

Then update `03-deployment.yaml` image line:
```yaml
image: ghcr.io/naidu72/jenkins:lts   # replace jenkins/jenkins:lts-jdk17
```

---

## Step 3 — Update ArgoCD Application

Edit `argocd/jenkins-app.yaml`:
- Confirm `repoURL` matches your pi-k8s repo URL
- Confirm `destination.server` matches your DFW Pi k3s cluster server URL

Check registered clusters in ArgoCD:
```bash
argocd cluster list
```

---

## Step 4 — Apply ArgoCD Application

```bash
# On sourceone (WSL2) — ArgoCD hub
kubectl apply -f apps/jenkins/argocd/jenkins-app.yaml -n argocd

# Watch sync status
argocd app get jenkins
argocd app sync jenkins   # force sync if needed
```

---

## Step 5 — Verify Deployment

```bash
# Watch pods come up on DFW Pi
kubectl get pods -n jenkins -w

# Expected output after ~2 minutes:
# NAME                       READY   STATUS    RESTARTS   AGE
# jenkins-<hash>             1/1     Running   0          2m

# Check PVC is bound
kubectl get pvc -n jenkins

# Check ingress
kubectl get ingress -n jenkins
kubectl get certificate -n jenkins
```

---

## Step 6 — First Login

1. Get the initial admin password:
```bash
kubectl exec -n jenkins deploy/jenkins -- \
  cat /var/jenkins_home/secrets/initialAdminPassword
```

2. Open: https://jenkins.naidu72.info/jenkins

3. **Required plugins to install via UI** (if NOT using custom image):
   - Configuration as Code
   - Kubernetes
   - Git, GitHub, GitHub Branch Source
   - Pipeline, Blue Ocean
   - (see plugins.txt for full list)

---

## Step 7 — Verify Kubernetes Cloud (Agent) Config

1. Jenkins UI → Manage Jenkins → Clouds → `k3s-cloud`
2. Click **Test Connection** — should show "Connected to Kubernetes ..."
3. The JCasC ConfigMap auto-configured this — just verify it loaded

---

## Step 8 — Run Your First Pipeline

Create a new Pipeline job and paste this Jenkinsfile:

```groovy
pipeline {
    agent {
        kubernetes {
            label 'k3s-agent'
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: jnlp
    image: jenkins/inbound-agent:latest-jdk17
    resources:
      requests:
        cpu: 200m
        memory: 256Mi
  - name: shell
    image: alpine:3.19
    command: ['sleep', 'infinity']
    tty: true
"""
        }
    }
    stages {
        stage('Hello from k3s Agent') {
            steps {
                container('shell') {
                    sh '''
                        echo "=== Running on Kubernetes agent pod ==="
                        echo "Node: $(hostname)"
                        echo "OS: $(uname -a)"
                        echo "=== SUCCESS ==="
                    '''
                }
            }
        }
    }
}
```

Watch the agent pod spin up:
```bash
kubectl get pods -n jenkins -w
```
You should see a new pod appear, run, then get deleted.

---

## Cloudflare Tunnel

Add a new public hostname in your Cloudflare Tunnel config:
```yaml
# In your existing tunnel config or via CF dashboard
- hostname: jenkins.naidu72.info
  service: http://ingress-nginx-controller.ingress-nginx.svc.cluster.local:80
```

The Tunnel already handles TLS termination at Cloudflare edge.
cert-manager will issue the internal cert for the Ingress.

---

## Troubleshooting

| Issue | Fix |
|---|---|
| Pod stuck in `Init:0/1` | Check init container logs: `kubectl logs -n jenkins deploy/jenkins -c init-permissions` |
| Jenkins UI not loading | Check readiness probe path matches `--prefix` value |
| Agent pods not spawning | Verify RBAC: `kubectl auth can-i create pods --as=system:serviceaccount:jenkins:jenkins -n jenkins` |
| JCasC not loading | Check env var `CASC_JENKINS_CONFIG` and ConfigMap mount |
| TLS cert not issuing | `kubectl describe certificate jenkins-tls -n jenkins` — check ClusterIssuer name |

---

## What's Next (Phase 2 Hands-on)

- [ ] Multibranch Pipeline from naidu72 GitHub repo
- [ ] SonarQube integration (deploy SonarQube on k3s)
- [ ] Nexus artifact repository
- [ ] Vault credentials injection via HashiCorp Vault plugin
- [ ] GitHub webhook → auto-trigger builds on push
- [ ] Jenkins + ArgoCD GitOps bridge (Jenkins builds image → updates Helm values → ArgoCD deploys)
