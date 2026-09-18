# Argo CD GitOps Lab — From Scratch

## Objective

In this lab:

```text
Kubernetes Cluster
      ↓
Install Argo CD
      ↓
Access Argo CD UI
      ↓
Connect Private GitHub Repository
      ↓
Create Argo CD Application
      ↓
Deploy Kubernetes Application
      ↓
Change Git → Argo CD Sync
      ↓
Manual Cluster Change → Argo CD Self-Heal
```

## 1. Prerequisites

You only need:

```text
✓ Running Kubernetes cluster
✓ kubectl configured to access the cluster
✓ GitHub account
```

Verify the cluster:

```bash
kubectl get nodes
```

Expected:

```text
NAME            STATUS   ROLES    AGE
worker-node-1   Ready    <none>   10m
worker-node-2   Ready    <none>   10m
```

---

# 2. Install Argo CD

Create the Argo CD namespace:

```bash
kubectl create namespace argocd
```

Install Argo CD:

```bash
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Check the pods:

```bash
kubectl get pods -n argocd
```

Wait until the pods are running:

```bash
kubectl get pods -n argocd -w
```

You should eventually see components such as:

```text
argocd-application-controller
argocd-applicationset-controller
argocd-dex-server
argocd-notifications-controller
argocd-redis
argocd-repo-server
argocd-server
```

---

# 3. Get Argo CD Admin Password

The default username is:

```text
admin
```

Get the initial password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

For a newline afterward:

```bash
echo
```

Save the password temporarily for login.

---

# 4. Access Argo CD UI

You have several options.

## Option 1 — Port Forwarding

This is easiest for a lab.

```bash
kubectl port-forward svc/argocd-server \
  -n argocd 8080:443
```

Open:

```text
https://localhost:8080
```

Your browser may display a certificate warning because Argo CD uses HTTPS with its own certificate by default.

Login with:

```text
Username: admin
Password: <password-from-secret>
```

Keep the terminal running while using the UI.

---

# 5. Alternative — NodePort

For a lab cluster where you want to access Argo CD through a worker node:

```bash
kubectl patch svc argocd-server \
  -n argocd \
  -p '{"spec":{"type":"NodePort"}}'
```

Check:

```bash
kubectl get svc argocd-server -n argocd
```

Example:

```text
NAME            TYPE       PORT(S)
argocd-server   NodePort   80:31234/TCP,443:31876/TCP
```

Get your node IP:

```bash
kubectl get nodes -o wide
```

Then access:

```text
https://<NODE-IP>:<HTTPS-NODEPORT>
```

Your firewall/security group must allow that NodePort if you're accessing it externally.

---

# 6. Alternative — LoadBalancer

If your Kubernetes environment supports `LoadBalancer` services:

```bash
kubectl patch svc argocd-server \
  -n argocd \
  -p '{"spec":{"type":"LoadBalancer"}}'
```

Check:

```bash
kubectl get svc argocd-server -n argocd
```

Watch until an external address is assigned:

```bash
kubectl get svc argocd-server -n argocd -w
```

Then access the Argo CD server using the external hostname/IP.

For a simple classroom demo, **port forwarding is easiest**.

---

# 7. Create Private GitHub Repository

Create a **private** repository named:

```text
nks-argocd
```

Clone it:

```bash
git clone git@github.com:<YOUR-GITHUB-USERNAME>/nks-argocd.git

cd nks-argocd
```

Create this structure:

```text
nks-argocd/
│
└── application/
    ├── deployment.yaml
    └── service.yaml
```

Create the folder:

```bash
mkdir application
```

---

# 8. Create Deployment Manifest

Create:

```text
application/deployment.yaml
```

Add:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nks-web
spec:
  replicas: 2

  selector:
    matchLabels:
      app: nks-web

  template:
    metadata:
      labels:
        app: nks-web

    spec:
      containers:
        - name: web
          image: nginx:1.27
          ports:
            - containerPort: 80
```

Notice that we're not hardcoding a namespace here. We'll let the Argo CD Application determine the destination namespace.

---

# 9. Create Service Manifest

Create:

```text
application/service.yaml
```

Add:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nks-web-service

spec:
  selector:
    app: nks-web

  ports:
    - protocol: TCP
      port: 80
      targetPort: 80

  type: ClusterIP
```

---

# 10. Push Application to GitHub

```bash
git add .

git commit -m "Add Kubernetes manifests"

git push origin main
```

At this point GitHub contains:

```text
nks-argocd
│
└── application
    ├── deployment.yaml
    └── service.yaml
```

But nothing has been deployed yet.

Argo CD doesn't know about this repository/application yet.

---

# 11. Give Argo CD Access to Private GitHub Repo

Because the repository is private, Argo CD needs authentication.

For a lab, you can use a GitHub Personal Access Token with access to this repository.

In the Argo CD UI go to:

```text
Settings
   ↓
Repositories
   ↓
Connect Repo
```

Select HTTPS authentication and enter approximately:

```text
Repository Type: git

Repository URL:
https://github.com/<YOUR-GITHUB-USERNAME>/nks-argocd.git

Username:
<YOUR-GITHUB-USERNAME>

Password:
<GITHUB-PERSONAL-ACCESS-TOKEN>
```

Connect the repository.

You want the repository connection to show as successful.

Do **not** put the PAT in your Kubernetes application manifests or commit it to Git.

---

# 12. Create Argo CD Application Manifest

Now add another folder:

```bash
mkdir argocd
```

Create:

```text
argocd/application.yaml
```

Your repository becomes:

```text
nks-argocd/
│
├── application/
│   ├── deployment.yaml
│   └── service.yaml
│
└── argocd/
    └── application.yaml
```

Add:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application

metadata:
  name: nks-web
  namespace: argocd

spec:
  project: default

  source:
    repoURL: https://github.com/<YOUR-GITHUB-USERNAME>/nks-argocd.git

    targetRevision: main

    path: application

  destination:
    server: https://kubernetes.default.svc

    namespace: dev

  syncPolicy:
    automated:
      prune: true
      selfHeal: true

    syncOptions:
      - CreateNamespace=true
```

Replace:

```text
<YOUR-GITHUB-USERNAME>
```

with your GitHub username.

---

# 13. Bootstrap Argo CD

Commit the Argo CD manifest:

```bash
git add .

git commit -m "Add Argo CD application"

git push origin main
```

There is an important concept here:

**Committing `argocd/application.yaml` alone will not create the Argo CD Application yet.**

Argo CD isn't currently monitoring that file.

So for the initial bootstrap, apply it manually:

```bash
kubectl apply -f argocd/application.yaml
```

This is your bootstrap step.

```text
ONE-TIME BOOTSTRAP

kubectl apply
      ↓
Argo CD Application
      ↓
Private GitHub Repo
      ↓
application/
      ↓
Deployment + Service
      ↓
dev namespace
```

---

# 14. Check Argo CD Application from CLI

Run:

```bash
kubectl get applications -n argocd
```

You should see something similar to:

```text
NAME      SYNC STATUS   HEALTH STATUS
nks-web   Synced        Healthy
```

Check your namespace:

```bash
kubectl get namespace dev
```

Then:

```bash
kubectl get all -n dev
```

You should see:

```text
deployment.apps/nks-web
pod/nks-web-xxxxx
pod/nks-web-yyyyy
service/nks-web-service
```

Check Deployment:

```bash
kubectl get deployment -n dev
```

Check Pods:

```bash
kubectl get pods -n dev
```

Check Service:

```bash
kubectl get service -n dev
```

---

# 15. Check Through Argo CD UI

Access the UI again:

```bash
kubectl port-forward svc/argocd-server \
  -n argocd 8080:443
```

Open:

```text
https://localhost:8080
```

You should see:

```text
nks-web

Sync Status
────────────
Synced

Health
──────
Healthy
```

Open `nks-web`.

Argo CD will visually show the resource relationship, roughly:

```text
nks-web
   │
   ├── Deployment
   │       │
   │       └── ReplicaSet
   │              │
   │              ├── Pod
   │              └── Pod
   │
   └── Service
```

This resource tree is one of the most useful parts of the Argo CD UI.

---

# 16. GitOps Exercise 1 — Change Image in Git

Now test automatic synchronization.

Your current Deployment has:

```yaml
image: nginx:1.27
```

Change it to another valid tag, for example:

```yaml
image: nginx:1.28
```

Then:

```bash
git add application/deployment.yaml

git commit -m "Update nginx image"

git push origin main
```

Now don't run:

```bash
kubectl apply
```

That's the point of the exercise.

Your flow is:

```text
Developer
    ↓
Change deployment.yaml
    ↓
git commit
    ↓
git push
    ↓
GitHub
    ↓
Argo CD detects difference
    ↓
Automatic Sync
    ↓
Deployment updated
    ↓
New Pods created
```

---

# 17. Watch Argo CD Perform the Deployment

In another terminal:

```bash
kubectl get pods -n dev -w
```

You should see the rolling update.

You can also check:

```bash
kubectl rollout status deployment/nks-web -n dev
```

Then verify the image:

```bash
kubectl get deployment nks-web \
  -n dev \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

You should see the new image.

In the Argo CD UI, you can also open the application and inspect the Deployment.

---

# 18. GitOps Exercise 2 — Change Replicas in Git

Currently:

```yaml
replicas: 2
```

Change:

```yaml
replicas: 4
```

Commit:

```bash
git add application/deployment.yaml

git commit -m "Scale application to 4 replicas"

git push origin main
```

Don't run `kubectl apply`.

Watch:

```bash
kubectl get pods -n dev -w
```

Eventually:

```bash
kubectl get deployment -n dev
```

should show the desired replica count.

This demonstrates:

```text
Git = Desired State

Git says:
replicas: 4

        ↓

Argo CD

        ↓

Kubernetes:
4 Pods
```

---

# 19. GitOps Exercise 3 — Manual Cluster Change

Now let's intentionally break the GitOps model.

Git says:

```yaml
replicas: 4
```

Manually change the live cluster:

```bash
kubectl scale deployment nks-web \
  --replicas=1 \
  -n dev
```

Immediately check:

```bash
kubectl get deployment nks-web -n dev
```

For a short period, you may see the manual change.

But your Argo CD configuration has:

```yaml
selfHeal: true
```

Argo CD compares:

```text
Git Desired State
replicas = 4

        VS

Cluster Live State
replicas = 1
```

Argo CD detects drift:

```text
Git
replicas: 4
        │
        ▼
     Argo CD
        │
        │ Drift detected
        ▼
Kubernetes
replicas: 1
        │
        ▼
SELF HEAL
        │
        ▼
replicas: 4
```

Watch:

```bash
kubectl get deployment nks-web -n dev -w
```

Argo CD should bring the Deployment back toward the desired Git state.

This is **self-healing**.

---

# 20. GitOps Exercise 4 — Manually Change the Image

Git says:

```yaml
image: nginx:1.28
```

Manually modify Kubernetes:

```bash
kubectl set image deployment/nks-web \
  web=nginx:1.27 \
  -n dev
```

Check:

```bash
kubectl get deployment nks-web \
  -n dev \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

You temporarily changed the live cluster.

Argo CD sees:

```text
Git                    Kubernetes

nginx:1.28             nginx:1.27
     │                      │
     └──────────┬───────────┘
                ↓
           OUT OF SYNC
                ↓
            SELF HEAL
                ↓
           nginx:1.28
```

After reconciliation, verify again:

```bash
kubectl get deployment nks-web \
  -n dev \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

---

# 21. GitOps Exercise 5 — Delete a Pod

Delete one of your Pods:

```bash
kubectl get pods -n dev
```

Then:

```bash
kubectl delete pod <POD-NAME> -n dev
```

A new Pod will appear.

But this particular behavior is primarily **Kubernetes Deployment/ReplicaSet reconciliation**, not an Argo CD self-heal demonstration.

The Deployment says:

```text
4 replicas required
```

so Kubernetes recreates the missing Pod.

Use the **replica/image manual changes** above when teaching Argo CD self-healing.

---

# 22. GitOps Exercise 6 — Test `prune`

This is another important Argo CD feature.

You configured:

```yaml
automated:
  prune: true
  selfHeal: true
```

Git currently contains:

```text
application/
├── deployment.yaml
└── service.yaml
```

Delete the Service manifest from Git:

```bash
git rm application/service.yaml

git commit -m "Remove service"

git push origin main
```

Argo CD now sees:

```text
Previously Git wanted:

Deployment
Service


Now Git wants:

Deployment
```

Because:

```yaml
prune: true
```

Argo CD can remove the Service from the cluster.

Check:

```bash
kubectl get service -n dev
```

Restore it afterward for the rest of your lab:

```bash
git restore --source=HEAD~1 application/service.yaml

git add application/service.yaml

git commit -m "Restore service"

git push origin main
```

---

# 23. Important Concepts Students Should Understand

There are three behaviors worth separating.

| Action                       | Who handles it?                  |
| ---------------------------- | -------------------------------- |
| Commit new image to Git      | Argo CD sync                     |
| Change replicas manually     | Argo CD self-heal                |
| Delete Service YAML from Git | Argo CD prune                    |
| Delete a Pod                 | Kubernetes Deployment/ReplicaSet |

The most important idea is:

```text
          Git Repository
         DESIRED STATE
               │
               │
               ▼
            Argo CD
               │
        Compare & Sync
               │
               ▼
       Kubernetes Cluster
          LIVE STATE
```

**Git becomes the source of truth.**

---

# 24. Useful CLI Commands

Check Argo CD:

```bash
kubectl get pods -n argocd
```

Check Applications:

```bash
kubectl get applications -n argocd
```

More details:

```bash
kubectl describe application nks-web -n argocd
```

Check application resources:

```bash
kubectl get all -n dev
```

Watch Pods:

```bash
kubectl get pods -n dev -w
```

Check image:

```bash
kubectl get deployment nks-web \
  -n dev \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

Check replicas:

```bash
kubectl get deployment nks-web -n dev
```

---

# 25. Final Repository Structure

At the end, your private `nks-argocd` repository should look like:

```text
nks-argocd/
│
├── application/
│   ├── deployment.yaml
│   └── service.yaml
│
└── argocd/
    └── application.yaml
```

The operational model is:

```text
                 PRIVATE GITHUB
                   nks-argocd
                       │
                       │
              ┌────────┴────────┐
              │                 │
        application/          argocd/
              │                 │
       deployment.yaml    application.yaml
       service.yaml
              │
              ▼
           Argo CD
              │
       Compare Desired
        vs Live State
              │
              ▼
      Kubernetes Cluster
              │
              ▼
        dev namespace
              │
       ┌──────┴──────┐
       ▼             ▼
   Deployment      Service
       │
       ▼
      Pods
```

### The one-time bootstrap rule

For this lab, remember:

```text
FIRST TIME

Install Argo CD
      ↓
Connect Private Repo
      ↓
kubectl apply -f argocd/application.yaml
      ↓
Argo CD takes control


AFTER THAT

Edit YAML
   ↓
Commit
   ↓
Push
   ↓
Argo CD
   ↓
Kubernetes
```

That final distinction is the core of the lab: **you use `kubectl` to bootstrap Argo CD, but after onboarding the application, normal application changes should flow through Git rather than direct `kubectl apply` commands.**
