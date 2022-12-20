---
layout: post
title:  "Home Server 4: Transmission with VPN"
date:   2022-12-19 22:35:30 -0500
categories: home-server self-hosted
---

Why stick with just Bitcoin? Let's run some other stuff on this machine, too. Let's get a BitTorrent server running! But, we don't want our ISP snooping on us, so let's do it within a VPN. However, VPNs can play havoc with local LAN services, so we will also need to setup a container so that only our BitTorrent client is operating within the VPN.

How do we do that? Why, Linux network namespaces! We could also use Docker, but I'm going the barebones route this time.

First, some housekeeping: I will be using [Mullvad](https://mullvad.net) for VPN, which supports Wireguard. If you use a different VPN provider, hopefully they support Wireguard. You should be able to adapt their Wireguard configs to this without too much difficulty. You will have a harder time if they only support OpenVPN.

First step is to install is to download the [wireguard config](https://mullvad.net/en/account/#/wireguard-config/) from Mullvad. Then scp (SSH file copy) to your machine.

```bash
scp mullvad-config.zip YOUR-USERNAME@host:/home/YOUR-USERNAME/mullvad-wireguard.zip
```

Then SSH back into your server and install some software and copy the wireguard configs.

```bash
sudo apt install unzip transmission-cli transmission-daemon wireguard
unzip mullvad-wireguard-config.zip -d mullvad_config
sudo mkdir -p /etc/wireguard

# Pick your Mullvad server and copy the config
cp mullvad_config/my-server.conf /etc/wireguard/wg0.conf
sudo chown root:root -R /etc/wireguard && sudo chmod 600 -R /etc/wireguard
```

Okay, you should have a basic VPN that can function now:

```bash
wg-quick up wg0
curl https://am.i.mullvad.net/connected
# => You are connected to Mullvad (server ch2-wireguard). Your IP address is 81.17.24.203
```

Alright, you're up and running! The only problem is that this covers the whole machine's network, not just one application. This is where namespaces come in. Linux has the ability to containerize network stacks into something called namespaces. This is a fundamental building block of Docker containers. We will be building one of these manually.

I may dockerize my server at a later time, but let's stick low level for the time being.

```bash
# Take down our previous VPN
wg-quick down wg0

# Comment out the lines starting with Address= and DNS=.
# However, remember or recordt these values because you will need them!
sudo nano /etc/wireguard/wg0.conf

touch mullvad-start mullvad-stop
chmod +x mullvad-start mullvad-stop
```

We're going to create a systemd service that will create our namespace at startup, and close it down.

### mullvad-start

Add this script below into the `mullvad-start` file. Make sure to replace the IP addresses with the ones from the commented out `Address` line from the wireguard config.

```bash
#!/usr/bin/env bash

# Create a wireguard namespace and move the wg0 interface into it
ip netns add wireguard
ip link add wg0 type wireguard
wg setconf wg0 /etc/wireguard/wg0.conf
ip link set wg0 netns wireguard
ip netns exec wireguard ip addr add 10.66.247.57/32 dev wg0
ip netns exec wireguard ip addr add fc00:bbbb:bbbb:bb01::3:f738/128 dev wg0
ip netns exec wireguard ip link set wg0 up
ip netns exec wireguard ip route add default dev wg0

# Create a tunnel between the wireguard namespace and the public namespace
ip link add transclient type veth peer name transserver
ip link set transserver netns wireguard
ip addr add 10.10.10.20/24 dev transclient
ip netns exec wireguard ip addr add 10.10.10.10/24 dev transserver
```

### mullvad-stop

Add this text to the `mullvad-stop` script.

```bash
#!/usr/bin/env bash

ip netns delete wireguard
ip link delete transclient
```

Install the executables.

```bash
sudo install mullvad-start /usr/local/bin/mullvad-start
sudo install mullvad-stop /usr/local/bin/mullvad-stop
```

Now we need to setup DNS.

```bash
sudo mkdir -p /etc/netns/wireguard
sudo touch /etc/netns/wireguard/resolv.conf
sudo chown root:root -R /etc/netns && sudo chmod 600 -R /etc/netns
sudo nano /etc/netns/wireguard/resolv.conf
```

Add this line to `resolv.conf`: `nameserver x.x.x.x`

Except, of course, replace the IP with the DNS from `wg0.conf` file, which you previously commented you.

You should now be able to connect to the VPN inside a container.

```bash
sudo mullvad-start
sudo ip netns exec wireguard curl https://am.i.mullvad.net/connected
# => You are connected to Mullvad (server wg0). Your IP address is x.x.x.x
sudo mullvad-stop
```

Let's setup a systemd service: `sudo nano /etc/systemd/system/mullvad.service`

```ini
[Unit]
Description=Start Wireguard VPN in separate namespace
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/mullvad-start
ExecStop=/usr/local/bin/mullvad-stop
RemainAfterExit=true

[Install]
WantedBy=multi-user.target
```

Enable it and modify the `transmission-daemon` service:

```bash
sudo systemctl enable mullvad
sudo systemctl edit transmission-daemon
```

Add the following lines and save:

```
[Service]
NetworkNamespacePath=/run/netns/wireguard
User=YOUR-USERNAME
```

Now let's start the services: `sudo systemctl restart mullvad transmission-daemon`

Last, but not least, add the following lines to the `stream {}` block in your `/etc/nginx/nginx.conf` config (or create the block if you don't have it):

```
stream {
	upstream transmission {
		server 10.10.10.10:9091;
	}

	server {
		listen 9091;
		proxy_pass transmission;
	}
}
```

Finally, you should be able to connect to your transmission web UI on another machine on your
network like this: `http://your-machine:9091`

If you get a message about rpc whitelists, stop the transmission-deamon service, then edit `~/.config/transmission-daemon/settings.json` and set the `rpc-whitelist` config options to `false` to disable rpc-whitelist. Make sure to set the `rpc-user` and `rpc-password` attribute, and change `rpc-authentication-required` to `true`.


Links:

https://volatilesystems.org/wireguard-in-a-separate-linux-network-namespace.html
https://askubuntu.com/questions/659267/how-do-i-override-or-configure-systemd-services
https://www.redhat.com/sysadmin/net-namespaces
