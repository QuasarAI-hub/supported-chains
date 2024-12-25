### System requirements

- OS: Ubuntu server 22 LTS
- CPU: 4 cores
- RAM: 4 Gb
- Storage: 1000Gb Nvme

### Update system and install build tools

Ensure your system is up to date and has all the necessary tools for the installation:

```bash
sudo apt update && sudo apt upgrade -y && sudo apt install curl tar wget clang pkg-config libssl-dev jq build-essential bsdmainutils git make ncdu gcc chrony liblz4-tool -y
```

### Install Go

Replace `VERSION` with the desired Go version

```bash
VERSION="1.23.0"
cd $HOME
wget "https://golang.org/dl/go$VERSION.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$VERSION.linux-amd64.tar.gz"
rm "go$VERSION.linux-amd64.tar.gz"
echo "export PATH=\$PATH:/usr/local/go/bin:\$HOME/go/bin" >> ~/.bash_profile
source ~/.bash_profile
```

### Install node

```bash
cd $HOME
mkdir -p $HOME/src
cd src
git clone https://github.com/cosmos/gaia
cd gaia
git checkout v21.0.0
make install
```

### Initialize Node

Replace <node_name>

```bash
~/go/bin/gaiad init <node_name> --chain-id="cosmoshub-4"
```

### Download genesis.json

```bash
cd $HOME/.gaia/config/
rm -f genesis.json
wget https://github.com/cosmos/mainnet/raw/master/genesis/genesis.cosmoshub-4.json.gz
gzip -d genesis.cosmoshub-4.json.gz
```

### Create Service

#### Cosmovisor:

If you don’t have cosmovisor

```bash
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@latest
```

You can find cosmovisor binary in `~/go/bin/` folder. After that you should create

```bash
mkdir -p $HOME/.gaia/cosmovisor/genesis/bin && mkdir -p $HOME/.gaia/cosmovisor/upgrades
cp $HOME/go/bin/gaiad $HOME/.gaia/cosmovisor/genesis/bin/gaiad 
```

Set up service:

```bash
SERVICE_FILE="/etc/systemd/system/gaiad.service"
DAEMON_HOME="$HOME/.gaia"

sudo bash -c "cat <<EOL > $SERVICE_FILE
[Unit]
Description=gaiad Daemon cosmovisor
After=network-online.target

[Service]
User=$USER
ExecStart=/home/$USER/go/bin/cosmovisor start
Restart=always
RestartSec=3
LimitNOFILE=4096
Environment=\"DAEMON_NAME=gaiad\"
Environment=\"DAEMON_HOME=$DAEMON_HOME\"
Environment=\"DAEMON_ALLOW_DOWNLOAD_BINARIES=false\"
Environment=\"DAEMON_RESTART_AFTER_UPGRADE=true\"
Environment=\"DAEMON_LOG_BUFFER_SIZE=512\"

[Install]
WantedBy=multi-user.target
EOL"

sudo chmod 644 $SERVICE_FILE
sudo systemctl daemon-reload
```

#### Simple service file:

Set up service:

```bash
SERVICE_FILE="/etc/systemd/system/gaiad.service"
USER=$(whoami)

sudo cat <<EOL > $SERVICE_FILE
[Unit]
Description=gaiad Daemon
After=network-online.target

[Service]
User=$USER
ExecStart=$HOME/go/bin/gaiad start
Restart=always
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOL

sudo chmod 644 $SERVICE_FILE
sudo systemctl daemon-reload
```

### Add peers

```bash
PEERS=d567c93fa5b646c8cca8ba0a2d7499bca6aeba52@78.46.79.242:26656,69fddfdb0df303964b0115bfd4b969ef49dcade6@3.217.133.209:26656,f6e3d89a3af4d3bc9bc56ebdb6ce7e96edb2cebe@65.109.127.254:26656,ebc272824924ea1a27ea3183dd0b9ba713494f83@195.3.221.175:26646,c7b5e2e7d84930128e7e5f594b299b20c8669463@51.210.155.88:23050
sed -i.bak -e "s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" $HOME/.gaia/config/config.toml
```

### Sync node:

```bash
SNAP_RPC="https://cosmoshub.rpc.quasarstaking.ai:443"

LATEST_HEIGHT=$(curl -s $SNAP_RPC/block | jq -r .result.block.header.height); \
BLOCK_HEIGHT=$((LATEST_HEIGHT - 2000)); \
TRUST_HASH=$(curl -s "$SNAP_RPC/block?height=$BLOCK_HEIGHT" | jq -r .result.block_id.hash)

sed -i.bak -E "s|^(enable[[:space:]]+=[[:space:]]+)._$|\1true| ; \
s|^(rpc_servers[[:space:]]+=[[:space:]]+)._$|\1\"$SNAP_RPC,$SNAP_RPC\"| ; \
s|^(trust_height[[:space:]]+=[[:space:]]+).*$|\1$BLOCK_HEIGHT| ; \
s|^(trust_hash[[:space:]]+=[[:space:]]+).*$|\1\"$TRUST_HASH\"|" $HOME/.gaia/config/config.toml
```

Check node status

```bash
curl -s http://localhost:26657/status | jq '.result.sync_info.catching_up'
```

### Start service

```bash
sudo systemctl enable gaiad.service && sudo systemctl start gaiad.service && journalctl -u gaiad.service -f
```

## If need upgrade

```bash
cd $HOME/src/gaia
git fetch origin && git checkout v22.0.0
make build
cd build
```

Depends of type to running a node (simple running or with cosmovisor)

Cosmovisor:

```bash
mkdir -p $HOME/.gaia/cosmovisor/upgrades/v22.0.0/bin/
mv gaiad $HOME/.gaia/cosmovisor/upgrades/v22.0.0/bin/
```

Simple running:

```bash
rm -f $HOME/go/bin/gaiad
mv gaiad $HOME/go/bin/
```

## Creating validator

Replace <wallet>

Creating a validator on the cosmoshub-4 network

```bash
gaiad tx staking create-validator \
  --amount=1000000uatom \
  --pubkey=$(gaiad tendermint show-validator) \
  --moniker="<validator-name>" \
  --identity="<identity>" \
  --website="<website>" \
  --details="<details>" \
  --commission-rate="0.05" \
  --commission-max-rate="0.2" \
  --commission-max-change-rate="0.01" \
  --min-self-delegation="1" \
  --from=<wallet-name> \
  --fees=20000uatom \
  --gas=auto \
  --gas-adjustment=1.5 \
  --chain-id="cosmoshub-4"

```
