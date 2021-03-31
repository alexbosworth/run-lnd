# Run LND

Notes on setting up and running [LND] instances.

Example commands are given from the perspective of running Ubuntu

## System Requirements

- EC2: T3 Micro Instance or better
- IP: A clear-net routing node should get a fairly static IP
- OS: Ubuntu is pretty common, any OS
- PORT: 9735 will be the standard P2P port, 10009 the standard gRPC port
- DISK: 25 GB+ (on AWS select the io2 storage and at least 200 IOPs)

- *Note: EC2 will only give you 5 IPs per region*
- *Note: When creating an EC2 instance you'll have to add rules to it's security group that allow access to ports 9735 and 10009*

### Disk:

If using Bitcoin Core on mainnet, setup a disk that can host the entire 
Blockchain and transaction index: 500 GB.

If using Neutrino lite-mode a separate disk is not necessary.

## Initial Setup

If on EC2:

```shell
# adjust privs on PEM file
sudo chmod 600 ~/PATH_TO_PEM_FILE 
```

Add an Elastic IP and associate it with the node

Connect:

```
ssh -i ~/path_to_downloaded_pem_file ubuntu@IP_OF_INSTANCE
```

Install your favorite editor, like emacs:

```shell
sudo apt update && sudo apt upgrade -y && sudo apt install -y emacs

# open and then quit
emacs

# change owner of emacs config
sudo chown -R ubuntu ~/.emacs.d
```

If running on a public instance, increase the file descriptors limit:

```shell
sudo emacs /etc/sysctl.conf
```

Add line:

```
fs.file-max=512000
```

```shell
# Save and reboot
sudo reboot
```

If using an attached disk for the full Blockchain and it has not yet been initialized set it up as 
something like `/blockchain`

```shell

# List storage
lsblk
# You will get the volume name appearing as something like nvme1n1

# Check on the storage to make sure it is empty
sudo file -s /dev/nvme1n1
# should show "/dev/nvme1n1: data" meaning empty

# Format the storage as ext4. It may take a second
sudo mkfs -t ext4 /dev/nvme1n1

# Make a directory for the volume and mount it
sudo mkdir /blockchain
sudo mount /dev/nvme1n1 /blockchain/
cd /blockchain

# Double check you have enough space
df -h .
# should show available space in the volume

# Automatically mount the partition, but first backup the existing config
sudo cp /etc/fstab /etc/fstab.bak
sudo emacs /etc/fstab

# Create entry in the file:
/dev/nvme1n1 /blockchain ext4 defaults,nofail 0 0

# Save and exit, then test:
sudo mount -a
# Should show no errors

# Take ownership of the directory:
sudo chown `whoami` /blockchain
```

Setup a local firewall:

```shell
# Check if UFW is installed
which ufw
sudo ufw logging on
sudo ufw enable
# PRESS Y
# Allow access to 9735 the P2P port and 10009 the gRPC port
sudo ufw status
sudo ufw allow OpenSSH
sudo ufw allow 9735
sudo ufw allow 10009
```

Setup network flood protection:

```shell
sudo iptables -N syn_flood
sudo iptables -A INPUT -p tcp --syn -j syn_flood
sudo iptables -A syn_flood -m limit --limit 1/s --limit-burst 3 -j RETURN
sudo iptables -A syn_flood -j DROP
sudo iptables -A INPUT -p icmp -m limit --limit 1/s --limit-burst 1 -j ACCEPT
sudo iptables -A INPUT -p icmp -m limit --limit 1/s --limit-burst 1 -j LOG --log-prefix PING-DROP:
sudo iptables -A INPUT -p icmp -j DROP
sudo iptables -A OUTPUT -p icmp -j ACCEPT
```

## Access Control

**On a remote instance, set it up to use hardware keys only to authenticate**

You can setup your SSH keys by editing `~/.ssh/authorized_keys`.

Use a `#` comment above the keys to comment on what they are

## Using Tor

If you want to run your node behind Tor? [Install Tor].

Instructions:

```shell
sudo apt-get update && sudo apt install -y apt-transport-https

# Edit package sources for installation
sudo emacs /etc/apt/sources.list.d/tor.list

deb https://deb.torproject.org/torproject.org focal main
deb-src https://deb.torproject.org/torproject.org focal main

# Get the GPG key for Tor and add it to GPG
sudo curl https://deb.torproject.org/torproject.org/A3C4F0F979CAA22CDBA8F512EE8CBC9E886DDD89.asc | sudo gpg --import
sudo gpg --export A3C4F0F979CAA22CDBA8F512EE8CBC9E886DDD89 | sudo apt-key add -

# Install the Tor package
sudo apt update && sudo apt install -y tor deb.torproject.org-keyring

# Add a user for Tor
sudo usermod -a -G debian-tor `whoami`
```

Then configure Tor:

```shell
# Edit the Tor configuration
sudo emacs /etc/tor/torrc
```

```
# Add these lines at the top of the file:

ControlPort 9051
CookieAuthentication 1
CookieAuthFileGroupReadable 1
Log notice stdout
SOCKSPort 9050
```

```shell
# Restart the Tor service
sudo service tor restart
```

Check if Tor is working

```shell
curl --socks5 localhost:9050 --socks5-hostname localhost:9050 -s https://check.torproject.org/ | cat | grep -m 1 Congratulations | xargs
```

This should echo `Congratulations`

## Install Bitcoin Core

Using Bitcoin Core as a chain backend? [Download Bitcoin Core].

Installation:

```
sudo apt install git build-essential libtool autotools-dev automake pkg-config libssl-dev libevent-dev bsdmainutils libboost-system-dev libboost-filesystem-dev libboost-chrono-dev libboost-program-options-dev libboost-test-dev libboost-thread-dev libminiupnpc-dev libzmq3-dev
git clone -b v0.21.0 https://github.com/bitcoin/bitcoin.git
cd bitcoin/
./autogen.sh 
./configure CXXFLAGS="--param ggc-min-expand=1 --param ggc-min-heapsize=32768" --enable-cxx --with-zmq --without-gui --disable-shared --with-pic --disable-tests --disable-bench --enable-upnp-default --disable-wallet
# This may take a while
make -j "$(($(nproc)+1))"
sudo make install
```

Setup directories on the Blockchain storage volume, and also create the 
[Bitcoin Core data directory] in order to setup the configuration file:

```shell
mkdir /blockchain/.bitcoin && mkdir /blockchain/.bitcoin/data && mkdir ~/.bitcoin
```

Download and use the [Bitcoin Core auth script] to generate credentials:

```shell
wget https://raw.githubusercontent.com/bitcoin/bitcoin/master/share/rpcauth/rpcauth.py
python ./rpcauth.py bitcoinrpc
# This will output the authentication string to add to bitcoin.conf
# Save the password, this will be used for LND configuration
```

Edit the configuration file. If you have an existing Bitcoin Core, use 
`getbestblockhash` to get the current chain tip hash.

```shell
emacs ~/.bitcoin/bitcoin.conf
```

Add this configuration:

```ini
# Set the best block hash here:
assumevalid=

# Run as a daemon mode without an interactive shell
daemon=1

# Set the data directory to the storage directory
datadir=/blockchain/.bitcoin/data

# Set the number of megabytes of RAM to use, set to like 50% of available memory
dbcache=3000

# Add visibility into mempool and RPC calls for potential LND debugging
debug=mempool
debug=rpc

# Turn off the wallet, it won't be used
disablewallet=1

# Don't bother listening for peers
listen=0

# Constrain the mempool to the number of megabytes needed:
maxmempool=100

# Limit uploading to peers
maxuploadtarget=1000

# Turn off serving SPV nodes
nopeerbloomfilters=1
peerbloomfilters=0

# Don't accept deprecated multi-sig style
permitbaremultisig=0

# Set the RPC auth to what was set above
rpcauth=

# Turn on the RPC server
server=1

# Reduce the log file size on restarts
shrinkdebuglog=1

# Set testnet if needed
testnet=1

# Turn on transaction lookup index
txindex=1

# Turn on ZMQ publishing
zmqpubrawblock=tcp://127.0.0.1:28332
zmqpubrawtx=tcp://127.0.0.1:28333
```

Using Tor? Add additional lines:

```ini
# put under [main] section
# Some mainnet peers
addnode=gyn2vguc35viks2b.onion
addnode=kvd44sw7skb5folw.onion
addnode=nkf5e6b7pl4jfd4a.onion
addnode=yu7sezmixhmyljn4.onion
addnode=3ffk7iumtx3cegbi.onion
addnode=3nmbbakinewlgdln.onion
addnode=4j77gihpokxu2kj4.onion
addnode=546esc6botbjfbxb.onion
addnode=5at7sq5nm76xijkd.onion
addnode=77mx2jsxaoyesz2p.onion
addnode=7g7j54btiaxhtsiy.onion
addnode=a6obdgzn67l7exu3.onion
addnode=ab64h7olpl7qpxci.onion
addnode=am2a4rahltfuxz6l.onion
addnode=azuxls4ihrr2mep7.onion
addnode=bitcoin7bi4op7wb.onion
addnode=bitcoinostk4e4re.onion
addnode=bk7yp6epnmcllq72.onion
addnode=bmutjfrj5btseddb.onion
addnode=ceeji4qpfs3ms3zc.onion
addnode=clexmzqio7yhdao4.onion
addnode=gb5ypqt63du3wfhn.onion
addnode=h2vlpudzphzqxutd.onion

# Only use Tor
onlynet=onion

# Connect to Tor proxy
proxy=127.0.0.1:9050
```

Start Bitcoin Core:

```shell
bitcoind
```

Add Bitcoin Core to crontab:

```shell
crontab -e
```

Add entry:

```
# Start Bitcoin Core on boot
@reboot bitcoind
```

Create an easy link to the debug log of Bitcoin Core:

```shell
# Mainnet:
ln -s /blockchain/.bitcoin/data/debug.log ~/bitcoind-mainnet.log

# Or Testnet:
ln -s /blockchain/.bitcoin/data/testnet3/debug.log ~/bitcoind-testnet.log
```

Create a file to rotate the logs

```shell
sudo emacs /etc/logrotate.d/bitcoin-debug

# Add these instructions
# Uncomment depending on mainnet or testnet:
# /blockchain/.bitcoin/data/debug.log
# /blockchain/.bitcoin/data/testnet3/debug.log
{
        rotate 5
        copytruncate
        daily
        missingok
        notifempty
        compress
        delaycompress
        sharedscripts
}
```

## Install Go

Building from source? [Install Go]

You can check if Go is installed and what version it is, and then install or update:

```shell
go version
# Should show Go version 1.15.7 or higher

# If an out of date Go is already installed
sudo rm -rf /usr/local/go

# If installing Go for the first time
sudo apt-get update && sudo apt-get -y upgrade

# Download Go
wget https://golang.org/dl/go1.15.7.linux-amd64.tar.gz

# Extract it
sudo tar -xvf go1.15.7.linux-amd64.tar.gz

# Install it and remove the download
sudo mv go /usr/local && rm go1.15.7.linux-amd64.tar.gz

# On a new install, make a directory for it
mkdir ~/go

# On a new install, setup the path to use the Go directory
emacs ~/.profile

# Place lines at the end of the file:
GOPATH=$HOME/go
PATH="$HOME/bin:$GOPATH/bin:$HOME/.local/bin:/usr/local/go/bin:$PATH"

# Add an alias if running on Testnet
alias lncli="lncli --network=testnet"

# Save and exit, then run profile
. ~/.profile
```

## Install LND

[Install LND] on the machine, then setup its configuration

```shell
# Get build tools
sudo apt-get install -y build-essential

# Clone the LND repo and install LND
cd ~/
git clone https://github.com/lightningnetwork/lnd.git
cd lnd
git checkout v0.12.1-beta
make && make install tags="autopilotrpc chainrpc invoicesrpc routerrpc signrpc walletrpc watchtowerrpc wtclientrpc"
mkdir ~/.lnd
emacs ~/.lnd/lnd.conf
```

Set configuration for LND: (Make sure to replace IP etc with correct IP)

```ini
[Application Options]
# Allow push payments
accept-keysend=1

# Public network name
alias=YOUR_ALIAS

# Allow gift routes
allow-circular-route=1

# Public hex color
color=#000000

# Log levels
debuglevel=CNCT=debug,CRTR=debug,HSWC=debug,NTFN=debug,RPCS=debug

# Public P2P IP (remove this if using Tor)
externalip=INSTANCE_IP

# Mark unpayable, unpaid invoices as deleted
gc-canceled-invoices-on-startup=1
gc-canceled-invoices-on-the-fly=1

# Avoid historical graph data sync
ignore-historical-gossip-filters=1

# Listen (not using Tor? Remove this)
listen=localhost

# Set the maximum amount of commit fees in a channel
max-channel-fee-allocation=1.0

# Set the max timeout blocks of a payment
max-cltv-expiry=5000

# Pending channel limit
maxpendingchannels=10

# Min inbound channel limit
minchansize=5000000

# gRPC socket binding
rpclisten=0.0.0.0:10009

# Avoid slow startup time
sync-freelist=1

# Avoid high startup overhead
stagger-initial-reconnect=1

# Delete and recreate RPC TLS certificate when details change or cert expires
tlsautorefresh=1

# Do not include IPs in the RPC TLS certificate
tlsdisableautofill=1

# Add DNS to the RPC TLS certificate
tlsextradomain=YOUR_DOMAIN_NAME

[Bitcoin]
# Turn on Bitcoin mode
bitcoin.active=1

# Set the channel confs to wait for channels
bitcoin.defaultchanconfs=2

# Forward fee rate in parts per million
bitcoin.feerate=1000

# Set bitcoin.testnet=1 or bitcoin.mainnet=1 as appropriate
bitcoin.mainnet=1

# Set the lower bound for HTLCs
bitcoin.minhtlc=1

# Set backing node, bitcoin.node=neutrino or bitcoin.node=bitcoind
bitcoin.node=bitcoind

[bitcoind]
# Configuration for using Bitcoin Core backend

# Set the password to what the auth script said
bitcoind.rpcpass=

# Set the username
bitcoind.rpcuser=bitcoinrpc

# Set the ZMQ listeners
bitcoind.zmqpubrawblock=tcp://127.0.0.1:28332
bitcoind.zmqpubrawtx=tcp://127.0.0.1:28333

[protocol]
# Enable large channels support
protocol.wumbo-channels=1

[routerrpc]
# Make sure that LND is the binary release or built with the routerrpc tag

# Set default chance of a hop success
routerrpc.apriorihopprob=0.5

# Start to ignore nodes if they return many failures (set to 1 to turn off)
routerrpc.aprioriweight=0.75

# Set minimum desired savings of trying a cheaper path
routerrpc.attemptcost=10
routerrpc.attemptcostppm=10

# Set the number of historical routing records
routerrpc.maxmchistory=10000

# Set the min confidence in a path worth trying
routerrpc.minrtprob=0.005

# Set the time to forget past routing failures
routerrpc.penaltyhalflife=6h0m0s

[routing]
# Set validation of channels off: only if using Neutrino
routing.assumechanvalid=1

[tor]
# Enable Tor if using
tor.active=1
tor.v3=1
```

If `bitcoin.node=neutrino` is set, add Neutrino options to lnd.conf:

```ini
[neutrino]
# Mainnet addpeers
neutrino.addpeer=btcd-mainnet.lightning.computer
neutrino.addpeer=mainnet1-btcd.zaphq.io
neutrino.addpeer=mainnet2-btcd.zaphq.io
neutrino.addpeer=mainnet3-btcd.zaphq.io
neutrino.addpeer=mainnet4-btcd.zaphq.io

# Testnet addpeers
neutrino.addpeer=btcd-testnet.ion.radar.tech
neutrino.addpeer=btcd-testnet.lightning.computer
neutrino.addpeer=lnd.bitrefill.com:18333
neutrino.addpeer=faucet.lightning.community
neutrino.addpeer=testnet1-btcd.zaphq.io
neutrino.addpeer=testnet2-btcd.zaphq.io
neutrino.addpeer=testnet3-btcd.zaphq.io
neutrino.addpeer=testnet4-btcd.zaphq.io

# Set fee data URL, change to btc-fee-estimates.json if mainnet
neutrino.feeurl=https://nodes.lightning.computer/fees/v1/btctestnet-fee-estimates.json
```

```shell
# Start LND with nohup for non-interactive operation
# Alternatively: use systemd https://gist.github.com/alexbosworth/171958cc9888b7ebf3a91e5c23a57464
nohup /home/ubuntu/go/bin/lnd > /dev/null 2> /home/ubuntu/.lnd/err.log &
```

Setup LND

```shell
openssl rand -hex 21 > ~/.lnd/wallet_password

cat ~/.lnd/wallet_password
# Copy this password

lncli create
# Follow prompts, use the wallet password as the initial password and set no cipher seed password
```

Edit crontab to run on startup and setup easy link of logs:

```shell
# Link if Mainnet
ln -s ~/.lnd/logs/bitcoin/mainnet/lnd.log ~/lnd-mainnet.log

# Link if Testnet
ln -s ~/.lnd/logs/bitcoin/testnet/lnd.log ~/lnd-testnet.log

# Setup crontab to start and unlock LND on boot
crontab -e
```

```
# Start LND on boot - or use systemd if you prefer: https://gist.github.com/alexbosworth/171958cc9888b7ebf3a91e5c23a57464
@reboot nohup /home/ubuntu/go/bin/lnd > /dev/null 2> /home/ubuntu/.lnd/err.log &

# Unlock wallet if locked
*/5 * * * * /home/ubuntu/.npm-global/bin/bos unlock /home/ubuntu/.lnd/wallet_password
```

```shell
## Connect the new node to some existing nodes to bootstrap the graph
# Testnet, connect to htlc.me, testnet.yalls.org
lncli connect 03c856d2dbec7454c48f311031f06bb99e3ca1ab15a9b9b35de14e139aa663b463@34.201.74.232:9735
lncli connect 027455aef8453d92f4706b560b61527cc217ddf14da41770e8ed6607190a1851b8@3.13.29.161:9735
# Mainnet, connect to some nodes, like:
lncli connect 03e50492eab4107a773141bb419e107bda3de3d55652e6e1a41225f06a0bbf2d56@3.13.48.80:9735

# Open channels to an initial node to bootstrap network connectivity
# testnet
lncli openchannel 03c856d2dbec7454c48f311031f06bb99e3ca1ab15a9b9b35de14e139aa663b463 500000
# mainnet
lncli openchannel 03e50492eab4107a773141bb419e107bda3de3d55652e6e1a41225f06a0bbf2d56 5000000
```

## Install Balance of Satoshis

This will need a [Node.js installation] to run:

```shell
curl -sL https://deb.nodesource.com/setup_14.x | sudo -E bash -
sudo apt-get install -y nodejs

# Avoid using sudo with NPM
mkdir ~/.npm-global
npm config set prefix '~/.npm-global'

# Update path
emacs ~/.profile

# Add line to the end
PATH="$HOME/.npm-global/bin:$PATH"

# Save and exit, update shell:
. ~/.profile

# Install balanceofsatoshis
npm i -g balanceofsatoshis
```

## Load Coins

```shell
bos chain-deposit
```

If you're using testnet, here are some faucets:

- [Coinfaucet]
- [YABTF]

[Bitcoin Core auth script]: https://github.com/bitcoin/bitcoin/blob/master/share/rpcauth/rpcauth.py
[Bitcoin Core data directory]: https://en.bitcoin.it/wiki/Data_directory
[Coinfaucet]: https://coinfaucet.eu/en/btc-testnet/
[Download Bitcoin Core]: https://bitcoincore.org/en/download/
[Install Go]: https://golang.org/doc/install
[Install LND]: https://github.com/lightningnetwork/lnd/blob/master/docs/INSTALL.md
[Install Tor]: https://2019.www.torproject.org/docs/installguide.html.en
[LND]: https://github.com/lightningnetwork/lnd
[Node.js installation]: https://nodejs.org/en/download/package-manager/
[YABTF]: https://testnet-faucet.mempool.co
