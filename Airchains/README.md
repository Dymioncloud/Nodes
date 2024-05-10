# Public Endpoints

    https://junction-rpc.dymion.cloud/
    https://junction-api.dymion.cloud/
    https://junction-grpc.dymion.cloud:443/
    
# Installation guide

### Update && Upgrade:

    sudo apt update && sudo apt upgrade -y
    
### Install package:

    sudo apt install curl tar wget clang pkg-config libssl-dev jq build-essential git make lz4 unzip ncdu -y
    
### Install GO:

    ver="1.21.5" 
    cd $HOME 
    wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz" 

    sudo rm -rf /usr/local/go 
    sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz" 
    rm "go$ver.linux-amd64.tar.gz"

    echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile
    source $HOME/.bash_profile    
    
### Download binary
    wget https://github.com/airchains-network/junction/releases/download/v0.1.0/junctiond
    chmod +x junctiond
    sudo mv junctiond /usr/local/bin
    
### Config & Addrbook & Genesis

    junctiond config chain-id junction
    junctiond init "Moniker" --chain-id junction

    sudo wget -O $HOME/.junction/config/genesis.json https://files.dymion.cloud/junction/genesis.json
    sudo wget -O $HOME/.junction/config/addrbook.json https://files.dymion.cloud/junction/addrbook.json
  
    sed -i.bak -e "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"0.025amf\"/;" ~/.junction/config/app.toml

### Pruning

    pruning="custom" && \
    pruning_keep_recent="100" && \
    pruning_keep_every="0" && \
    pruning_interval="10" && \
    sed -i -e "s/^pruning *=.*/pruning = \"$pruning\"/" $HOME/.junction/config/app.toml && \
    sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"$pruning_keep_recent\"/" $HOME/.junction/config/app.toml && \
    sed -i -e "s/^pruning-keep-every *=.*/pruning-keep-every = \"$pruning_keep_every\"/" $HOME/.junction/config/app.toml && \
    sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"$pruning_interval\"/" $HOME/.junction/config/app.toml
### Live peers

    PEERS=$(curl -sS https://junction-rpc.dymion.cloud/net_info | \
    jq -r '.result.peers[] | "\(.node_info.id)@\(.remote_ip):\(.node_info.listen_addr)"' | \
    awk -F ':' '{printf "%s:%s%s", $1, $(NF), NR==NF?"":","}')
    echo "$PEERS"

    sed -i 's|^persistent_peers *=.*|persistent_peers = "'$PEERS'"|' $HOME/.junction/config/config.toml

### Snapshot (Every 24 hours)

    junctiond tendermint unsafe-reset-all --home ~/.junction/ --keep-addr-book
    curl https://files.dymion.cloud/junction/data.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.junction
    
### Create Service:

    sudo tee /etc/systemd/system/junctiond.service > /dev/null <<EOF
    [Unit]
    Description=junction
    After=network-online.target
    [Service]
    User=$USER
    ExecStart=$(which junctiond) start
    Restart=always
    RestartSec=3
    LimitNOFILE=65535
    [Install]
    WantedBy=multi-user.target
    EOF
    sudo systemctl daemon-reload
    sudo systemctl enable junctiond

### State-sync (Optional)
    sudo systemctl stop junctiond
    junctiond tendermint unsafe-reset-all --home ~/.junction/ --keep-addr-book
    SNAP_RPC="https://junction-rpc.dymion.cloud:443"

    LATEST_HEIGHT=$(curl -s $SNAP_RPC/block | jq -r .result.block.header.height); \
    BLOCK_HEIGHT=$((LATEST_HEIGHT - 2000)); \
    TRUST_HASH=$(curl -s "$SNAP_RPC/block?height=$BLOCK_HEIGHT" | jq -r .result.block_id.hash)
    echo $LATEST_HEIGHT $BLOCK_HEIGHT $TRUST_HASH

    sed -i.bak -E "s|^(enable[[:space:]]+=[[:space:]]+).*$|\1true| ; \
    s|^(rpc_servers[[:space:]]+=[[:space:]]+).*$|\1\"$SNAP_RPC,$SNAP_RPC\"| ; \
    s|^(trust_height[[:space:]]+=[[:space:]]+).*$|\1$BLOCK_HEIGHT| ; \
    s|^(trust_hash[[:space:]]+=[[:space:]]+).*$|\1\"$TRUST_HASH\"|" ~/.junction/config/config.toml
    more ~/.junction/config/config.toml | grep 'rpc_servers'
    more ~/.junction/config/config.toml | grep 'trust_height'
    more ~/.junction/config/config.toml | grep 'trust_hash'

    sudo systemctl restart junctiond
    
### Check Logs:

    journalctl -u junctiond -f -o cat

