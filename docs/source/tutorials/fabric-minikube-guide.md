[//]: # (##############################################################################################)
[//]: # (Copyright Accenture. All Rights Reserved.)
[//]: # (SPDX-License-Identifier: Apache-2.0)
[//]: # (##############################################################################################)

# Hyperledger Fabric Minikube Deployment Guide

This guide provides detailed instructions for deploying a Hyperledger Fabric network on minikube using Hyperledger Bevel. It addresses common misalignments in sample configurations and provides a step-by-step walkthrough aligned with how the Helm charts expect configurations.

## Overview

Deploying Fabric on minikube requires specific configuration adjustments compared to multi-cluster cloud deployments. The key differences are:

- Single Kubernetes cluster (no multi-cluster setup)
- Internal cluster DNS addressing (no external URLs)
- No HAProxy needed (proxy: none)
- Simplified networking model

## Key Misalignments in Sample Configurations

Before starting, understand these common issues with sample configurations:

### 1. Proxy Settings
**Problem**: `network-fabricv2.yaml` uses `proxy: haproxy` with external URLs
```yaml
# WRONG for minikube
proxy: haproxy
uri: peer0.carrier-net.org3proxy.blockchaincloudpoc.com:443
```

**Solution**: Use `proxy: none` with internal cluster DNS
```yaml
# CORRECT for minikube
proxy: none
uri: peer0.carrier-net:7051
```

### 2. Addressing
**Problem**: Multi-cluster samples use external addresses with port :443

**Solution**: Minikube should use internal Kubernetes service DNS format: `<service>.<namespace>:<port>`

### 3. Cloud Provider
**Problem**: Samples show `cloud_provider: aws`

**Solution**: Must be `cloud_provider: minikube`

### 4. Vault Integration
**Problem**: Production samples assume external Hashicorp Vault

**Solution**: Minikube can use local Vault setup with your machine's local IP address

## Prerequisites

Before proceeding, ensure you have:

1. **Minikube** installed and running with sufficient resources
2. **Hashicorp Vault** running locally
3. **Git repository** - A forked copy of Bevel
4. **Docker** installed
5. **kubectl** configured

For detailed prerequisite setup, refer to the [main minikube setup guide](./bevel-minikube-setup.md).

### Start Minikube

```bash
minikube start --memory 12000 --cpus 4 --kubernetes-version=1.23.1
```

### Start Hashicorp Vault

```bash
# Start Vault in dev mode (for testing)
vault server -dev

# In another terminal, enable secrets v2
export VAULT_ADDR='http://<Your-Local-IP>:8200'  # NOT localhost!
export VAULT_TOKEN="<vault_root_token>"
vault secrets enable -version=2 -path=secretsv2 kv
```

**Important**: Use your machine's actual local IP address (e.g., 192.168.1.10), NOT `localhost` or `127.0.0.1`. Find it using:
```bash
# Linux/Mac
ip addr show | grep "inet " | grep -v 127.0.0.1

# Mac alternative
ifconfig | grep "inet " | grep -v 127.0.0.1
```

## Step-by-Step Deployment

### Step 1: Prepare Build Directory

Create a build directory and copy necessary certificates:

```bash
cd /path/to/bevel

# Create build directory
mkdir -p build

# Copy minikube certificates
cp ~/.minikube/ca.crt build/
cp ~/.minikube/profiles/minikube/client.key build/
cp ~/.minikube/profiles/minikube/client.crt build/

# Copy kubeconfig
cp ~/.kube/config build/
```

### Step 2: Update Kubeconfig

Edit `build/config` and update the certificate paths to be relative:

```yaml
# Change FROM:
certificate-authority: /home/user/.minikube/ca.crt
client-certificate: /home/user/.minikube/profiles/minikube/client.crt
client-key: /home/user/.minikube/profiles/minikube/client.key

# Change TO:
certificate-authority: ca.crt
client-certificate: client.crt
client-key: client.key
```

### Step 3: Create Network Configuration

Copy the minikube-specific sample:

```bash
cp platforms/hyperledger-fabric/configuration/samples/network-minikube.yaml build/network.yaml
```

### Step 4: Configure network.yaml

Edit `build/network.yaml` with your specific values:

#### Network-Level Configuration

```yaml
network:
  type: fabric
  version: 2.2.2  # or 2.5.4

  env:
    type: "local"           # Tag for minikube environment
    proxy: none             # CRITICAL: No proxy for single cluster
    retry_count: 50
    external_dns: disabled  # Minikube doesn't need external DNS
    annotations:
      service: {}
      deployment: {}
      pvc: {}

  docker:
    url: "ghcr.io/hyperledger"
    # Comment out for public repos
    #username: "docker_username"
    #password: "docker_password"
```

#### Orderer Configuration

```yaml
  consensus:
    name: raft

  orderers:
    - orderer:
      type: orderer
      name: orderer1
      org_name: supplychain
      uri: orderer1.supplychain-net:7050  # Internal K8s DNS only
```

#### Organization Configuration

For the orderer organization:

```yaml
organizations:
  - organization:
      name: supplychain
      country: UK
      state: London
      location: London
      subject: "O=Orderer,L=51.50/-0.13/London,C=GB"
      external_url_suffix: develop.local.com  # Ignored when proxy: none
      org_status: new

      cloud_provider: minikube  # CRITICAL: must be minikube

      k8s:
        region: "minikube"
        context: "minikube"
        config_file: "/home/bevel/build/config"

      vault:
        url: "http://192.168.X.X:8200"  # Replace with YOUR local IP
        root_token: "your_vault_root_token"
        secret_path: "secretsv2"

      gitops:
        git_protocol: "https"
        git_url: "https://github.com/<username>/bevel.git"
        branch: "local"
        release_dir: "platforms/hyperledger-fabric/releases/dev"
        component_dir: "platforms/hyperledger-fabric/releases/k8sComponent"
        chart_source: "platforms/hyperledger-fabric/charts"
        git_repo: "github.com/<username>/bevel.git"
        username: "<github_username>"
        password: "<github_token>"
        email: "<github_email>"
        private_key: "/home/bevel/build/gitops"  # Only needed for SSH

      services:
        ca:
          name: ca
          subject: "/C=GB/ST=London/L=London/O=Orderer/CN=ca.supplychain-net"
          type: ca
          grpc:
            port: 7054

        consensus:
          name: raft

        orderers:
          - orderer:
            name: orderer1
            type: orderer
            consensus: raft
            grpc:
              port: 7050
            ordererAddress: orderer1.supplychain-net:7050  # Internal only
```

For peer organizations:

```yaml
  - organization:
      name: carrier
      country: GB
      state: London
      location: London
      subject: "O=Carrier,OU=Carrier,L=51.50/-0.13/London,C=GB"
      external_url_suffix: develop.local.com
      org_status: new
      orderer_org: supplychain  # Name of orderer organization

      # Same cloud_provider, k8s, vault, gitops as above
      cloud_provider: minikube
      # ... (repeat same config) ...

      services:
        ca:
          name: ca
          subject: "/C=GB/ST=London/L=London/O=Carrier/CN=ca.carrier-net"
          type: ca
          grpc:
            port: 7054

        peers:
          - peer:
            name: peer0
            type: anchor  # At least one anchor peer per org
            gossippeeraddress: peer0.carrier-net:7051  # Internal address
            peerAddress: peer0.carrier-net:7051        # Internal address
            cli: enabled  # Useful for debugging
            grpc:
              port: 7051
            events:
              port: 7053
            couchdb:
              port: 5984
            metrics:
              enabled: false
              port: 9443
```

#### Channel Configuration

```yaml
channels:
  - channel:
    consortium: SupplyChainConsortium
    channel_name: AllChannel
    channel_status: new
    orderers:
      - supplychain
    participants:
      - organization:
        name: carrier
        type: creator  # First org creates the channel
        org_status: new
        peers:
          - peer:
            name: peer0
            gossipAddress: peer0.carrier-net:7051
            peerAddress: peer0.carrier-net:7051
        ordererAddress: orderer1.supplychain-net:7050

      - organization:
        name: manufacturer
        type: joiner
        org_status: new
        peers:
          - peer:
            name: peer0
            gossipAddress: peer0.manufacturer-net:7051
            peerAddress: peer0.manufacturer-net:7051
        ordererAddress: orderer1.supplychain-net:7050
```

### Step 5: Create Git Branch

```bash
cd /path/to/bevel
git checkout -b local
git push --set-upstream origin local
```

### Step 6: Run Deployment

#### Option A: Using Bevel Build Container (Recommended)

```bash
# From your bevel directory
docker run -it -v $(pwd):/home/bevel/ --network="host" \
  ghcr.io/hyperledger/bevel-build:latest /bin/bash

# Inside the container
cd /home/bevel

# Setup git config (if not already done)
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# Run the deployment script
./run.sh
```

When prompted, provide the path: `/home/bevel/build/network.yaml`

#### Option B: Direct Ansible Execution

If you have Ansible installed locally:

```bash
# Full deployment including prerequisites
ansible-playbook platforms/shared/configuration/site.yaml \
  -e "@build/network.yaml"

# OR, just Fabric deployment (if prerequisites already setup)
ansible-playbook platforms/hyperledger-fabric/configuration/deploy-network.yaml \
  -e "@build/network.yaml"
```

### Step 7: Monitor Deployment

In another terminal, monitor the pod creation:

```bash
# Watch all pods across namespaces
kubectl get pods --all-namespaces -w

# Check specific namespace
kubectl get pods -n supplychain-net
kubectl get pods -n carrier-net
kubectl get pods -n manufacturer-net

# Check pod details if issues arise
kubectl describe pod <pod-name> -n <namespace>
```

### Step 8: Verify Deployment

After deployment completes (15-30 minutes typically):

```bash
# Check all pods are running
kubectl get pods --all-namespaces

# Verify channel creation
kubectl logs -n carrier-net $(kubectl get pods -n carrier-net -l app=peer0 -o jsonpath='{.items[0].metadata.name}') | grep -i "joined channel"

# Access CLI pod (if enabled)
kubectl exec -it $(kubectl get pods -n carrier-net -l app=peer0-cli -o jsonpath='{.items[0].metadata.name}') -n carrier-net -- bash
peer channel list
```

## Exposing the Fabric Network for External Applications

By default, the minikube configuration uses `proxy: none`, which means services are only accessible within the Kubernetes cluster using internal DNS. To connect external applications to your Fabric network, you have several options:

### Option 1: Using Nginx Ingress (Recommended for Testing)

This approach uses minikube's built-in Nginx ingress controller to expose services via host-based routing.

#### Enable Minikube Ingress

```bash
# Enable the ingress addon
minikube addons enable ingress

# Verify ingress controller is running
kubectl get pods -n ingress-nginx
```

#### Create Ingress Resources

Since the Bevel charts only create Ingress when `proxy.provider` is "haproxy", you'll need to create custom Ingress resources:

```bash
# Create ingress configuration
cat > build/fabric-ingress.yaml <<'EOF'
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: orderer1-ingress
  namespace: supplychain-net
  annotations:
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
spec:
  ingressClassName: nginx
  rules:
  - host: orderer1.supplychain-net.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: orderer1
            port:
              number: 7050
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: peer0-ingress
  namespace: carrier-net
  annotations:
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
spec:
  ingressClassName: nginx
  rules:
  - host: peer0.carrier-net.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: peer0
            port:
              number: 7051
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ca-ingress
  namespace: carrier-net
  annotations:
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
spec:
  ingressClassName: nginx
  rules:
  - host: ca.carrier-net.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ca
            port:
              number: 7054
EOF

# Apply the ingress resources
kubectl apply -f build/fabric-ingress.yaml
```

#### Configure DNS

Add entries to your `/etc/hosts` (or `C:\Windows\System32\drivers\etc\hosts` on Windows):

```bash
# Get minikube IP
MINIKUBE_IP=$(minikube ip)
echo "Minikube IP: $MINIKUBE_IP"

# Add to /etc/hosts
sudo bash -c "cat >> /etc/hosts <<EOF
$MINIKUBE_IP orderer1.supplychain-net.local
$MINIKUBE_IP peer0.carrier-net.local
$MINIKUBE_IP peer0.manufacturer-net.local
$MINIKUBE_IP ca.carrier-net.local
$MINIKUBE_IP ca.manufacturer-net.local
EOF"
```

#### Test Connectivity

```bash
# Test orderer
openssl s_client -connect orderer1.supplychain-net.local:443 -servername orderer1.supplychain-net.local

# Test peer
openssl s_client -connect peer0.carrier-net.local:443 -servername peer0.carrier-net.local
```

#### Configure Client Connection Profile

```json
{
  "orderers": {
    "orderer1.supplychain-net": {
      "url": "grpcs://orderer1.supplychain-net.local:443",
      "grpcOptions": {
        "ssl-target-name-override": "orderer1.supplychain-net"
      }
    }
  },
  "peers": {
    "peer0.carrier-net": {
      "url": "grpcs://peer0.carrier-net.local:443",
      "grpcOptions": {
        "ssl-target-name-override": "peer0.carrier-net"
      }
    }
  }
}
```

### Option 2: Using NodePort Services

NodePort exposes services on high-numbered ports (30000-32767) directly on the minikube node.

#### Upgrade Services to NodePort

```bash
# Upgrade peer service
helm upgrade peer0 ./platforms/hyperledger-fabric/charts/fabric-peernode \
  --namespace carrier-net \
  --reuse-values \
  --set peer.serviceType=NodePort \
  --set peer.ports.grpc.nodePort=30051 \
  --set peer.ports.events.nodePort=30053

# Upgrade orderer service
helm upgrade orderer1 ./platforms/hyperledger-fabric/charts/fabric-orderernode \
  --namespace supplychain-net \
  --reuse-values \
  --set orderer.serviceType=NodePort \
  --set orderer.ports.grpc.nodeport=30050

# Upgrade CA service
helm upgrade ca ./platforms/hyperledger-fabric/charts/fabric-ca-server \
  --namespace carrier-net \
  --reuse-values \
  --set service.serviceType=NodePort \
  --set service.ports.tcp.nodeport=30054
```

#### Verify NodePort Assignments

```bash
# Check services
kubectl get svc -n supplychain-net
kubectl get svc -n carrier-net

# Get minikube IP
MINIKUBE_IP=$(minikube ip)
echo "Services accessible at: $MINIKUBE_IP"
echo "  - Orderer: $MINIKUBE_IP:30050"
echo "  - Peer0: $MINIKUBE_IP:30051"
echo "  - CA: $MINIKUBE_IP:30054"
```

#### Configure Client Connection Profile

```json
{
  "orderers": {
    "orderer1.supplychain-net": {
      "url": "grpcs://192.168.49.2:30050",
      "grpcOptions": {
        "ssl-target-name-override": "orderer1.supplychain-net"
      }
    }
  },
  "peers": {
    "peer0.carrier-net": {
      "url": "grpcs://192.168.49.2:30051",
      "grpcOptions": {
        "ssl-target-name-override": "peer0.carrier-net"
      }
    }
  }
}
```

### Option 3: Using Port Forwarding (Quick Testing)

Port forwarding is the quickest method for testing but requires keeping terminal sessions open.

```bash
# Forward peer gRPC port
kubectl port-forward -n carrier-net svc/peer0 7051:7051 &

# Forward peer events port
kubectl port-forward -n carrier-net svc/peer0 7053:7053 &

# Forward orderer port
kubectl port-forward -n supplychain-net svc/orderer1 7050:7050 &

# Forward CA port
kubectl port-forward -n carrier-net svc/ca 7054:7054 &
```

Your application can then connect to `localhost:7051`, `localhost:7050`, etc.

To stop port forwarding:
```bash
# Kill all port-forward processes
pkill -f "kubectl port-forward"
```

### Option 4: Using Minikube Tunnel (LoadBalancer)

Minikube tunnel allows LoadBalancer services to work on minikube.

```bash
# Start tunnel (requires sudo and must stay running)
minikube tunnel
```

In another terminal:

```bash
# Upgrade services to LoadBalancer type
helm upgrade peer0 ./platforms/hyperledger-fabric/charts/fabric-peernode \
  --namespace carrier-net \
  --reuse-values \
  --set peer.serviceType=LoadBalancer

# Check assigned external IPs
kubectl get svc -n carrier-net
```

### Comparison of Exposure Methods

| Method | Complexity | Stability | Use Case |
|--------|-----------|-----------|----------|
| **Nginx Ingress** | Medium | High | External apps with DNS |
| **NodePort** | Low | High | Simple external access |
| **Port Forward** | Very Low | Low | Quick testing only |
| **Minikube Tunnel** | Medium | Medium | LoadBalancer testing |

**Recommendation**: Use **Nginx Ingress** for a production-like setup, or **NodePort** for simplicity.

### Obtaining Certificates for External Applications

External applications need TLS certificates to connect. Retrieve them from Vault:

```bash
# Set Vault credentials
export VAULT_ADDR='http://192.168.X.X:8200'
export VAULT_TOKEN="your_vault_root_token"

# Create directory for certificates
mkdir -p build/crypto/{orderer,peer,ca}

# Get orderer TLS CA certificate
vault kv get -field=ca.crt \
  secretsv2/crypto/ordererOrganizations/supplychain-net/orderers/orderer1.supplychain-net/tls \
  > build/crypto/orderer/ca.crt

# Get peer TLS CA certificate
vault kv get -field=ca.crt \
  secretsv2/crypto/peerOrganizations/carrier-net/peers/peer0.carrier-net/tls \
  > build/crypto/peer/ca.crt

# Get admin user certificate (for signing transactions)
vault kv get -field=user.crt \
  secretsv2/crypto/peerOrganizations/carrier-net/users/admin/msp \
  > build/crypto/peer/admin.crt

# Get admin user private key
vault kv get -field=user.key \
  secretsv2/crypto/peerOrganizations/carrier-net/users/admin/msp \
  > build/crypto/peer/admin.key

# Get CA root certificate
vault kv get -field=ca.carrier-net-cert.pem \
  secretsv2/crypto/peerOrganizations/carrier-net/ca \
  > build/crypto/ca/ca-cert.pem
```

### Complete Connection Profile Example

Here's a complete connection profile for external applications:

```json
{
  "name": "fabric-minikube-network",
  "version": "1.0.0",
  "client": {
    "organization": "Carrier",
    "connection": {
      "timeout": {
        "peer": { "endorser": "300" },
        "orderer": "300"
      }
    }
  },
  "channels": {
    "allchannel": {
      "orderers": ["orderer1.supplychain-net"],
      "peers": {
        "peer0.carrier-net": {
          "endorsingPeer": true,
          "chaincodeQuery": true,
          "ledgerQuery": true,
          "eventSource": true
        }
      }
    }
  },
  "organizations": {
    "Carrier": {
      "mspid": "carrierMSP",
      "peers": ["peer0.carrier-net"],
      "certificateAuthorities": ["ca.carrier-net"]
    },
    "Supplychain": {
      "mspid": "supplychainMSP",
      "orderers": ["orderer1.supplychain-net"]
    }
  },
  "orderers": {
    "orderer1.supplychain-net": {
      "url": "grpcs://orderer1.supplychain-net.local:443",
      "tlsCACerts": {
        "path": "./build/crypto/orderer/ca.crt"
      },
      "grpcOptions": {
        "ssl-target-name-override": "orderer1.supplychain-net",
        "hostnameOverride": "orderer1.supplychain-net"
      }
    }
  },
  "peers": {
    "peer0.carrier-net": {
      "url": "grpcs://peer0.carrier-net.local:443",
      "tlsCACerts": {
        "path": "./build/crypto/peer/ca.crt"
      },
      "grpcOptions": {
        "ssl-target-name-override": "peer0.carrier-net",
        "hostnameOverride": "peer0.carrier-net"
      }
    }
  },
  "certificateAuthorities": {
    "ca.carrier-net": {
      "url": "https://ca.carrier-net.local:443",
      "caName": "ca-carrier",
      "tlsCACerts": {
        "path": "./build/crypto/ca/ca-cert.pem"
      },
      "httpOptions": {
        "verify": true
      }
    }
  }
}
```

Note: Adjust URLs based on your chosen exposure method (Ingress vs NodePort vs localhost).

## Deploying Chaincode (Optional)

If you want to deploy chaincode after the network is running:

### Step 1: Update network.yaml

Add chaincode configuration under the peer service:

```yaml
services:
  peers:
    - peer:
      name: peer0
      # ... other config ...
      chaincodes:
        - name: "supplychain"
          version: "1"
          maindirectory: "cmd"
          lang: "golang"
          repository:
            username: "<github_username>"
            password: "<github_token>"
            url: "github.com/<username>/bevel-samples.git"
            branch: main
            path: "examples/supplychain-app/fabric/chaincode_rest_server/chaincode/"
          arguments: '\"init\",\"\"'
          endorsements: ""
```

### Step 2: Run Chaincode Operations Playbook

```bash
ansible-playbook platforms/hyperledger-fabric/configuration/chaincode-ops.yaml \
  -e "@build/network.yaml" \
  -e "add_new_org='false'"
```

## Critical Configuration Points

### Addressing Rules for Minikube

**ALWAYS use internal Kubernetes DNS format**:
- Orderers: `<orderer-name>.<namespace>:<port>`
  - Example: `orderer1.supplychain-net:7050`
- Peers: `<peer-name>.<namespace>:<port>`
  - Example: `peer0.carrier-net:7051`

**NEVER use**:
- External domains (e.g., `*.blockchaincloudpoc.com`)
- Localhost or 127.0.0.1
- Port 443 (use standard Fabric ports)

### Port Reference

Standard Fabric ports:
- CA: 7054
- Orderer: 7050
- Peer: 7051
- Peer events: 7053
- CouchDB: 5984

### Vault Configuration

- **URL**: Must use your machine's local IP (NOT localhost)
- Find your IP: `ip addr show` or `ifconfig`
- Format: `http://192.168.X.X:8200`
- Must be accessible from both your machine and minikube cluster

### GitOps Configuration

- Create a dedicated `local` branch in your forked repo
- Use GitHub token (not password) if 2FA is enabled
- Ensure the token has write access to the repository
- The gitops path should match your repository structure

## Common Issues and Solutions

### Issue: Connection Refused to Vault

**Symptom**: Ansible fails with "Connection refused" to Vault

**Solution**:
1. Verify Vault is running: `vault status`
2. Check you're using local IP, not localhost
3. Ensure Vault is accessible: `curl http://<your-ip>:8200/v1/sys/health`

### Issue: Pods in CrashLoopBackOff

**Symptom**: Pods repeatedly crash

**Solution**:
1. Check logs: `kubectl logs <pod-name> -n <namespace>`
2. Verify certificates were generated in Vault
3. Check resource constraints: `kubectl describe pod <pod-name> -n <namespace>`

### Issue: Channel Creation Fails

**Symptom**: Channel jobs fail or timeout

**Solution**:
1. Verify orderer is running and healthy
2. Check orderer logs: `kubectl logs <orderer-pod> -n supplychain-net`
3. Ensure all peer organizations can reach orderer at internal address

### Issue: Chaincode Installation Fails

**Symptom**: Chaincode install/approve jobs fail

**Solution**:
1. Verify GitHub credentials in network.yaml
2. Check chaincode repository path is correct
3. Ensure chaincode dependencies can be downloaded from within cluster

### Issue: GitOps Push Fails

**Symptom**: Ansible fails when pushing to Git

**Solution**:
1. Verify GitHub token has write permissions
2. Ensure `local` branch exists
3. Check git config is set inside container

## Clean Up

To completely remove the Fabric network:

```bash
# Remove all Fabric resources
ansible-playbook platforms/hyperledger-fabric/configuration/reset-network.yaml \
  -e "@build/network.yaml"

# OR use the shared reset playbook
ansible-playbook platforms/shared/configuration/site.yaml \
  -e "@build/network.yaml" \
  -e "reset=true"

# Delete minikube cluster
minikube delete

# Stop Vault
# Ctrl+C if running in dev mode, or:
killall vault
```

## Network Architecture on Minikube

When deployed, your network will have the following structure:

```
Minikube Cluster
│
├── Namespace: supplychain-net
│   ├── CA Server (ca)
│   ├── Orderer1 (orderer1)
│   └── Orderer1 CouchDB (if needed)
│
├── Namespace: carrier-net
│   ├── CA Server (ca)
│   ├── Peer0 (peer0)
│   ├── Peer0 CouchDB
│   └── Peer0 CLI (if enabled)
│
└── Namespace: manufacturer-net
    ├── CA Server (ca)
    ├── Peer0 (peer0)
    ├── Peer0 CouchDB
    └── Peer0 CLI (if enabled)
```

All components communicate via internal Kubernetes service DNS.

## Next Steps

After successful deployment:

1. **Explore the CLI**: Use the peer CLI pods to interact with the network
2. **Install Chaincode**: Deploy your smart contracts
3. **Add Organizations**: Extend the network with new organizations
4. **Add Peers**: Scale organizations by adding more peers
5. **Test Applications**: Connect client applications to the network

## Additional Resources

- [Hyperledger Bevel Documentation](https://hyperledger-bevel.readthedocs.io/)
- [Fabric Network Configuration Guide](https://hyperledger-bevel.readthedocs.io/en/latest/operations/fabric_networkyaml.html)
- [General Minikube Setup](./bevel-minikube-setup.md)
- [Fabric Operations Guide](../../platforms/hyperledger-fabric/configuration/README.md)

## Troubleshooting Checklist

Before seeking help, verify:

- [ ] Minikube is running with sufficient resources
- [ ] Vault is accessible at the configured URL
- [ ] Kubeconfig paths are relative in build/config
- [ ] All addresses use internal K8s DNS format
- [ ] cloud_provider is set to "minikube" for all organizations
- [ ] proxy is set to "none"
- [ ] Git branch "local" exists and is pushed
- [ ] GitHub token has write permissions
- [ ] No external URLs or domains in orderer/peer addresses
- [ ] Vault has secretsv2 enabled
