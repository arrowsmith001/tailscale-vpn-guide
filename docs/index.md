# How to use Tailscale + a commercial VPN on the same device

![alt text](banner.png)

If you are a Tailscale user, you probably enjoy the benefit of having all your devices and services networked from anywhere in the world. If you are concerned with online privacy and censorship, you may also choose to use commercial VPNs, such as Mullvad, Nord, etc.

The difficulty arises with using both simultaneously on a single device. They tend to conflict, killing your internet connectivity, and rendering one or both services useless. Is it even possible to get a working setup with both?

Thankfully it is, but with some caveats.

This article will outline:

- the requirements of the setup
- the general idea behind it, with advantages and disadvantages
- and a step-by-step guide on how to implement it yourself.

## Requirements

### Basic requirements

- You will, of course, require <u>**a working Tailscale setup**</u>.
- You will also require <u>**a subscription to a commercial VPN that supports wireguard**</u> - the major VPNs tend to do so, but check that yours does!

*This setup has been tested to work with both Mullvad and NordVPN. Of these two providers, obtaining the wireguard configuration file was easier with Mullvad.

### Additional requirement

Additionally, you will require <u>**a server/device**</u> which: 

- is always-on
- has reliable internet connectivity
- is a Tailscale node on your tailnet

This could be a VPS running in the cloud, an old laptop (or Raspberry Pi) sitting in a corner somewhere at home, or potentially even a NAS. By all means use an existing server if you already have one - we do not require much server real estate to do what we want - however, it must be connected to your tailnet.

In this guide, I use a Linux server running Ubuntu 24.04. While many features of this guide will be specific to Linux, I hope to illustrate the broad ideas so you can hopefully adapt it to your setup.

If you are able to fulfil this requirement, then read on to find out how you may implement this for yourself!

## Solution outline

### The general idea

Tailscale supports the use of a device as an "[exit node](https://tailscale.com/docs/features/exit-nodes)". This means that when a device on a tailnet is being used as an exit node, it is responsible for handling all the internet traffic of a client device.

In our setup, our server will be made available to be used as an exit node by any other device on our tailnet. Being an exit node, that server will, by definition, be handling that device's internet traffic.

And crucially, with the addition of some network rules, we can also make that server pass through any tailscale traffic it receives straight through to the commercial VPN that sends it out to the wider internet.

This fulfils the requirements of the setup. On any given device, you will fundamentally be using Tailscale as your VPN. But by using your server as an exit node, you will implicitly be using your commercial VPN to handle your internet traffic.

### Some advantages & features of this setup

The main notable aspect of this setup, as per the initial motivation, is that it allows you to primarily use Tailscale as your VPN app, safe in the knowledge that your internet traffic is being routed through a commercial VPN in the background.

A neat feature of this is that you can switch off your device's use of the commercial VPN by simply turning off your exit node, or using a different exit node.

Another really cool aspect of this is that it gets around the device limit most VPN providers enforce. You only need one device with a commercial VPN connection - that being your server. Your other devices are only using that connection implicitly. Therefore, your one VPN connection is handling a whole tailnet's worth of devices - and even Tailscale's personal plan allows unlimited devices!

And finally, particularly on mobile, you can take advantage of app split tunneling in Tailscale if - for whatever reason - you wish to exclude certain apps from using the VPN. Some commercial VPN apps have this anyway, but with Tailscale, you at least don't lose this crucial feature.

### The main disadvantage of this setup

This is a Tailscale-centred setup. By using it, you generally forfeit any specific features offered natively by the apps offered by your commercial VPN provider.

This, unfortunately, includes convenient country-switching. We will be configuring the VPN on our own server using wireguard. Each configuration file specifies a particular VPN server in a particular country. This can be changed, of course, but requires you to log in to the server and mess around, which is usually less convenient than using the app.

So before committing to it, check that this setup suits your individual needs! And if so, read on to find out how to implement this setup yourself step-by-step...

## Step-by-step guide

To summarize, we will first be obtaining the wireguard configuration file from our commercial VPN provider. 

Then we will configure and execute a shell script which sets up the appropriate network rules to fulfil our needs.

Once we are convinced that everything is working as expected, we will automate the execution of that script on server startup using systemd. 

### Step 1: Install relevant packages on your server

- [Install wireguard](https://www.wireguard.com/install/) to enable use of the `wg-quick` command
- Verify that systemd is installed by running `systemctl --version`

> If you do not have systemd for whatever reason, consider switching your Linux distribution to Ubuntu or something similar.


### Step 2: Configure wireguard

#### Step 2-1: Obtain the configuration file

The method for obtaining the wireguard configuration file will differ depending on what VPN service you use. You're looking to generate a file that looks like [this](https://wiresock.net/documentation/wireguard/config.html#sample-configuration-for-a-server).

Below are links to all the documented ways that I know of to obtain this file, as of the time of writing this article. If your VPN service is _not_ one of those listed below, try searching the documentation for your service for "wireguard configuration", or try searching for it in a search engine.

> Note: you will need to fetch this file whenever you want to switch servers, including switching to a server based in another country.

- [Wireguard configuration for Mullvad](wireguard-config/mullvad.md)
- [Wireguard configuration for NordVPN](wireguard-config/nord.md) 

Save this file on your server under `/etc/wireguard/wg-vpn.conf`.

#### Step 2-2: Edit the file

Add `Table = off` under the `[Interface]` section.
```
[Interface]
...
Table = off
```
Like so:
```
[Interface]
PrivateKey = <server-private-key>
Address = 10.0.0.1/24
ListenPort = 51820
Table = off     # <--------------- THIS LINE

[Peer]
PublicKey = <client-public-key>
PresharedKey = <preshared-key>
AllowedIPs = 10.0.0.2/32
```
> _**This is an absolutely crucial step**, which if omitted will break your entire setup. There's no way around it - every time you obtain a new config file, you must make this small edit._

### Step 3: Save network config script

It is possible to automate all the network configuration with one helpful script! 

Open your favourite editor:

```
sudo nano /usr/local/bin/ts-exitnode-vpn.sh
```

Paste the following content, and save:
```
#!/bin/sh

# Copy this to `/usr/local/bin/ts-exitnode-vpn.sh`, then reload daemon and restart ts-exitnode-vpn.service to apply

# Public network interface & public IP address, automatically determined
PUB_IF=$(ip route get 1.1.1.1 | awk '{for(i=1;i<=NF;i++) if($i=="dev") print $(i+1)}')
PUB_IP=$(ip addr show dev "$PUB_IF" | awk '/inet / {print $2}' | cut -d/ -f1)          

SSH_PORT=22                            # SSH port - by default this should be 22. If it's different for you, change this!
TAILSCALE_IF="tailscale0"              # Tailscale network interface
TAILSCALE_SUBNET="100.64.0.0/10"       # Tailscale subnet

VPN_TABLE="vpn"                           # Custom routing table
VPN_IF="wg-vpn"                           # WireGuard interface
VPN_CONF="/etc/wireguard/$VPN_IF.conf"    # The location of your config file

# Automatically extract the VPN peer IP from the Endpoint line in the config
VPN_PEER=$(grep '^Endpoint' "$VPN_CONF" | \
           cut -d'=' -f2 | \
           cut -d':' -f1 | \
           tr -d ' ')

# ----------------- ENABLE IP FORWARDING -----------------
sysctl -w net.ipv4.ip_forward=1
sysctl -w net.ipv4.conf.all.rp_filter=0
sysctl -w net.ipv4.conf.default.rp_filter=0

# ----------------- START WIREGUARD -----------------
wg-quick up $VPN_IF

# Automatically determines the new public IP address of your server, assigned by the VPN service
VPN_ADDR=$(ip -4 addr show dev "$VPN_IF" | awk '/inet / {print $2}' | cut -d/ -f1)  

# ----------------- CREATE VPN ROUTING TABLE -----------------
TABLE_ID=200
grep -q " $VPN_TABLE" /etc/iproute2/rt_tables 2>/dev/null || \
echo "$TABLE_ID $VPN_TABLE" >> /etc/iproute2/rt_tables
ip route replace default dev "$VPN_IF" table "$VPN_TABLE"

# ----------------- PROTECT SSH -----------------
iptables -t mangle -A OUTPUT -p tcp --sport $SSH_PORT -j MARK --set-mark 1
ip rule add fwmark 1 lookup main priority 50

# ----------------- MARK EXIT-NODE TRAFFIC -----------------
# Any packet from Tailscale subnet entering tailscale0 is marked
iptables -t mangle -A PREROUTING -i $TAILSCALE_IF -s $TAILSCALE_SUBNET -j MARK --set-mark 2

# ----------------- POLICY ROUTING FOR EXIT-NODE -----------------
ip rule add fwmark 2 lookup $VPN_TABLE priority 100

# ----------------- FORWARDING RULES -----------------
# Allow Tailscale -> VPN
iptables -A FORWARD -i $TAILSCALE_IF -o $VPN_IF -j ACCEPT
# Allow VPN -> Tailscale return traffic
iptables -A FORWARD -i $VPN_IF -o $TAILSCALE_IF -m state --state ESTABLISHED,RELATED -j ACCEPT

# ----------------- NAT (MASQUERADE) -----------------
# Ensure exit-node traffic leaving VPN interface is NATed
iptables -t nat -A POSTROUTING -o $VPN_IF -s $TAILSCALE_SUBNET -j MASQUERADE

# ----------------- EXCLUDE VPN ENDPOINT -----------------
# Allow packets to VPN server to bypass routing
iptables -t mangle -A OUTPUT -d $VPN_PEER/32 -j MARK --set-mark 0
```

### Step 4: Test the script

Before we automate the script to execute at system boot, we should test it first.

First, advertise your server as an exit node for your tailnet. On the server, run:
```
sudo tailscale up --advertise-exit-node
```

Then, let's execute that script:
```
sudo sh -x /usr/local/bin/ts-exitnode-vpn.sh
```

Next, <u>**choose another device on your tailnet**</u> that will use your server as an exit node. Ideally, this device should be one where it is convenient to run terminal commands on.

Run the following command on your device to baseline the public IP address:
```
curl icanhazip.com
```

Next, <u>**set your server as the exit node**</u> for that device:
```
sudo tailscale set --exit-node=YOUR_SERVER_TAILNET_NAME
```

Finally, run that earlier command on your device one more time:

```
curl icanhazip.com
```

Your IP address should now be different to before, indicating that the VPN tunnel is working!

For extra confirmation, visit [https://whatismyipaddress.com/](https://whatismyipaddress.com/) in your browser to confirm that your IP location has changed to the country you expect.

> If the final `curl` command fails, or hangs indefinitely, something has gone wrong. Rebooting your server will erase the effects of the script, so you can troubleshoot and try again.

### Step 5: Automate script with systemd

```
sudo nano /etc/systemd/system/ts-exitnode-vpn.service
```
Paste this content:
```
[Unit]
Description=Tailscale Exit Node via VPN
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/ts-exitnode-vpn.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```
Next, reload the systemd daemon and enable the service:
```
sudo systemctl daemon-reload
sudo systemctl enable ts-exitnode-vpn.service
```
The script should now run whenever the server boots.