# net

Super lightweight network manager.

## Usage

RTFM:

```
usage: net [<command>] [<args>] [--config=<config>] [--iface=<interface>]
           [--no-vpn] [--verbose] [-h] [--help]

Shorthands:
  If no positional arguments are given the command is "list".
  If one positional argument is given the command is "connect".

Commands:
  list:
    List available connections.
  scan [open]:
    Scan for access points.  With "open" only shop open networks.
  connect <connection> [<password>]:
    If <connection> is present in the configuration file then use that,
    otherwise connect to an access point with SSID <connection>, using the
    password <password> if specified.
  stop [<interface> [<interface> ...]]:
    Bring down the connection.  Brings down all interfaces if called with no
    arguments.
  dns [<dns> [<dns> ...]]:
    Change DNS server.  No argument or "dhcp" requests DNS servers via DHCP.
  mac [<mac>]:
    Change the MAC address of the interface specified by --iface.  If no address
    is given, one is chosen at random.
  vpn [<name>] [stop|tmpstop [delay]]:
    Connect to, or disconnect (temporarily [default = 5m] in case of tmpstop)
    from, VPN.  Lists available VPNs if called without argument.
  genkey:
    Generate a WireGuard key pair.
  show [<connection>]:
    Show configuration options.  If no connection is specified, all are show.
  help:
    You're reading it.

Options:
  --config=<config>:
    Select configurations file.  If <config> is "-" no configuration file is
    used.  Defaults to "~/.net/config".
  --iface=<interface>:
    Select networking interface.  Overridden by configuration file if specified.
    Defaults to first WiFi capable interface found.
  --no-vpn:
    Don't connect to a VPN.  Acts as if the connection configuration did not
    have a `vpn` field.
  --verbose:
    Print every executed command (and the result) to stdout.
```

The simplest usage is probably connecting to a wireless network:

```
$ net connect MyWirelessNetwork MySecretPassphrase
Connecting to MyWirelessNetwork
Sending DHCP request
Address acquired: 192.168.1.42
```

### Configuration

The file `~/.net.conf` holds a list of configured networks (in YAML).  An
example is included in `.net.conf.example`:

```
common: # Default settings
  mac: 00:??:??:??:??:?? # Make last 5 bytes random
  dns: 8.8.8.8, 8.8.4.4
  hostname: <name>s-MacBook-Pro # <name> is a table of generic names
  vpn: myvpn

ignored:
  interfaces:
  - br[0-9]+
  - docker[0-9]+ # Docker
  - tap[0-9]+
  - tun[0-9]+
  - vboxnet[0-9]+ # Virtualbox
  - veth.*
  - vmnet[0-9]+ # VMWare
  - wg.* # Probably WireGuard

vpn:
  myvpn:
    type: openvpn
    config: |
      client
      dev tun

      proto udp
      remote my-server-1 1194

      resolv-retry infinite
      nobind
      persist-key
      persist-tun

      ca ca.crt
      cert client.crt
      key client.key

      comp-lzo
      verb 3
  myvpn2:
    type: wireguard
    address: 10.0.0.1/8
    interface: wg0
    gateway: True
    config: |
      [Interface]
      ListenPort = 51820
      PrivateKey = QLa1x8ttCEl23cCIGpndDv9CIZ7Al7G7Kuj9yG0PIVk=

      [Peer]
      Endpoint = 198.51.100.1:51820
      PublicKey = cPybMYBdfrj0wp+FlvWoFfL2fI1kc7dhtKB+cqvNPCA=
      AllowedIPs = 0.0.0.0/0

office:
  vpn: myvpn
  routes:
    - 192.168.0.0/16 -> 192.168.1.1 # Local network

crappy-hotel-wifi:
  ssid: FreeWiFi
  # Pin access point address to avoid switching between a gazillion equally
  # crappy ones.  This tends to give a more reliable connection.
  ap-addr: 00:11:22:33:44:55
  vpn: myvpn # Connect to VPN when away from home

wired:
  dns: dhcp
  mac: default # The default is to pick a random Macbook Pro MAC address
  hostname: # Do not send a hostname

# Alias chooses different configuration for each host
static: static-$(hostname)

static-laptop:
  interface: eth0
  addr: 192.168.0.42/24
  gateway: 192.168.0.1
  routes:
    - default

static-desktop:
  interface: eth0
  addr: 192.168.0.43/24
  gateway: 192.168.0.1
  routes:
    - default

eduroam:
  ssid: eduroam
  wpa: |
    network={
      identity="YOUR-ID-HERE"
      password="YOUR-PASSPHRASE-HERE"
      key_mgmt=WPA-EAP
    }

my-home-network:
  ssid: SSID-HERE
  psk: PASSPHRASE-HERE
  vpn: # Do not connect to VPN when at home
```

Using this config file you can connect to `my-home-network` using the command:

```
$ net my-home-network
```

Notice that the section `common` does not define a network but rather settings
common to *all* network configurations (in this case using Google's DNS servers,
randomizing the MAC address and hostname [`<name>` will be replaced by an actual
name] and connecting to a VPN).

The `ignored` and `vpn` sections do not define networks either.  The `ignored`
section contains a list of interfaces to be ignored by e.g. `net stop` and the
`vpn` section contains the OpenVPN configurations for each VPN.

## Installing

Put `net` in your `PATH`.

## `Bash` completion

Put `_net_bash_completion` in your path and add the line

```
_net_bash_completion
```

to `~/.bash_completion`.

## Dependencies

| Dependency             | Debian package                       |
|------------------------|--------------------------------------|
| `/bin/ip`              | `iproute2`                           |
| `/sbin/ethtool`        | `ethtool`                            |
| `/sbin/iw`             | `iw`                                 |
| `/sbin/udhcpc`         | `udhcpc`                             |
| `/sbin/wpa_cli`        | `wpasupplicant`                      |
| `/sbin/wpa_supplicant` | `wpasupplicant`                      |
| `/usr/bin/chattr`      | `e2fsprogs`                          |
| `/usr/bin/expand`      | `coreutils`                          |
| `/usr/bin/cut`         | `coreutils`                          |
| `/usr/bin/pkill`       | `procps`                             |
| `/usr/sbin/openvpn`    | `openvpn`                            |
| `/usr/bin/wg`          | `https://www.wireguard.com/install/` |
| Python package `yaml`  | `python3-yaml` / PyPI `pyyaml`       |

It is also a good idea to uninstall resolvconf, as it overwrites the DNS settings.

**udhcpc** is part of the busybox suite, and can be installed, and used,
on non-Debian systems, which doesn't have a separate package, by:
* Install busybox with udhcpc compiled in
* Hard-link busybox as udhcpc
```
# ln /bin/busybox /usr/local/bin/udhcpc
```
* Install a client script somewhere and make it executable - [udhcpc script](default.script) - [udhcpc README](https://udhcp.busybox.net/README.udhcpc)
* Set _udhcpc-config:_ under _common:_
```
common:
  udhcpc-config: /etc/udhcpc/default.script
```

### Bootstrapping

`net` will run even without `udhcpc` or `python3-yaml`, with limited
functionality.  In the former case information acquired via DHCP may not be
reflected on the system, and in the latter the configuration file will not be
used.  However it may be enough to get online in order to fulfill the
dependencies (tested on Debian 11).

## Contributors

If you want to contribute, feel free to make a pull request on [Github](https://github.com/Pwnies/net), please read [CONTRIBUTING](CONTRIBUTING) and [the license](UNLICENSE) first.

This project was originally developed by [br0ns](https://github.com/br0ns) and [TethysSvensson](https://github.com/TethysSvensson).

For a complete list; check the log.
