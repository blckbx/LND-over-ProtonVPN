# LND-over-ProtonVPN

### Preconditions
System Requirements:
- Bolt LND Node
- LND datadir: `/data/lnd`
- ProtonVPN v1.0.3-2 installed
- `sudo apt install natpmpc`
- Connected to ProtonVPN server capable of NAT-MMP (🔁)

⚠️ Note: This is not yet tested thoroughly!

This proposal tries to create a sustainable environment for tunneling a hybrid LND node over ProtonVPN.
For this to work we need to get hold of some circumstances that come with running ProtonVPN (as of Nov 2023):

ProtonVPN for Linux comes with port-forwarding support for selected VPN servers (v1.0.3-2). By default ProtonVPN assigns forwarded-ports for 60 seconds if not kept alive. `natpmpc` provides us with an opportunity to keep the assigned port alive for a longer time period.

### Current Issues
This means we need to prepare for two problematic scenarios:
1) Keep-alive script stops unexpectedly and therefore assigned port is being rotated
2) System restarts are inevitable which leads us to 1.

What are the consequences of these scenarios:
Rotated ports invalidates the current LND and network config (firewall setting) we run the node with (listening port and set external port). Therefore we need to restart LND and adjust firewall config with the new VPN config (IP/port) that has been assigned. To prevent having to modify `lnd.conf` every time new VPN data is assigned, we exclude necessary parameters from `lnd.conf` and run them as startup flags.

### Proposed Solution
To manage this automatically, we create two shell scripts: 
- `protonkeepalive.sh`: runs as cronjob every 50 secs to keepalive the assigned VPN port by fetching data from `natpmpc` and saving VPN IP/port to file: `proton.json`
- `protonlnd.sh`: reading VPN IP/port from file, constructing startup flags for `lnd`: `--listen=0.0.0.0:(vpnport) ---externalip=(vpnip):(vpnport)`
- Systemd service `lnd.service` requires new `ExecStart` command: `ExecStart=/usr/bin/sh /usr/local/bin/protonlnd.sh`

### Shell Scripts
Now here are the shell scripts:

`protonkeepalive.sh`
```sh
#!/bin/sh

# Keep-alive script for ProtonVPN's server port
# script has to run every < 60s to prevent port rotation

# Add the following to sudo crontab -e
# * * * * * sleep 50; /bin/sh /usr/local/bin/protonkeepalive.sh 2&>1 | /usr/bin/logger -t protonvpn

# fetch vpn ip and port from natpmpc
VPNIP=$(/usr/bin/natpmpc -a 1 0 udp 60 -g 10.2.0.1 | grep "Public" | awk '{ print $5 }')
VPNPORT=$(/usr/bin/natpmpc -a 1 0 tcp 60 -g 10.2.0.1 | grep "Mapped" | awk '{ print $4 }')
echo "ProtonVPN public IP/Port: ${VPNIP}:${VPNPORT}"

# json file for current config
# create if non-existent
jsonConfig="/data/lnd/proton.json"
[ ! -f $jsonConfig ] && /usr/bin/touch $jsonConfig

# read config and compare port numbers
# exit if ports match (= unchanged)
currentVPNPORT=$(/usr/bin/jq -r '.vpnport' $jsonConfig)
[ "$currentVPNPORT" = "$VPNPORT" ] && echo "VPN config still valid" && exit 0

# if there has been a new port assigned,
# save new config to json
echo "New VPN config found:"
/usr/bin/jq -n --arg vpnip "${VPNIP}" --arg vpnport "${VPNPORT}" "{vpnip: \"${VPNIP}\", vpnport: \"${VPNPORT}\"}" | /usr/bin/tee $jsonConfig

# ufw: open new port, rm old port
/usr/sbin/ufw allow $VPNPORT
/usr/sbin/ufw delete allow $currentVPNPORT

```

`protonlnd.sh`
```sh
#!/bin/sh

# LND startup script called by systemd service: lnd.service
# reading vpn config from json file

# json config
jsonConfig="/data/lnd/proton.json"

# read vars
VPNIP=$(/usr/bin/jq -r '.vpnip' $jsonConfig)
VPNPORT=$(/usr/bin/jq -r '.vpnport' $jsonConfig)

# verify and construct lnd startup command
[ -z "$VPNIP" ] || [ -z "$VPNPORT" ] \
        && echo "VPNIP or VPNPORT empty" \
        || /usr/local/bin/lnd --listen=0.0.0.0:$VPNPORT --externalip=$VPNIP:$VPNPORT

```

### Future Improvements:
- more safety checks for edge cases (empty config file, empty vars, ...)
- split-tunneling for real hybrid mode (Tor outside of the tunnel)
- process for SHTF at runtime

Ressources:
1. https://protonvpn.com/support/linux-vpn-setup
2. https://protonvpn.com/support/port-forwarding-manual-setup/#linux
3. https://www.reddit.com/r/ProtonVPN/comments/10owypt/successful_port_forward_on_debian_wdietpi_using/?rdt=34775
