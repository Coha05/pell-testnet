# Install Guide
**Recommended Hardware: 4 Cores, 8GB RAM, 200GB of storage (NVME)**
### 1. Install required packages
```
sudo apt update && \
sudo apt install curl git jq build-essential gcc unzip wget lz4 -y
```
```
WASMVM_VERSION=v2.1.2
export LD_LIBRARY_PATH=~/.pellcored/lib
mkdir -p $LD_LIBRARY_PATH
wget "https://github.com/CosmWasm/wasmvm/releases/download/$WASMVM_VERSION/libwasmvm.$(uname -m).so" -O "$LD_LIBRARY_PATH/libwasmvm.$(uname -m).so"
export LD_LIBRARY_PATH=$HOME/.pellcored/lib:$LD_LIBRARY_PATH
source ~/.bashrc
```
### 2. Install Go
```
cd $HOME && \
ver="1.23.0" && \
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz" && \
sudo rm -rf /usr/local/go && \
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz" && \
rm "go$ver.linux-amd64.tar.gz" && \
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile && \
source $HOME/.bash_profile && \
go version
```
### 3. Install `Pell` binary
```
cd $HOME
wget -O $HOME/pellcored https://github.com/0xPellNetwork/network-config/releases/download/v1.0.20-ignite/pellcored-v1.0.20-linux-amd64
chmod +x pellcored
cp pellcored $HOME/go/bin/pellcored
source $HOME/.bash_profile
pellcored version
```
### 4. Set up variables
```
# Customize if you need
echo 'export MONIKER="My_Node"' >> ~/.bash_profile
echo 'export CHAIN_ID="ignite_186-1"' >> ~/.bash_profile
echo 'export WALLET_NAME="wallet"' >> ~/.bash_profile
echo 'export RPC_PORT="26657"' >> ~/.bash_profile
source $HOME/.bash_profile
```
### 5. Initialize the node
```
pellcored init $MONIKER --chain-id $PELL_CHAIN_ID
```
### 6. Download genesis and addrbook
```
wget -O $HOME/.pellcored/config/genesis.json https://snapshot.spidernode.net/pell-tessnet-genesis.json
wget -O $HOME/.pellcored/config/addrbook.json  https://snapshot.spidernode.net/pell-testnet-addrbook.json
```
### 7. Config pruning
```
sed -i -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.pellcored/config/app.toml 
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.pellcored/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"10\"/" $HOME/.pellcored/config/app.toml
```
### 8. Set minimum gas price and disable indexing
```
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "0apell"|g' $HOME/.pellcored/config/app.toml
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.pellcored/config/config.toml
```
### 9. Create service file and run the service
```
sudo nano /etc/systemd/system/pelld.service
```
```
[Unit]
Description=Pell node
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.pellcored
ExecStart=$(which pellcored) start --home $HOME/.pellcored
Environment=LD_LIBRARY_PATH=$HOME/.pellcored/lib/
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
```
### 10. Download snapshot
```
wget https://snapshot.spidernode.net/pell_snapshot.lz4 | lz4 -dc - | tar -xf - -C $HOME/.pellcored
```
### 11. Start the service
```
sudo systemctl daemon-reload
sudo systemctl enable pelld
sudo systemctl restart pelld
```
```
sudo journalctl -u pellcored -f
```

### 12. Create or recover wallet
```
pellcored keys add $WALLET
```
```
pellcored keys add $WALLET --recover
```
### 13. Register validator
```
cat > ./validator.json << EOF
{
    "pubkey": $(pellcored tendermint show-validator),
    "amount": "1000000000000000000apell",
    "moniker": "<your_node_name>",
    "identity": "optional identity signature (ex. UPort or Keybase)",
    "website": "validator's (optional) website",
    "security": "validator's (optional) security contact email",
    "details": "validator's (optional) details",
    "commission-rate": "0.1",
    "commission-max-rate": "0.2",
    "commission-max-change-rate": "0.01",
    "min-self-delegation": "1"
}
EOF


pellcored tx staking create-validator ./validator.json \
--chain-id=ignite_186-1 \
--fees=0.000001pell \
--gas=1000000 \
--from=my_validator_key
```
