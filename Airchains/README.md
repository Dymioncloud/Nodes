## Public Endpoints

    https://junction-rpc.dymion.cloud/
    https://junction-api.dymion.cloud/
    https://junction-grpc.dymion.cloud/
    
# Manual Install

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

### Create Service:

    sudo tee /etc/systemd/system/junction.service > /dev/null <<EOF
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
    
### Check Logs:
    sudo systemctl enable junctiond
    sudo systemctl start junctiond && journalctl -u junctiond -f -o cat

