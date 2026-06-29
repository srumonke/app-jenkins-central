# Demo: Centralized Helm Chart Pattern in Harness

This demo shows how to replicate your Jenkins-based centralized Helm chart deployment using **Harness native features**.

## What This Demonstrates

✅ **Separate chart and values repos** (like your current setup)  
✅ **Values file layering** (base → environment-specific)  
✅ **Multi-environment deployment** (dev vs prod with different configs)  
✅ **No custom scripting** (Harness expressions replace Groovy)  
✅ **Native rollback** (no manual rollback logic)

## Files Included

| File | Purpose |
|------|---------|
| **app-jenkins-central/** | Shared Helm chart (mimics your app-v1 chart) |
| **app-jenkins-ref-app/** | App-specific values files |
| **QUICK-START.md** | 5-minute setup guide |
| **DEMO-SETUP.md** | Detailed walkthrough with explanations |
| **harness-service.yaml** | Service definition (can import into Harness) |
| **harness-pipeline.yaml** | Pipeline definition (can import into Harness) |

## Quick Start

1. **Test locally**:
   ```bash
   cd demo
   helm template demo-app app-jenkins-central/ \
     -f app-jenkins-ref-app/helm-values/values.yaml \
     -f app-jenkins-ref-app/helm-values/values-dev.yaml
   ```

2. **Push to GitHub**:
   - Push `app-jenkins-central/` to one repo
   - Push `app-jenkins-ref-app/` to another repo

3. **Setup in Harness**:
   - Create GitHub connector
   - Create Service with 3 manifests (chart + 2 values)
   - Create Pipeline with Deploy stage
   - Run!

## The Key Pattern

### Your Jenkins Setup
```
helm upgrade --install my-app helm-charts/charts/app-v1 \
  -f helm-values/values.global.yaml \
  -f helm-values/values.dev.yaml \
  --set global.deployment.git.sha=$GIT_SHA
```

### Harness Equivalent
```yaml
Service:
  manifests:
    - HelmChart from app-jenkins-central
    - Values: values.yaml
    - Values: values-<+env.name>.yaml  # Dynamically resolves!
  
Pipeline:
  commandFlags:
    --set global.image.tag=<+artifact.tag>
```

**No scripting required!** Harness expressions handle the dynamic file selection.

## What Makes This "Harness Native"

1. **Service abstraction**: Define chart + values once, deploy to any environment
2. **Values overlay**: Harness automatically merges multiple Values manifests
3. **Expressions**: `<+env.name>` replaces scripting logic
4. **Built-in rollback**: No need for custom rollback code
5. **GitOps ready**: Can enable GitOps mode for drift detection
6. **Secret management**: Native integration, no SOPS plugin needed

## Folder Structure

```
demo/
│
├── app-jenkins-central/              ← Push to GitHub repo 1
│   ├── Chart.yaml                    # Chart metadata
│   ├── values.yaml                   # Chart defaults
│   └── templates/
│       ├── deployment.yaml           # K8s Deployment
│       └── service.yaml              # K8s Service
│
├── app-jenkins-ref-app/              ← Push to GitHub repo 2
│   └── helm-values/
│       ├── values.yaml               # Base app config
│       ├── values-dev.yaml           # Dev overrides (replicas: 1)
│       └── values-prod.yaml          # Prod overrides (replicas: 3)
│
└── Documentation
    ├── README.md                     # This file
    ├── QUICK-START.md                # Fast setup (5 min)
    ├── DEMO-SETUP.md                 # Detailed guide
    ├── harness-service.yaml          # Import into Harness
    └── harness-pipeline.yaml         # Import into Harness
```

## Next Steps After Demo

Once you validate this works, you can:

1. **Add more templates**: Add ingress.yaml, hpa.yaml to the chart
2. **Add more values files**: values-staging.yaml, values-pre.yaml
3. **Add secrets**: Use Harness Secret Manager instead of SOPS
4. **Add approvals**: Insert Manual Approval step before prod
5. **Enable progressive delivery**: Use Harness canary/blue-green
6. **Add multi-deployment**: Create separate Services for alt/alt-euc1

## Migration Path

1. ✅ **Demo** (you are here): Validate the pattern works
2. **Pilot**: Migrate 1-2 low-risk apps using this pattern
3. **Scale**: Create templates/standards for teams
4. **Migrate**: Move remaining apps in batches
5. **Decommission**: Turn off Jenkins jobs

## Support

For questions on:
- **Demo setup**: See DEMO-SETUP.md
- **Quick testing**: See QUICK-START.md
- **Harness features**: See harness-service.yaml comments
- **Original Jenkins pattern**: See ../DEPLOY-PIPELINE.md and ../current setup.md

## Key Differences from Jenkins

| Aspect | Jenkins | This Demo |
|--------|---------|-----------|
| Chart location | helm-charts repo | app-jenkins-central repo |
| Values merging | Bash + helm -f flags | Harness Service manifests |
| Environment selection | Groovy parameters | Harness expressions |
| Secrets | SOPS + helm-secrets plugin | Harness Secret Manager |
| Multi-deployment | Loop in Groovy | Multiple Services |
| Rollback | Custom script | Native Harness |
| Image tag injection | --set in script | <+artifact.tag> expression |

The end result is **less code, more configuration**, which makes it easier to maintain and audit.
