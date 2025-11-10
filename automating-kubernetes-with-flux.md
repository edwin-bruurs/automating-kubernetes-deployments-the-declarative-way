---
marp: true
theme: gaia
class:
  - invert
paginate: true

style: |
  section {
    background-image: url(images/background.jpg);
    background-repeat: no-repeat;
    background-size: 100% 100%;
    font-family: "Montserrat";
    color: #fff;
  }
  h1,h2 {
    color: #75fbe3;
    text-align: center;
  }
  p, li {
    font-size: 70%;
  }
  footer {
    font-family: "Montserrat";
    font-size: 20px;
    margin: 5% 5%;
  }
  pre {
    code {
      font-size: 0.4em;
    }
  }
  table {
    font-size: 60%;
  }
---
<!-- 
_class: [invert, lead] 
paginate: false
-->
## Automating Kubernetes Deployments the Declarative Way

![bg right:35% 60%](images/fluxcd.svg)

<!-- _footer: Edwin Bruurs | CM.com DevDays 2025-->

<!--
- Welcome everyone to the workshop on automating Kubernetes deployments
- Today we'll focus on GitOps and specifically Flux
- This is a hands-on workshop, so please have your laptops ready
- We'll be working locally with your preferred tooling (Rancher Desktop, Minikube, Kind, etc.)
- By the end, you'll be able to set up a complete GitOps workflow
- Ask questions anytime - this is interactive!
-->

---
## Agenda

- Introduction to GitOps
- What is Flux?
- Flux Architecture
- Hands-on: Setting up Flux
- Deploying Applications
- Q&A

<!--
- We have about 1.5 hours for this workshop
- We start with a bit of theory and concepts
- Second part is hands-on with Flux installation and setup
- Third part is on deploying applications using Flux
- We'll take a 10-minute break halfway through
- All materials will be available on GitHub after the workshop
-->

---
## The Problem: Manual Deployments

- **kubectl apply -f** everywhere
- No audit trail of who changed what
- Difficult to rollback changes
- Configuration drift between environments
- No single source of truth

<!--
- Show of hands: who uses kubectl apply directly in production?
- Let's talk about the pain points we all experience
- Manual deployments lead to "works on my machine" syndrome
- Configuration drift is when dev/staging/prod environments become different
- When something breaks at 3 AM, how do you know what changed?
- These problems get exponentially worse as teams grow
-->
---

## What is GitOps?

**GitOps = Infrastructure as Code + Pull Requests + CI/CD**

- Git as the single source of truth
- Declarative infrastructure and applications
- Automated deployment through Git commits
- Continuous reconciliation
- Easy rollbacks with Git revert

<!--
- GitOps term coined by Weaveworks in 2017
- It's not just about automation - it's about culture change
- Everything in Git: apps, config, infrastructure
- Declarative means you describe WHAT you want, not HOW to get there
- Reconciliation: system continuously checks if reality matches desired state
- Git revert is your "undo button" for production
- Key principle: Git commit = deployment trigger
-->

---
## Why GitOps?
**Version Control** - Every change is tracked
**Auditability** - Every change is a Git commit
**Collaboration** - Review changes via Pull Requests
**Disaster Recovery** - Git is your backup
**Consistency** - Same process for all environments

<!--
- Version control: git log shows you everything that happened
- Auditability: compliance teams love this - complete audit trail
- Collaboration: your deployment process becomes code review
- DR story: company lost entire cluster, restored from Git in 30 minutes
- Consistency: no more "special snowflake" deployments
-->

---
## Introducing Flux

**Flux is a GitOps operator for Kubernetes**

- Open source (CNCF Graduated Project)
- Continuous delivery from Git repositories
- Automatic synchronization
- Progressive delivery with Flagger
- Multi-tenancy support

<!--
- Flux is one of the most popular GitOps tools
- CNCF Graduated (June 2022) - same level as Kubernetes, Prometheus
- Originally created by Weaveworks, now community-driven
- Version 2 (Flux v2) is a complete rewrite - what we use today
- Supports Git, Helm, OCI registries as sources
- Works with any Kubernetes distribution
- Used by companies like Adobe, Mercedes-Benz, SAP and CM.com
-->

---
## Flux Architecture

![bg 80%](images/fluxcd-controllers.png)

<!--
- Flux runs INSIDE your Kubernetes cluster
- Pull-based model: Flux pulls from Git, not pushed to
- This is different from traditional CI/CD (Jenkins, GitLab CI)
- Controllers watch Git repo for changes
- When changes detected, controllers reconcile cluster state
- No external system needs cluster credentials - more secure
- Works even if cluster is behind firewall
- Flux can manage itself - it's bootstrapped
-->

---
## Flux Core Components

![bg right:33% 80%](images/fluxcd-controllers.png)
- **Source Controller** - Manages Git/Helm repositories
- **Kustomize Controller** - Applies Kustomize configurations
- **Helm Controller** - Manages Helm releases
- **Notification Controller** - Event forwarding and webhooks
- **Image Automation** - Updates container images

<!--
- Each controller is a separate deployment in flux-system namespace
- Source Controller: pulls from Git/Helm repos, OCI registries
- Kustomize Controller: applies raw manifests and Kustomize overlays
- Helm Controller: manages Helm chart deployments (HelmRelease CRD)
- Notification Controller: sends alerts to Slack, Teams, etc.
- Image Automation: scans registries, updates image tags in Git
- All controllers are loosely coupled - can use independently
- They communicate via Kubernetes CRDs (Custom Resource Definitions)
-->

---

## Prerequisites

- Kubernetes cluster (Rancher Desktop, kind, k3s or cloud)
- kubectl installed and configured
- GitHub account and personal access token
- Flux CLI installed

```bash
# Install (macOS)
brew install kubectl
brew install --cask rancher
brew install fluxcd/tap/flux

# Install (Windows)
choco install flux
choco install kubernetes-cli
choco install rancher-desktop

# Verify installation
flux --version
```

<!--
Speaker Notes:
- I'll use Rancher Desktop (opensource Docker Desktop alternative) for this workshop
- GitHub PAT needs repo permissions - I'll show you how
- Flux CLI is required - not just kubectl
- Flux works the same regardless of Kubernetes distribution
- Check who needs help with prerequisites before continuing
- Flux CLI can bootstrap, manage, troubleshoot Flux
-->

---

## Lab Setup

We'll create a local Kubernetes cluster:

```bash
# Using Rancher Desktop
rdctl start --kubernetes.enabled --kubernetes.version v1.33.5 --container-engine.name containerd

# Set kubeconfig for this workshop
export KUBECONFIG=$HOME/.kube/config

# Verify cluster
kubectl cluster-info
kubectl get nodes
```

**Be careful if you already have a cluster running! Some settings can reset existing your existing cluster**
<!--
Speaker Notes:
- Let's do this together - follow along on your laptops
- Rancher Desktop creates a cluster in about 30-60 seconds
- This gives us a real Kubernetes cluster to work with
- One node is enough for this workshop
- In production, you'd have multiple nodes across zones
- Make sure everyone has a running cluster before proceeding
-->

---

## Flux Installation

1. Export GitHub credentials:

```bash
# GitHub Personal Access Token (classic) with permissions repo
export GITHUB_TOKEN=<your-token> # ghp_XXXXXXXXXXXXXXXXXXXXXX
export GITHUB_USER=<your-github-username> # edwin-bruurs_cmcom
```

2. Run pre-flight check:

```bash
flux check --pre
```

<!--
Speaker Notes:
- GitHub token: Settings > Developer settings > Personal access tokens
- Token needs: repo, admin:repo_hook permissions
- NEVER commit tokens to Git - use environment variables
- Pre-flight check verifies: Kubernetes version, connectivity, etc.
- If pre-flight fails, don't proceed - fix issues first
- Common issue: KUBECONFIG not set correctly
- Flux can also work with GitLab, Bitbucket, Gitea
-->


---
## Bootstrap Flux

```bash
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=workshop-devdays-2025 \
  --branch=main \
  --path=./clusters/local \
  --personal \
  --private
```

**What just happened?**
- Created GitHub repository
- Created deploy token and stored in Kubernetes secret `kubectl get secret -n flux-system flux-system`
- Installed Flux components
- Configured GitOps workflow

**Since Flux 2.6 GitHub Apps are preferred over Personal Access Tokens for better security and management. Consider using GitHub Apps in production setups.**

<!--
Speaker Notes:
- Bootstrap is a one-time setup command
- Creates a new GitHub repo called "workshop-devdays-2025"
- --personal flag is for personal GitHub accounts (not org)
- Installs all Flux controllers in flux-system namespace
- Creates manifests in the repo that define Flux itself
- This is "Flux managing Flux" - very meta!
- Path defines where cluster config lives in repo
- Can bootstrap multiple clusters to same repo, different paths
- Wait for this to complete - takes about 1-2 minutes
- Check GitHub to see the new repository
-->

---

## Verify Flux Installation

```bash
# Check Flux components
flux check

# View Flux pods
kubectl get pods -n flux-system

# Watch reconciliation
flux get sources git
flux get kustomizations
```

<!--
Speaker Notes:
- flux check shows health of all components
- Should see all controllers running (green checkmarks)
- Four main pods: source, kustomize, helm, notification controllers
- flux get sources git shows the Git repository being watched
- flux get kustomizations shows what's being applied
- Reconciliation happens every 1 minute by default (configurable)
- If something is wrong, flux check will tell you what
- Common issue: GitHub token permissions insufficient
-->

---
## Repository Structure

Clone the created repository: 
```bash
git clone https://github.com/${GITHUB_USER}/workshop-devdays-2025.git
```

```text
workshop-devdays-2025/
└── clusters/
    └── local/
        └── flux-system/
            ├── gotk-components.yaml
            ├── gotk-sync.yaml
            └── kustomization.yaml

```

<!--
Speaker Notes:
- Let's look at the structure Flux created
- clusters/local: environment-specific configuration
- flux-system: Flux's own configuration (it manages itself!)
- gotk-components.yaml: Flux controllers manifests
- gotk-sync.yaml: Git repository source definition
- kustomization.yaml: Kustomize configuration for Flux
- You can customize these files to change Flux behavior
-->

---
## Flux Deployment Methods

Flux supports two primary ways to deploy applications:

**1. Kustomize-based deployments**
- Raw Kubernetes YAML manifests
- Kustomize overlays for environment-specific configs
- Native Kubernetes resources

**2. Helm-based deployments**
- Helm charts from repositories
- Values files for customization
- Packaged applications

<!--
Speaker Notes:
- Flux is flexible - choose the method that fits your needs
- Kustomize: simpler, more transparent, good for custom apps
- Helm: powerful for third-party apps, reusable packages
- You can mix both in the same cluster
- Most teams use Kustomize for their apps, Helm for third-party applications
- Decision factors: team expertise, app complexity, reusability
- Kustomize is closer to "plain YAML" - easier to understand
- Helm adds abstraction layer - templates with variables
- Both are first-class citizens in Flux
- We'll explore both in detail
-->

---
## Kustomize vs Helm: When to Use Which?

| Aspect              | Kustomize                   | Helm                           |
| ------------------- | --------------------------- | ------------------------------ |
| **Complexity**      | Simple, declarative         | Templates, more complex        |
| **Learning Curve**  | Low                         | Medium-High                    |
| **Best For**        | Custom apps, simple configs | Third-party apps, complex apps |
| **Reusability**     | Environment overlays        | Chart packaging                |
| **Templating**      | Patches & overlays          | Go templates                   |
| **Version Control** | Direct YAML files           | Chart versions                 |

**Recommendation:** Start with Kustomize, add Helm when needed

<!--
Speaker Notes:
- This table helps you decide which to use
- Kustomize: you see exactly what gets deployed
- Helm: templates can be harder to debug
- Learning curve: Kustomize is YAML + patches
- Helm requires learning templating language
- Best for: Kustomize for apps you build
- Helm for apps you consume (Redis, PostgreSQL, Grafana)
- Reusability: Kustomize uses base + overlays pattern
- Helm uses chart values for different deployments
- Templating: Kustomize patches existing resources
- Helm generates resources from templates
- Version control: Kustomize = readable diffs in Git
- Helm charts = versioned packages
- At CM.com we use both: Kustomize for services, Helm for infrastructure
- Don't overcomplicate: start simple with Kustomize
- Add Helm only when you need packaged charts
- Many teams successfully use only Kustomize
-->

---
## Create Your First Application

Add the following files to your repository under `apps/base/podinfo/`:
```yaml
# apps/base/podinfo/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: podinfo
resources:
  - namespace.yaml
  - github.com/stefanprodan/podinfo/kustomize
```

```yaml
# apps/base/podinfo/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: podinfo
```

<!--
- Now let's create our first application using Kustomize
- This follows the Kustomize pattern: base + overlays
- base: shared configuration across all environments
- DRY principle: Don't Repeat Yourself

- We're using podinfo - a simple demo app perfect for testing
- Navigate to your cloned repository locally
- Create the directory structure: mkdir -p apps/base/podinfo
- The kustomization.yaml will be the entry point for Flux's Kustomize
- kustomization.yaml in base lists resources
- Notice we're setting a namespace - all resources will go there
- We're using Kustomize's remote resources feature here
- The resources section pulls podinfo manifests from Stefan's repo
- namespace.yaml ensures the namespace exists before deployment
- Alternative: you could copy the manifests into your repo
-->

---
## Create the overlays
Let's assume we want to create a development and production overlay for the podinfo application.
Instead of deploying the podinfo application in the podinfo namespace we deploy it in both the development and production namespaces.

Create the following files:

```yaml
# apps/development/podinfo/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base/podinfo
namespace: development
```

```yaml
# apps/production/podinfo/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base/podinfo
namespace: production
```

<!--
- Overlay references the base
- patches.yaml contains environment-specific changes
- Example patches: increase replicas from 1 to 3
- Can patch: replicas, resources, images, env vars
- Strategic merge patch or JSON patch formats
- Production might add: more memory, anti-affinity rules
- Staging might use: different database connection strings
- This is how you maintain DRY across environments
- Test overlays: kustomize build production/nginx
-->
---

## Deploying an Application

Let's instruct Flux to deploy our development and production overlays by creating the following Kustomization resources

```yaml
# clusters/local/apps-development.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: development
  namespace: flux-system
spec:
  interval: 5m
  path: ./apps/development
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
```

---
## Deploying an Application

```yaml
# clusters/local/apps-production.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: production
  namespace: flux-system
spec:
  interval: 5m
  path: ./apps/production
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
```

<!--
Speaker Notes:
- Kustomization (Flux) is different from kustomization.yaml (Kustomize tool)
- This tells Flux WHAT to deploy from the GitRepository
- interval: how often to reconcile (5 minutes)
- path: where in the repo to find manifests
- prune: delete resources if removed from Git (be careful!)
- sourceRef links to our GitRepository from previous slide
- Flux will apply everything in ./kustomize directory
- If manifests change in Git, Flux automatically updates cluster
- This is the "GitOps magic" - Git commit = deployment
-->

---
## Some command to speed up the process

Reconcile immediately instead of waiting for the next interval (1 minute for the GitRepository and 10 minutes for the Kustomization):
```bash
flux reconcile kustomization --with-source flux-system
```

---

## Hands-on Exercise #1

**Deploy only one replica in development**

Update your overlay to deploy only one replica in the development environment while keeping the production environment with the default number of replicas.

* **Tip:** Use a Kustomize patch in the development overlay to modify the replica count.
* **Tip:** What about the HorizontalPodAutoscaler?

<!--
Speaker Notes:
- Now you'll do this yourself - hands-on time!
- I'll walk around to help anyone who gets stuck
- Time box: 5 minutes for this exercise
- We'll review solutions together afterward
-->

---
## Solution Exercise #1

Multiple options are possible, Ill be using the inline patch (JSON 6902) method.

```yaml
# apps/development/kustomization.yaml

# Add the following section to the existing file
...
patches:
  - target:
      kind: Deployment
      name: podinfo
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 1
  - target:
      kind: HorizontalPodAutoscaler
      name: podinfo
    patch: |-
      - op: replace
        path: /spec/minReplicas
        value: 1
```

---
## Solution Exercise #1

And the strategic merge patch solution.

```yaml
# apps/development/kustomization.yaml

# Add the following section to the existing file
...
patches:
  - patch: |-
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: podinfo
      spec:
        replicas: 1
  - patch: |-
      apiVersion: autoscaling/v2
      kind: HorizontalPodAutoscaler
      metadata:
        name: podinfo
      spec:
        minReplicas: 1
```

---
## Helm Integration

Flux can manage Helm releases, let's define a Helm Repository for Bitnami charts.

```yaml
# repositories/bitnami.yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: bitnami
  namespace: flux-system
spec:
  interval: 1h
  url: https://charts.bitnami.com/bitnami
```

```yaml
# repositories/kustomization.yaml
apiVersion: kustomize.k8s.io/v1beta1
kind: Kustomization
resources:
  - bitnami.yaml
```

<!--
Speaker Notes:
- Many apps are packaged as Helm charts
- Flux has first-class Helm support
- HelmRepository watches a Helm chart repository
- Similar to GitRepository but for Helm charts
- interval: how often to check for new chart versions
- Can use public repos (Bitnami, stable) or private
- Helm Controller will manage releases
- Next slide shows how to deploy a chart
- This gives you GitOps for Helm too
-->
---

## Helm Release Deployment

We will deploy Sealed Secrets as part of the core infrastructure for the applications.
Both our development and production overlays will make use of Sealed Secrets to manage sensitive information.

```yaml
# infrastructure/sealed-secrets.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: sealed-secrets
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: sealed-secrets
  namespace: sealed-secrets
spec:
  interval: 5m
  chart:
    spec:
      chart: sealed-secrets
      version: "2.5.*"
      sourceRef:
        kind: HelmRepository
        name: bitnami
```

<!--
- HelmRelease is like "helm install" as a resource
- chart.spec.chart: which chart to deploy
- sourceRef: which HelmRepository to use from
- values: override chart default values
- Flux runs helm upgrade on every reconciliation
- Can specify chart version or use latest
- Helm Controller handles rollbacks on failures
- Status shows if release is healthy
- This is GitOps for Helm - values in Git
- Much better than helm install commands
-->

---
## Helm Release Deployment

Create the Kustomization combining the resources needed.

```yaml
# infrastructure/kustomization.yaml
apiVersion: kustomize.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - sealed-secrets.yaml
```

This is optional; Flux Kustomization assumes `kustomize create --autodetect` when no `kustomization.yaml` is present in the directory.

---
## Wire it all together

Finally we need to instruct Flux to deploy the repository and infrastructure Kustomization.

```yaml
# clusters/local/infrastructure.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: infrastructure
  namespace: flux-system
spec:
  dependsOn:
    - name: repositories
  interval: 5m
  path: ./infrastructure # Looks for kustomization.yaml or `kustomize create --autodetect`
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
```
**Create the repositories Kustomization as well if you haven't already done so.**
**Apply all changes and reconcile immediately.**

---
### Wire it all together
```yaml
# clusters/local/repositories.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: repositories
  namespace: flux-system
spec:
  interval: 5m
  path: ./repositories
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
```

---
## Secrets Management

- Never commit secrets to Git
- Use Sealed Secrets or External Secrets
- Rotate credentials regularly

<!--
Speaker Notes:
- Security is critical in GitOps
- Secrets in Git is a common mistake - DON'T DO IT
- Sealed Secrets: encrypt secrets, safe to commit
- External Secrets: reference secrets from vault, AWS Secrets Manager
- Rotate credentials often - don't hardcode
-->

---
## Managing Secrets with Sealed Secrets

You can store the output of the following command in your overlay.

```bash
# Create sealed secret
echo -n mypassword | kubectl \
  create secret generic mysecret \
  --namespace development \
  --dry-run=client \
  --from-file=password=/dev/stdin \
  -o yaml | kubeseal \
  --controller-namespace sealed-secrets \
  --controller-name sealed-secrets \
  --format yaml > sealed-secret.yaml
```

<!--
Speaker Notes:
- Sealed Secrets is one popular solution
- Controller runs in cluster with private key
- You encrypt with public key (safe to share)
- Only cluster can decrypt with private key
- kubeseal CLI does the encryption
- SealedSecret resource safe to commit to Git
- Controller decrypts to create actual Secret
- Alternative: Mozilla SOPS with age/gpg
- Or: External Secrets Operator (recommended for production)
- Backup the sealing key securely!
-->

---

## Monitoring Flux

```bash
# Check reconciliation status
flux get all

# Get specific resources
flux get sources git
flux get kustomizations
flux get helmreleases

# Suspend/Resume reconciliation
flux suspend kustomization <name>
flux resume kustomization <name>
```

<!--
Speaker Notes:
- Flux CLI is your main troubleshooting tool
- flux get all: overview of everything Flux manages
- Shows status: Ready, Reconciling, Failed
- flux logs shows controller logs
- suspend: temporarily stop reconciliation (for maintenance)
- resume: start reconciliation again
- Use suspend when making manual changes
- Or when debugging issues
- Can also use kubectl describe on Flux resources
- Prometheus metrics available for monitoring
-->

---

## Troubleshooting

```bash
# View events
flux events

# Check logs
flux logs --level=error

# Reconcile immediately
flux reconcile kustomization flux-system --with-source

# Describe resource
kubectl describe gitrepository <name> -n flux-system
```

<!--
Speaker Notes:
- When things go wrong (and they will)
- flux events shows recent Flux activity
- flux logs --level=error shows only errors
- --all-namespaces to see everything
- reconcile --with-source forces immediate check
- Useful when you can't wait for interval
- kubectl describe shows detailed status and events
- Common issues we'll cover on next slide
- Check .status.conditions for detailed info
- Status tells you exactly what's wrong
-->

---

## Flux vs Other GitOps Tools

| Feature          | Flux                 | ArgoCD          |
| ---------------- | -------------------- | --------------- |
| Architecture     | Pull-based           | Pull-based      |
| UI               | CLI + Web (optional) | Web-first       |
| Complexity       | Lightweight          | Feature-rich    |
| Image Automation | Built-in             | Requires add-on |
| Helm Support     | Native               | Native          |

<!--
Speaker Notes:
- Common question: Flux vs ArgoCD?
- Both are excellent tools
- ArgoCD: better UI, more features out of box
- Flux: more lightweight, better image automation
- ArgoCD UI: great for visualization, troubleshooting
- Flux: CLI-first, optional UI (Freelens with Flux extension)
- Choose based on your team's needs
- Some teams use both: ArgoCD for apps, Flux for platform
- Both are CNCF projects, both production-ready
- We chose Flux for this workshop because it's simpler to start
- ArgoCD might be better if you need multi-cluster UI
-->

---

## Migration Strategy

**From manual deployments to Flux:**

1. **Audit** - Document current deployment process
2. **Prepare** - Move manifests to Git
3. **Test** - Run Flux in dry-run mode
4. **Migrate** - Start with non-critical apps
5. **Validate** - Monitor and verify
6. **Scale** - Roll out to all applications

<!--
Speaker Notes:
- You can't migrate overnight
- Start with audit: how do you deploy today?
- Document everything: apps, configs, secrets
- Prepare: move manifests to Git (might need cleanup)
- Test in dev cluster first
- Migrate: pick least critical app to start
- Learn from mistakes in low-stakes environment
- Validate: did it work? What broke?
- Scale: gradually migrate more apps
- Typical migration: 3-6 months for large org
- Have rollback plan for each migration
- Communicate with teams throughout
-->

---
## Features 

- Notifications
- Image automation
- Multi-Tenancy
- Remote cluster
- OCI Support
- Progressive Delivery with Flagger (Canary, A/B Testing, Blue/Green deployments)

---
## Resources

📚 **Documentation**
- https://fluxcd.io/flux/
- https://fluxcd.io/flux/guides/

🎓 **Learning**
- Flux tutorials and workshops
- CNCF GitOps Working Group

💬 **Community**
- CNCF Slack #flux channel
- GitHub Discussions

<!--
Speaker Notes:
- All these resources are excellent
- Flux docs are very comprehensive
- Guides section has step-by-step tutorials
- CNCF GitOps WG: monthly meetings, open to all
- Slack: active community, helpful people
- GitHub Discussions: search before asking
- Flux YouTube channel has great videos
- Weekly dev meetings if you want to contribute
- Consider contributing back to Flux
- Documentation improvements always welcome
-->

---

<!-- 
_class: [invert, lead] 
paginate: false
-->

## Thank You!

**Questions?**

![bg right:35% 60%](images/fluxcd.svg)

<!-- _footer: Edwin Bruurs | CM.com DevDays 2025 -->

<!--
Speaker Notes:
- Thank you for attending!
- Open floor for questions
- Any topic we covered or didn't cover
- Share your experiences if you've used GitOps
- All materials will be on GitHub
- Feel free to reach out after workshop
- Connect on LinkedIn, GitHub
- Would love to hear about your GitOps journey
- Don't forget to fill out feedback form
- Help me improve for next time
- Stay in touch through Slack/community
-->
