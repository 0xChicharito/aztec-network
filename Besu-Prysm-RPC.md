# Besu-Prysm-RPC

### Hardware Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| **CPU** | 4 cores | 4+ cores |
| **RAM** | 12GB | 16GB+ |
| **Storage** | 1TB SSD | 1TB+ SSD |

it will take 640gb of storage after getting synced. tested both Aztec and RPC nodes on a $10 vps and it works properly. (Servarica KVM FAT Slice 4)
### Install Required Packages

```bash
sudo apt-get update && sudo apt-get upgrade -y
```
```bash
sudo apt install curl iptables build-essential git wget lz4 jq make gcc nano automake autoconf tmux htop nvme-cli libgbm1 pkg-config libssl-dev libleveldb-dev tar clang bsdmainutils ncdu unzip libleveldb-dev ufw screen gawk pv -y
```

### Install Docker (Latest Version)

Run this script to install the latest Docker version:

```bash
#!/bin/bash
set -e

if [ ! -f /etc/os-release ]; then
  echo "Not Ubuntu or Debian"
  exit 1
fi

sudo apt update -y && sudo apt upgrade -y
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc docker-ce docker-ce-cli docker-buildx-plugin docker-compose-plugin; do
  sudo apt-get remove --purge -y $pkg 2>/dev/null || true
done
sudo apt-get autoremove -y
sudo rm -rf /var/lib/docker /var/lib/containerd /etc/docker /etc/apt/sources.list.d/docker.list /etc/apt/keyrings/docker.gpg

sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg lsb-release
sudo install -m 0755 -d /etc/apt/keyrings

. /etc/os-release
repo_url="https://download.docker.com/linux/$ID"
curl -fsSL "$repo_url/gpg" | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] $repo_url $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update -y && sudo apt upgrade -y
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

if sudo docker run hello-world; then
  sudo docker rm $(sudo docker ps -a --filter "ancestor=hello-world" --format "{{.ID}}") --force 2>/dev/null || true
  sudo docker image rm hello-world 2>/dev/null || true
  sudo systemctl enable docker
  sudo systemctl restart docker
  clear
  echo -e "• Docker Installed ✓"
fi
```
## Installation Steps
## Method 1: Automated setup
just run this cmd in terminal and it'll install the RPC
```bash
screen -S rpc bash -c "wget -O besu-prysm.sh https://raw.githubusercontent.com/AminMAD86/Besu-Prysm-1script-installation/refs/heads/main/Besu-Prysm && chmod +x besu-prysm.sh && ./besu-prysm.sh"
```
description: it'll open a screen, create directories, generate jwt token, download the snapshot, create docker-compose.yml and start the node. it'll take about 6 hours to sync.
## Method 2: Manual setep
### Step 1: Create Directories

```bash
mkdir -p /root/ethereum/execution
mkdir -p /root/ethereum/consensus
```

### Step 2: Generate JWT Secret

The JWT secret is used for authenticated communication between execution and consensus clients:

```bash
openssl rand -hex 32 > /root/ethereum/jwt.hex
```

Verify it was created correctly:
```bash
cat /root/ethereum/jwt.hex
# Should show a 64-character hex string
```
### Step 3: Download and Extract Besu Snapshot

**Important**: Run this in a `screen` session to prevent interruption if your terminal disconnects. or run it directly on terminal if you prefer.

#### 3.1: Start a screen session
```bash
screen -S besu-snapshot
```

#### 3.2: Download and extract snapshot
```bash
cd /root/ethereum/execution
wget -q -O - "https://snapshots.publicnode.com/ethereum-sepolia-besu-9349545.tar.lz4" | pv | lz4 -d | tar -xf -
```
#### 3.3: Wait for completion
The download will take approximately **1-2 hours** depending on your vps internet speed

Once completed, exit the screen session `Ctrl+a ,d`
### Step 4: Create Docker Compose Configuration

Navigate to the ethereum directory and create the `docker-compose.yml` file:

```bash
cd /root/ethereum
nano docker-compose.yml
```

Paste the following configuration:
```
services:
  besu:
    image: hyperledger/besu:latest
    container_name: besu
    network_mode: host
    restart: unless-stopped
    ports:
      - 30303:30303
      - 30303:30303/udp
      - 8545:8545
      - 8546:8546
      - 8551:8551
    volumes:
      - /root/ethereum/execution:/data
      - /root/ethereum/jwt.hex:/data/jwt.hex
    command:
      - --network=sepolia
      - --data-path=/data
      - --sync-mode=SNAP
      - --rpc-http-enabled=true
      - --rpc-http-host=0.0.0.0
      - --rpc-http-port=8545
      - --rpc-http-api=ETH,NET,WEB3
      - --rpc-http-cors-origins=*
      - --host-allowlist=*
      - --rpc-http-max-active-connections=500
      - --rpc-http-max-request-content-length=52428800
      - --engine-rpc-enabled=true
      - --engine-host-allowlist=*
      - --engine-jwt-secret=/data/jwt.hex
      - --engine-rpc-port=8551
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  prysm:
    image: gcr.io/offchainlabs/prysm/beacon-chain:v6.1.2
    container_name: prysm
    network_mode: host
    restart: unless-stopped
    volumes:
      - /root/ethereum/consensus:/data
      - /root/ethereum/jwt.hex:/data/jwt.hex
    depends_on:
      - besu
    ports:
      - 4000:4000
      - 3500:3500
    command:
      - --sepolia
      - --execution-endpoint=http://127.0.0.1:8551
      - --jwt-secret=/data/jwt.hex
      - --checkpoint-sync-url=https://sepolia.beaconstate.info
      - --genesis-beacon-api-url=https://sepolia.beaconstate.info
      - --accept-terms-of-use
      - --datadir=/data
      - --rpc-host=0.0.0.0
      - --rpc-port=4000
      - --grpc-gateway-corsdomain=*
      - --grpc-gateway-host=0.0.0.0
      - --grpc-gateway-port=3500
      - --monitoring-port=8081
      - --disable-monitoring
      - --subscribe-all-data-subnets
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```
Save and exit: `Ctrl+X`, then `Y`, then `Enter`
### Step 5: Start the Node

```bash
docker compose up -d
```

### Step 6: Monitor the Logs

```bash
# Watch both services
docker compose logs -f
```
To exit log view: Press `Ctrl+C
### Check if both clients are synced:

**Besu sync status:**
it will take about 4 hours. or even less if you downloaded the newest snapshot.
```bash
curl -X POST -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}' \
  http://localhost:8545
```

**Expected when synced:** `{"jsonrpc":"2.0","id":1,"result":false}`

**Prysm sync status:**
```bash
curl http://localhost:3500/eth/v1/node/syncing
```

**Expected when synced:** `{"data":{"is_syncing":false,"el_offline":false}}`
## Firewall Configuration

Configure your firewall to allow necessary ports while maintaining security:

### Basic Firewall Setup

```bash
# Enable firewall
sudo ufw allow 22
sudo ufw allow ssh
sudo ufw enable
```

### Allow P2P Ports (Required for Node Operation)

```bash
# Besu P2P ports (required for peer discovery)
sudo ufw allow 30303/tcp   # Besu P2P TCP
sudo ufw allow 30303/udp   # Besu P2P UDP

# Prysm P2P ports (required for peer discovery)
sudo ufw allow 13000/tcp   # Prysm P2P TCP
sudo ufw allow 12000/udp   # Prysm P2P UDP
```

### Local Access (Aztec Sequencer on Same Server)

```bash
# Allow localhost access to RPC endpoints
sudo ufw allow from 127.0.0.1 to any port 8545 proto tcp  # Besu
sudo ufw allow from 127.0.0.1 to any port 3500 proto tcp  # Prysm
```

### Remote Access (Aztec Sequencer on Different Server)

allow access from specific IPs:

```bash
# Replace <your-aztec-server-ip> with the actual IP of your Aztec server
sudo ufw allow from <your-vps-ip> to any port 8545 proto tcp
sudo ufw allow from <your-vps-ip> to any port 3500 proto tcp
```

### Deny Public Access (Recommended for Production)

```bash
# Explicitly deny public access to RPC endpoints
sudo ufw deny 8545/tcp   # Besu
sudo ufw deny 3500/tcp   # Prysm
```

### Check Firewall Status

```bash
# View current firewall rules
sudo ufw status verbose

# View numbered rules (for easy deletion)
sudo ufw status numbered

# Delete a rule (replace X with rule number)
sudo ufw delete X
```
### Example Firewall Configuration

For a typical setup with Aztec sequencer on the same server:

```bash
# Enable firewall
sudo ufw allow 22
sudo ufw allow ssh
sudo ufw enable

# Allow P2P ports (required)
sudo ufw allow 30303/tcp
sudo ufw allow 30303/udp
sudo ufw allow 13000/tcp
sudo ufw allow 12000/udp

# Allow localhost access
sudo ufw allow from 127.0.0.1 to any port 8545 proto tcp
sudo ufw allow from 127.0.0.1 to any port 3500 proto tcp

# Deny public access to RPC
sudo ufw deny 8545/tcp
sudo ufw deny 3500/tcp

# Check status
sudo ufw status verbose
```
## Connecting Aztec Sequencer

Once both clients are synced, configure your Aztec sequencer to connect to:

```bash
# Execution RPC endpoint
ETHEREUM_RPC_URL=http://localhost:8545

# Consensus RPC endpoint
BEACON_RPC_URL=http://localhost:3500
```
