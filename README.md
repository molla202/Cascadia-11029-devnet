# Update
```
apt update && apt upgrade -y
apt install curl iptables build-essential git wget jq make gcc nano tmux htop nvme-cli pkg-config libssl-dev libleveldb-dev tar clang bsdmainutils ncdu unzip libleveldb-dev -y
```
# Go
```
ver="1.20.3"
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz"
rm "go$ver.linux-amd64.tar.gz"
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile
source $HOME/.bash_profile
go version
```
## setup
```
cd $HOME
curl -L https://github.com/CascadiaFoundation/cascadia/releases/download/v0.1.9/cascadiad -o cascadiad
sudo chmod u+x cascadiad
```

# Prepare binaries for Cosmovisor
```
mkdir -p $HOME/.cascadiad/cosmovisor/genesis/bin
mv cascadiad $HOME/.cascadiad/cosmovisor/genesis/bin/
```
# set vars
Not: write moniker and wallet name
```
echo "export WALLET="wallet"" >> $HOME/.bash_profile
echo "export MONIKER="test"" >> $HOME/.bash_profile
echo "export CASCADIA_CHAIN_ID="cascadia_11029-1"" >> $HOME/.bash_profile
echo "export CASCADIA_PORT="40"" >> $HOME/.bash_profile
source $HOME/.bash_profile
```
# Create application symlinks
```
sudo ln -s $HOME/.cascadiad/cosmovisor/genesis $HOME/.cascadiad/cosmovisor/current -f
sudo ln -s $HOME/.cascadiad/cosmovisor/current/bin/cascadiad /usr/local/bin/cascadiad -f
```

# Download and install Cosmovisor (Install Cosmovisor and create a service)
```
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.5.0
```
# Create service
```
sudo tee /etc/systemd/system/cascadiad.service > /dev/null << EOF
[Unit]
Description=cascadia node service
After=network-online.target

[Service]
User=$USER
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.cascadiad"
Environment="DAEMON_NAME=cascadiad"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:$HOME/.cascadiad/cosmovisor/current/bin"

[Install]
WantedBy=multi-user.target
EOF
```

# config and init app
```
cascadiad config node tcp://localhost:${CASCADIA_PORT}657
cascadiad config keyring-backend os
cascadiad config chain-id cascadia_11029-1
cascadiad init $MONIKER --chain-id cascadia_11029-1
```
# download genesis and addrbook
```
wget -O $HOME/.cascadiad/config/genesis.json https://testnet-files.itrocket.net/cascadia/genesis.json
wget -O $HOME/.cascadiad/config/addrbook.json https://testnet-files.itrocket.net/cascadia/addrbook.json
```
# set seeds and peers
```
SEEDS=""
PEERS="d1ed80e232fc2f3742637daacab454e345bbe475@54.204.246.120:26656,0c96a6c328eb58d1467afff4130ab446c294108c@34.239.67.55:26656"
sed -i 's|^seeds *=.*|seeds = "'$SEEDS'"|; s|^persistent_peers *=.*|persistent_peers = "'$PEERS'"|' $HOME/.cascadiad/config/config.toml
```
# set custom ports in app.toml
```
sed -i.bak -e "s%:1317%:${CASCADIA_PORT}317%g;
s%:8080%:${CASCADIA_PORT}080%g;
s%:9090%:${CASCADIA_PORT}090%g;
s%:9091%:${CASCADIA_PORT}091%g;
s%:8545%:${CASCADIA_PORT}545%g;
s%:8546%:${CASCADIA_PORT}546%g;
s%:6065%:${CASCADIA_PORT}065%g" $HOME/.cascadiad/config/app.toml
```
# set custom ports in config.toml file
```
sed -i.bak -e "s%:26658%:${CASCADIA_PORT}658%g;
s%:26657%:${CASCADIA_PORT}657%g;
s%:6060%:${CASCADIA_PORT}060%g;
s%:26656%:${CASCADIA_PORT}656%g;
s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):${CASCADIA_PORT}656\"%;
s%:26660%:${CASCADIA_PORT}660%g" $HOME/.cascadiad/config/config.toml
```
# config pruning
```
sed -i 's|^pruning *=.*|pruning = "custom"|g' $HOME/.cascadiad/config/app.toml
sed -i 's|^pruning-keep-recent  *=.*|pruning-keep-recent = "100"|g' $HOME/.cascadiad/config/app.toml
sed -i 's|^pruning-interval *=.*|pruning-interval = "10"|g' $HOME/.cascadiad/config/app.toml
sed -i 's|^snapshot-interval *=.*|snapshot-interval = 0|g' $HOME/.cascadiad/config/app.toml
```
# set minimum gas price, enable prometheus and disable indexing
```
sed -i 's|^minimum-gas-prices *=.*|minimum-gas-prices = "0.025aCC"|g' $HOME/.cascadiad/config/app.toml
sed -i 's|^prometheus *=.*|prometheus = true|' $HOME/.cascadiad/config/config.toml
```
# reset and download snapshot
```
.
```
# enable and start service
```
sudo systemctl daemon-reload
sudo systemctl enable cascadiad
sudo systemctl restart cascadiad && sudo journalctl -u cascadiad -fo cat
```
