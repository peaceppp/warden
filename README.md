# warden tutorial
chaind id:buenavista-1

# minimum requirements
- 4 or more physical CPU cores
- At least 500GB of SSD disk storage
- At least 16GB of memory (RAM)
- At least 120mbps network bandwidth
  
# official links
### [official document](https://docs.wardenprotocol.org/developers/installation)

# explorer
### [explorer](https://explorer.kjnodes.com/warden-testnet)

# install depencies
```bash
sudo apt -q update
sudo apt -qy install curl git jq lz4 build-essential
sudo apt -qy upgrade
```
# install go
```bash
sudo rm -rf /usr/local/go
curl -Ls https://go.dev/dl/go1.21.10.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
eval $(echo 'export PATH=$PATH:/usr/local/go/bin' | sudo tee /etc/profile.d/golang.sh)
eval $(echo 'export PATH=$PATH:$HOME/go/bin' | tee -a $HOME/.profile)
```
# download and build binaries
```bash
# clone project repository
cd $HOME
rm -rf wardenprotocol
git clone https://github.com/warden-protocol/wardenprotocol.git
cd wardenprotocol
git checkout v0.3.0

# Build binaries
make build

# Prepare binaries for Cosmovisor
mkdir -p $HOME/.warden/cosmovisor/genesis/bin
mv build/wardend $HOME/.warden/cosmovisor/genesis/bin/
rm -rf build

# Create application symlinks
sudo ln -s $HOME/.warden/cosmovisor/genesis $HOME/.warden/cosmovisor/current -f
sudo ln -s $HOME/.warden/cosmovisor/current/bin/wardend /usr/local/bin/wardend -f
```
# Install Cosmovisor and create a service
```bash
# Download and install Cosmovisor
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.5.0

# Create service
sudo tee /etc/systemd/system/warden.service > /dev/null << EOF
[Unit]
Description=warden node service
After=network-online.target

[Service]
User=$USER
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.warden"
Environment="DAEMON_NAME=wardend"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:$HOME/.warden/cosmovisor/current/bin"

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable warden.service
```
# Initialize the node
```
# Set node configuration
wardend config chain-id buenavista-1
wardend config keyring-backend test
wardend config node tcp://localhost:17857

# Initialize the node
wardend init $MONIKER --chain-id buenavista-1

# Download genesis and addrbook
curl -Ls https://snapshots.kjnodes.com/warden-testnet/genesis.json > $HOME/.warden/config/genesis.json
curl -Ls https://snapshots.kjnodes.com/warden-testnet/addrbook.json > $HOME/.warden/config/addrbook.json

# Add seeds
sed -i -e "s|^seeds *=.*|seeds = \"3f472746f46493309650e5a033076689996c8881@warden-testnet.rpc.kjnodes.com:17859\"|" $HOME/.warden/config/config.toml

# Set minimum gas price
sed -i -e "s|^minimum-gas-prices *=.*|minimum-gas-prices = \"0.0025uward\"|" $HOME/.warden/config/app.toml

# Set pruning
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-keep-every *=.*|pruning-keep-every = "0"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "19"|' \
  $HOME/.warden/config/app.toml

# Set custom ports
sed -i -e "s%^proxy_app = \"tcp://127.0.0.1:26658\"%proxy_app = \"tcp://127.0.0.1:17858\"%; s%^laddr = \"tcp://127.0.0.1:26657\"%laddr = \"tcp://127.0.0.1:17857\"%; s%^pprof_laddr = \"localhost:6060\"%pprof_laddr = \"localhost:17860\"%; s%^laddr = \"tcp://0.0.0.0:26656\"%laddr = \"tcp://0.0.0.0:17856\"%; s%^prometheus_listen_addr = \":26660\"%prometheus_listen_addr = \":17866\"%" $HOME/.warden/config/config.toml
sed -i -e "s%^address = \"tcp://0.0.0.0:1317\"%address = \"tcp://0.0.0.0:17817\"%; s%^address = \":8080\"%address = \":17880\"%; s%^address = \"0.0.0.0:9090\"%address = \"0.0.0.0:17890\"%; s%^address = \"0.0.0.0:9091\"%address = \"0.0.0.0:17891\"%; s%:8545%:17845%; s%:8546%:17846%; s%:6065%:17865%" $HOME/.warden/config/app.toml
```
# snapshot using kj nodes or nodes guru
```
curl -L https://snapshots.kjnodes.com/warden-testnet/snapshot_latest.tar.lz4 | tar -Ilz4 -xf - -C $HOME/.warden
[[ -f $HOME/.warden/data/upgrade-info.json ]] && cp $HOME/.warden/data/upgrade-info.json $HOME/.warden/cosmovisor/genesis/upgrade-info.json
```
# Start service and check the logs
```
sudo systemctl start warden.service && sudo journalctl -u warden.service -f --no-hostname -o cat
```
# add wallet
```
wardend keys add wallet
```
# recover wallet
```
wardend keys add wallet --recover
```
# query ballance
```
wardend q bank balances $(wardend keys show wallet -a)
```
# get sync info
```
wardend status 2>&1 | jq .SyncInfo
```
# validator management
please check sync before create validator

# create validator
```
wardend tx staking create-validator \
--amount 1000000uward \
--pubkey $(wardend tendermint show-validator) \
--moniker "YOUR_MONIKER_NAME" \
--identity "YOUR_KEYBASE_ID" \
--details "YOUR_DETAILS" \
--website "YOUR_WEBSITE_URL" \
--chain-id buenavista-1 \
--commission-rate 0.05 \
--commission-max-rate 0.20 \
--commission-max-change-rate 0.05 \
--min-self-delegation 1 \
--from wallet \
--gas-adjustment 1.4 \
--gas auto \
--gas-prices 0.0025uward \
-y
```
# edit validator
```
wardend tx staking edit-validator \
--new-moniker "YOUR_MONIKER_NAME" \
--identity "YOUR_KEYBASE_ID" \
--details "YOUR_DETAILS" \
--website "YOUR_WEBSITE_URL" \
--chain-id buenavista-1 \
--commission-rate 0.05 \
--from wallet \
--gas-adjustment 1.4 \
--gas auto \
--gas-prices 0.0025uward \
-y
```
# unjail validator
```
wardend tx slashing unjail --from wallet --chain-id buenavista-1 --gas-adjustment 1.4 --gas auto --gas-prices 0.0025uward -y
```

# Token management


# WITHDRAW REWARDS FROM ALL VALIDATORS
```
wardend tx distribution withdraw-all-rewards --from wallet --chain-id buenavista-1 --gas-adjustment 1.4 --gas auto --gas-prices 0.0025uward -y
```
# WITHDRAW COMMISSION AND REWARDS FROM YOUR VALIDATOR
```
wardend tx distribution withdraw-rewards $(wardend keys show wallet --bech val -a) --commission --from wallet --chain-id buenavista-1 --gas-adjustment 1.4 --gas auto --gas-prices 0.0025uward -y
```
# DELEGATE TOKENS TO YOURSELF
```
wardend tx staking delegate $(wardend keys show wallet --bech val -a) 1000000uward --from wallet --chain-id buenavista-1 --gas-adjustment 1.4 --gas auto --gas-prices 0.0025uward -y
```
# DELEGATE TOKENS TO VALIDATOR
```
wardend tx staking delegate <TO_VALOPER_ADDRESS> 1000000uward --from wallet --chain-id buenavista-1 --gas-adjustment 1.4 --gas auto --gas-prices 0.0025uward -y
```
# REDELEGATE TOKENS TO ANOTHER VALIDATOR
```
wardend tx staking redelegate $(wardend keys show wallet --bech val -a) <TO_VALOPER_ADDRESS> 1000000uward --from wallet --chain-id buenavista-1 --gas-adjustment 1.4 --gas auto --gas-prices 0.0025uward -y
```
# UNBOND TOKENS FROM YOUR VALIDATOR
```
wardend tx staking unbond $(wardend keys show wallet --bech val -a) 1000000uward --from wallet --chain-id buenavista-1 --gas-adjustment 1.4 --gas auto --gas-prices 0.0025uward -y
```
# SEND TOKENS TO THE WALLET
```
wardend tx bank send wallet <TO_WALLET_ADDRESS> 1000000uward --from wallet --chain-id buenavista-1 --gas-adjustment 1.4 --gas auto --gas-prices 0.0025uward -y
```

# Governance


# LIST ALL PROPOSALS
```
wardend query gov proposals
```
# VIEW PROPOSAL BY ID
```
wardend query gov proposal 1
```
# VOTE ‘YES’
```
wardend tx gov vote 1 yes --from wallet --chain-id buenavista-1 --gas-adjustment 1.4 --gas auto --gas-prices 0.0025uward -y
```
# VOTE ‘NO’
```
wardend tx gov vote 1 no --from wallet --chain-id buenavista-1 --gas-adjustment 1.4 --gas auto --gas-prices 0.0025uward -y
```
# VOTE ‘ABSTAIN’
```
wardend tx gov vote 1 abstain --from wallet --chain-id buenavista-1 --gas-adjustment 1.4 --gas auto --gas-prices 0.0025uward -y
```
# VOTE ‘NOWITHVETO’
```
wardend tx gov vote 1 NoWithVeto --from wallet --chain-id buenavista-1 --gas-adjustment 1.4 --gas auto --gas-prices 0.0025uward -y
```
# ⚡️ Utility
```
UPDATE PORTS
CUSTOM_PORT=110
sed -i -e "s%^proxy_app = \"tcp://127.0.0.1:26658\"%proxy_app = \"tcp://127.0.0.1:${CUSTOM_PORT}58\"%; s%^laddr = \"tcp://127.0.0.1:26657\"%laddr = \"tcp://127.0.0.1:${CUSTOM_PORT}57\"%; s%^pprof_laddr = \"localhost:6060\"%pprof_laddr = \"localhost:${CUSTOM_PORT}60\"%; s%^laddr = \"tcp://0.0.0.0:26656\"%laddr = \"tcp://0.0.0.0:${CUSTOM_PORT}56\"%; s%^prometheus_listen_addr = \":26660\"%prometheus_listen_addr = \":${CUSTOM_PORT}66\"%" $HOME/.warden/config/config.toml
sed -i -e "s%^address = \"tcp://0.0.0.0:1317\"%address = \"tcp://0.0.0.0:${CUSTOM_PORT}17\"%; s%^address = \":8080\"%address = \":${CUSTOM_PORT}80\"%; s%^address = \"0.0.0.0:9090\"%address = \"0.0.0.0:${CUSTOM_PORT}90\"%; s%^address = \"0.0.0.0:9091\"%address = \"0.0.0.0:${CUSTOM_PORT}91\"%" $HOME/.warden/config/app.toml
```
UPDATE INDEXER
# Disable indexer
```
sed -i -e 's|^indexer *=.*|indexer = "null"|' $HOME/.warden/config/config.toml
```
# Enable indexer
```
sed -i -e 's|^indexer *=.*|indexer = "kv"|' $HOME/.warden/config/config.toml
```
# UPDATE PRUNING
```
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-keep-every *=.*|pruning-keep-every = "0"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "19"|' \
  $HOME/.warden/config/app.toml
```


# 🚨 Maintenance
# GET VALIDATOR INFO
```
wardend status 2>&1 | jq .ValidatorInfo
```
# GET SYNC INFO
```
wardend status 2>&1 | jq .SyncInfo
```
# GET NODE PEER
```
echo $(wardend tendermint show-node-id)'@'$(curl -s ifconfig.me)':'$(cat $HOME/.warden/config/config.toml | sed -n '/Address to listen for incoming connection/{n;p;}' | sed 's/.*://; s/".*//')
```
# CHECK IF VALIDATOR KEY IS CORRECT
```
[[ $(wardend q staking validator $(wardend keys show wallet --bech val -a) -oj | jq -r .consensus_pubkey.key) = $(wardend status | jq -r .ValidatorInfo.PubKey.value) ]] && echo -e "\n\e[1m\e[32mTrue\e[0m\n" || echo -e "\n\e[1m\e[31mFalse\e[0m\n"
```
# GET LIVE PEERS
curl -sS http://localhost:17857/net_info | jq -r '.result.peers[] | "\(.node_info.id)@\(.remote_ip):\(.node_info.listen_addr)"' | awk -F ':' '{print $1":"$(NF)}'
```
# SET MINIMUM GAS PRICE
```
sed -i -e "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"0.0025uward\"/" $HOME/.warden/config/app.toml
```
# ENABLE PROMETHEUS
```
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.warden/config/config.toml
```
# RESET CHAIN DATA
```
wardend tendermint unsafe-reset-all --keep-addr-book --home $HOME/.warden --keep-addr-book
```
# REMOVE NODE
Please, before proceeding with the next step! All chain data will be lost! Make sure you have backed up your priv_validator_key.json!

```cd $HOME
sudo systemctl stop warden.service
sudo systemctl disable warden.service
sudo rm /etc/systemd/system/warden.service
sudo systemctl daemon-reload
rm -f $(which wardend)
rm -rf $HOME/.warden
rm -rf $HOME/wardenprotocol
```

# ⚙️ Service Management
# RELOAD SERVICE CONFIGURATION
```
sudo systemctl daemon-reload
```
# ENABLE SERVICE
```
sudo systemctl enable warden.service
```
# DISABLE SERVICE
```
sudo systemctl disable warden.service
```
# START SERVICE
```
sudo systemctl start warden.service
```
# STOP SERVICE
```
sudo systemctl stop warden.service
```
# RESTART SERVICE
```
sudo systemctl restart warden.service
```
# CHECK SERVICE STATUS
```
sudo systemctl status warden.service
```
CHECK SERVICE LOGS
```
sudo journalctl -u warden.service -f --no-hostname -o cat
```
