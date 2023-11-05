# LND-over-ProtonVPN

### Preconditions
System Requirements:
- Bolt LND Node
- LND datadir: `/data/lnd`
- ProtonVPN v1.0.3-2 installed
- `sudo apt install natpmpc`
- Connected to ProtonVPN server capable of NAT-MMP (🔁)

⚠️ **Note: This is not tested thoroughly yet!**

This proposal tries to create a sustainable environment for tunneling a hybrid LND node over ProtonVPN.
For this to work we need to get hold of some circumstances that come with running ProtonVPN (as of Nov 2023):

ProtonVPN for Linux comes with port-forwarding support for selected VPN servers (v1.0.3-2). By default ProtonVPN assigns forwarded-ports for 60 seconds if not kept alive. `natpmpc` provides us with an opportunity to keep the assigned port alive for a longer time period.

### Current Issues
This means we need to prepare for two problematic scenarios:
1) Keep-alive script stops unexpectedly and therefore assigned port is being rotated
2) System restarts are inevitable which leads us to 1.

What are the consequences of these scenarios:
Rotated ports invalidates the current LND and network config (firewall setting) we run the node with (listening port and set external port). Therefore we need to restart LND and adjust firewall config with the new VPN config (IP/port). To prevent having to modify `lnd.conf` every time new VPN data is assigned, we exclude necessary parameters from `lnd.conf` and run them as startup flags in systemd service.

### Proposed Solution
To manage this automatically, we create a shell script and modify systemd service: 
- We put two shell scripts in `sudo mkdir /usr/local/bin/proton`
- `protonkeepalive.sh`: runs as cronjob every 50 secs to keepalive the assigned VPN port by fetching data from `natpmpc` and saving VPN IP/port to env file: `proton`
- `protonprecheck.sh`: runs some mandatory checks on proton config file before starting up lnd. If checks fail, lnd won't start up.
- `lnd.service`: reading VPN IP/port from environment file, constructing startup flags for `lnd --listen=0.0.0.0:(vpnport) ---externalip=(vpnip):(vpnport)`

### Shell Scripts
Now here are the shell scripts:

Setup:
```bash
$ sudo mkdir /usr/local/bin/proton
$ sudo touch /usr/local/bin/proton/protonkeepalive.sh
$ sudo touch /usr/local/bin/proton/protonprecheck.sh
$ sudo chmod +x /usr/local/bin/proton/protonprecheck.sh

$ sudo touch /data/lnd/proton

$ sudo crontab -e
# add line:
* * * * * /usr/bin/sleep 50; /bin/sh /usr/local/bin/proton/protonkeepalive.sh 2&>1 | /usr/bin/logger -t protonvpn
```

`protonkeepalive.sh`
```sh
#!/bin/sh

# Keep-alive script for ProtonVPN's server port
# script has to run every < 60s to prevent port rotation

# Add the following to sudo crontab -e
# * * * * * /usr/bin/sleep 50; /bin/sh /usr/local/bin/proton/protonkeepalive.sh 2&>1 | /usr/bin/logger -t protonvpn

# fetch vpn ip and port from natpmpc
VPNIP=$(/usr/bin/natpmpc -a 1 0 udp 60 -g 10.2.0.1 | grep "Public" | awk '{ print $5 }')
VPNPORT=$(/usr/bin/natpmpc -a 1 0 tcp 60 -g 10.2.0.1 | grep "Mapped" | awk '{ print $4 }')
/usr/bin/echo "ProtonVPN public IP/Port: ${VPNIP}:${VPNPORT}"

# env file for current config
# create file if non-existent
envConfig="/data/lnd/proton"
[ ! -f $envConfig ] && /usr/bin/touch $envConfig

# read config and compare port numbers
# exit if ports match (= unchanged)
currentVPNPORT=$(/usr/bin/grep "VPNPORT" $envConfig | /usr/bin/cut -d '=' -f2)
[ "$currentVPNPORT" = "$VPNPORT" ] && /usr/bin/echo "VPN config still valid" && exit 0

# if there has been a new port assigned,
# save new config to env file
/usr/bin/echo "New VPN config found:"
/usr/bin/echo -e "VPNIP=${VPNIP}\nVPNPORT=${VPNPORT}" | /usr/bin/tee $envConfig


# ufw: open new port, rm old port
/usr/sbin/ufw allow $VPNPORT
/usr/sbin/ufw delete allow $currentVPNPORT

```

`protonprecheck.sh`
```sh
#!/bin/sh

# Simple pre-check for empty env vars
# in /data/lnd/proton

# define env file
envConfig="/data/lnd/proton"

# exit on missing file
[ ! -f $envConfig ] && exit 1

# exit on missing key/value pairs
VPNIP=$(/usr/bin/grep "VPNIP" $envConfig | /usr/bin/cut -d '=' -f2)
VPNPORT=$(/usr/bin/grep "VPNPORT" $envConfig | /usr/bin/cut -d '=' -f2)

[ -z "$VPNIP" ] && exit 1
[ -z "$VPNPORT" ] && exit 1

/usr/bin/echo "proton config pre-check OK"
```

`lnd.service`
```
[Service]
EnvironmentFiles=/data/lnd/proton
ExecStartPre=/usr/local/bin/proton/protonprecheck.sh
ExecStart=/usr/local/bin/lnd --listen=0.0.0.0:${VPNPORT} --externalip=${VPNIP}:${VPNPORT}
```

### Future Improvements:
- split-tunneling for real hybrid mode (Tor outside of the tunnel)
- process for SHTF at runtime (connection loss, random port rotation)

Ressources:
1. https://protonvpn.com/support/linux-vpn-setup
2. https://protonvpn.com/support/port-forwarding-manual-setup/#linux
3. https://www.reddit.com/r/ProtonVPN/comments/10owypt/successful_port_forward_on_debian_wdietpi_using/?rdt=34775
