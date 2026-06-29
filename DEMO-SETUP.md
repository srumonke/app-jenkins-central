# Minimal Working Demo - Jenkins to Harness Migration

This demo replicates your centralized Helm chart pattern using Harness native features.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ app-jenkins-central (Shared Chart Repo)                     │
│  ├── Chart.yaml                                             │
│  ├── values.yaml (defaults)                                 │
│  └── templates/                                             │
│      ├── deployment.yaml                                    │
│      └── service.yaml                                       │
└─────────────────────────────────────────────────────────────┘
                            ↓
                   Referenced by Harness Service
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ app-jenkins-ref-app (App-specific Values)                   │
│  └── helm-values/                                           │
│      ├── values.yaml (base)                                 │
│      ├── values-dev.yaml (dev overrides)                    │
│      └── values-prod.yaml (prod overrides)                  │
└─────────────────────────────────────────────────────────────┘
                            ↓
                   Merged by Harness Pipeline
                            ↓
                      Deployed to K8s
```

## Step 1: Push Code to GitHub

### Repository 1: app-jenkins-central
```bash
cd demo/app-jenkins-central
git init
git add .
git commit -m "Initial central chart"
git remote add origin https://github.com/<YOUR_ORG>/app-jenkins-central.git
git push -u origin main
```

### Repository 2: app-jenkins-ref-app
```bash
cd demo/app-jenkins-ref-app
git init
git add .
git commit -m "Initial app values"
git remote add origin https://github.com/<YOUR_ORG>/app-jenkins-ref-app.git
git push -u origin main
```

## Step 2: Test Locally

```bash
cd demo

# Render dev environment
helm template demo-app app-jenkins-central/ \
  -f app-jenkins-ref-app/helm-values/values.yaml \
  -f app-jenkins-ref-app/helm-values/values-dev.yaml \
  --set global.image.tag=1.25

# Render prod environment
helm template demo-app app-jenkins-central/ \
  -f app-jenkins-ref-app/helm-values/values.yaml \
  -f app-jenkins-ref-app/helm-values/values-prod.yaml \
  --set global.image.tag=1.25
```

**Expected output**: You should see replicas=1 for dev, replicas=3 for prod.

## Step 3: Setup Harness

### 3.1 Create GitHub Connector

1. Go to **Account Settings > Connectors > New Connector > GitHub**
2. Name: `github_connector`
3. URL Type: `Account`
4. URL: `https://github.com/<YOUR_ORG>`
5. Authentication: Personal Access Token or GitHub App
6. Test and Save

### 3.2 Create Environments

#### Dev Environment
1. Go to **Environments > New Environment**
2. Name: `dev`
3. Type: `Pre-Production`
4. Add Infrastructure:
   - Name: `dev-k8s`
   - Type: `Kubernetes`
   - Connector: Select your K8s cluster connector
   - Namespace: `dev`

#### Prod Environment
1. Go to **Environments > New Environment**
2. Name: `prod`
3. Type: `Production`
4. Add Infrastructure:
   - Name: `prod-k8s`
   - Type: `Kubernetes`
   - Connector: Select your K8s cluster connector
   - Namespace: `prod`

### 3.3 Create Service

1. Go to **Services > New Service**
2. Name: `Demo App`
3. Deployment Type: `Kubernetes`
4. Click **Add Manifest**

#### Manifest 1: Helm Chart (from app-jenkins-central)
```yaml
Type: Helm Chart
Store: GitHub
Connector: github_connector
Repository: app-jenkins-central
Branch: main
Folder Path: /
Chart Name: app-chart
Helm Version: V3
```

#### Manifest 2: Base Values (from app-jenkins-ref-app)
```yaml
Type: Values
Store: GitHub
Connector: github_connector
Repository: app-jenkins-ref-app
Branch: main
Folder Path: helm-values
Files:
  - values.yaml
```

#### Manifest 3: Environment Values (from app-jenkins-ref-app)
```yaml
Type: Values
Store: GitHub
Connector: github_connector
Repository: app-jenkins-ref-app
Branch: main
Folder Path: helm-values
Files:
  - values-<+env.name>.yaml
```

**Key Point**: The `<+env.name>` expression automatically picks the right file based on the environment (dev → values-dev.yaml, prod → values-prod.yaml).

### 3.4 Create Pipeline

1. Go to **Pipelines > New Pipeline**
2. Name: `Demo App Deploy`
3. Type: `Deployment`
4. Add Stage:
   - Type: `Deploy`
   - Service: `Demo App`
   - Environment: `<+input>` (runtime selection)
   - Infrastructure: `<+input>` (runtime selection)

5. Execution Steps:
   - Add Step: `Helm Deploy` or `K8s Apply`
   - Configure:
     ```yaml
     Name: Deploy App
     Timeout: 10m
     Command Flags (optional):
       --set global.image.tag=<+artifact.tag>
     ```

6. Add Rollback:
   - Add Step: `K8s Rollback`

## Step 4: Run the Demo

### Deploy to Dev
1. Click **Run Pipeline**
2. Select Environment: `dev`
3. Select Infrastructure: `dev-k8s`
4. Click **Run**

**What happens**:
- Harness clones `app-jenkins-central` (chart)
- Harness clones `app-jenkins-ref-app` (values)
- Merges: chart defaults → values.yaml → values-dev.yaml
- Deploys to `dev` namespace with **replicas=1**

### Deploy to Prod
1. Click **Run Pipeline**
2. Select Environment: `prod`
3. Select Infrastructure: `prod-k8s`
4. Click **Run**

**What happens**:
- Same chart
- Same base values
- Merges values-prod.yaml instead
- Deploys to `prod` namespace with **replicas=3**

## Step 5: Verify

```bash
# Check dev deployment
kubectl get deployment demo-app -n dev
# Should show READY: 1/1

# Check prod deployment
kubectl get deployment demo-app -n prod
# Should show READY: 3/3

# Check rendered values
kubectl get deployment demo-app -n dev -o yaml | grep replicas
kubectl get deployment demo-app -n prod -o yaml | grep replicas
```

## How This Replicates Your Jenkins Setup

| Jenkins Approach | Harness Native Approach |
|------------------|-------------------------|
| Groovy script loops through helm-values dirs | Harness Service with multiple Values manifests |
| `helm -f file1 -f file2 -f file3` | Harness merges Values manifests in order |
| `--set global.account=<account>` | Harness expressions: `<+env.name>` |
| helm-secrets plugin for SOPS | Harness Secret Manager |
| Parallel deployment loop | Separate Services or looping strategy |
| Custom rollback logic | Native Harness failure strategy |
| Jenkins parameters (account, env, gitSha) | Harness pipeline variables & expressions |

## Key Harness Features Used

✅ **Service as abstraction**: Chart + values defined once, reused across environments  
✅ **Values overlay**: Multiple Values manifests merged in order  
✅ **Expressions**: `<+env.name>` dynamically selects values-dev.yaml or values-prod.yaml  
✅ **Git integration**: Both repos stay separate (chart vs values)  
✅ **Native secrets**: No SOPS needed, use Harness Secret Manager  
✅ **Rollback**: Automatic on failure, no custom scripting  

## Next Steps

1. **Add secrets**: Use Harness secrets instead of SOPS
   - Create secret in Harness
   - Reference as `<+secrets.getValue("my_secret")>`

2. **Add more environments**: Just create new values-staging.yaml, values-pre.yaml

3. **Multi-deployment support**: Create separate Harness Services for each deployment (alt, alt-euc1)

4. **Progressive delivery**: Use Harness native canary/blue-green instead of Argo Rollouts

5. **Approvals**: Add Manual Approval step between dev and prod

## Troubleshooting

### Values not merging correctly
- Check manifest order in Service (they merge top to bottom)
- Verify `<+env.name>` matches your environment identifier

### Chart not found
- Verify GitHub connector has access
- Check repository name and folder path
- Ensure Chart.yaml exists at the root

### Deployment fails
- Check Harness delegate can reach K8s cluster
- Verify namespace exists or enable auto-create
- Check service account permissions
