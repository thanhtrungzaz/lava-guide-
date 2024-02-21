# lava-guide-
Hardware Requirements

Recommended : 4CPU 8RAM 160GB


# Install dependencies for building from source
sudo apt update           
sudo apt install -y curl git jq lz4 build-essential

# Install Go
sudo rm -rf /usr/local/go
curl -L https://go.dev/dl/go1.21.6.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> $HOME/.bash_profile
source .bash_profile


Node Installation

# Clone project repository
cd && rm -rf lava      
git clone https://github.com/lavanet/lava       
cd lava   
git checkout v0.35.0   

# Build binary
export LAVA_BINARY=lavad     
make install

# Set node CLI configuration
lavad config chain-id lava-testnet-2    
lavad config keyring-backend test    
lavad config node tcp://localhost:19957    

# Initialize the node
lavad init "YOUR_MONIKER" --chain-id lava-testnet-2    

# Download genesis and addrbook files
curl -L https://snapshots-testnet.nodejumper.io/lava-testnet/genesis.json > $HOME/.lava/config/genesis.json      
curl -L https://snapshots-testnet.nodejumper.io/lava-testnet/addrbook.json > $HOME/.lava/config/addrbook.json     

# Set seeds
sed -i -e 's|^seeds *=.*|seeds = "3a445bfdbe2d0c8ee82461633aa3af31bc2b4dc0@prod-pnet-seed-node.lavanet.xyz:26656,e593c7a9ca61f5616119d6beb5bd8ef5dd28d62d@prod-pnet-seed-node2.lavanet.xyz:26656,ade4d8bc8cbe014af6ebdf3cb7b1e9ad36f412c0@testnet-seeds.polkachu.com:19956"|' $HOME/.lava/config/config.toml   

# Set minimum gas price
sed -i -e 's|^minimum-gas-prices *=.*|minimum-gas-prices = "0.000000001ulava"|' $HOME/.lava/config/app.toml   

# Set pruning
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "17"|' \
  $HOME/.lava/config/app.toml     

# Change ports
sed -i -e "s%:1317%:19917%; s%:8080%:19980%; s%:9090%:19990%; s%:9091%:19991%; s%:8545%:19945%; s%:8546%:19946%; s%:6065%:19965%" $HOME/.lava/config/app.toml   

sed -i -e "s%:26658%:19958%; s%:26657%:19957%; s%:6060%:19960%; s%:26656%:19956%; s%:26660%:19961%" $HOME/.lava/config/config.toml    

# Download latest chain data snapshot
curl "https://snapshots-testnet.nodejumper.io/lava-testnet/lava-testnet_latest.tar.lz4" | lz4 -dc - | tar -xf - -C "$HOME/.lava"    

# Create a service
sudo tee /etc/systemd/system/lavad.service > /dev/null << EOF
[Unit]   
Description=Lava node service   
After=network-online.target   
[Service]  
User=$USER    
ExecStart=$(which lavad) start   
Restart=on-failure   
RestartSec=10   
LimitNOFILE=65535   
[Install]    
WantedBy=multi-user.target   
EOF       

sudo systemctl daemon-reload     

sudo systemctl enable lavad.service     

# Start the service and check the logs

sudo systemctl start lavad.service    

sudo journalctl -u lavad.service -f --no-hostname -o cat   

Create Validator   

# create wallet

lavad keys add wallet   

## console output:
  name: wallet  
  type: local  
  address: lava@1us3tv59r3wz57ydjafkzgpy0pccae2a2e4k5en   
  pubkey: '{"@type":"/cosmos.crypto.secp256k1.PubKey","key":"Auq9WzVEs5pCoZgr2WctjI7fU+lJCH0I3r6GC1oa0tc0"}'  
  mnemonic: ""   

#!!! SAVE SEED PHRASE (example)   
kite upset hip dirt pet winter thunder slice parent flag sand express suffer chest custom pencil mother bargain remember patient other curve cancel sweet   

#!!! SAVE PRIVATE VALIDATOR KEY   
cat $HOME/.lava/config/priv_validator_key.json   

# wait util the node is synced, should return FALSE  

lavad status 2>&1 | jq .SyncInfo.catching_up  

# faucet some tokens with the command below or ask in discord, if the command doesn't work    

curl -X POST -d '{"address": "YOUR_WALLET_ADDRESS", "coins": ["10000000ulava"]}' https://faucet-api.lavanet.xyz/faucet/    

# verify the balance

lavad q bank balances $(lavad keys show wallet -a)    
 
## console output:
  balances:  
  - amount: "10000000"  
    denom: ulava  

# create validator
lavad tx staking create-validator \    
--amount=9000000ulava \    
--pubkey=$(lavad tendermint show-validator) \    
--moniker="$NODE_MONIKER" \   
--chain-id=lava-testnet-2 \   
--commission-rate=0.1 \   
--commission-max-rate=0.2 \   
--commission-max-change-rate=0.05 \   
--min-self-delegation=1 \   
--fees=10000ulava \   
--from=wallet \   
-y   
  
# make sure you see the validator details

lavad q staking validator $(lavad keys show wallet --bech val -a)


