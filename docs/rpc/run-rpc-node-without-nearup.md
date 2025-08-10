---
id: run-rpc-node-without-nearup
title: Run an RPC Node
sidebar_label: Run a Node 🚀
sidebar_position: 2
description: Complete guide to run a NEAR RPC Node with native compilation and Kubernetes deployment
---

# Run a NEAR RPC Node

This comprehensive guide covers running a NEAR RPC node both natively and with Kubernetes, including real-world performance insights and troubleshooting tips.

The following instructions are applicable across localnet, testnet, and mainnet environments.

<blockquote class="info">
<strong>Heads up</strong><br /><br />

Running an RPC node is very similar to running a [validator node](/validator/running-a-node) as both use the same `nearcore` release. The main difference is that validator nodes require `validator_key.json` for block and chunk validation.

</blockquote>

## Prerequisites

### System Requirements

**Minimum Requirements:**
- **CPU:** 8+ cores (Intel i7/i9 or AMD Ryzen 7/9)
- **RAM:** 32GB+ (64GB recommended for mainnet)
- **Storage:** 500GB+ SSD (NVMe preferred)
- **Network:** 100+ Mbps with unlimited bandwidth

**Recommended Production Setup:**
- **AWS:** `m5.2xlarge` or `m5.4xlarge`
- **CPU:** 16+ cores
- **RAM:** 64GB+
- **Storage:** 1TB+ NVMe SSD

### Software Dependencies

- [Rust](https://www.rust-lang.org/) - Latest stable version
- [Git](https://git-scm.com/)
- [Docker](https://docker.com/) (for Kubernetes deployment)
- [jq](https://jqlang.github.io/jq/) - JSON processor for API queries

**Installation:**

**MacOS:**
```bash
brew install cmake protobuf clang llvm awscli git jq
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

**Ubuntu/Debian:**
```bash
apt update && apt install -y \
  git binutils-dev libcurl4-openssl-dev zlib1g-dev libdw-dev \
  libiberty-dev cmake gcc g++ python3 docker.io protobuf-compiler \
  libssl-dev pkg-config clang llvm cargo awscli curl jq

curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

---

## Choosing Your NEAR Version

### Version Selection Strategy

**Production Mainnet:**
- Use **latest stable release** from [NEAR releases](https://github.com/near/nearcore/releases)
- Current recommended: `2.6.5` (as of August 2025)
- Check version compatibility: `curl -s https://rpc.mainnet.near.org | jq .version`

**Testnet Development:**
- Use **latest release candidate** (RC versions)
- More experimental features available

**Version Checking:**
```bash
# Check current mainnet version
curl -s https://rpc.mainnet.near.org \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":"1","method":"status","params":[]}' \
  | jq -r '.result.version'

# List all available releases
curl -s https://api.github.com/repos/near/nearcore/releases | jq -r '.[].tag_name' | head -10
```

---

## Native Deployment

### 1. Clone and Build NEAR Core

```bash
# Clone repository
git clone https://github.com/near/nearcore
cd nearcore
git fetch origin --tags

# Checkout stable release (recommended for mainnet)
git checkout tags/2.6.5 -b mainnet-node

# Build (takes 20-40 minutes depending on hardware)
make release
```

**Build Optimization:**
```bash
# For machines with limited RAM
CARGO_BUILD_JOBS=4 make release

# For high-performance builds
CARGO_BUILD_JOBS=16 make release
```

### 2. Initialize Node Configuration

**Mainnet:**
```bash
./target/release/neard --home ~/.near init \
  --chain-id mainnet \
  --download-genesis \
  --download-config \
  --account-id your-node-name
```

**Testnet:**
```bash
./target/release/neard --home ~/.near init \
  --chain-id testnet \
  --download-genesis \
  --download-config
```

### 3. Configure for RPC

Edit `~/.near/config.json`:

```json
{
  "genesis_file": "genesis.json",
  "genesis_records_file": null,
  "validator_key_file": "validator_key.json",
  "node_key_file": "node_key.json",
  "rpc": {
    "addr": "0.0.0.0:3030",
    "prometheus": {
      "addr": "0.0.0.0:3031"
    }
  },
  "network": {
    "addr": "0.0.0.0:24567",
    "boot_nodes": "ed25519:CGiu5DuDxwkk6uE6NxJ22jYvq2hQ74LL8W8cmaf81ZkN@3.255.170.133:24567,ed25519:7fPwmfuDAAGLz8UG8oJC5Pv6zz2wAJ2RfVZTTFLV7aS8@5.9.147.217:24567"
  },
  "tracked_shards": [0],
  "epoch_sync_enabled": true
}
```

### 4. System Optimization

```bash
# Optimize kernel parameters
sudo sysctl -w net.core.rmem_max=8388608
sudo sysctl -w net.core.wmem_max=8388608
sudo sysctl -w net.ipv4.tcp_rmem="4096 87380 8388608"
sudo sysctl -w net.ipv4.tcp_wmem="4096 16384 8388608"
sudo sysctl -w net.ipv4.tcp_slow_start_after_idle=0

# Make permanent
echo 'net.core.rmem_max = 8388608' | sudo tee -a /etc/sysctl.conf
echo 'net.core.wmem_max = 8388608' | sudo tee -a /etc/sysctl.conf
```

### 5. Run the Node

```bash
# Start node
./target/release/neard --home ~/.near run

# Or with custom logging
RUST_LOG=near=info ./target/release/neard --home ~/.near run
```

---

## Kubernetes Deployment

### Prerequisites

- Kubernetes cluster (EKS, GKE, or AKS recommended)
- kubectl configured
- Persistent storage class available

### 1. Deployment Manifest

Create `near-rpc-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: near-rpc
  namespace: near
  labels:
    app: near-rpc
spec:
  replicas: 1
  selector:
    matchLabels:
      app: near-rpc
  template:
    metadata:
      labels:
        app: near-rpc
    spec:
      # Use dedicated high-performance nodes
      nodeSelector:
        node-type: high-performance
      
      initContainers:
      - name: sysctl-optimization
        image: busybox:1.35
        command: ['sh', '-c']
        args:
        - |
          sysctl -w net.ipv4.tcp_rmem="4096 87380 8388608"
          sysctl -w net.ipv4.tcp_wmem="4096 16384 8388608" 
          sysctl -w net.ipv4.tcp_slow_start_after_idle=0
          sysctl -w net.core.rmem_max=8388608
          sysctl -w net.core.wmem_max=8388608
          echo "Network parameters optimized"
        securityContext:
          privileged: true
      
      containers:
      - name: near-rpc
        image: nearprotocol/nearcore:2.6.5
        command: ["/bin/bash", "-c"]
        args:
        - |
          echo "Initializing NEAR RPC node..."
          
          # Initialize if no config exists
          if [ ! -f "/data/.near/config.json" ]; then
            neard --home /data/.near init \
              --chain-id mainnet \
              --account-id rpc-node \
              --fast
            
            # Configure active boot nodes (updated regularly)
            sed -i 's/"boot_nodes": ""/"boot_nodes": "ed25519:CGiu5DuDxwkk6uE6NxJ22jYvq2hQ74LL8W8cmaf81ZkN@3.255.170.133:24567,ed25519:7fPwmfuDAAGLz8UG8oJC5Pv6zz2wAJ2RfVZTTFLV7aS8@5.9.147.217:24567,ed25519:EFNjoPzZB4HTvGzLg3KYSsj2649jJnLf7nkNesb8eFjP@136.243.81.236:24567"/' /data/.near/config.json
            
            # Enable Prometheus metrics
            sed -i 's/"prometheus_addr": null/"prometheus_addr": "0.0.0.0:3031"/' /data/.near/config.json
            
            # Enable Epoch Sync for faster synchronization
            sed -i '$s/}/,"epoch_sync_enabled": true}/' /data/.near/config.json
          fi
          
          echo "Starting NEAR RPC node..."
          exec neard --home /data/.near run
          
        ports:
        - name: rpc
          containerPort: 3030
        - name: network
          containerPort: 24567
        - name: metrics
          containerPort: 3031
          
        resources:
          requests:
            cpu: "8"
            memory: "32Gi"
          limits:
            cpu: "16"
            memory: "64Gi"
            
        volumeMounts:
        - name: near-data
          mountPath: /data/.near
          
        # Health checks
        livenessProbe:
          exec:
            command:
            - /bin/bash
            - -c
            - "curl -f http://localhost:3030/status || exit 1"
          initialDelaySeconds: 300
          periodSeconds: 60
          
        readinessProbe:
          exec:
            command:
            - /bin/bash
            - -c
            - "curl -f http://localhost:3030/status || exit 1"
          initialDelaySeconds: 60
          periodSeconds: 30
          
      volumes:
      - name: near-data
        persistentVolumeClaim:
          claimName: near-data-pvc
        
      terminationGracePeriodSeconds: 300
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: near-data-pvc
  namespace: near
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Ti
  storageClassName: fast-ssd
---
apiVersion: v1
kind: Service
metadata:
  name: near-rpc-service
  namespace: near
spec:
  selector:
    app: near-rpc
  ports:
    - name: rpc
      port: 3030
      targetPort: 3030
    - name: metrics
      port: 3031
      targetPort: 3031
  type: LoadBalancer
```

### 2. Deploy to Kubernetes

```bash
# Create namespace
kubectl create namespace near

# Apply deployment
kubectl apply -f near-rpc-deployment.yaml

# Monitor deployment
kubectl get pods -n near -w
```

### 3. Monitor Logs

```bash
# Follow logs
kubectl logs -f -n near deployment/near-rpc

# Check sync status
kubectl exec -n near deployment/near-rpc -- curl -s http://localhost:3030/status
```

---

## Boot Nodes Configuration

### Understanding Boot Nodes

Boot nodes are initial connection points for your NEAR node to discover and connect to the network. Using active, geographically distributed boot nodes improves sync performance.

### Finding Active Boot Nodes

**Method 1: Query Active Peers**
```bash
# Get current active peers from mainnet RPC
curl -s https://rpc.mainnet.near.org \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":"1","method":"network_info","params":[]}' \
  | jq -r '.result.active_peers[].id' | head -5
```

**Method 2: Community Resources**
- [NEAR Official Boot Nodes](https://github.com/near/nearcore/blob/master/chain-configs/mainnet/config.json)
- [Community Node Lists](https://near-nodes.io)

### Current Recommended Boot Nodes (August 2025)

**Mainnet:**
```json
"boot_nodes": "ed25519:CGiu5DuDxwkk6uE6NxJ22jYvq2hQ74LL8W8cmaf81ZkN@3.255.170.133:24567,ed25519:7fPwmfuDAAGLz8UG8oJC5Pv6zz2wAJ2RfVZTTFLV7aS8@5.9.147.217:24567,ed25519:EFNjoPzZB4HTvGzLg3KYSsj2649jJnLf7nkNesb8eFjP@136.243.81.236:24567,ed25519:4s2ApPCSmQMeUHB8TnUubof74APNPU4s3dwUUXoY8XdG@54.250.184.95:24567,ed25519:Hnsh6tFSrt9tEUvvn1sKyesjCsWcXvSZPKm7jyRP3dxV@141.94.160.175:24567"
```

**Testnet:**
```json
"boot_nodes": "ed25519:4k9csx6zMiXy4waUvRMPTkEtAS2RFKLVScocR5HwN53P@34.73.25.182:24567,ed25519:4keFArc3M4SE1debUQWi3F1jiuFZSWThgVuA2Ja2p3Jv@34.94.158.10:24567,ed25519:D2t1KTLJuwKDhbcD9tMXcXaydMNykA99Cedz7SkJkdj2@35.234.138.23:24567"
```

---

## Expected Logs and Sync Process

### Normal Sync Progression

**1. Initialization Phase (0-5 minutes):**
```
2025-08-10T07:15:47.120889Z  INFO neard: version="2.6.5" build="1.77.0" latest_protocol=77
2025-08-10T07:15:47.120889Z  INFO config: Validating Config, extracted from config.json...
2025-08-10T07:15:47.120889Z  INFO db_opener: Opening NodeStorage path="/data/.near/data"
```

**2. Network Connection (5-15 minutes):**
```
2025-08-10T07:15:47.120889Z  INFO stats: # 9820210 Waiting for peers 0 peers ⬇ 0 B/s ⬆ 0 B/s
2025-08-10T07:16:47.120889Z  INFO stats: # 9820210 Downloading headers 100.00% 30 peers ⬇ 2.30 MB/s ⬆ 1.94 MB/s
```

**3. State Sync Phase (2-12 hours depending on hardware):**
```
2025-08-10T07:17:47.120889Z  INFO near_client::sync::state::shard: Running state sync for shard 0
2025-08-10T07:18:47.120889Z  INFO stats: State 36B8K2ikiwT3MMK5BpbLeNJrSbvxf2QCKqEXqtxP5EFE[0: parts][1: parts] 30 peers ⬇ 3.42 MB/s ⬆ 1.21 MB/s CPU: 42%, Mem: 9.01 GB
```

**4. Final Application Phase (1-4 hours):**
```
2025-08-10T07:19:47.120889Z  INFO stats: State 36B8K2ikiwT3MMK5BpbLeNJrSbvxf2QCKqEXqtxP5EFE[0: apply in progress][1: done] (0 downloads, 4 computations) CPU: 307%, Mem: 19.5 GB
```

**5. Full Sync Complete:**
```
2025-08-10T07:20:47.120889Z  INFO stats: # 158951706 V/1 0/0/40 peers ⬇ 1.23 MB/s ⬆ 0.89 MB/s 0.00 bps CPU: 15%, Mem: 2.1 GB
```

### Performance Metrics

**Real-World Sync Times (Based on Our Experience):**

| Hardware Spec | Network | Total Sync Time | Data Size | Avg Speed |
|---------------|---------|----------------|-----------|-----------|
| AWS m5.2xlarge | 100Mbps | 8-12 hours | ~350GB | 3.5 MB/s |
| AWS m5.4xlarge | 1Gbps | 4-6 hours | ~350GB | 7-10 MB/s |
| Bare Metal 32C/64GB | 1Gbps | 2-4 hours | ~350GB | 15+ MB/s |

**Progress Indicators:**
- **0-25%:** Initial state download (fast)
- **25-75%:** Bulk state sync (steady 3-5 MB/s)
- **75-95%:** State application (CPU intensive, slower progress)
- **95-100%:** Final catch-up (fast)

### Troubleshooting Common Issues

**No Peers Connected:**
```bash
# Check boot nodes connectivity
nc -zv 3.255.170.133 24567

# Update boot nodes in config.json
# Restart node
```

**Slow Sync Speed:**
- Check network bandwidth
- Verify SSD performance (`iostat -x 1`)
- Ensure sufficient RAM (32GB+ recommended)

**High Memory Usage:**
- Normal during state application phase
- Monitor with `kubectl top pods` (Kubernetes) or `htop`
- Consider scaling to larger instance

---

## Monitoring and Maintenance

### Prometheus Metrics

Access metrics at `http://your-node:3031/metrics`:

```bash
# Key metrics to monitor
curl -s http://localhost:3031/metrics | grep -E "(near_block_height_head|near_peers_total)"
```

### Health Checks

```bash
# Check node status
curl -s http://localhost:3030/status | jq '.sync_info'

# Verify RPC functionality
curl -s http://localhost:3030 \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":"1","method":"block","params":{"finality":"final"}}' \
  | jq '.result.header.height'
```

### Backup Strategy

```bash
# Stop node gracefully
kubectl scale deployment/near-rpc --replicas=0

# Backup data (example with AWS)
kubectl run backup-pod --image=amazon/aws-cli --restart=Never -- \
  aws s3 sync /data/.near s3://your-backup-bucket/near-backup-$(date +%Y%m%d)

# Restore from backup
aws s3 sync s3://your-backup-bucket/near-backup-20250810 /data/.near
```

---

## Advanced Configuration

### Epoch Sync (Recommended)

Enable faster synchronization in `config.json`:

```json
{
  "epoch_sync_enabled": true,
  "epoch_sync": {
    "timeout_total": 300,
    "timeout_per_part": 60
  }
}
```

### Light Client Mode

For reduced resource usage:

```json
{
  "tracked_shards": [],
  "archive": false
}
```

### Custom Network Configuration

```json
{
  "network": {
    "addr": "0.0.0.0:24567",
    "external_address": "your-public-ip:24567",
    "boot_nodes": "your,custom,boot,nodes",
    "max_num_peers": 100
  }
}
```

---

## Production Checklist

**Pre-Deployment:**
- [ ] Hardware meets minimum requirements
- [ ] Network bandwidth sufficient (100+ Mbps)
- [ ] Storage monitoring configured
- [ ] Boot nodes verified active
- [ ] Backup strategy implemented

**Post-Deployment:**
- [ ] Prometheus metrics accessible
- [ ] RPC endpoints responding
- [ ] Peer count > 20
- [ ] Sync progressing normally
- [ ] Resource usage within limits

**Ongoing Maintenance:**
- [ ] Monitor sync progress daily
- [ ] Update boot nodes quarterly
- [ ] Backup data weekly
- [ ] Upgrade NEAR version monthly
- [ ] Review resource usage trends

---

## Getting Support

**Community Resources:**
- [GitHub Issues](https://github.com/near/nearcore/issues)
- [Stack Overflow](https://stackoverflow.com/questions/tagged/nearprotocol)

**Monitoring Tools:**
- [NEAR Explorer](https://nearblocks.io)

---

*Last updated: August 2025 | Version: 2.6.5 | Tested on AWS EKS, m5.2xlarge*
