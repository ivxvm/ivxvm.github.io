+++
title = "Clearnet netns"
date = 2026-11-04
+++

When using VPNs I often find myself willing to have a simple escape hatch that I can use at any moment to route certain specific traffic through normal clearnet connection. Fortunately, Linux has especially nice tools for this. My personal favorite at the moment is network namespaces (netns).

Lets say my internet goes through `eno1` (ethernet) and VPN is `wg-CH-123` (wireguard connection managed by `wg-quick` service). I want website tabs in my browser to route through VPN by default, but I also want to be able to open specific new tabs going through clearnet, I also want to route torrents through clearnet.

Another important concern here is that even though for now it might only be a few apps that I want to tunnel outside of VPN, it is possible that in future I might want to add more. Working with per-app workarounds would not be the best solution, it's much more promising to find a way to put arbitrary apps under a single umbrella such that this split tunneling setup scales for any number of apps later. Fortunately, in Linux there is a thing called "network namespaces" which allows to achieve this.

There are many alternative setups with network namespaces. For example, it's possible to create network namespace that only knows about ethernet connection and isn't aware of wireguard at all. In this case traffic originating from network namespace will go through ethernet even in presence of active wireguard. This kind of netns would be the one linked to host network via something called macvlan. It's probably the most simple approach for basic usecases, but it has a downside: it complicates connections originating from host and going into namespace. 

In my setup in addition to torrent app I also want to run tinyproxy inside netns to use it in browser to route only specific browser tabs through clearnet. For usecases where connection from host into netns to specific port is needed, there exists another netns link type which is called veth - a bridge consisting of two veth links is created: one on the host side and another inside of ns. Through this bridge traffic is routed between netns and host network. There is an important caveat however: with macvlan, netns traffic completely ignores any routing rules existing in host network, while veth traffic will go through the whole host routing chain. 

When using wireguard with wg-quick, host network will get a bunch of routing rules that redirect all traffic through wg connection excluding packets marked with fwmark flag. Which means that with veth (as opposed to macvlan), traffic coming from netns will be affected as well. This is why in this scenario additional routing rule has to be created to automark all packets originating from netns with fwmark, so that they bypass wg connection.

# Example setup

Keep in mind that network namespaces and nftables don't survive reboot on their own, so if you want automatic setup, you need to execute this code from a service (easy to do with systemd, openrc, and probably any other init system).

```sh
NS="clearnet"
UPLINK="eno1" # network interface for clearnet traffic, ethernet in this case
CLEARNET_DNS="9.9.9.9" # you may want to use DNS of your ISP instead
SUBNET="10.200.0.0/24"

VETH_HOST="veth-cn-host"
VETH_NS="veth-cn-ns"

MARK="0xca6c"

# forwarded ports for apps like torrents, tor, etc
# you may add more and then add corresponding rules to nftables
FWD_PORT_1="9001"
FWD_PORT_2="32854"

### Create netns

ip netns add ${NS} || return 1
ip link add ${VETH_HOST} type veth peer name ${VETH_NS}
ip link set ${VETH_NS} netns ${NS}

ip addr add 10.200.0.1/24 dev ${VETH_HOST}
ip link set ${VETH_HOST} up
ip -n ${NS} addr add 10.200.0.2/24 dev ${VETH_NS}
ip -n ${NS} link set ${VETH_NS} up
ip -n ${NS} link set lo up
ip -n ${NS} route add default via 10.200.0.1

### Configure routing (nftables)

sysctl -qw net.ipv4.ip_forward=1

nft delete table ip clearnet 2>/dev/null
nft -f - <<EOF
table ip clearnet {
    chain pre {
        type filter hook prerouting priority -149; policy accept;
        iifname "${VETH_HOST}" meta mark set ${MARK}
    }
    chain portfwd {
        type nat hook prerouting priority dstnat; policy accept;        
        # forwarded ports, add new constants and rules when needed
        iifname "${UPLINK}" tcp dport ${FWD_PORT_1} dnat to 10.200.0.2:${FWD_PORT_1}
        iifname "${UPLINK}" tcp dport ${FWD_PORT_2} dnat to 10.200.0.2:${FWD_PORT_2}        
        # force dns requests go through clearnet dns
        iifname "${VETH_HOST}" ip daddr != ${CLEARNET_DNS} udp dport 53 counter dnat to ${CLEARNET_DNS}
        iifname "${VETH_HOST}" ip daddr != ${CLEARNET_DNS} tcp dport 53 counter dnat to ${CLEARNET_DNS}
    }
    chain post {
        type nat hook postrouting priority 100; policy accept;
        ip saddr ${SUBNET} oifname != "${VETH_HOST}" masquerade
    }
}
EOF

```

# Example usage

To run apps inside of clearnet netns:

```sh
sudo -E ip netns exec vps-ns runuser -u ivxvm -- flatpak run org.qbittorrent.qBittorrent
```

However this requires sudo which doesn't work well with usual userspace infrastructure like KDE autostarts and application launchers. It's possible to have a wrapper that works without sudo, but the setup is a bit more involved:

```sh
#!/bin/sh
[ -n "$SUDO_USER" ] && [ "$SUDO_USER" != root ] || { echo "run via sudo as a normal user" >&2; exit 1; }

HOME_DIR=$(getent passwd "$SUDO_USER" | cut -d: -f6)
exec ip netns exec clearnet runuser -u "$SUDO_USER" -- \
  env HOME="$HOME_DIR" USER="$SUDO_USER" LOGNAME="$SUDO_USER" "$@"

## ^ /usr/local/libexec/clearnet-exec
```

```sh
Defaults!/usr/local/libexec/clearnet-exec env_keep+="DISPLAY WAYLAND_DISPLAY XAUTHORITY XDG_RUNTIME_DIR DBUS_SESSION_BUS_ADDRESS"
ivxvm ALL=(root) NOPASSWD: /usr/local/libexec/clearnet-exec

## ^ /etc/sudoers.d/clearnet
```

```sh
#!/bin/sh
exec sudo /usr/local/libexec/clearnet-exec "$@"

## ^ ~/bin/clearnet
```

With setup above you can do things like this, and you can put it into application launchers and autostarts:

```sh
clearnet flatpak run org.qbittorrent.qBittorrent
```

But don't forget to put `~/bin` on your `PATH`.

Some apps like qBittorrent, are capable of creating KDE autostart entry for themselves, by default it's configured as following:

```sh
# Program:
sh
# Arguments:
-c 'date >> /tmp/qbt-autostart.log; flatpak run org.qbittorrent.qBittorrent >> /tmp/qbt-autostart.log 2>&1'
```

To make it launch inside of clearnet netns, it's enough to modify it slightly:

```sh
# Program:
sh
# Arguments:
-c 'sleep 5; date >> /tmp/qbt-autostart.log; /home/ivxvm/bin/clearnet flatpak run org.qbittorrent.qBittorrent >> /tmp/qbt-autostart.log 2>&1'
```

Extra sleep is added to give netns script/service some time to launch properly after boot.

# Clearnet browser tabs

I'll only cover Firefox here, but it's possible that similar technology exist for some other browsers.

Firefox has container tabs which make all cookies and other data isolated from other containers and from non-containerized tabs. With [Firefox Multi-Account Containers](https://addons.mozilla.org/en-US/firefox/addon/multi-account-containers/) plugin developed by Firefox itself, it's also possible to configure proxy per container. This is exactly how I route specific tabs through clearnet. Inside of clearnet netns I run tinyproxy with the following config:

```sh
Port 4321
Listen 10.200.0.2
Allow 10.200.0.1
Timeout 600
MaxClients 50
PidFile "/tmp/tinyproxy-clearnet.pid"
LogFile "/tmp/tinyproxy-clearnet.log"

## ^ ~/.config/tinyproxy-clearnet.conf
```

I start tinyproxy on boot by using KDE Plasma autostart entry, sleep is just a bit of safety net to make sure service or script setting up clearnet netns itself runs first, so that clearnet network namespace is already available:

```sh
# Program:
sh
# Arguments:
-c 'sleep 5; /home/ivxvm/bin/clearnet tinyproxy -c /home/ivxvm/.config/tinyproxy-clearnet.conf'
```

Then in Firefox in container proxy settings I set its IP and port as a proxy: 
```
http://10.200.0.2:4321
```

# Further reading

* [https://www.man7.org/linux/man-pages/man7/network_namespaces.7.html](https://www.man7.org/linux/man-pages/man7/network_namespaces.7.html)
* [https://developers.redhat.com/articles/2026/04/03/introduction-to-linux-interfaces-for-virtual-networking](https://developers.redhat.com/articles/2026/04/03/introduction-to-linux-interfaces-for-virtual-networking)
* [https://en.wikipedia.org/wiki/Nftables](https://en.wikipedia.org/wiki/Nftables)
* [https://support.mozilla.org/en-US/kb/how-use-firefox-containers](https://support.mozilla.org/en-US/kb/how-use-firefox-containers)
* [https://tinyproxy.github.io](https://tinyproxy.github.io)
