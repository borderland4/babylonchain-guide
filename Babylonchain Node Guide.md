### Minimum hardware requirements:
- 4 CPU Cores
- 4 GB RAM
- 50 GB SSD


## Install Babylon

### Install dependencies:
```
sudo apt update
sudo apt install -y curl git jq lz4 build-essential
```

### Install Go
```
sudo rm -rf /usr/local/go
curl -L https://go.dev/dl/go1.21.6.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> $HOME/.bash_profile
source .bash_profile
```

### Node Installation

**Clone project repository**
```
cd && rm -rf babylon
git clone https://github.com/babylonchain/babylon
cd babylon
git checkout v0.7.2
```

**Build binary**
```
make install
```

**Set node CLI configuration**
```
babylond config chain-id bbn-test-2
babylond config keyring-backend test
babylond config node tcp://localhost:20657
```

**Initialize the node**
> [!IMPORTANT]  
> Replace `BabylonChainNodeName` with your own

```
babylond init "BabylonChainNodeName" --chain-id bbn-test-2
```

**Download genesis and addrbook files**
```
curl -L https://snapshots-testnet.borderland.one/babylon-testnet/genesis.json > $HOME/.babylond/config/genesis.json
curl -L https://snapshots-testnet.borderland.one/babylon-testnet/addrbook.json > $HOME/.babylond/config/addrbook.json
```

**Set seeds**
```
sed -i -e 's|^seeds *=.*|seeds = "03ce5e1b5be3c9a81517d415f65378943996c864@18.207.168.204:26656,a5fabac19c732bf7d814cf22e7ffc23113dc9606@34.238.169.221:26656,ade4d8bc8cbe014af6ebdf3cb7b1e9ad36f412c0@testnet-seeds.polkachu.com:20656"|' $HOME/.babylond/config/config.toml
```

**Set minimum gas price**
```
sed -i -e 's|^minimum-gas-prices *=.*|minimum-gas-prices = "0.001ubbn"|' $HOME/.babylond/config/app.toml
```

**Set pruning**
```
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "17"|' \
  $HOME/.babylond/config/app.toml
```

**Set additional configs**
```
sed -i 's|^network *=.*|network = "mainnet"|g' $HOME/.babylond/config/app.toml
```

**Change ports**
```
sed -i -e "s%:1317%:20617%; s%:8080%:20680%; s%:9090%:20690%; s%:9091%:20691%; s%:8545%:20645%; s%:8546%:20646%; s%:6065%:20665%" $HOME/.babylond/config/app.toml
sed -i -e "s%:26658%:20658%; s%:26657%:20657%; s%:6060%:20660%; s%:26656%:20656%; s%:26660%:20661%" $HOME/.babylond/config/config.toml
```

**Download latest chain data snapshot**
```
curl "https://snapshots-testnet.borderland.one/babylon-testnet/babylon-testnet_latest.tar.lz4" | lz4 -dc - | tar -xf - -C "$HOME/.babylond"
```

**Create a service**
```
sudo tee /etc/systemd/system/babylond.service > /dev/null << EOF
[Unit]
Description=Babylon node service
After=network-online.target
[Service]
User=$USER
ExecStart=$(which babylond) start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
```
```
sudo systemctl daemon-reload
sudo systemctl enable babylond.service
```

**Start the service and check the logs**
```
sudo systemctl start babylond.service
sudo journalctl -u babylond.service -f --no-hostname -o cat
```

- If you encounter errors, don't worry; the node is syncing. Just give it some time.
- Press `Ctrl+C` to exit the log (this won’t stop the node).

You can stop following the guide here if your goal is just to create a node.

### Creating a Validator

**Create wallet**
```
babylond keys add wallet
```

> [!IMPORTANT]  
> Copy and Save the seed phrase. This will be used to recover your testnet wallet when needed

**Wait until the node is synced**

Let the node run for a while (it may take more than 30 minutes).

After that, enter the command below:
```
babylond status 2>&1 | jq .SyncInfo.catching_up
```

If the output is `false` → then the node is synced.

**Faucet some tokens**

Join [**Babylon's Discord server**](https://discord.com/invite/babylonglobal) and navigate to the faucet section. There, enter your node wallet address that you created.
> [!IMPORTANT]  
> Replace `yourWalletAddress` with your own address
> 
```
!faucet yourWalletAddress
```

**Verify the balance**

If you have used faucet in Discord, your wallet will have some tokens.
```
babylond q bank balances $(babylond keys show wallet -a)
```

**Create validator wallet**
```
babylond create-bls-key $(babylond keys show wallet -a)
```
```
cat $HOME/.babylond/config/priv_validator_key.json
```
The output will be your validator wallet’s address & key.

Save the output to a file so that you can access it later on.

**Restart a node service**
```
sudo systemctl restart babylond
```

**Set your validator moniker**
> [!IMPORTANT]  
> Replace `YourValidatorMoniker` with your own
```
NODE_MONIKER="YourValidatorMoniker"
```

**Create validator**
> [!IMPORTANT]  
> Replace `YourNodeDetails` with your own
```
babylond tx checkpointing create-validator \
--amount=10ubbn \
--pubkey=$(babylond tendermint show-validator) \
--moniker="$NODE_MONIKER" \
--details="YourNodeDetails" \
--chain-id=bbn-test-2 \
--commission-rate=0.1 \
--commission-max-rate=0.2 \
--commission-max-change-rate=0.05 \
--min-self-delegation=1 \
--fees=2000ubbn \
--from=wallet \
-y
```

**Wait for ~30 min for the validator to appear**

It takes time before your validator appear. Then type in:
```
babylond q staking validator $(babylond keys show wallet --bech val -a)
```

If an error keeps happening type wait for awhile and try again.

**Check your wallet status**

Type in your node wallet address [here](https://babylon.explorers.guru/). You should see your wallet with some tokens in it with some transaction activities.
