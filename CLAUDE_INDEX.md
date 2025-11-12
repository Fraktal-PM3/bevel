# Hyperledger Bevel - Quick Index to CLAUDE.md

## Document Location
**File:** `/home/adrian/Code/LTU/bevel/CLAUDE.md`
**Size:** 40KB | 1,266 lines
**Updated:** 2025-11-12

## Quick Navigation

### If you need to understand...

| Topic | Section | Lines |
|-------|---------|-------|
| Project layout | 1. PROJECT STRUCTURE OVERVIEW | ~40 |
| Platform directories | 2. PLATFORMS ARCHITECTURE | ~80 |
| How Helm charts work | 3. HELM CHARTS ORGANIZATION | ~150 |
| Ansible & roles | 4. ANSIBLE PLAYBOOKS & ROLES | ~200 |
| network.yaml format | 5. NETWORK.YAML CONFIGURATION | ~140 |
| Shared components | 6. SHARED CONFIGURATION COMPONENTS | ~150 |
| Deployment process | 7. BUILD & DEPLOYMENT PROCESS | ~120 |
| Testing | 8. TESTING STRUCTURE | ~80 |
| Component flows | 9. COMPONENT INTERACTION FLOWS | ~120 |
| Design patterns | 10. KEY ARCHITECTURAL PATTERNS | ~80 |
| Configuration options | 11. CONFIGURATION REFERENCE | ~80 |
| How to extend | 12. EXTENDING BEVEL | ~60 |
| Troubleshooting | 13. TROUBLESHOOTING GUIDE | ~50 |
| File paths | 15. QUICK REFERENCE | ~80 |

## Key Concepts

### The Core Pattern
```
network.yaml (input) 
  ↓ Ansible playbooks
  ↓ Roles generate Jinja2 templates  
  ↓ Generated values.yaml
  ↓ Helm lint validation
  ↓ Git commit
  ↓ Flux auto-deploy
  ↓ Kubernetes resources
```

### Main Files to Know
- **Core:** `/platforms/shared/configuration/site.yaml` (main orchestration)
- **Schema:** `/platforms/network-schema.json` (validates all configs)
- **Shared roles:** `/platforms/shared/configuration/roles/` (used by all platforms)
- **Helm charts:** `/platforms/{platform}/charts/` (K8s deployment specs)
- **Ansible roles:** `/platforms/{platform}/configuration/roles/` (platform logic)

### Directory Tree (Critical Paths)
```
bevel/
├── platforms/
│   ├── shared/
│   │   ├── configuration/
│   │   │   ├── site.yaml (MAIN)
│   │   │   ├── roles/ (shared logic)
│   │   │   └── setup-*.yaml (prerequisites)
│   │   └── charts/ (ingress, storage, vault)
│   ├── hyperledger-fabric/
│   │   ├── charts/ (CA, peer, orderer, chaincode ~20 charts)
│   │   ├── configuration/
│   │   │   ├── deploy-network.yaml
│   │   │   ├── roles/create/ (component deployment)
│   │   │   └── samples/ (example network.yaml)
│   │   └── releases/ (generated values, GitOps)
│   ├── hyperledger-besu/
│   ├── quorum/
│   ├── r3-corda/
│   ├── r3-corda-ent/
│   ├── hyperledger-indy/
│   ├── substrate/
│   └── network-schema.json
└── build/ (generated artifacts)
```

## Running Commands

### Basic Deployment
```bash
# From project root
ansible-playbook platforms/shared/configuration/site.yaml \
  -e "@./platforms/hyperledger-fabric/configuration/samples/network-fabricv2.yaml"
```

### With Reset
```bash
ansible-playbook platforms/shared/configuration/site.yaml \
  -e "@./network.yaml" -e "reset=true"
```

### Dry Run
```bash
ansible-playbook -i platforms/shared/inventory/ansible_provisioners \
  platforms/shared/configuration/site.yaml \
  -e "@./network.yaml" --check
```

## Understanding Each Layer

### Layer 1: Configuration Input
**File:** `network.yaml`
- Type: fabric, besu, quorum, indy, etc.
- Organizations: k8s config, Vault, Git, services
- Services: CA, peers, orderers, channels, chaincode
- See: `/platforms/hyperledger-fabric/configuration/samples/`

### Layer 2: Orchestration
**File:** `site.yaml` + setup/deploy playbooks
- Reads network.yaml
- Calls platform-specific playbooks
- Coordinates execution order
- Manages conditional logic

### Layer 3: Role Logic
**Dir:** `roles/create/`, `roles/setup/`, `roles/delete/`
- Implement specific tasks
- Generate Helm values from templates
- Interact with Kubernetes and Vault
- Commit to Git

### Layer 4: Templates
**Files:** `roles/create/{component}/templates/*.tpl`
- Jinja2 templates
- Read variables from network.yaml
- Generate values-*.yaml files

### Layer 5: Helm Charts
**Dir:** `charts/{component-name}/`
- Kubernetes resource definitions
- Default values.yaml
- ConfigMaps and secrets
- Jobs, deployments, statefulsets

### Layer 6: GitOps
**Dir:** `releases/{env}/{org_name}/`
- Generated values files
- Committed to Git
- Flux controller watches
- Helm install triggered

### Layer 7: Kubernetes
- Namespaces, RBAC
- Pods, services, storage
- ConfigMaps, secrets
- Ingress routes

## Shared Components (Every Organization)

1. **Namespace** - `{org_name}-net`
2. **ServiceAccount** - `vault-auth`
3. **RBAC** - Role, RoleBinding
4. **StorageClass** - `{org_name}-bevel-storageclass`
5. **Vault secrets** - At path `secretsv2/fabric/{org_name}/`
6. **Flux deployment** - Watches Git for changes
7. **Ingress** - HAProxy or Ambassador (optional)

## Platform-Specific Implementations

| Platform | Charts | Primary Role | Key Difference |
|----------|--------|--------------|-----------------|
| Fabric | ~20 | CA, Peer, Orderer, Channel | Complex channel/chaincode ops |
| Besu | ~5 | Node, Genesis, Validator | Simpler node management |
| Quorum | ~5 | Node, Genesis, Validator | Similar to Besu |
| Indy | ~5 | Node, Genesis | Ledger agent focus |
| Corda OSS | ~10 | Node, Network map | Enterprise features |
| Corda Ent | ~10 | Node, CENM | Full enterprise suite |
| Substrate | ~5 | Node, Genesis | Validator sets |

All use shared roles and configuration patterns.

## Testing & Validation Points

### Before Running
1. Validate network.yaml against schema
   - Tool: `ajv-cli`
   - File: `/platforms/network-schema.json`

### During Deployment
1. Helm lint each generated values file
   - Integrated in: `create/helm_component` role
2. Pod readiness checks
   - Role: `check/helm_component`

### After Deployment
1. Verify Flux status
2. Check pod states
3. Verify Vault secrets accessible
4. Test component endpoints

## Critical Junctions

**Junction 1:** Network Schema
- All network.yaml files validated against `/platforms/network-schema.json`
- Platform-specific conditional schemas
- Prevents invalid configurations

**Junction 2:** Shared Helm Component
- Role: `create/shared_helm_component`
- Every component uses this role
- Generates values file, runs lint, commits to Git
- **All platforms depend on this**

**Junction 3:** Flux GitOps
- Generated values files committed to Git
- Flux controller watches `/platforms/{platform}/releases/`
- Git becomes source of truth
- Flux automatically runs `helm install`

**Junction 4:** Vault Integration
- Every pod authenticates to Vault
- ServiceAccount JWT tokens
- Kubernetes auth method
- Policies restrict by organization

## Troubleshooting Quick Links

| Issue | Check Section |
|-------|---------------|
| Pod stuck in Pending | 13. TROUBLESHOOTING GUIDE |
| Playbook fails | 13. TROUBLESHOOTING GUIDE |
| Helm lint errors | 13. TROUBLESHOOTING GUIDE |
| Vault access denied | 6.3. Vault Authentication Flow |
| Network.yaml invalid | 5.1. Schema & Validation |
| Component not deploying | 7.3. Helm Deployment Coordination |

## Most Important Files to Read First

1. **CLAUDE.md** (this documentation)
2. `/platforms/network-schema.json` (understand validation)
3. `/platforms/shared/configuration/site.yaml` (main entry point)
4. `/platforms/shared/configuration/setup-environment.yaml` (prerequisites)
5. `/platforms/hyperledger-fabric/configuration/deploy-network.yaml` (example flow)
6. `/platforms/shared/configuration/roles/create/helm_component/tasks/main.yaml` (core pattern)

## Architecture Diagram (Text Form)

```
┌─────────────────────────────────────────────────────────────┐
│ Input: network.yaml (YAML configuration file)              │
└────────────────┬────────────────────────────────────────────┘
                 │
                 ▼
        ┌────────────────────┐
        │ site.yaml          │ Main orchestration
        │ (Ansible playbook) │
        └────────┬───────────┘
                 │
        ┌────────▼──────────────────────────────┐
        │ setup-environment.yaml                 │ Install tools
        │ setup-k8s-environment.yaml             │ Setup K8s
        │ deploy-network.yaml (platform-specific)│ Deploy DLT
        └────────┬──────────────────────────────┘
                 │
      ┌──────────▼──────────────┐
      │ Ansible Roles           │
      ├─────────────────────────┤
      │ create/ca_server        │ Generates values
      │ create/peers            │ from Jinja2
      │ create/orderers         │ templates
      │ create/channels         │
      │ create/chaincode        │
      └──────────┬──────────────┘
                 │
       ┌─────────▼────────────┐
       │ Generated values.yaml │
       │ (Jinja2 -> YAML)      │
       └─────────┬────────────┘
                 │
         ┌───────▼──────────┐
         │ Helm Lint Check  │
         │ (Validate YAML)  │
         └────────┬─────────┘
                  │
         ┌────────▼─────────┐
         │ Git Commit       │
         │ (Push to repo)   │
         └────────┬─────────┘
                  │
         ┌────────▼──────────┐
         │ Flux Controller   │
         │ (Watches Git)     │
         └────────┬──────────┘
                  │
    ┌─────────────▼──────────────┐
    │ Helm Install               │
    │ (Install chart with values)│
    └─────────────┬──────────────┘
                  │
    ┌─────────────▼───────────────────┐
    │ Kubernetes                      │
    │ ├─ Namespaces                   │
    │ ├─ RBAC                         │
    │ ├─ Pods (CA, peers, orderers)   │
    │ ├─ Services                     │
    │ ├─ Storage                      │
    │ ├─ ConfigMaps                   │
    │ └─ Secrets (via Vault)          │
    └─────────────┬───────────────────┘
                  │
         ┌────────▼────────┐
         │ Running Network │
         │ (DLT Platform)  │
         └─────────────────┘
```

## Environment Variables & Paths

**Ansible:**
- `KUBECONFIG`: Path to K8s config
- `VAULT_ADDR`: Vault URL
- `VAULT_TOKEN`: Vault root token

**Git:**
- `GIT_DIR`: Repository root
- `GIT_BRANCH`: Target branch

**Kubernetes:**
- Pod namespaces: `{org_name}-net`
- Service accounts: `vault-auth`

## Next Steps for Future Claude Instances

1. Read CLAUDE.md sections 1-5 for overview
2. Identify the DLT platform (Fabric, Besu, etc.)
3. Find platform-specific files in `/platforms/{platform}/`
4. Review relevant sample network.yaml in `/samples/`
5. Trace execution through playbooks and roles
6. Check shared roles for common patterns
7. Understand Vault and Flux integration

---

**This index should help you quickly locate information in CLAUDE.md and understand Hyperledger Bevel's architecture.**

For detailed information on any topic, open CLAUDE.md and navigate to the relevant section listed above.
