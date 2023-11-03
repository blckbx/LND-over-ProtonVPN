# LND-over-ProtonVPN

This proposal tries to create a sustainable environment for tunneling a hybrid LND node over ProtonVPN.
For this to work we need to get hold of some circumstances that come with running ProtonVPN (as of Nov 2023):

ProtonVPN for Linux comes with port-forwarding support for selected VPN servers (v1.0.3-2). By default ProtonVPN assigns forwarded-ports for 60 seconds if not kept alive. `natpmpc` provides us with an opportunity to keep the assigned port alive for a longer time period.

This means we need to prepare for two problematic scenarios:
1) Keep-alive script stops unexpectedly and we lose our assigned port
2) System restart is required which leads us to 1.

What are the consequences of these scenarios:
Rotated ports invalidates the current LND and network config (firewall setting) we run the node with (listening port and set external port). Therefore we need to restart LND and adjust firewall config with the new VPN config (IP/port) that has been assigned. To prevent having to modify `lnd.conf` every time new VPN data is assigned, we exclude necessary parameters from `lnd.conf` and run them as startup flags.

To manage this automatically, we create two shell scripts: 
- `protonportfwd.sh`: runs as cronjob every 50 secs to keepalive the assigned VPN port by fetching data from `natpmpc` and saving VPN IP/port to file
- `protonlnd.sh`: reading VPN IP/port from file, constructing startup parameters to run lnd with flags: `--listen=0.0.0.0:(vpnport)` and `---externalip=(vpnip):(vpnport)`
- Systemd service `lnd.service` requires new `ExecStart` command: `ExecStart=/usr/bin/sh /usr/local/bin/protonlnd.sh`


Now here are the shell scripts:

`protonportfwd.sh`
```sh
#!/bin/sh

# Keep-alive script for ProtonVPN's server port
# script has to run every < 60s to prevent port rotation

# Add the following to sudo crontab -e
# * * * * * sleep 50; /bin/sh /usr/local/bin/protonportfwd.sh 2&>1 | /usr/bin/logger -t protonvpn

# fetch vpn ip and port from natpmpc
VPNIP=$(/usr/bin/natpmpc -a 1 0 udp 60 -g 10.2.0.1 | grep "Public" | awk '{ print $5 }')
VPNPORT=$(/usr/bin/natpmpc -a 1 0 tcp 60 -g 10.2.0.1 | grep "Mapped" | awk '{ print $4 }')
echo "ProtonVPN public IP/Port: ${VPNIP}:${VPNPORT}"


# json file for current config
jsonConfig="/data/lnd/protonportfwd.json"


# read config and compare port numbers
# exit if ports match (= unchanged)
currentVPNPORT=$(/usr/bin/jq -r '.vpnport' $jsonConfig)
[ "$currentVPNPORT" = "$VPNPORT" ] && echo "no new vpn config found" && exit 1


# if there has been a new port assigned,
# save new config to json
echo "new vpn config found:"
/usr/bin/jq -n --arg vpnip "${VPNIP}" --arg vpnport "${VPNPORT}" "{vpnip: \"${VPNIP}\", vpnport: \"${VPNPORT}\"}" | /usr/bin/tee $jsonConfig


# ufw: open port
/usr/sbin/ufw allow $VPNPORT
```

`protonlnd.sh`
```sh
#!/bin/sh

# LND startup script called by systemd service: lnd.service
# reading vpn config from json file

# json config
jsonConfig="/data/lnd/protonportfwd.json"

# read vars
VPNIP=$(/usr/bin/jq -r '.vpnip' $jsonConfig)
VPNPORT=$(/usr/bin/jq -r '.vpnport' $jsonConfig)

# debug
#echo "${VPNIP}:${VPNPORT}"

# verify and construct lnd startup command
[ -z "$VPNIP" ] || [ -z "$VPNPORT" ] \
        && echo "VPNIP or VPNPORT empty" \
        || echo "/usr/local/bin/lnd --listen=0.0.0.0:$VPNPORT --externalip=$VPNIP:$VPNPORT"

```

Ressources:
1. https://protonvpn.com/support/linux-vpn-setup
2. https://protonvpn.com/support/port-forwarding-manual-setup/#linux
3. https://www.reddit.com/r/ProtonVPN/comments/10owypt/successful_port_forward_on_debian_wdietpi_using/?rdt=34775
