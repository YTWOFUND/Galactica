# Galactica

### Galactica node Installation Instructions.

[Official documentation](https://galactica.com)

System requirements:</br>
CPU: Quad Core or larger AMD or Intel (amd64) CPU
RAM:32GB RAM
SSD:1TB NVMe Storage
100MBps bidirectional internet connection
OS: Ubuntu 20.04 or 22.04</br>

You can take a weaker server

### Network configurations: </br>
Outbound - allow all traffic </br>
Inbound - open the following ports :</br>
1317 - REST </br>
26657 - TENDERMINT_RP </br>
26656 - Cosmos </br>
Running as a Provider? Add these specific ports: </br>
22221 - provider port </br>
22231 - provider port </br>
22241 - provider port </br>

### Installing the Babylon Node

1. Preparing the server/Required packages installation</br>
```
sudo apt update
sudo apt upgrade -y
sudo apt-get install libclang-dev
sudo apt install -y unzip logrotate git jq sed wget curl coreutils systemd
```
### Go installation.
```
sudo rm -rf /usr/local/go
curl -L https://go.dev/dl/go1.21.6.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> $HOME/.bash_profile
source .bash_profile
go version
```

### Download and build binaries
```
cd $HOME
rm -rf galactica
git clone https://github.com/Galactica-corp/galactica
cd galactica
git checkout v0.1.2
make build
mv $HOME/galactica/build/galacticad $HOME/go/bin
```

# Config and init app
```
galacticad config node tcp://localhost:${GALACTICA_PORT}657
galacticad config keyring-backend os
galacticad config chain-id galactica_9302-1
galacticad init "your moniker" --chain-id galactica_9302-1
```

# Download genesis and addrbook
```
wget -O $HOME/.galactica/config/genesis.json https://testnet-files.itrocket.net/galactica/genesis.json
wget -O $HOME/.galactica/config/addrbook.json https://testnet-files.itrocket.net/galactica/addrbook.json
```

# Set seeds and peers
```
SEEDS="52ccf467673f93561c9d5dd4434def32ef2cd7f3@galactica-testnet-seed.itrocket.net:46656"
PEERS="c9993c738bec6a10cfb8bb30ac4e9ae6f8286a5b@galactica-testnet-peer.itrocket.net:11656,6b846b316d704d78f3f9ca46d86cebc5a22de8ae@161.97.111.249:26656,d572caf3a63d6c730fe0a5c586fd93e70683b727@167.86.127.19:656,e926f2e20568e61646558a2b8fd4a4e46d77903f@141.95.110.124:26656,7d8c640a24a1f15e98d45982bfd02dd0316c46e8@213.136.85.27:26656,f3cd6b6ebf8376e17e630266348672517aca006a@46.4.5.45:27456,9990ab130eac92a2ed1c3d668e9a1c6e811e8f35@148.251.177.108:27456,8949fb771f2859248bf8b315b6f2934107f1cf5a@168.119.241.1:26656,dc4ed6e614725dffc41874e762a1b1ce4cdc3304@161.97.74.20:46656,e38c22e44893e75e388f3bcea2a075124d75ccd3@89.110.89.244:26656,c722e6dc5f762b0ef19be7f8cc8fd67cdf988946@49.12.96.14:26656,3afb7974589e431293a370d10f4dcdb73fa96e9b@157.90.158.222:26656"
sed -i -e "s/^seeds *=.*/seeds = \"$SEEDS\"/; s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" $HOME/.galactica/config/config.toml

sed -i -e "s/^pruning *=.*/pruning = \"custom\"/" $HOME/.galactica/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"100\"/" $HOME/.galactica/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"50\"/" $HOME/.galactica/config/app.toml

sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "10agnet"|g' $HOME/.galactica/config/app.toml
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.galactica/config/config.toml
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.galactica/config/config.toml
```

# Create service file
```
sudo tee /etc/systemd/system/galacticad.service > /dev/null <<EOF
[Unit]
Description=Galactica node
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.galactica
ExecStart=$(which galacticad) start --home $HOME/.galactica --chain-id galactica_9302-1
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable galacticad
```

# Reset and download snapshot
```
galacticad tendermint unsafe-reset-all --home $HOME/.galactica
if curl -s --head curl https://testnet-files.itrocket.net/galactica/snap_galactica.tar.lz4 | head -n 1 | grep "200" > /dev/null; then
  curl https://testnet-files.itrocket.net/galactica/snap_galactica.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.galactica
    else
  echo no have snap
fi
```

# enable and start service
```
sudo systemctl start galacticad
sudo journalctl -u galacticad -f
```

### Becoming a Validator

# Create wallet key new
```
galacticad keys add $WALLET
```

(OPTIONAL) RECOVER EXISTING KEY
```
galacticad keys add $WALLET --recover
```

# check sync status, once your node is fully synced, the output from above will print "false"
```
galacticad keys add $WALLET --recover
```

### We receive tokens in the [site](https://faucet-reticulum.galactica.com/)

# before creating a validator, you need to fund your wallet and check balance
```
galacticad q bank balances $WALLET_ADDRESS 
```
# Create validator
```
galacticad tx staking create-validator \
--amount 1000000agnet \
--from $WALLET \
--commission-rate 0.1 \
--commission-max-rate 0.2 \
--commission-max-change-rate 0.01 \
--min-self-delegation 1 \
--pubkey $(galacticad tendermint show-validator) \
--moniker "$MONIKER" \
--identity "FFB0AA51A2DF5955" \
--details "I love YTWO❤️" \
--chain-id galactica_9302-1 \
--gas 200000 --gas-prices 10agnet \
-y 
```

### Update
```
No update

Current network:galactica_9302-1
Current version:v0.1.2
```

### Useful commands

Check balance
```
galacticad q bank balances $WALLET_ADDRESS 
```

CHECK SERVICE LOGS
```
sudo journalctl -u galacticad -f
```

RESTART SERVICE
```
sudo journalctl -u galacticad -f
```

GET VALIDATOR INFO
```
galacticad status 2>&1 | jq
```

DELEGATE TOKENS TO YOURSELF
```
galacticad tx staking delegate <TO_VALOPER_ADDRESS> 1000000agnet --from $WALLET --chain-id galactica_9302-1 --gas 200000 --gas-prices 10agnet -y 	
```

REMOVE NODE
Please, before proceeding with the next step! All chain data will be lost! Make sure you have backed up your priv_validator_key.json!
```
sudo systemctl stop galacticad && sudo systemctl disable galacticad && sudo rm /etc/systemd/system/galacticad.service && sudo systemctl daemon-reload && rm -rf $HOME/.galacticad && rm -rf galacticad && sudo rm -rf $(which galacticad) 
```
