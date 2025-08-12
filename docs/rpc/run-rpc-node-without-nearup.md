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

If you are looking to learn how to compile and run a NEAR RPC node natively for one of the following networks, this guide is for you.

- [`testnet`](/rpc/run-rpc-node-without-nearup#testnet)
- [`mainnet`](/rpc/run-rpc-node-without-nearup#mainnet)
- [`kubernetes`](/rpc/run-rpc-node-without-nearup#kubernetes-deployment)

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

When building your NEAR node you will have several branch options to choose from depending on your desired use:

- `master` : _(**Experimental**)_
  - Use this if you want to play around with the latest code and experiment. This branch is not guaranteed to be in a fully working state and there is absolutely no guarantee it will be compatible with the current state of *mainnet* or *testnet*.
- [`Latest stable release`](https://github.com/near/nearcore/tags) : _(**Stable**)_
  - Use this if you want to run a NEAR node for *mainnet*. For *mainnet*, please use the latest stable release. This version is used by mainnet validators and other nodes and is fully compatible with the current state of *mainnet*.
- [`Latest release candidates`](https://github.com/near/nearcore/tags) : _(**Release Candidates**)_
  - Use this if you want to run a NEAR node for *testnet*. For *testnet*, we first release a RC version and then later make that release stable. For testnet, please run the latest RC version.

**Version Selection Strategy:**

**Production Mainnet:**
- Use **latest stable release** from [NEAR releases](https://github.com/near/nearcore/releases)
- Current recommended: `2.6.5` (as of August 2025)
- Check version compatibility: `curl -s https://rpc.mainnet.near.org | jq .version`

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

## `testnet`

### 1. Clone `nearcore` project from GitHub

First, clone the [`nearcore` repository](https://github.com/near/nearcore).

```bash
git clone https://github.com/near/nearcore
cd nearcore
git fetch origin --tags
```

Checkout to the branch you need if not `master` (default). Latest release is recommended. Please check the [releases page on GitHub](https://github.com/near/nearcore/releases).

```bash
git checkout tags/2.6.5 -b mynode
```

### 2. Compile `nearcore` binary

In the `nearcore` folder run the following commands:

```bash
make release
```

This will start the compilation process. It will take some time depending on your machine power (e.g. i9 8-core CPU, 32 GB RAM, SSD takes approximately 25 minutes). Note that compilation will need over 1 GB of memory per virtual core the machine has. If the build fails with processes being killed, you might want to try reducing number of parallel jobs, for example: `CARGO_BUILD_JOBS=8 make release`.

**Build Optimization:**
```bash
# For machines with limited RAM
CARGO_BUILD_JOBS=4 make release

# For high-performance builds
CARGO_BUILD_JOBS=16 make release
```

The binary path is `target/release/neard`

### 3. Initialize working directory

The NEAR node requires a working directory with a couple of configuration files. Generate the initial required working directory by running:

```bash
./target/release/neard --home ~/.near init --chain-id testnet --download-genesis --download-config rpc
```

> You can specify trusted boot nodes that you'd like to use by pass in a flag during init: `--boot-nodes ed25519:4k9csx6zMiXy4waUvRMPTkEtAS2RFKLVScocR5HwN53P@34.73.25.182:24567,ed25519:4keFArc3M4SE1debUQWi3F1jiuFZSWThgVuA2Ja2p3Jv@34.94.158.10:24567,ed25519:D2t1KTLJuwKDhbcD9tMXcXaydMNykA99Cedz7SkJkdj2@35.234.138.23:24567`

> You can skip the `--home` argument if you are fine with the default working directory in `~/.near`. If not, pass your preferred location.

This command will create the required directory structure and will generate `config.json`, `node_key.json`, and `genesis.json` for `testnet` network.
- `config.json` - Configuration parameters which are responsive for how the node will work. This file should contain the following fields critical for RPC nodes:
  - `"tracked_shards": [0]` - to track all shards.
- `genesis.json` - A file with all the data the network started with at genesis. This contains initial accounts, contracts, access keys, and other records which represents the initial state of the blockchain.
- `node_key.json` -  A file which contains a public and private key for the node. Also includes an optional `account_id` parameter which is required to run a validator node (not covered in this doc).
- `data/` -  A folder in which a NEAR node will write it's state.

> **Heads up**
> The genesis file for `testnet` is big (6GB +) so this command will be running for a while and no progress will be shown.

### 4. Configure for RPC (Optional)

Edit `~/.near/config.json` to optimize for RPC usage:

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
    "boot_nodes": "ed25519:4k9csx6zMiXy4waUvRMPTkEtAS2RFKLVScocR5HwN53P@34.73.25.182:24567,ed25519:4keFArc3M4SE1debUQWi3F1jiuFZSWThgVuA2Ja2p3Jv@34.94.158.10:24567,ed25519:D2t1KTLJuwKDhbcD9tMXcXaydMNykA99Cedz7SkJkdj2@35.234.138.23:24567"
  },
  "tracked_shards": [0],
  "epoch_sync_enabled": true
}
```

### 5. System Optimization (Optional)

For better performance, optimize kernel parameters:

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

### 6. Get data backup

⚠️ **FREE SNAPSHOT SERVICE BY FASTNEAR WILL BE DEPRECATED STARTING JUNE 1ST, 2025. We strongly recommend all node operators to use [Epoch Sync](../intro/epoch_sync.md) when possible.**

The node is ready to be started. However, you must first sync up with the network. This means your node needs to download all the headers and blocks that other nodes in the network already have.

While snapshot-based syncing was previously the recommended default, we now recommend **Epoch Sync**—a faster, more lightweight method that allows a node to catch up from genesis without downloading a large state snapshot.

### 7. Run the node

To start your node simply run the following command:

```bash
./target/release/neard --home ~/.near run
```

That's all. The node is running and you can see log outputs in your console. It will download a bit of missing data since the last backup was performed but it shouldn't take much time.

---

## `mainnet`

### 1. Clone `nearcore` project from GitHub

First, clone the [`nearcore` repository](https://github.com/near/nearcore).

```bash
git clone https://github.com/near/nearcore
cd nearcore
git fetch origin --tags
```

Next, checkout the release branch you need (recommended) if you will not be using the default `master` branch. Please check the [releases page on GitHub](https://github.com/near/nearcore/releases) for the latest release.

```bash
git checkout tags/2.6.5 -b mynode
```

### 2. Compile `nearcore` binary

In the `nearcore` folder run the following commands:

```bash
make release
```

This will start the compilation process. It will take some time depending on your machine power (e.g. i9 8-core CPU, 32 GB RAM, SSD takes approximately 25 minutes). Note that compilation will need over 1 GB of memory per virtual core the machine has. If the build fails with processes being killed, you might want to try reducing number of parallel jobs, for example: `CARGO_BUILD_JOBS=8 make release`.

**Build Optimization:**
```bash
# For machines with limited RAM
CARGO_BUILD_JOBS=4 make release

# For high-performance builds
CARGO_BUILD_JOBS=16 make release
```

The binary path is `target/release/neard`

### 3. Initialize working directory

The NEAR node requires a working directory with a couple of configuration files. Generate the initial required working directory by running:

```bash
./target/release/neard --home ~/.near init --chain-id mainnet --download-genesis --download-config rpc
```

> You can specify trusted boot nodes that you'd like to use by pass in a flag during init: `--boot-nodes ed25519:CGiu5DuDxwkk6uE6NxJ22jYvq2hQ74LL8W8cmaf81ZkN@3.255.170.133:24567,ed25519:7fPwmfuDAAGLz8UG8oJC5Pv6zz2wAJ2RfVZTTFLV7aS8@5.9.147.217:24567,ed25519:EFNjoPzZB4HTvGzLg3KYSsj2649jJnLf7nkNesb8eFjP@136.243.81.236:24567,ed25519:4s2ApPCSmQMeUHB8TnUubof74APNPU4s3dwUUXoY8XdG@54.250.184.95:24567,ed25519:Hnsh6tFSrt9tEUvvn1sKyesjCsWcXvSZPKm7jyRP3dxV@141.94.160.175:24567`

> You can skip the `--home` argument if you are fine with the default working directory in `~/.near`. If not, pass your preferred location.

This command will create the required directory structure by generating a `config.json`, `node_key.json`, and downloads a `genesis.json` for `mainnet`.
- `config.json` - Configuration parameters which are responsive for how the node will work. This file should contain the following fields critical for RPC nodes:
  - `"tracked_shards": [0]` - to track all shards.
- `genesis.json` - A file with all the data the network started with at genesis. This contains initial accounts, contracts, access keys, and other records which represents the initial state of the blockchain.
- `node_key.json` -  A file which contains a public and private key for the node. Also includes an optional `account_id` parameter which is required to run a validator node (not covered in this doc).
- `data/` -  A folder in which a NEAR node will write it's state.

### 4. Configure for RPC (Optional)

Edit `~/.near/config.json` to optimize for RPC usage:

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
    "boot_nodes": "ed25519:CGiu5DuDxwkk6uE6NxJ22jYvq2hQ74LL8W8cmaf81ZkN@3.255.170.133:24567,ed25519:7fPwmfuDAAGLz8UG8oJC5Pv6zz2wAJ2RfVZTTFLV7aS8@5.9.147.217:24567,ed25519:EFNjoPzZB4HTvGzLg3KYSsj2649jJnLf7nkNesb8eFjP@136.243.81.236:24567,ed25519:4s2ApPCSmQMeUHB8TnUubof74APNPU4s3dwUUXoY8XdG@54.250.184.95:24567,ed25519:Hnsh6tFSrt9tEUvvn1sKyesjCsWcXvSZPKm7jyRP3dxV@141.94.160.175:24567"
  },
  "tracked_shards": [0],
  "epoch_sync_enabled": true
}
```

### 5. System Optimization (Optional)

For better performance, optimize kernel parameters:

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

### 6. Get data backup

⚠️ **FREE SNAPSHOT SERVICE BY FASTNEAR WILL BE DEPRECATED STARTING JUNE 1ST, 2025. We strongly recommend all node operators to use [Epoch Sync](../intro/epoch_sync.md) when possible.**

The node is ready to be started. However, you must first sync up with the network. This means your node needs to download all the headers and blocks that other nodes in the network already have.

While snapshot-based syncing was previously the recommended default, we now recommend **Epoch Sync**—a faster, more lightweight method that allows a node to catch up from genesis without downloading a large state snapshot.

### 7. Run the node

To start your node simply run the following command:

```bash
./target/release/neard --home ~/.near run
```

That's all. The node is running and you can see log outputs in your console. It will download a bit of missing data since the last backup was performed but it shouldn't take much time.

---

## Kubernetes Deployment

For production deployments, we recommend using Kubernetes for better scalability, monitoring, and maintenance. 

**📋 For detailed Kubernetes deployment instructions, see:** [**Run RPC Node on Kubernetes**](run-rpc-node-on-kubernetes.md)

### Quick Overview

Kubernetes deployment offers several advantages:
- **Persistent storage** with high-performance EBS volumes
- **Health monitoring** and automatic restarts
- **Resource management** and scaling
- **Integrated monitoring** with Prometheus/Grafana
- **Network optimization** via init containers

**Key considerations for Kubernetes:**
- Use `m5.2xlarge` or larger instances
- Configure `gp3-high-iops` storage class (16k IOPS)
- Allocate 1TB+ persistent storage
- Enable epoch sync for faster synchronization
- Monitor with Prometheus metrics on port 3031

---

## Running a node in `light` mode

Running a node in `light` mode allows the operator to access chain level data, not state level data. You can also use the `light` node to submit transactions or verify certain proofs. To run a node in a `light` mode that doesn't track any shards, the only change required is to update the `config.json` whereby `tracked_shards` is set to an empty array.

```json
"tracked_shards": []
```

---

## Expected Logs and Sync Process

### Normal Sync Progression

Understanding the sync process helps you monitor your node's progress:

**1. Initialization Phase (0-5 minutes):**
```
2025-08-10T07:15:47.120889Z  INFO neard: version="2.6.5" build="1.77.0" latest_protocol=77
2025-08-10T07:15:47.120889Z  INFO config: Validating Config, extracted from config.json...
```

**2. Epoch Sync (1-3 hours):**
```
2025-08-10T07:17:47.120889Z  INFO stats: [EPOCH] EpochSyncStatus { source_peer_height: 159013592... }
```

**3. Block Download (1-2 hours):**
```
2025-08-10T07:18:47.120889Z  INFO stats: #158993649 Downloading blocks 99.95% (77649 left...)
```

**4. State Sync (2-6 hours):**
```
2025-08-10T07:19:47.120889Z  INFO stats: State HffeFY...[0: parts][1: parts][6: done]...
```

**5. Fully Synced:**
```
2025-08-10T07:20:47.120889Z  INFO stats: #159205451 HuuU77... 300 validators 32 peers ⬇ 4.59 MB/s
```

### Performance Expectations

**Real-World Sync Times:**

| Hardware Spec | Network | Total Sync Time | Final Data Size | Avg Speed |
|---------------|---------|----------------|----------------|-----------|
| AWS m5.2xlarge | 100Mbps | 6-12 hours | ~265GB | 3-5 MB/s |
| AWS m5.4xlarge | 1Gbps | 4-8 hours | ~265GB | 7-10 MB/s |
| Bare Metal 32C/64GB | 1Gbps | 2-4 hours | ~265GB | 15+ MB/s |

---

## Monitoring and Maintenance

### Health Checks

```bash
# Check node status
curl -s http://localhost:3030/status | jq '.sync_info'

# Verify RPC functionality  
curl -s http://localhost:3030 \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":"1","method":"block","params":{"finality":"final"}}' \
  | jq '.result.header.height'

# Monitor Prometheus metrics
curl -s http://localhost:3031/metrics | grep -E "(near_block_height_head|near_peers_total)"
```

### Common Issues

**Slow Sync Speed:**
- Check network bandwidth and connectivity
- Verify SSD performance (`iostat -x 1`)
- Ensure sufficient RAM (32GB+ recommended)
- Update boot nodes if peers < 10

**High Memory Usage:**
- Normal during state application phase (can reach 20GB+)
- Monitor with `htop` or system monitoring tools
- Consider upgrading to larger instance if consistently hitting limits

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

### Boot Nodes Configuration

**Finding Active Boot Nodes:**
```bash
# Get current active peers from mainnet RPC
curl -s https://rpc.mainnet.near.org \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":"1","method":"network_info","params":[]}' \
  | jq -r '.result.active_peers[].id' | head -5
```

**Current Recommended Boot Nodes (August 2025):**

**Mainnet:**
```json
"boot_nodes": "ed25519:CGiu5DuDxwkk6uE6NxJ22jYvq2hQ74LL8W8cmaf81ZkN@3.255.170.133:24567,ed25519:7fPwmfuDAAGLz8UG8oJC5Pv6zz2wAJ2RfVZTTFLV7aS8@5.9.147.217:24567,ed25519:EFNjoPzZB4HTvGzLg3KYSsj2649jJnLf7nkNesb8eFjP@136.243.81.236:24567,ed25519:4s2ApPCSmQMeUHB8TnUubof74APNPU4s3dwUUXoY8XdG@54.250.184.95:24567,ed25519:Hnsh6tFSrt9tEUvvn1sKyesjCsWcXvSZPKm7jyRP3dxV@141.94.160.175:24567"
```

**Testnet:**
```json
"boot_nodes": "ed25519:4k9csx6zMiXy4waUvRMPTkEtAS2RFKLVScocR5HwN53P@34.73.25.182:24567,ed25519:4keFArc3M4SE1debUQWi3F1jiuFZSWThgVuA2Ja2p3Jv@34.94.158.10:24567,ed25519:D2t1KTLJuwKDhbcD9tMXcXaydMNykA99Cedz7SkJkdj2@35.234.138.23:24567"
```

---

>Got a question?
<a href="https://stackoverflow.com/questions/tagged/nearprotocol">
  <h8>Ask it on StackOverflow!</h8></a>

---

*Last updated: August 2025 | Version: 2.6.5 | Tested with real mainnet deployment*
