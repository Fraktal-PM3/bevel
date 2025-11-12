# Hyperledger Bevel - Comprehensive Architecture Guide

Last Updated: 2025-11-12
Version: Test/Development Build

## Executive Summary

Hyperledger Bevel is an automation framework and Helm chart collection for deploying production-ready Distributed Ledger Technology (DLT) platforms to Kubernetes clusters across multiple cloud providers. It follows a GitOps approach where infrastructure and configuration are managed through Git repositories, with Flux handling continuous deployment.

**Supported Platforms:** Hyperledger Fabric, R3 Corda (OSS & Enterprise), Hyperledger Indy, Quorum, Hyperledger Besu, Substrate

---

## 1. PROJECT STRUCTURE OVERVIEW

### Root Directory Layout
```
bevel/
├── README.md                          # Main project documentation
├── CONTRIBUTING.md                    # Contribution guidelines
├── LICENSE                           # Apache 2.0
├── Dockerfile                        # Build image (Ansible + tools)
├── Dockerfile.jdk8                   # Alternative with JDK8
├── run.sh                            # Helper script to run provisioning
├── platforms/                        # Core platform implementations (detailed below)
├── automation/                       # Jenkins CI/CD integration
├── docs/                             # ReadTheDocs documentation
├── build/                            # Generated artifacts during deployment
└── .github/                          # GitHub Actions workflows
```

### Key Directories
- **/platforms**: Contains all DLT platform implementations
- **/automation**: Jenkins pipeline definitions
- **/docs**: Sphinx-based ReadTheDocs documentation
- **/build**: Temporary directory for generated files (gitops values, certs, etc.)

---

## 2. PLATFORMS ARCHITECTURE

### 2.1 Directory Structure Pattern

Each platform follows a consistent structure:

```
platforms/
├── shared/                           # SHARED components (used by all platforms)
│   ├── charts/                       # Common Helm charts (Vault, StorageClass, HAProxy)
│   ├── configuration/                # Shared Ansible playbooks and roles
│   ├── images/                       # Common Dockerfiles
│   ├── inventory/                    # Ansible inventory templates
│   └── scripts/                      # Common shell scripts
│
├── hyperledger-fabric/               # Fabric-specific implementation
├── hyperledger-besu/                 # Besu-specific implementation
├── hyperledger-indy/                 # Indy-specific implementation
├── quorum/                           # Quorum-specific implementation
├── r3-corda/                         # Corda OSS implementation
├── r3-corda-ent/                     # Corda Enterprise implementation
├── substrate/                        # Substrate implementation
│
└── network-schema.json               # JSON Schema for network.yaml validation
```

### 2.2 Platform-Specific Structure (Common Pattern)

Each DLT platform directory follows this pattern:

```
platform-name/
├── README.md                         # Platform-specific documentation
├── charts/                           # Helm charts for platform components
│   ├── README.md                     # Chart usage guide
│   ├── component-name/
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   ├── templates/                # Kubernetes resource templates
│   │   ├── conf/                     # Config files for the component
│   │   └── files/                    # Static files (certificates, configs)
│   └── [multiple component charts]
│
├── configuration/                    # Ansible playbooks and roles
│   ├── deploy-network.yaml           # Main deployment playbook
│   ├── add-new-channel.yaml          # Operations playbooks
│   ├── cleanup.yaml
│   ├── roles/
│   │   ├── create/                   # Role directories for creating components
│   │   │   ├── ca_server/
│   │   │   ├── peers/
│   │   │   ├── orderers/
│   │   │   ├── channels/
│   │   │   └── [component-specific roles]
│   │   ├── delete/                   # Cleanup roles
│   │   ├── setup/                    # Setup roles
│   │   └── upgrade/                  # Upgrade roles
│   ├── samples/                      # Sample network.yaml files
│   │   ├── network-fabricv2.yaml
│   │   ├── network-fabric-add-cli.yaml
│   │   └── [variant configurations]
│   └── collections/                  # Ansible collections
│
├── images/                           # Platform-specific Dockerfiles
├── releases/                         # GitOps release directory (generated)
│   ├── dev/                          # Per-environment directories
│   ├── stg/
│   └── [environment-name]/
└── scripts/                          # Platform-specific shell scripts
```

---

## 3. HELM CHARTS ORGANIZATION

### 3.1 Shared Charts (`/platforms/shared/charts/`)

These are prerequisites and common utilities:

- **bevel-storageclass**: Creates persistent volume storage classes
- **bevel-vault-mgmt**: Manages Vault authentication and secret engines
- **haproxy-ingress**: HAProxy ingress controller (alternative to Ambassador)
- **cert-manager**: Certificate lifecycle management
- **letsencrypt-cert/issuer**: Let's Encrypt integration
- **bevel-scripts**: Common utility scripts
- **bevel-image-automation**: Image pull policy automation

### 3.2 Platform Charts - Fabric Example

Hyperledger Fabric has ~20 specialized Helm charts:

**Component Charts:**
- `fabric-ca-server`: Certificate Authority deployment
- `fabric-peernode`: Peer node deployment with CouchDB
- `fabric-orderernode`: Ordering service node
- `fabric-genesis`: Genesis block generation
- `fabric-cli`: CLI tools pod

**Channel & Chaincode Operations:**
- `fabric-channel-create`: Create channel transactions
- `fabric-channel-join`: Join peers to channels
- `fabric-chaincode-install`: Install chaincode
- `fabric-chaincode-lifecycle`: Chaincode approval/commit (Fabric 2.x)
- `fabric-chaincode-invoke`: Invoke chaincode methods

**Operational Tools:**
- `fabric-operations-console`: IBM Fabric Operations Console
- `fabric-cacti-connector`: Hyperledger Cacti connector
- `fabric-catools`: CA tools utility

### 3.3 Helm Chart Internal Structure

Each chart contains:

```
chart-name/
├── Chart.yaml                        # Chart metadata
├── values.yaml                       # Default values (configurable)
├── requirements.yaml                 # Helm dependencies
├── README.md                         # Chart documentation
├── templates/
│   ├── deployment.yaml              # or StatefulSet
│   ├── service.yaml                 # Kubernetes Service
│   ├── configmap.yaml               # ConfigMaps for configs
│   ├── secret.yaml                  # Secrets (via Vault integration)
│   ├── job-cleanup.yaml             # Cleanup jobs
│   ├── rbac.yaml                    # RBAC definitions
│   ├── _helpers.tpl                 # Template macros
│   └── ingress.yaml                 # Ingress routes
├── conf/                            # Configuration files
│   ├── fabric-ca-server-config.yaml # e.g., CA server config
│   └── [component-specific configs]
└── files/                           # Static files
    ├── orderer.crt                  # TLS certificates
    └── [other static files]
```

### 3.4 Values File Generation & Templating

**Process Flow:**

1. **Ansible Role**: `create/helm_component` (shared role)
   - File: `/platforms/shared/configuration/roles/create/helm_component/tasks/main.yaml`
   - Consumes a Jinja2 template file

2. **Template Files**: Located in platform-specific roles
   - File: `/platforms/hyperledger-fabric/configuration/roles/create/[component]/templates/values.tpl`
   - Uses Jinja2 templating to generate `values-[component].yaml`

3. **Generated Output**: Stored in GitOps release directory
   - Example: `/platforms/hyperledger-fabric/releases/dev/org1/ca/values-ca.yaml`

4. **Helm Lint**: Validates generated values files
   - Role: `helm_lint` (shared configuration role)
   - Uses `helm template` to validate syntax

5. **Git Commit**: Pushes generated values to Git
   - Role: `git_push` (shared configuration role)
   - Enables GitOps continuous deployment

---

## 4. ANSIBLE PLAYBOOKS & ROLES

### 4.1 Main Entry Points

**Shared Playbooks** (`/platforms/shared/configuration/`):

```
site.yaml
├── setup-environment.yaml            # Install/configure tools (Helm, kubectl, vault)
├── setup-k8s-environment.yaml         # Install Flux, Ambassador, Cert Manager on cluster
└── [conditional imports based on network.type]
    ├── hyperledger-fabric/deploy-network.yaml
    ├── quorum/deploy-network.yaml
    └── [platform-specific deployment]
```

**Execution:**
```bash
ansible-playbook site.yaml -e "@./platforms/hyperledger-fabric/configuration/network.yaml"
```

### 4.2 Playbook Execution Flow (Fabric Example)

**File:** `/platforms/shared/configuration/site.yaml`

```
site.yaml (main entry)
│
├─→ setup-environment.yaml
│   └─ Installs: Helm, kubectl, Vault, aws-cli, ansible dependencies
│
├─→ setup-k8s-environment.yaml
│   ├─ Create namespaces & RBAC
│   ├─ Deploy Flux (GitOps controller)
│   ├─ Deploy Ambassador or HAProxy (ingress)
│   ├─ Deploy Cert Manager
│   └─ Configure cloud provider auth (AWS/Azure)
│
├─→ hyperledger-fabric/deploy-network.yaml (conditional)
│   ├─ create/namespace (per organization)
│   ├─ create/secrets (Vault credentials)
│   ├─ shared/setup/storageclass (per org)
│   ├─ create/ca_server (per organization)
│   ├─ create/orderers (if org has orderers)
│   ├─ create/peers (if org has peers)
│   ├─ create/genesis (blockchain genesis block)
│   ├─ create/osnchannels or channels (channel creation)
│   ├─ create/channels_join (peer joins channels)
│   └─ create/chaincode (install/approve/commit chaincode)
│
└─→ (Reset mode only)
    └─ cleanup.yaml
        └─ delete-network.yaml (remove all resources)
```

### 4.3 Role Architecture

Roles follow Ansible best practices with nested task inclusion:

**Pattern 1: Create/Deploy Roles**
```
roles/create/component_name/
├── tasks/
│   ├── main.yaml                    # Entry point
│   ├── nested_main.yaml             # Loop processing (per org/peer/etc)
│   └── [helper tasks]
├── templates/
│   └── values.tpl                   # Jinja2 template for values file
└── vars/
    └── main.yaml                    # Role variables
```

**Pattern 2: Shared Utility Roles**
```
shared/configuration/roles/
├── create/
│   ├── shared_helm_component/        # Generic Helm values generation
│   ├── job_component/                # Kubernetes Job template generation
│   ├── shared_k8s_secrets/           # Create K8s secrets from network.yaml
│   └── namespace/                    # Create namespaces & RBAC
├── check/
│   ├── directory/                    # Verify/create directories
│   ├── helm_component/               # Wait for pod/service readiness
│   ├── k8_component/                 # General K8s resource checks
│   └── setup/                        # Pre-requisite validation
├── delete/
│   ├── k8s_resources/                # Delete K8s resources
│   ├── flux/                         # Flux cleanup
│   └── [platform-specific delete]
├── setup/
│   ├── aws-cli/                      # AWS CLI installation
│   ├── aws-auth/                     # AWS authentication
│   ├── helm/                         # Helm installation
│   ├── kubectl/                      # kubectl installation
│   ├── vault/                        # Vault CLI installation
│   ├── flux/                         # Flux deployment
│   ├── ambassador/                   # Ambassador stack
│   ├── cactus-connector/             # Hyperledger Cacti
│   └── [cloud provider specific]
├── git_push/                         # Commit & push to Git
├── git_status/                       # Check Git status
└── helm_lint/                        # Validate Helm values
```

**Fabric-Specific Roles:**
```
hyperledger-fabric/configuration/roles/
├── create/
│   ├── ca_server/                    # CA setup + key generation
│   ├── peers/                        # Peer node deployment
│   ├── orderers/                     # Orderer node deployment
│   ├── genesis/                      # Genesis block generation
│   ├── channels/                     # Channel creation (Fabric 2.2)
│   ├── osnchannels/                  # OSN channel creation (Fabric 2.5)
│   ├── channels_join/                # Peer join channel + anchor peer
│   ├── chaincode/
│   │   ├── approve/                  # Chaincode approval (Fabric 2.x)
│   │   ├── commit/                   # Chaincode commit (Fabric 2.x)
│   │   ├── install/                  # Chaincode installation
│   │   └── invoke/                   # Chaincode invocation
│   ├── new_organization/             # Add org to existing network
│   ├── new_peer/                     # Add peer to organization
│   ├── new_orderer/                  # Add orderer to network
│   ├── external_chaincode_server/    # External chaincode server setup
│   ├── console_assets/               # Operations console assets
│   └── [operational roles]
├── operator/                         # Bevel Fabric Operator roles
├── k8_component/                     # K8s resource creation
├── helm_component/                   # Fabric-specific Helm template
├── delete/                           # Cleanup and deletion
└── upgrade/                          # Certificate refresh, version upgrade
```

### 4.4 Role Invocation Pattern

Roles are included from playbooks with variable context:

```yaml
- name: Create CA for organization
  include_role:
    name: "create/ca_server"           # Role name relative to roles_path
  vars:
    component_ns: "{{ org.name }}-net"
    component: "{{ org.name }}"
    kubernetes: "{{ org.k8s }}"         # K8s config from network.yaml
    vault: "{{ org.vault }}"            # Vault config from network.yaml
    ca: "{{ org.services.ca }}"         # CA config from network.yaml
    values_dir: "{{playbook_dir}}/../../{{org.gitops.release_dir}}/{{ org.name }}"
  loop: "{{ network['organizations'] }}"
  loop_control:
    loop_var: org
  when: org.services.ca is defined
```

### 4.5 Key Shared Roles

**shared_helm_component**: 
- Generates Helm values files from templates
- Runs Helm lint validation
- Called by every component creation role

**shared_k8s_secrets**:
- Creates K8s Secret objects containing sensitive data
- Stores Vault tokens, Docker credentials, Git SSH keys
- Used before component deployment

**check/helm_component**:
- Waits for pods to reach Running state
- Uses kubectl port forwarding checks
- Implements retry logic with configurable timeout

---

## 5. NETWORK.YAML CONFIGURATION

### 5.1 Schema & Validation

**Schema File:** `/platforms/network-schema.json`
- JSON Schema format (draft-07)
- Validates structure for each DLT type
- Includes platform-specific conditional schemas

**Validation in network.yaml:**
```yaml
# yaml-language-server: $schema=../../../../platforms/network-schema.json
network:
  type: fabric                         # Triggers Fabric-specific schema
  version: 2.5.4
  env:
    type: "dev"
    proxy: haproxy                     # Can be: haproxy, ambassador, none
    retry_count: 20
    external_dns: enabled              # For auto DNS registration
    annotations:
      service: []
      deployment: {}
      pvc: {}
  docker:
    url: "ghcr.io/hyperledger"
    username: "docker_username"
    password: "docker_password"
```

### 5.2 Network Structure

```yaml
network:
  type: fabric
  version: 2.5.4
  env: { ... }
  docker: { ... }
  
  # DLT-specific configurations
  consensus:
    name: raft                         # or kafka for Fabric
  
  orderers:                            # Remote orderer references
    - orderer:
        name: orderer1
        org_name: supplychainorg
        uri: orderer1.supplychain-net.example.com:443
  
  channels:
    - channel:
        consortium: SupplyChainConsortium
        channel_name: mainchannel
        channel_status: new
        chaincodes:
          - "my-chaincode"
        orderers:
          - supplychainorg
        participants:
          - organization:
              name: manufacturer
              type: creator
              org_status: new
              peers:
                - peer:
                    name: peer0
                    type: anchor
                    peerstatus: new
  
  organizations:                       # Organization-specific configs
    - organization:
        name: supplychainorg
        type: orderer                  # or peer
        org_status: new
        # Cloud provider config
        k8s:
          context: "aws-context"
          config_file: "/path/to/kubeconfig"
          region: "us-east-1"
        # Vault config
        vault:
          url: "http://vault.example.com:8200"
          root_token: "s.xxxx"
          secret_path: "secretsv2"
          auth_path: "supplychain"
        # Git config for GitOps
        gitops:
          git_protocol: "https"
          git_url: "https://github.com/user/repo"
          git_username: "user"
          git_password: "token"
          email: "user@example.com"
          private_key: "/path/to/gitops"
          branch: "main"
          release_dir: "platforms/hyperledger-fabric/releases"
          chart_source: "platforms/hyperledger-fabric/charts"
        # Services (components)
        services:
          ca:
            name: ca
            subject: "/C=US/ST=SC/L=Raleigh/O=supplychainorg"
          peers:
            - peer:
                name: peer0
                type: anchor
          orderers:
            - orderer:
                name: orderer1
                type: orderer
```

### 5.3 Network.yaml Usage in Roles

Variables are extracted by roles and used in multiple ways:

```yaml
# 1. In task loops
loop: "{{ network['organizations'] }}"
loop_control:
  loop_var: org

# 2. In when conditions
when: org.org_status is not defined or org.org_status == 'new'

# 3. In variable substitution
vars:
  namespace: "{{ org.name | lower }}-net"
  kubernetes_context: "{{ org.k8s.context }}"

# 4. In template variable passing
template_vars:
  org_name: "{{ org.name }}"
  vault_addr: "{{ org.vault.url }}"
  ca_name: "{{ org.services.ca.name }}"
```

### 5.4 Sample Files Location

**Path:** `/platforms/hyperledger-fabric/configuration/samples/`

Key samples:
- `network-fabricv2.yaml` - Basic Fabric 2.5.x with 2 orgs
- `network-fabric-add-cli.yaml` - Add CLI pod to existing network
- `network-fabric-add-new-channel.yaml` - Add channel to running network
- `network-fabric-add-ordererorg.yaml` - Add orderer org
- `network-fabric-add-peer.yaml` - Add peer to existing org
- `network-fabric-remove-organization.yaml` - Remove org from network
- `network-operator-fabric.yaml` - Fabric Operator deployment mode
- `network-proxy-none.yaml` - No proxy configuration

---

## 6. SHARED CONFIGURATION COMPONENTS

### 6.1 Common Service Architecture

Every organization requires these shared components:

**1. Kubernetes Resources:**
- Namespace: `{org_name}-net`
- ServiceAccount: `vault-auth`
- RBAC (Role/RoleBinding) for Vault access
- StorageClass: `{org_name}-bevel-storageclass`

**2. Vault Integration:**
- Path: `{network_type}/{org_name}/`
- Auth method: Kubernetes auth (OIDC via Vault JWT)
- Secret engines: 
  - KV v2 for credentials
  - PKI for certificate generation

**3. GitOps (Flux) Integration:**
- Flux deployment in namespace
- Git SSH key for authentication
- HelmRelease CRDs for deployments
- Flux monitors Git for changes → auto-deploy

**4. Ingress Controller:**
- Option 1: HAProxy (for Fabric, recommended for HA)
- Option 2: Ambassador Edge Stack
- Option 3: None (direct K8s service)

### 6.2 Shared Roles Deep Dive

**Namespace Creation (`create/namespace`):**
```
Tasks:
├─ Create namespace
├─ Create ServiceAccount
├─ Create Role (read secrets from Vault path)
├─ Create RoleBinding
└─ Store K8s CA certificate in Vault
```

**Secrets Creation (`create/shared_k8s_secrets`):**
```
Creates K8s Secrets for:
├─ Vault token (roottoken secret)
├─ Docker registry credentials
├─ Git SSH key (for Flux)
└─ Custom credentials from network.yaml
```

**Flux Setup (`setup/flux`):**
```
Tasks:
├─ Add Flux Helm repository
├─ Create namespace & RBAC for Flux
├─ Deploy Flux controller (uses HelmRelease CRD)
├─ Configure Git SSH authentication
└─ Wait for Flux pod to be ready
```

**Storage Class (`setup/storageclass`):**
```
Creates PersistentVolumeClaim storage for:
├─ Orderer data
├─ Peer world state (CouchDB)
├─ CA certificates
└─ Ledger data
```

### 6.3 Vault Authentication Flow

```
1. Kubernetes Auth Method Setup
   ├─ Vault admin configures auth/kubernetes
   ├─ Points to K8s API server URL
   └─ Configures JWT review endpoint

2. ServiceAccount JWT Token
   ├─ K8s automatically creates JWT in secret
   ├─ Mounted in pod at /var/run/secrets/kubernetes.io/serviceaccount/token
   └─ Valid for pod's entire lifetime

3. Pod Authentication
   ├─ Pod calls: vault login -method=kubernetes \
   │   -path=auth/{org_name} \
   │   role=vault-role \
   │   jwt={token}
   ├─ Vault validates JWT via K8s API
   └─ Returns client token with policy

4. Secret Access
   ├─ Pod uses client token for future requests
   ├─ Can read secrets at: secret/{org_name}/...
   └─ Permissions defined in Vault policy
```

---

## 7. BUILD & DEPLOYMENT PROCESS

### 7.1 Deployment Sequence Diagram

```
User runs: ansible-playbook site.yaml -e "@network.yaml"
│
├─→ Parse network.yaml
│   └─ Load organizations, services, k8s configs
│
├─→ setup-environment.yaml (on controller machine)
│   ├─ Install Helm
│   ├─ Install kubectl
│   ├─ Install Vault CLI
│   ├─ Install aws-cli (if AWS provider)
│   └─ Verify accessibility to kubeconfig files
│
├─→ setup-k8s-environment.yaml (for each K8s cluster)
│   ├─ Create namespace
│   ├─ Create RBAC
│   ├─ Deploy Flux (GitOps controller)
│   ├─ Deploy Ambassador/HAProxy (ingress)
│   ├─ Deploy Cert Manager
│   └─ Store cluster CA cert in Vault
│
├─→ deploy-network.yaml (platform-specific)
│   ├─ For each organization:
│   │   ├─ create/namespace
│   │   ├─ create/secrets
│   │   ├─ create/storageclass
│   │   ├─ create/ca_server
│   │   │   ├─ Generates Helm values
│   │   │   ├─ Commits to Git
│   │   │   └─ Flux auto-deploys
│   │   ├─ Wait 6 minutes for CA certificates
│   │   ├─ create/orderers (if orderer org)
│   │   │   ├─ Generates orderer configs
│   │   │   ├─ Generates Helm values per orderer
│   │   │   └─ Flux auto-deploys
│   │   ├─ create/peers (if peer org)
│   │   │   ├─ Generates peer configs
│   │   │   ├─ Generates Helm values per peer
│   │   │   └─ Flux auto-deploys
│   │   └─ create/secrets (stores peer certs in Vault)
│   │
│   ├─ Genesis Block Creation
│   │   ├─ Collect MSP configs from Vault
│   │   ├─ Generate genesis block
│   │   ├─ Helm install fabric-genesis
│   │   └─ Flux deploys genesis job
│   │
│   ├─ Channel Creation & Join
│   │   ├─ Create channels
│   │   ├─ Join peers to channels
│   │   ├─ Set anchor peers
│   │   └─ Verify channel membership
│   │
│   └─ Chaincode Operations
│       ├─ Install chaincode
│       ├─ Approve chaincode
│       ├─ Commit chaincode
│       └─ Invoke test transactions
│
└─→ Completion
    └─ All components deployed via GitOps (Flux)
        ├─ Helm releases managed by Flux HelmRelease CRD
        ├─ Git becomes source of truth
        └─ Future changes made via Git PR/merge
```

### 7.2 Build Directory Structure

**Location:** `/bevel/build/` (created during deployment)

```
build/
├── ca/
│   ├── ca-key.pem                   # CA private key
│   └── ca-cert.pem                  # CA certificate
├── orderers/
│   ├── orderer1-key.pem
│   ├── orderer1-cert.pem
│   └── [more orderers]
├── orgs/
│   ├── org1/
│   │   ├── peers/
│   │   │   ├── peer0-key.pem
│   │   │   └── peer0-cert.pem
│   │   └── admin/
│   │       └── admin-msp/
│   └── [more orgs]
└── genesis/
    └── genesis.block                # Blockchain genesis block
```

### 7.3 Helm Deployment Coordination

**Flux HelmRelease Resources:**

Located in: `/platforms/hyperledger-fabric/releases/{env}/{org_name}/{component}/`

```yaml
# Example: ca/ca-values.yaml
apiVersion: helm.fluxcd.io/v1
kind: HelmRelease
metadata:
  name: org1-ca
  namespace: org1-net
spec:
  releaseName: org1-ca
  chart:
    repository: https://github.com/org/repo
    name: fabric-ca-server
    version: 1.0
  values:
    global:
      vault:
        address: "http://vault:8200"
        role: "org1-vault-role"
    ca:
      name: "ca"
  wait: true
  timeout: 5m
```

**Flux Workflow:**
```
Git Commit (values-ca.yaml)
    ↓
Flux detects change (polls Git)
    ↓
Flux generates HelmRelease CRD
    ↓
Helm controller reads HelmRelease
    ↓
Runs: helm upgrade --install org1-ca fabric-ca-server -f values-ca.yaml
    ↓
Kubernetes creates CA StatefulSet/Pod
    ↓
CA comes up, stores certs in Vault
```

---

## 8. TESTING STRUCTURE

### 8.1 Available Tests

**Network Schema Validation:**
- **File:** `.github/workflows/test_networkschema.yml`
- **Tool:** `ajv-cli` (JSON Schema validator)
- **Purpose:** Validate network.yaml against network-schema.json before playbook execution

**Ansible Lint:**
- **Tool:** `ansible-lint`
- **Purpose:** Check playbook syntax and best practices

**Helm Lint:**
- **Tool:** `helm lint`
- **Purpose:** Validate Helm chart syntax and values
- **Role:** `helm_lint` (checks each generated values file)

### 8.2 Test Execution Points

1. **network.yaml Validation** (optional in site.yaml - commented out)
   ```yaml
   # In site.yaml:
   # - import_playbook: validate-network-schema.yaml
   #   when: reset is undefined or reset == 'false'
   ```

2. **Helm Chart Linting** (during deployment)
   - Runs automatically when `create/helm_component` role executes
   - Validates each generated values file before git commit

3. **Component Readiness Checks**
   - Role: `check/helm_component`
   - Waits for pods to reach Running state
   - Verifies service endpoints are accessible

### 8.3 Manual Testing

**Verify Deployment:**
```bash
# Check namespace
kubectl get ns

# Check pods
kubectl get pods -n {org_name}-net

# Check Helm releases
helm list -n {org_name}-net

# Check Vault secrets
vault kv list secretsv2/fabric/{org_name}/

# Check Flux status
kubectl get helmrelease -n {org_name}-net
```

---

## 9. COMPONENT INTERACTION FLOWS

### 9.1 CA Server Deployment (Detailed)

```
Role: create/ca_server
│
├─ Task 1: Copy custom CA config (if provided)
│   └─ cp ${ca.configpath} → charts/ca/conf/
│
├─ Task 2: Get Kubernetes server URL
│   └─ kubectl config view → get API server URL
│
├─ Task 3: Generate CA values file
│   └─ include_role: shared_helm_component
│       ├─ Template: ca-values.tpl
│       ├─ Output: releases/dev/{org}/ca/values-ca.yaml
│       └─ Contains: vault, k8s, docker, storage configs
│
├─ Task 4: Helm lint validation
│   └─ helm template fabric-ca-server -f values-ca.yaml
│
├─ Task 5: Git commit & push
│   ├─ git add releases/
│   ├─ git commit -m "[ci skip] Pushing CA Server files"
│   └─ git push
│
├─ Task 6: Flux detects change
│   └─ HelmRelease CRD triggers Helm install
│
├─ Task 7: Monitor pod readiness
│   └─ kubectl wait --for=condition=Ready pod/ca-0
│
├─ Task 8: Fetch CA certificate from Vault
│   ├─ vault kv get secretsv2/fabric/{org}/ca/
│   └─ Save to build/ca-cert.pem
│
└─ Complete: CA is running and certificate available
```

### 9.2 Peer Node Deployment (Detailed)

```
Role: create/peers
│
├─ Loop per peer in organization:
│   │
│   ├─ Reset pod (if refresh certs)
│   │   └─ kubectl delete pod peer{n}
│   │
│   ├─ Generate peer values file
│   │   ├─ Template: peer-values.tpl
│   │   ├─ Variables:
│   │   │   ├─ peer.name, peer.type
│   │   │   ├─ couchdb credentials (random)
│   │   │   ├─ gossip addresses
│   │   │   └─ vault, ingress configs
│   │   └─ Output: values-peer{n}.yaml
│   │
│   ├─ Lint & commit to git
│   │   ├─ helm lint
│   │   ├─ git push
│   │   └─ Flux auto-deploy
│   │
│   ├─ Wait for peer pod running
│   │   ├─ kubectl wait pod/peer{n}
│   │   └─ Verify service endpoints
│   │
│   └─ Store secrets in Vault
│       ├─ vault kv put secretsv2/fabric/{org}/peer{n}/
│       │   ├─ private key
│       │   ├─ certificate
│       │   ├─ couchdb password
│       │   └─ msp config
│       └─ Complete: Peer operational
│
└─ All peers deployed
```

### 9.3 Channel Creation Flow (Fabric 2.5.x)

```
Role: create/osnchannels
│
├─ Loop per channel:
│   │
│   ├─ Generate channel create transaction
│   │   ├─ Read org configs from Vault
│   │   ├─ Create channel.pb protobuf
│   │   └─ Store in ConfigMap
│   │
│   ├─ Deploy fabric-osnadmin-channel-create Helm chart
│   │   ├─ Job mounts channel.pb
│   │   ├─ Job runs: osnadmin channel create ...
│   │   └─ Creates channel on orderers
│   │
│   └─ Verify channel existence
│       └─ osnadmin channel list
│
├─ Loop per peer to join channel:
│   │
│   ├─ Generate peer join transaction
│   │   └─ Store in ConfigMap
│   │
│   ├─ Deploy fabric-channel-join chart
│   │   ├─ Job mounts transaction
│   │   ├─ Job runs: peer channel join ...
│   │   └─ Peer joins channel
│   │
│   └─ If peer.type == anchor:
│       ├─ Generate anchor peer update
│       ├─ Deploy fabric-anchorpeer-update
│       └─ Submit anchor peer transaction
│
└─ All peers in all channels
```

---

## 10. KEY ARCHITECTURAL PATTERNS

### 10.1 Template Rendering Pattern

```
Ansible Role
    ↓
Template Variables (from network.yaml)
    ↓
Jinja2 Template File (.tpl)
    ↓
Generated YAML File (values-*.yaml)
    ↓
Helm lint validation
    ↓
Git commit
    ↓
Flux detects change
    ↓
Helm install → Kubernetes
```

### 10.2 Multi-Organization Pattern

Each organization is isolated:
- Dedicated K8s cluster (or namespace on shared cluster)
- Dedicated Vault path
- Dedicated Git release directory
- Independent Flux deployment
- Cross-org communication via network endpoints

```
Network spanning multiple organizations:

Organization A              Organization B
├─ K8s Cluster A          ├─ K8s Cluster B
├─ Vault A                ├─ Vault B
├─ Git branch: org-a      ├─ Git branch: org-b
├─ Namespace: org-a-net   ├─ Namespace: org-b-net
├─ CA, Peers, Orderers    └─ CA, Peers
│
└─ Connected via:
   ├─ Orderer endpoints (known)
   ├─ Peer gossip addresses (shared)
   └─ MSP certificates (exchanged)
```

### 10.3 Vault Secret Hierarchy

```
Vault KV v2 Structure:

secret/fabric/org1/
├─ ca/
│   ├─ rootca_pem
│   └─ rootca_key
├─ peer0/
│   ├─ private_key
│   ├─ certificate
│   ├─ msp_config_json
│   └─ tls_cert
├─ orderer/
│   └─ [similar structure]
├─ admin/
│   ├─ msp_config_json
│   └─ password
└─ [other identities]
```

### 10.4 GitOps Release Structure

```
platforms/hyperledger-fabric/releases/
├─ dev/                         # Environment
│   ├─ flux-dev/
│   │   └─ flux-values.yaml     # Flux config
│   ├─ org1-tf/                 # Organization
│   │   ├─ ca/
│   │   │   ├─ ca-values.yaml
│   │   │   └─ ca-storage.yaml
│   │   ├─ orderer/
│   │   │   ├─ orderer-values.yaml
│   │   │   └─ orderer-storage.yaml
│   │   └─ org1-tf/             # Namespace
│   │       ├─ peer0-values.yaml
│   │       ├─ peer1-values.yaml
│   │       ├─ channel-values.yaml
│   │       └─ chaincode-values.yaml
│   └─ [other orgs]
├─ stg/
└─ prod/
```

---

## 11. CONFIGURATION REFERENCE

### 11.1 Environment Type Indicators

**network.env.type values:**
- `dev`: Development environment
- `test`: Testing environment (CI/CD)
- `operator`: Fabric Operator mode (alternative deployment pattern)
- Custom tags allowed for multi-environment setups

### 11.2 Cloud Provider Support

**Tested Providers:**
- AWS EKS (full support)
- Azure AKS (full support)
- Minikube (local development)
- Any K8s cluster (with kubeconfig)

**Provider-specific:**
- AWS: EBS volumes, IAM auth, load balancers
- Azure: Managed disks, RBAC
- Others: Generic K8s resources

### 11.3 Proxy/Ingress Options

1. **HAProxy** (recommended for HA)
   - Layer 4 TCP routing
   - TLS termination
   - Health checks
   - Load balancing across replicas

2. **Ambassador Edge Stack**
   - Layer 7 HTTP/gRPC routing
   - API gateway features
   - Service mesh integration

3. **None**
   - Direct K8s service exposure
   - Suitable for development

### 11.4 Secret Management Options

1. **Hashicorp Vault** (recommended for production)
   - Kubernetes auth integration
   - PKI for certificate lifecycle
   - Audit logging

2. **Kubernetes Secrets** (built-in)
   - Simpler setup
   - No external dependencies
   - Less security (base64 encoding only)

---

## 12. EXTENDING BEVEL

### 12.1 Adding a New Component

1. **Create Helm Chart:**
   - Create: `platforms/{platform}/charts/{component-name}/`
   - Add: `Chart.yaml`, `values.yaml`, `templates/`

2. **Create Ansible Role:**
   - Create: `platforms/{platform}/configuration/roles/create/{component-name}/`
   - Add: `tasks/main.yaml` (calls `include_role: helm_component`)
   - Add: `templates/{component-name}-values.tpl`

3. **Add to Playbook:**
   - Include role in: `platforms/{platform}/configuration/deploy-network.yaml`
   - Add loop over organizations/components
   - Add readiness checks

### 12.2 Adding a New Platform

1. **Create Platform Directory:**
   - Copy structure from existing platform
   - Create: `charts/`, `configuration/`, `images/`, `releases/`

2. **Create Roles:**
   - Implement: `create/`, `delete/`, `setup/` roles
   - Follow same patterns as other platforms

3. **Create Playbooks:**
   - Create: `deploy-network.yaml`, `cleanup.yaml`
   - Include in: `site.yaml` conditional logic

4. **Update Schema:**
   - Add platform definition to: `network-schema.json`
   - Add conditional schema validation

---

## 13. TROUBLESHOOTING GUIDE

### 13.1 Common Issues & Solutions

**Pod stuck in Pending:**
- Check: `kubectl describe pod -n {org}-net {pod-name}`
- Check PVC: `kubectl get pvc -n {org}-net`
- Check StorageClass: `kubectl get sc`
- Check Vault token expiration

**Ansible playbook fails:**
- Check: `ansible-playbook -vvv` for verbose output
- Check: network.yaml syntax (against schema)
- Check: kubeconfig path and permissions
- Check: Vault token validity

**Helm lint fails:**
- Check: values-*.yaml for required fields
- Check: template files for syntax errors
- Run: `helm template {chart} -f values.yaml`

**Pod can't access Vault:**
- Check: K8s JWT token in pod
- Check: Vault JWT review endpoint
- Check: Service account role bindings
- Check: Vault policy for role

---

## 14. GIT REPOSITORY STRUCTURE

Typical Bevel Git repo layout:

```
repo/
├─ platforms/
│   ├─ hyperledger-fabric/
│   │   ├─ configuration/
│   │   │   └─ samples/
│   │   │       └─ network-fabric-*.yaml  (editable)
│   │   └─ releases/                      (generated)
│   │       ├─ dev/
│   │       └─ prod/
│   └─ shared/
│       └─ configuration/
│
├─ .gitops/
│   └─ flux-config/                       (optional)
│
└─ docs/
    └─ architecture.md                    (documentation)
```

**Important:**
- **DO NOT CHECK IN:** kubeconfig files, Vault tokens, private keys
- **DO CHECK IN:** network.yaml samples, generated values files (for GitOps)
- **Git hooks:** Can validate network.yaml before commit

---

## 15. QUICK REFERENCE

### Key File Paths (Fabric)

| Purpose | Path |
|---------|------|
| Main entry playbook | `/platforms/shared/configuration/site.yaml` |
| Deploy playbook | `/platforms/hyperledger-fabric/configuration/deploy-network.yaml` |
| Sample config | `/platforms/hyperledger-fabric/configuration/samples/network-fabricv2.yaml` |
| Schema validation | `/platforms/network-schema.json` |
| Shared roles | `/platforms/shared/configuration/roles/` |
| CA chart | `/platforms/hyperledger-fabric/charts/fabric-ca-server/` |
| Peer chart | `/platforms/hyperledger-fabric/charts/fabric-peernode/` |
| CA role | `/platforms/hyperledger-fabric/configuration/roles/create/ca_server/` |

### Important Commands

```bash
# Validate network config
ajv validate -s platforms/network-schema.json \
  -d platforms/hyperledger-fabric/configuration/samples/network-fabricv2.yaml

# Run deployment (dry-run)
ansible-playbook -i hosts site.yaml \
  -e "@./network.yaml" --check

# Run deployment
ansible-playbook -i hosts site.yaml \
  -e "@./network.yaml"

# Reset network
ansible-playbook -i hosts site.yaml \
  -e "@./network.yaml" -e "reset=true"

# Check Flux status
kubectl get helmrelease -A

# View generated values
kubectl get configmap -n {org}-net

# Tail logs
kubectl logs -n {org}-net -f {pod-name}
```

---

## 16. DOCUMENTATION REFERENCES

- **Main Docs:** https://hyperledger-bevel.readthedocs.io/
- **GitHub:** https://github.com/hyperledger/bevel
- **Discord:** https://discord.gg/hyperledger
- **Contributing:** `/CONTRIBUTING.md`

---

## Summary

Hyperledger Bevel is a sophisticated infrastructure-as-code framework built on:

1. **Ansible** for orchestration and templating
2. **Helm** for Kubernetes deployment
3. **Flux** for GitOps continuous deployment
4. **Vault** for secret management
5. **Git** as the single source of truth

The architecture separates concerns:
- **Shared components**: Common across all platforms
- **Platform-specific components**: DLT-specific logic
- **Generated artifacts**: Values files, configs (from templates)
- **Deployed resources**: Running in Kubernetes (via Flux)

Understanding the relationship between network.yaml → Ansible roles → Helm templates → Generated values → Flux deployment → Kubernetes is crucial for working with Bevel effectively.

