# Installation
# **Auto Install:**
```
curl https://revonode.xyz/sc/selfchain | bash
```
# Manual Install:
# Server preparation
- Updating packages
```
sudo apt update && sudo apt upgrade -y
```
- Install developer tools and necessary packages
```
sudo apt install curl build-essential pkg-config libssl-dev git wget jq make gcc tmux chrony -y
```
- Installing GO
```
ver="1.20"
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz"
rm "go$ver.linux-amd64.tar.gz"
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile
source $HOME/.bash_profile
go version
```
# Node installation
- Clone the project repository with the node, go to the project folder and collect the binary files
```
cd $HOME
wget https://snapshots.indonode.net/selfchain/selfchaind
sudo chmod +x selfchaind
sudo mv selfchaind /usr/local/bin
```
- Checking the version
```
selfchaind version
#0.2.2
```
- Creating Variables
```
MONIKER_SELFCHAIN=type in your name
CHAIN_ID_SELFCHAIN=self-dev-1
PORT_SELFCHAIN=37
```
- Save variables, reload .bash_profile and check variable values
```
echo "export MONIKER_SELFCHAIN="${MONIKER_SELFCHAIN}"" >> $HOME/.bash_profile
echo "export CHAIN_ID_SELFCHAIN="${CHAIN_ID_SELFCHAIN}"" >> $HOME/.bash_profile
echo "export PORT_SELFCHAIN="${PORT_SELFCHAIN}"" >> $HOME/.bash_profile
source $HOME/.bash_profile

echo -e "\nmoniker_SELFCHAIN > ${MONIKER_SELFCHAIN}.\n"
echo -e "\nchain_id_SELFCHAIN > ${CHAIN_ID_SELFCHAIN}.\n"
echo -e "\nport_SELFCHAIN > ${PORT_SELFCHAIN}.\n"
```
- Setting up the config
```
selfchaind config chain-id self-dev-1
selfchaind config keyring-backend test
sed -i -e "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"0.005uself\"/" $HOME/.selfchain/config/app.toml
```
- Initialize the node
```
selfchaind init $MONIKER_SELFCHAIN --chain-id $CHAIN_ID_SELFCHAIN
```
- Loading the genesis file and address book
```
wget -O $HOME/.selfchain/config/genesis.json  https://raw.githubusercontent.com/hotcrosscom/selfchain-genesis/main/networks/devnet/genesis.json
curl -Ls https://snapshots.indonode.net/selfchain/addrbook.json > $HOME/.selfchain/config/addrbook.json
```
- Adding seeds and peers
```
SEEDS="94a7baabb2bcc00c7b47cbaa58adf4f433df9599@157.230.119.165:26656,d3b5b6ca39c8c62152abbeac4669816166d96831@165.22.24.236:26656
PEERS="94a7baabb2bcc00c7b47cbaa58adf4f433df9599@157.230.119.165:26656,d3b5b6ca39c8c62152abbeac4669816166d96831@165.22.24.236:26656
sed -i.bak -e "s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" ~/.selfchaind/config/config.toml
```
- Setting up pruning
```
PRUNING="custom"
PRUNING_KEEP_RECENT="100"
PRUNING_INTERVAL="19"

sed -i -e "s/^pruning *=.*/pruning = \"$PRUNING\"/" $HOME/.selfchain/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \
\"$PRUNING_KEEP_RECENT\"/" $HOME/.selfchain/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \
\"$PRUNING_INTERVAL\"/" $HOME/.selfchain/config/app.toml
sed -i -e 's|^indexer *=.*|indexer = "null"|' $HOME/.selfchain/config/config.toml
sed -i 's|^prometheus *=.*|prometheus = true|' $HOME/.selfchain/config/config.toml
```
- Create a service file
```
sudo tee /etc/systemd/system/selfchaind.service > /dev/null <<EOF
[Unit]
Description=selfchaind testnet node
After=network-online.target
[Service]
User=$USER
ExecStart=$(which selfchaind) start
Restart=always
RestartSec=3
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
```
- Start the service and check the logs
```
sudo systemctl daemon-reload && \
sudo systemctl enable selfchaind && \
sudo systemctl restart selfchaind && \
sudo journalctl -u selfchaind -f -o cat
```
- We are waiting for the end of synchronization, you can check the synchronization with the command
```
selfchaind status 2>&1 | jq .SyncInfo
```
If the output shows false, the sync is complete.

# Creating a wallet and validator
- Create a wallet
```
selfchaind keys add $MONIKER_SELFCHAIN
```
Save the mnemonic phrase in a safe place!

If you participated in previous testnets, restore the wallet with the command and enter the mnemonic phrase
```
selfchaind keys add $MONIKER_SELFCHAIN --recover
```
- Create a variable with the address of the wallet and validator
```
WALLET_SELFCHAIN=$(selfchaind keys show $MONIKER_SELFCHAIN -a)
VALOPER_SELFCHAIN=$(selfchaind keys show $MONIKER_SELFCHAIN --bech val -a)

echo "export WALLET_SELFCHAIN="${WALLET_SELFCHAIN}"" >> $HOME/.bash_profile
echo "export VALOPER_SELFCHAIN="${VALOPER_SELFCHAIN}"" >> $HOME/.bash_profile
source $HOME/.bash_profile
echo -e "\nwallet_SELFCHAIN > ${WALLET_SELFCHAIN}.\n"
echo -e "\nvaloper_SELFCHAIN > ${VALOPER_SELFCHAIN}.\n"
```
- Checking your balance
```
selfchaind q bank balances $WALLET_SELFCHAIN
```
- After the synchronization is completed and the wallet is replenished, we create a validator
```
selfchaind tx staking create-validator \
  --amount 1000000uself \
  --pubkey $(selfchaind tendermint show-validator) \
  --moniker "MONIKER" \
  --identity="YOUR_KEYBASE_ID" \
  --details="YOUR_DETAILS" \
  --website="YOUR_WEBSITE_URL" \
  --chain-id self-dev-1 \
  --commission-rate="0.05" \
  --commission-max-rate="0.20" \
  --commission-max-change-rate="0.01" \
  --min-self-delegation "1" \
  --gas-prices="0.005uself" \
  --gas="auto" \
  --gas-adjustment="1.5" \
  --from wallet \
  -y
```
# Deleting a node
- Before deleting a node, make sure that the files from the ~/.selfchaind/config directory are saved To remove a node, use the following commands
```
sudo systemctl stop selfchaind
sudo systemctl disable selfchaind
sudo rm -rf $HOME/.selfchain
sudo rm -rf /etc/systemd/system/selfchaind.service
sudo rm -rf /usr/local/bin/selfchaind
sudo systemctl daemon-reload
```
<img src="https://miro.medium.com/v2/resize:fit:1400/1*vktX69M0K8K7tVZwrD2bZQ.png">

# **Contact Me🔥☕**
<p align="left">
<a href="https://www.github.com/0xGilang"><img height="40" width="40" align="center" src="https://avatars.githubusercontent.com/u/94946818?v=4"></a>
<a href="https://www.facebook.com/Gilangbuanasultoni" target="blank"><img align="center" src="https://raw.githubusercontent.com/rahuldkjain/github-profile-readme-generator/master/src/images/icons/Social/facebook.svg" alt="AryaTrickers2020" height="30" width="40" /></a>
<a href="https://wa.me/6282131561458?text=Halo+Bang+Gilang" target="blank"><img align="center" src="https://raw.githubusercontent.com/rahuldkjain/github-profile-readme-generator/master/src/images/icons/Social/whatsapp.svg" alt="6289694295787" height="30" width="40" /></a>
<a href="https://twitter.com/Gilangsultoni"><img height="40" width="40" align="center" src="https://static.dezeen.com/uploads/2023/07/x-logo-twitter-elon-musk_dezeen_2364_col_0.jpg"></a>

------
------
