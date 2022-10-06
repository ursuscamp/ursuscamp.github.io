---
layout: post
title:  "Home Server 3: Bitcoin Core"
date:   2022-10-05 20:36:52 -0400
categories: bitcoin server
---
The time has come! We will finally be able to install Bitcoin Core, and when we are finised, we can send a totally original and unique "running Bitcoin" tweet.

Start by SSH'ing into your server, then on your main machine, go to the [bitcoincore.org downloads page](https://bitcoincore.org/en/download/). Right click `Linux (tgz)` link and copy the URL.

On your server, run `sudo apt install wget` to install `wget`, a terminal application that can download files over the web. After it installs, run `wget https://bitcoincore.org/bin/bitcoin-core-23.0/bitcoin-23.0-x86_64-linux-gnu.tar.gz`, except use the URL you copied from the Bitcoin Core website, which may different.

That will download Bitcoin Core from the website, and leave the file in your local folder:

```
user@server:~/$ ls
bitcoin-23.0-x86_64-linux-gnu.tar.gz
```

Unzip the file:

```
user@server:~/$ tar xvf bitcoin-23.0-x86_64-linux-gnu.tar.gz
bitcoin-23.0/
bitcoin-23.0/README.md
bitcoin-23.0/bin/
bitcoin-23.0/bin/bitcoin-cli
bitcoin-23.0/bin/bitcoin-qt
bitcoin-23.0/bin/bitcoin-tx
bitcoin-23.0/bin/bitcoin-util
bitcoin-23.0/bin/bitcoin-wallet
bitcoin-23.0/bin/bitcoind
bitcoin-23.0/bin/test_bitcoin
bitcoin-23.0/include/
bitcoin-23.0/include/bitcoinconsensus.h
bitcoin-23.0/lib/
bitcoin-23.0/lib/libbitcoinconsensus.so
bitcoin-23.0/lib/libbitcoinconsensus.so.0
bitcoin-23.0/lib/libbitcoinconsensus.so.0.0.0
bitcoin-23.0/share/
bitcoin-23.0/share/man/
bitcoin-23.0/share/man/man1/
bitcoin-23.0/share/man/man1/bitcoin-cli.1
bitcoin-23.0/share/man/man1/bitcoin-qt.1
bitcoin-23.0/share/man/man1/bitcoin-tx.1
bitcoin-23.0/share/man/man1/bitcoin-util.1
bitcoin-23.0/share/man/man1/bitcoin-wallet.1
bitcoin-23.0/share/man/man1/bitcoind.1
```

We can adapt the official [installation instruction](https://bitcoin.org/en/full-node#linux-instructions) for our use.

Install the software directly into your `/usr/local/bin` folder.

```
user@server:~/$ sudo install -m 0755 -o root -g root -t /usr/local/bin bitcoin-23.0/bin/*
```

You can now run `bitcoind`, but don't start it yet! We're going to create a systemd service to start the bitcoin node at system boot.

```
user@server:~/$ sudo nano /etc/systemd/system/bitcoind.service
```

Paste the following into the file:

```
[Unit]
Description=Bitcoin daemon
Documentation=https://github.com/bitcoin/bitcoin/blob/master/doc/init.md

# https://www.freedesktop.org/wiki/Software/systemd/NetworkTarget/
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/local/bin/bitcoind -daemon \
                            -pid=/run/bitcoind/bitcoind.pid \
                            -conf=/home/USERNAME/.bitcoin/bitcoin.conf \
                            -datadir=/home/USERNAME/.bitcoin

# Make sure the config directory is readable by the service user
PermissionsStartOnly=true
#ExecStartPre=/bin/chgrp bitcoin /etc/bitcoin

# Process management
####################

Type=forking
PIDFile=/run/bitcoind/bitcoind.pid
Restart=on-failure
TimeoutStartSec=infinity
TimeoutStopSec=600

# Directory creation and permissions
####################################

# Run as bitcoin:bitcoin
User=USERNAME
Group=USERNAME

# /run/bitcoind
RuntimeDirectory=bitcoind
RuntimeDirectoryMode=0710

# /etc/bitcoin
ConfigurationDirectory=bitcoin
ConfigurationDirectoryMode=0710

# /var/lib/bitcoind
StateDirectory=bitcoind
StateDirectoryMode=0710

# Hardening measures
####################

# Provide a private /tmp and /var/tmp.
PrivateTmp=true

# Mount /usr, /boot/ and /etc read-only for the process.
ProtectSystem=full

# Deny access to /home, /root and /run/user
#ProtectHome=true

# Disallow the process and all of its children to gain
# new privileges through execve().
NoNewPrivileges=true

# Use a new /dev namespace only populated with API pseudo devices
# such as /dev/null, /dev/zero and /dev/random.
PrivateDevices=true

# Deny the creation of writable and executable memory mappings.
MemoryDenyWriteExecute=true

[Install]
WantedBy=multi-user.target
```

Search for every instance of `USERNAME` in the config file and replace it with your username. If you're not unsure what this file is for, it's a just description that tells systemd (Ubuntu's service manager) how, why and when to start and stop the Bitcoin daemon.

Lastly, we are going to configure bitcoin using a configuration file:

```
user@server:~/$ mkdir -p ~/.bitcoin
user@server:~/$ sudo nano ~/.bitcoin/bitcoin.conf
```

Paste this content into the file:

```
server=1
txindex=1
daemon=1
rpcport=8332
rpcbind=0.0.0.0
rpcallowip=127.0.0.1
rpcallowip=10.0.0.0/8
rpcallowip=172.0.0.0/8
rpcallowip=192.0.0.0/8
zmqpubrawblock=tcp://0.0.0.0:28332
zmqpubrawtx=tcp://0.0.0.0:28333
zmqpubhashblock=tcp://0.0.0.0:28334
whitelist=127.0.0.1
rpcuser=username
rpcpassword=password
```

Make sure to replace `rpcuser` and `rpcpassword` with an appropriate username and password. **Note:** As you can see, at this time, we are not setting up Tor. That may come later.

Finally, everything is ready to go. Just run `sudo systemctl enable bitcoind` and `sudo systemctl start bitcoind`.

At this point, Bitcoin should be running and beginning to download the blockchain. You can run `sudo systemctl status bitcoind` to see how the service is running. Alternative, you can check the Bitcoin debug log to see how the download is doing:

```
user@server:~/$ tail -f ~/.bitcoin/debug.log
2022-10-06T01:13:19Z New outbound peer connected: version: 70016, blocks=757289, peer=1444 (block-relay-only)
2022-10-06T01:20:49Z New outbound peer connected: version: 70016, blocks=757289, peer=1445 (block-relay-only)
2022-10-06T01:21:21Z New outbound peer connected: version: 70016, blocks=757289, peer=1446 (block-relay-only)
2022-10-06T01:27:07Z UpdateTip: new best=000000000000000000005049f1ef7798dbd3ba2c47752d21bee0d91db06f972b height=757290 version=0x220ac000 log2_work=93.768231 tx=769965790 date='2022-10-06T01:26:28Z' progress=1.000000 cache=74.8MiB(516512txo)
2022-10-06T01:27:35Z UpdateTip: new best=000000000000000000002e69ef654077c22f7fe6451c777c1780a033c50d2b85 height=757291 version=0x2e95c004 log2_work=93.768242 tx=769965875 date='2022-10-06T01:27:07Z' progress=1.000000 cache=74.8MiB(516869txo)
2022-10-06T01:29:12Z UpdateTip: new best=00000000000000000002832a316e84e2ff86f71fcfb219c860f8762c5b73b5e5 height=757292 version=0x2c4fc000 log2_work=93.768254 tx=769966199 date='2022-10-06T01:28:55Z' progress=1.000000 cache=74.8MiB(517081txo)
2022-10-06T01:29:14Z New outbound peer connected: version: 70015, blocks=757292, peer=1447 (block-relay-only)
2022-10-06T01:31:40Z UpdateTip: new best=0000000000000000000713bd733ab4a7d42e7a3853e78829ed78b98d60d7ca46 height=757293 version=0x20000000 log2_work=93.768265 tx=769966586 date='2022-10-06T01:31:18Z' progress=1.000000 cache=75.0MiB(518131txo)
2022-10-06T01:33:03Z UpdateTip: new best=00000000000000000006fb159899fe535dfde78f18107ee026edb8b3a50285ab height=757294 version=0x20000004 log2_work=93.768277 tx=769966789 date='2022-10-06T01:32:29Z' progress=1.000000 cache=75.0MiB(518729txo)
2022-10-06T01:41:37Z UpdateTip: new best=000000000000000000039db1fbbe91942ef8453293443514fc444670358425da height=757295 version=0x20002004 log2_work=93.768288 tx=769968052 date='2022-10-06T01:41:34Z' progress=1.000000 cache=75.5MiB(522075txo)
```

This is what an updated node's log looks like. Yours may look different because it's doing a node sync, but you should at least see text indicating that the daemon is doing something. You can stop following the log by typing `Ctrl-C`.

You can check the status of the block download by typing:

```
user@server:~/$ bitcoin-cli getblockchaininfo
{
  "chain": "main",
  "blocks": 757295,
  "headers": 757295,
  "bestblockhash": "000000000000000000039db1fbbe91942ef8453293443514fc444670358425da",
  "difficulty": 31360548173144.85,
  "time": 1665020494,
  "mediantime": 1665019588,
  "verificationprogress": 0.9999979987229801,
  "initialblockdownload": false,
  "chainwork": "00000000000000000000000000000000000000003681016072e90e118f6b4420",
  "size_on_disk": 489570147653,
  "pruned": false,
  "warnings": ""
}
```

This is mine. You will see `"initalblockdownload"` as `true`, and you can see the size in disk in bytes. The `true` value indicates that the initial block sync is still in progress. On my fiber connection, it took 4-5 hours.

## Conclusion

That's it, we are all done! We are running Bitcoin Core and a full node. Don't forget to tweet some version of "running Bitcoin".

If you want to explore your node a bit, you can use the `bitcoin-cli` command to execute commands against the node, as above. Take a look at the [RPC API reference](https://developer.bitcoin.org/reference/rpc/index.html) for a listing of commands you can run with `bitcoin-cli`.

Next time, we will install an electrum server, which indexes the blockchain for use by wallets.