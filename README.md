# artela
Update Packages and Install Prerequisites
```bash
sudo apt update && apt upgrade -y
sudo apt install curl git jq lz4 build-essential unzip fail2ban ufw -y
```
Set and enable Firewall (allow port 9100)
```bash
sudo ufw default allow outgoing
sudo ufw default deny incoming
sudo ufw allow ssh
sudo ufw allow 9100
sudo ufw enable
```
Install GO
```bash
ver="1.20.7"
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz"
rm "go$ver.linux-amd64.tar.gz"
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile
source $HOME/.bash_profile
go version
```
Node Installation
```bash
cd $HOME
rm -rf artela
git clone https://github.com/artela-network/artela artela
cd artela
git checkout v0.4.7-rc4
make install
```
Set Configuration for your node
```bash
artelad config chain-id artela_11822-1
artelad config keyring-backend file
```
Init your node (change <MyNode> with your own node name)
```bash
artelad init <MyNode> --chain-in artela_11822-1
```
Add Genesis File and Addrbook
```bash
curl -Ls https://snapshots.indonode.net/artela-t/genesis.json > $HOME/.artelad/config/genesis.json
curl -Ls https://snapshots.indonode.net/artela-t/addrbook.json > $HOME/.artelad/config/addrbook.json
```
Configure Seeds and Peers
```bash
PEERS="$(curl -sS https://rpc.artela-t.indonode.net:443/net_info | jq -r '.result.peers[] | "(.node_info.id)@(.remote_ip):(.node_info.listen_addr)"' | awk -F ':' '{print $1":"$(NF)}' | sed -z 's|
|,|g;s|.$||')"
sed -i -e "s|^persistent_peers *=.*|persistent_peers = "$PEERS"|" $HOME/.artelad/config/config.toml
```
Set Pruning, Enable Prometheus, Gas Prices, and Indexer
```bash
PRUNING="custom"
PRUNING_KEEP_RECENT="100"
PRUNING_INTERVAL="19"

sed -i -e "s/^pruning *=.*/pruning = "$PRUNING"/" $HOME/.artelad/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = "$PRUNING_KEEP_RECENT"/" $HOME/.artelad/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = "$PRUNING_INTERVAL"/" $HOME/.artelad/config/app.toml
sed -i 's|^prometheus *=.*|prometheus = true|' $HOME/.artelad/config/config.toml
```
Set Service file
```bash
sudo tee /etc/systemd/system/artelad.service > /dev/null <<EOF
[Unit]
Description=artela testnet node
After=network-online.target
[Service]
User=$USER
ExecStart=$(which artelad) start
Restart=always
RestartSec=3
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable artelad
```
Download Latest Snapshot
```bash
curl -L https://snapshots.indonode.net/artela-t/artela-t-latest.tar.lz4 | tar -Ilz4 -xf - -C $HOME/.artelad
[[ -f $HOME/.artelad/data/upgrade-info.json ]] && cp $HOME/.artelad/data/upgrade-info.json $HOME/.artelad/cosmovisor/genesis/upgrade-info.json
```
Start the Node
```bash
sudo systemctl restart artelad
sudo journalctl -fu artelad -o cat
```
