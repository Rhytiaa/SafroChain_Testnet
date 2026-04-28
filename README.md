# Safrochain Testnet Installation Guide (`safro-testnet-1`)


## 📋 Hardware Requirements

| Component | Minimum | Recommended |
| :--- | :--- | :--- |
| **CPU** | 4 Cores | 8 Cores |
| **RAM** | 8 GB | 16 GB |
| **SSD** | 200 GB NVMe | 500 GB NVMe |
| **OS** | Ubuntu 22.04 | Ubuntu 22.04 |

## 🛠 Installation Steps

### 1. System Update & Dependencies
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip snapd -y
```

### 2. Install Go
```bash
cd $HOME
VER="1.23.9"
wget "https://golang.org/dl/go$VER.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$VER.linux-amd64.tar.gz"
rm "go$VER.linux-amd64.tar.gz"
[ ! -f ~/.bash_profile ] && touch ~/.bash_profile
echo "export PATH=$PATH:/usr/local/go/bin:~/go/bin" >> ~/.bash_profile
source $HOME/.bash_profile
```

### 3. Configure Environment Variables
Replace `<YOUR_MONIKER>` with your preferred node name.
```bash
echo "export MONIKER="<YOUR_MONIKER>"" >> $HOME/.bash_profile
echo "export WALLET="wallet"" >> $HOME/.bash_profile
echo "export SAFRO_PORT="53"" >> $HOME/.bash_profile
echo "export SAFRO_CHAIN_ID="safro-testnet-1"" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

### 4. Build Binary & Library Setup
Safrochain requires specific `libwasmvm` libraries to be present.

```bash
# Clone and build
cd $HOME
rm -rf safrochain-node
git clone https://github.com/Safrochain-Org/safrochain-node.git
cd safrochain-node
git checkout v0.1.0
make install

# Setup libwasmvm
mkdir -p $HOME/safrochain-node/lib
cd $HOME/safrochain-node/lib
wget https://github.com/CosmWasm/wasmvm/releases/download/v2.0.1/libwasmvm.x86_64.so -O libwasmvm.so
chmod +x libwasmvm.so
```

### 5. Install Cosmovisor
```bash
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@latest

# Prepare directories
mkdir -p $HOME/.safrochain/cosmovisor/genesis/bin
mkdir -p $HOME/.safrochain/cosmovisor/upgrades
cp $HOME/go/bin/safrochaind $HOME/.safrochain/cosmovisor/genesis/bin/
```

### 6. Initialize Node
```bash
# Initialize
safrochaind init $MONIKER --chain-id $SAFRO_CHAIN_ID

# Download Genesis and Addrbook
wget -O $HOME/.safrochain/config/genesis.json https://vault2.astrostake.xyz/testnet/safrochain/genesis.json
wget -O $HOME/.safrochain/config/addrbook.json https://vault2.astrostake.xyz/testnet/safrochain/addrbook.json

# Configuration (Seeds & Gas)
SEEDS="2242a526e7841e7e8a551aabc4614e6cd612e7fb@88.99.211.113:26656"
sed -i -e "s/^seeds *=.*/seeds = \"$SEEDS\"/" $HOME/.safrochain/config/config.toml
sed -i -e "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"0.001usaf\"/" $HOME/.safrochain/config/app.toml
```

### 7. Custom Port Configuration
```bash
sed -i.bak -e "s%:1317%:${SAFRO_PORT}317%g; s%:8080%:${SAFRO_PORT}080%g; s%:9090%:${SAFRO_PORT}090%g; s%:9091%:${SAFRO_PORT}091%g; s%:26658%:${SAFRO_PORT}658%g; s%:26657%:${SAFRO_PORT}657%g; s%:6060%:${SAFRO_PORT}060%g; s%:26656%:${SAFRO_PORT}656%g; s%:26660%:${SAFRO_PORT}660%g" $HOME/.safrochain/config/config.toml
```

### 8. Create Systemd Service
```bash
sudo tee /etc/systemd/system/safrochaind.service > /dev/null <<EOF
[Unit]
Description=Safrochain Node (Cosmovisor)
After=network-online.target

[Service]
User=$USER
WorkingDirectory=$HOME/safrochain-node
ExecStart=$(which cosmovisor) run start --home $HOME/.safrochain
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
Environment="DAEMON_NAME=safrochaind"
Environment="DAEMON_HOME=$HOME/.safrochain"
Environment="LD_LIBRARY_PATH=$HOME/safrochain-node/lib"
Environment="DAEMON_RESTART_AFTER_UPGRADE=true"
Environment="UNSAFE_SKIP_BACKUP=true"

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable safrochaind
```

### 9. Snapshot & Start
```bash
# Reset data
safrochaind tendermint unsafe-reset-all --home $HOME/.safrochain --keep-addr-book

# Download Snapshot
SNAP_NAME=$(curl -s https://ss-t.safrochain.nodestake.org/ | egrep -o ">20.*\.tar.lz4" | tr -d ">")
curl -o - -L https://ss-t.safrochain.nodestake.org/${SNAP_NAME} | lz4 -c -d - | tar -x -C $HOME/.safrochain

# Start
sudo systemctl restart safrochaind && sudo journalctl -u safrochaind -fo cat
```
#####Thanks NodeStake
---

## 🔑 Validator Setup

### Create Wallet
```bash
safrochaind keys add $WALLET
```

### Create Validator
Create a `validator.json` file and run the following:
```bash
safrochaind tx staking create-validator /path/to/validator.json \
  --from $WALLET \
  --chain-id $SAFRO_CHAIN_ID \
  --gas auto --gas-adjustment 1.5 --fees 300usaf \
  -y
```

---

## 🗑 Uninstall
```bash
sudo systemctl stop safrochaind
sudo systemctl disable safrochaind
sudo rm /etc/systemd/system/safrochaind.service
rm -rf $HOME/.safrochain $HOME/safrochain-node
sed -i '/SAFRO_/d' $HOME/.bash_profile
```
