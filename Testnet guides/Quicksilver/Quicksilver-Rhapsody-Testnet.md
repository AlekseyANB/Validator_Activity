Officiall link is here https://github.com/ingenuity-build/testnets

Explorer: 
  - https://testnet.explorer.testnet.run/Quicksilver
  - https://quicksilver.explorers.guru/

# Instructions
Update the packeges
```
sudo apt update && sudo apt upgrade -y
```
Install what we need
```
sudo apt-get install make git jq curl gcc g++ mc nano -y
```
Install GO
```
wget -O go1.18.linux-amd64.tar.gz https://go.dev/dl/go1.18.linux-amd64.tar.gz
rm -rf /usr/local/go && tar -C /usr/local -xzf go1.18.linux-amd64.tar.gz && rm go1.18.linux-amd64.tar.gz
cat <<'EOF' >> $HOME/.bash_profile
export GOROOT=/usr/local/go
export GOPATH=$HOME/go
export GO111MODULE=on
export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin
EOF
. $HOME/.bash_profile
cp /usr/local/go/bin/go /usr/bin
go version
```
Install Quicksilver
```
cd $HOME && git clone https://github.com/ingenuity-build/quicksilver.git --branch v0.1.10
cd quicksilver
git checkout v0.1.10
make build
/root/quicksilver/build/quicksilverd version
cp $HOME/quicksilver/build/quicksilverd /usr/local/bin
. $HOME/.bash_profile
quicksilverd version
```
### Let's add variables
Add your moniker instead of <Your_Moniker>. Enter by one command.
```
NODE_MONIKER=<Your_Moniker> ; \
echo $NODE_MONIKER ; \
echo 'export NODE_MONIKER='\"${NODE_MONIKER}\" >> $HOME/.bash_profile
```
Add your wallet name instead of <Your_Wallet_Name>. Enter by one command.
```
YOUR_TEST_WALLET=<Your_Wallet_Name> ; \
echo $YOUR_TEST_WALLET ; \
echo 'export YOUR_TEST_WALLET='\"${YOUR_TEST_WALLET}\" >> $HOME/.bash_profile
```
Add chainName. Enter by one command.
```
CHAIN_ID=quicktest-3 ; \
echo $CHAIN_ID ; \
echo 'export chainName='\"${CHAIN_ID}\" >> $HOME/.bash_profile
```
### Init and make chandges configuration
Init
```
quicksilverd init $NODE_MONIKER --chain-id $chainName
```
Include seeds in `config.toml`. By one command
```
SEEDS="dd3460ec11f78b4a7c4336f22a356fe00805ab64@seed.quicktest-1.quicksilver.zone:26656" ; \
sed -i -e "/seeds =/ s/= .*/= \"$SEEDS\"/"  $HOME/.quicksilverd/config/config.toml
```
Download genesis. By one command
```
GENESIS_URL="https://raw.githubusercontent.com/ingenuity-build/testnets/main/rhapsody/genesis.json" ; \
NODE_HOME=$HOME/.quicksilverd ; \
curl -sSL $GENESIS_URL > $NODE_HOME/config/genesis.json
```
Set minimum gas price
```
sed -i -e "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"0uqck\"/" $HOME/.quicksilverd/config/app.toml
```
Comment out the line `log_level=info`
```
sed -i.bak 's/^log_level/# log_level/' $HOME/.quicksilverd/config/config.toml
```
### Create test keys with the `--keyring-backend=test` flag. 
You should reed about this flag here:
- https://docs.cosmos.network/v0.44/core/cli.html#root-command
- Details about usage of `--keyring-backend` https://docs.cosmos.network/v0.44/run-node/keyring.html
- see the code https://docs.cosmos.network/v0.44/core/cli.html#root-command
```
quicksilverd keys add $YOUR_TEST_WALLET
# add flag `--recover` if you already have mnemonic 
```
### Сreate a service file
```
sudo tee <<EOF >/dev/null /etc/systemd/system/quicksilverd.service
[Unit]
Description=Quicksilver Cosmos daemon
After=network-online.target

[Service]
User=$USER
ExecStart=$(which quicksilverd) start --log_level=info
Restart=on-failure
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```
Update variables
```
source ~/.bash_profile
```
## Start the service to synchronize from first block
```
sudo systemctl enable quicksilverd
sudo systemctl daemon-reload
sudo systemctl restart quicksilverd
sudo systemctl status quicksilverd
```
### Logs and status
Logs
```
journalctl -u quicksilverd -f --output cat
```
Status of sinchronization
```
quicksilverd status 2>&1 | jq .SyncInfo
```
If you want to know your node_id.
```
curl localhost:26657/status | jq '.result.node_info.id'
```
## Synchronize by State Sync
If it takes a long time to sync from the first block, then use the quick State Sync synchronization.
Actual network State Sync from community:
- https://github.com/StakeTake/guidecosmos/blob/main/quicksilver/quicktest-3/README.md
- https://github.com/Staketab/cosmos-tools/tree/main/state-sync
- https://github.com/kj89/testnet_manuals/tree/main/quicksilver#to-synchronize-your-quicksilver-node-to-latest-block-you-have-to-use-state-sync-provided-below

In this guide, I will use the service from [kj89](https://github.com/kj89)

Stop the service
```
sudo systemctl stop quicksilverd
```
Resetting downloaded blockchain data
```
quicksilverd unsafe-reset-all
```
add State Sync data to `config.toml`
```
# variables
INTERVAL=1500
LATEST_HEIGHT=$(curl -s http://seed.quicktest-1.quicksilver.zone:26657/block | jq -r .result.block.header.height)
BLOCK_HEIGHT=$(($(($LATEST_HEIGHT / $INTERVAL)) * $INTERVAL))
TRUST_HASH=$(curl -s "http://seed.quicktest-1.quicksilver.zone:26657/block?height=$BLOCK_HEIGHT" | jq -r .result.block_id.hash)
SYNC_RPC="node02.quicktest-1.quicksilver.zone:26657,node03.quicktest-1.quicksilver.zone:26657,node04.quicktest-1.quicksilver.zone:26657"

# Ever by one command
sed -i.bak -E "s|^(enable[[:space:]]+=[[:space:]]+).*$|\1true| ; \
s|^(rpc_servers[[:space:]]+=[[:space:]]+).*$|\1\"$SYNC_RPC\"| ; \
s|^(trust_height[[:space:]]+=[[:space:]]+).*$|\1$BLOCK_HEIGHT| ; \
s|^(trust_hash[[:space:]]+=[[:space:]]+).*$|\1\"$TRUST_HASH\"|" ~/.quicksilverd/config/config.toml
```
Start the service
```
sudo systemctl restart quicksilverd
sudo systemctl status quicksilverd
```
## Faucet
Before upgrade to validator status you should to request a faucet grant.
- Here is the guide https://github.com/ingenuity-build/testnets#faucet
### Validator Tx
If your node is fully synchronized, then run the tx to upgrade to validator status:
```
quicksilverd tx staking create-validator \
 --amount=<amount>uqck \
 --pubkey=$(quicksilverd tendermint show-validator) \
 --from=$YOUR_TEST_WALLET \
 --moniker=$NODE_MONIKER \
 --chain-id=$chainName \
 --details="" \
 --website="" \
 --identity="" \
 --commission-rate=0.1 \
 --commission-max-rate=0.5 \
 --commission-max-change-rate=0.1 \
 --min-self-delegation=1 \
 --gas-prices=0.025uqck \
 --gas-adjustment=1.4
```
To see how many tokens are in your wallet, enter:
```
quicksilverd q bank balances <quick1...your..wallet...>
```
To send tokens type the command:
```
quicksilverd tx bank send <sender> <receiver> <amount>uqck --chain-id=$chainName
```
Find out your wallet
```
quicksilverd keys show $YOUR_TEST_WALLET -a
```
Find out your validator
```
quicksilverd keys show $YOUR_TEST_WALLET --bech val -a
```
To delegate more tokens
```
quicksilverd tx staking delegate $(quicksilverd keys show $YOUR_TEST_WALLET --bech val -a) <amount>uqck \
--chain-id=$chainName \
--from=$YOUR_TEST_WALLET \
--gas auto \
--fees=200uqck
```
## Unjail
Transaction to Unjail
```
quicksilverd tx slashing unjail \
--chain-id $chainName \
--from $YOUR_TEST_WALLET \
--gas=auto \
--gas-adjustment=1.4 \
--gas-prices="0.025uqck"
```
But if you are in prison, then you should analyze the reasons. The server may not be able to handle the load.

### Governance
https://testnet.explorer.testnet.run/Quicksilver/gov

How to vote:

```
quicksilverd tx gov vote 2 yes --from=$(quicksilverd keys show $WALLET -a) --chain-id=$chainName
```















