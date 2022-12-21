---
layout: post
title:  "Home Server 4: Electrum Server"
date:   2022-11-04 20:54:00 -0400
categories: bitcoin server
---
Alright, time to move onto the next step of our Bitcoin server installation! In order for our server to be useful with a wallet, we will need to setup a server application that implements the Electrum protocol.

# What is Electrum?

Electrum is a desktop Bitcoin wallet. Additionally, there is a server protocol originated from the Electrum project that implements a number of functions useful for Bitcoin wallets. Bitcoin Core has many useful commands, but it doesn't have the indexing functionality required for a modern, full-featured Bitcoin wallet. That's where an Electrum server comes in. It scans the blockchain, and indexes it across multiple sets of data such as addresses and transactions.

Without this functionality, adding new wallets to the Bitcoin Core node would be required, along with full blockchain rescans, etc. It is not very performant for these tasks. An Electrum server will handle all of that for us.

# Electrs

[Electrs](https://github.com/romanz/electrs) is a popular Electrum server implementation for use with non-public clients (you, plus friends and family). This is what we will be installing. There are no binary packages for this, so we will be compiling it ourself. That's easier than it sounds, don't worry!

## Compiling

Begin by installing the necessary prerequisites on your server:

```
$ sudo apt update
$ sudo apt install cmake build-essential clang rustc cargo
```

Now we will use `git` to clone the repository and download the code.

```
$ mkdir code
$ cd code
$ git clone https://github.com/romanz/electrs
$ cd electrs
```

Finally, let's build the application.

```
$ cargo build --locked --release
```

It will take a few minutes, even on a decent machine. When it's finished, you will see a file called `target/release/electrs`.

Now that the application is built, install it into an executable folder on your `PATH`.

```
$ sudo install target/release/electrs /usr/local/bin
```

# Configuration 

Time to configure electrs. We're going to setup a folder to store our configuration, Bitcoin index data and SSH keys.

```
$ cd ~
$ mkdir .electrs
$ cd .electrs
$ wget https://raw.githubusercontent.com/romanz/electrs/master/doc/config_example.toml -O config.toml
```

Open `config.toml` in your text editor (`nano` or `vim`) and edit the field `db_dir`. Change it the new electrs folder, something like: `db_dir = "/home/user/.electrs/db"`. Leave everything else as is.

## Startup

We will create a systemd service file to start the application at system restart.

```
$ sudo nano /etc/systemd/system/electrs.service
```

Input the following:

```
[Unit]
Description=Electrs
After=bitcoind.service

[Service]
ExecStart=/usr/local/bin/electrs
User=user
Group=user
Type=simple
KillMode=process
TimeoutSec=60
Restart=always
RestartSec=60

[Install]
WantedBy=multi-user.target
```

Of course, repalce `User` and `Group` with your username. Now run `sudo systemctl enable electrs.service` to start the service. `systemctl status electrs` should show the server boot up and start indexing the blockchain!

## Privacy

Okay, the last and final step is setting up encryption. Electrs is configured by default to only accept connections from the local machine. We will use nginx to setup an SSL tunnel so that any connection from outside the machine are encrypted.

Let's create an SSL certificate. Run the following commands, and answer any questions with the default option.

```
$ cd ~/.electrs
$ openssl genrsa -out server.key 2048
$ openssl req -x509 -new -nodes -key server.key -sha256 -days 1825 -out server.crt
```

Now install nginx and edit the config file:

```
$ sudo apt install nginx
$ sudo nano /etc/nginx/nginx.conf
```

Add the following section to this config file. Make sure to edit `ssl_certificate` and `ssl_certificate_key` to the location of the key and certificate you just made.

```
stream {
        upstream electrs {
                server 127.0.0.1:50001;
        }

        server {
                listen 50002 ssl;
                proxy_pass electrs;

                ssl_certificate /home/user/.electrs/server.crt;
                ssl_certificate_key /home/user/.electrs/server.key;
                ssl_session_cache shared:SSL:1m;
                ssl_session_timeout 4h;
                ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
                ssl_prefer_server_ciphers on;
        }
}
```

Now `sudo systemctl restart nginx` and you should have a fully functional electrum server, with SSL!

Just point any desktop wallet, such as [Sparrow Wallet](https://sparrowwallet.com) to your machine's local network IP address, and use 50002 as the port. Make sure the SSL/TLS option is selected, if available.

**Note:** Connecting to this electrum outside instance outside of your local network is outside the scope of this tutorial. It will probably require some port forwarding setup in your home router. Look up relevant tutorials your router model.