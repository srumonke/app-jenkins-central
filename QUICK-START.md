# Quick Start - 5 Minute Demo

## What You Have

```
demo/
├── app-jenkins-central/          # Shared Helm chart (push to GitHub repo 1)
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       ├── deployment.yaml
│       └── service.yaml
│
├── app-jenkins-ref-app/          # App values (push to GitHub repo 2)
│   └── helm-values/
│       ├── values.yaml           # Base
│       ├── values-dev.yaml       # Dev overrides
│       └── values-prod.yaml      # Prod overrides
│
├── harness-service.yaml          # Import this into Harness
└── harness-pipeline.yaml         # Import this into Harness
```

## Quick Test (Local)

```bash
cd demo

# Test dev deployment
helm template demo-app app-jenkins-central/ \
  -f app-jenkins-ref-app/helm-values/values.yaml \
  -f app-jenkins-ref-app/helm-values/values-dev.yaml

# Test prod deployment
helm template demo-app app-jenkins-central/ \
  -f app-jenkins-ref-app/helm-values/values.yaml \
  -f app-jenkins-ref-app/helm-values/values-prod.yaml
```

**Expected**: Dev shows replicas=1, Prod shows replicas=3

## Push to GitHub

```bash
# Repo 1: Central chart
cd app-jenkins-central
git init
git add .
git commit -m "Central chart"
git remote add origin https://github.com/<ORG>/app-jenkins-central.git
git push -u origin main

# Repo 2: App values
cd ../app-jenkins-ref-app
git init
git add .
git commit -m "App values"
git remote add origin https://github.com/<ORG>/app-jenkins-ref-app.git
git push -u origin main
```

## Harness Setup (3 steps)

### 1. Create GitHub Connector
- **Account Settings** → **Connectors** → **New Connector** → **GitHub**
- Name: `github_connector`
- URL: `https://github.com/<YOUR_ORG>`
- Auth: Personal Access Token
- Test & Save

### 2. Create Service
- **Services** → **New Service**
- Name: `Demo App`
- Type: `Kubernetes`

**Add 3 Manifests**:

1. **Helm Chart** (from central repo)
   - Type: Helm Chart
   - Repo: `app-jenkins-central`
   - Path: `/`
   - Chart: `app-chart`

2. **Base Values** (from app repo)
   - Type: Values
   - Repo: `app-jenkins-ref-app`
   - Path: `helm-values`
   - Files: `values.yaml`

3. **Env Values** (from app repo)
   - Type: Values
   - Repo: `app-jenkins-ref-app`
   - Path: `helm-values`
   - Files: `values-<+env.name>.yaml` ← **This is the magic!**

### 3. Create Pipeline
- **Pipelines** → **New Pipeline**
- Name: `Deploy Demo App`
- Add Stage: **Deploy**
  - Service: `Demo App`
  - Environment: `<+input>` (select at runtime)
  - Infrastructure: `<+input>` (select at runtime)
- Add Step: **K8s Apply** or **Helm Deploy**
- Add Rollback Step: **K8s Rollback**

## Run It

1. Click **Run Pipeline**
2. Select env: `dev` → Deploys with replicas=1
3. Select env: `prod` → Deploys with replicas=3

## The Key Insight

The expression `values-<+env.name>.yaml` automatically resolves to:
- `values-dev.yaml` when deploying to dev environment
- `values-prod.yaml` when deploying to prod environment

**This replaces your Jenkins loop that applies different -f flags per environment!**

## Comparison

### Jenkins Way
```groovy
// applyManifest.groovy
valueFiles = [
  "helm-charts/charts/app-v1/values.yaml",
  "helm-values/values.global.yaml",
  "helm-values/values.${account}.yaml",
  "helm-values/values.${account}-${env}.yaml"
]
helm upgrade --install ${name} \
  helm-charts/charts/app-v1 \
  ${valueFiles.collect { "-f $it" }.join(' ')}
```

### Harness Way
```yaml
# Service definition
manifests:
  - HelmChart: app-jenkins-central
  - Values: values.yaml
  - Values: values-<+env.name>.yaml  # Auto-selects based on environment
```

No scripting, just configuration!
