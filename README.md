# Vad

Vad is our experimental command line interface (CLI) for OnionVPN, an onion routing-based VPN tunnel that provides better bulk transfer performance than Tor and offers additional security features over a VPN:
First, intermediate VPN nodes see only encrypted traffic; second, protection against AS-level attackers with a new path selection algorithm; and third, onion services with a novel cryptographic NAT traversal algorithm using the Noise protocol framework.

See our paper [OnionVPN: Onion Routing-Based VPN-Tunnels with Onion Services](https://florian.adamsky.it/research/publications/2024/onion-vpn.pdf) for a detailed description
of the approach and results of this tool. Please cite it as following:

```
@inproceedings{pahl:onionvpn:2024,
  title     = {{OnionVPN: Onion Routing-Based VPN-Tunnels with Onion Services}},
  author    = {Pahl, Sebastian; Kaiser, Daniel; Engel, Thomas; Adamsky, Florian},
  booktitle = {Proceedings of the 23th Workshop on Privacy in the Electronic Society (WPES)},
  date      = {2024},
}
```

The script is an alternative CLI for Mullvad that is based on network namespaces, supports up to ten hops and does not need a daemon.
NordVPN, ProtonVPN and Surfshark are only rudimentary supported.
It aims to be very user friendly.
It is based on [this](https://www.wireguard.com/netns#sample-script) script.
It also supports features that make Onion Services possible, even without port-forwarding, but these are **HIGHLY EXPERIMENTAL!**
Currently, it has some rough edges.

Other VPN providers or self-hosted VPNs are not directly supported, but you can integrate them.
You will need two things for this:

1. You need a key pair, an IP address and optionally the provider's DNS server (you can also use e.g. `8.8.8.8`, but that is not recommended).
   The public key must be uploaded to the provider.
   How to do this and to get all of this data is provider dependent.
   If you have it, you can add it via `vad --static DNS IPV4 IPV6 PRIVATE_KEY_FILENAME`.
1. A WireGuard relay/server list from the provider, in which each relay has at least one IP address and a public key.
   Currently, you can get a list for NordVPN, ProtonVPN (partial), and Surfshark (partial) with `vad update` (after you added one peer).
   If you want to create custom list or a list another VPN provider, look at `/etc/vad/<provider>.json`.

With network namespaces all programs from users, except programs that run with root privileges, are forced to use the VPN interface to connect to the Internet, without complex `iptable` rules.
An so-called "kill switch" is already integrated.
After an `vad up`, if the VPN does not work anymore, no traffic will go out of the normal interfaces.
The hop configuration can also be rebuild without traffic leaks.
The physical devices will stay inaccessible until an `vad down`.

## Assumptions

1. Only works under Linux (requires network namespaces);
1. You have a Mullvad account;
1. You only want to use WireGuard relays (not e.g. OpenVPN);
1. You don't have a network interface with the name `vad*` in your root namespace;
1. You don't have other network namespaces with the name `physical` or `vad-*`; and;
1. You don't want to use Socks Proxies for Multihop.

## Dependencies

1. `ip`
1. `iw` (for wifi)
1. `kill`
1. `ln`
1. `mkdir`
1. `pass` (optionally)
1. `ping`
1. `sudo`
1. `sysctl`
1. `systemctl`
1. `udevadm`
1. `wg`
1. `python-3.10`
1. `python-cryptography`
1. `python-noiseprotocol`
1. `python-numpy`
1. `python-prettytable`
1. `python-pyasn`
1. `python-pycountry`
1. `python-requests`
1. `python-termcolor`
1. `python-yaml`

## Install Dependencies

Arch Linux based:

```
$ pacman -Syu --noconfirm python-cryptography python-noiseprotocol python-numpy python-prettytable python-pyasn python-pycountry python-requests python-termcolor python-yaml iw sudo wireguard-tools
```

Debian based (ubuntu 22.04, 24.04; bookworm):

```
$ apt install -y -q python3-pip python3-cryptography python3-numpy python3-prettytable python3-pyasn python3-pycountry python3-requests python3-termcolor python3-yaml sudo psmisc wireguard-tools iproute2 iw wpasupplicant dhcpcd5 procps
$ pip install --break-system-packages noiseprotocol
$ pip install noiseprotocol # 22.04
```

## Untested

There could be problems with other configured WireGuard/VPN interfaces.

## Features by Example

If you have any problems, use `vad down` it will rollback all changes from `vad up`.

First use:

```sh
$ vad init   # Will ask for your account number and saves it
$ vad up
```

If you do not have a configuration file under `/etc/vad/1-physical.yaml`, `vad init` will create a default configuration for you.
The name `1-physical` refers to the source namespace `physical` and target namespace `1` (which is the root namespace, where normally your network interfaces reside).
At the moment it assumes that you are using NetworkManager under systemd, in the future it might automatically find a suitable configuration for you.
The configuration will look like this:

```yaml
post_down:
- systemctl revert wpa_supplicant
- systemctl revert NetworkManager
- systemctl start NetworkManager
post_up:
- mkdir -p /etc/systemd/system/wpa_supplicant.service.d
- printf '[Service]\nNetworkNamespacePath=/var/run/netns/physical\nBindPaths=/etc/netns/physical/resolv.conf:/etc/resolv.conf' > /etc/systemd/system/wpa_supplicant.service.d/override.conf
- mkdir -p /etc/systemd/system/NetworkManager.service.d
- printf '[Service]\nNetworkNamespacePath=/var/run/netns/physical\nBindPaths=/etc/netns/physical/resolv.conf:/etc/resolv.conf' > /etc/systemd/system/NetworkManager.service.d/override.conf
- systemctl daemon-reload
- systemctl start NetworkManager
pre_down:
- systemctl stop NetworkManager
- systemctl stop wpa_supplicant
pre_up:
- systemctl stop NetworkManager
- systemctl stop wpa_supplicant
```

During `vad up`, a physical namespace is created and all physical interfaces are moved there.
NetworkManager is also moved to the physical namespace.
After `vad up` you can manage your interfaces with NetworkManager as usual.

Show status information:

```sh
$ vad show
$ vad show --wireguard-interfaces
$ vad
```

The up command can be called multiple times:

```sh
$ vad up de    # Build one hop tunnel to a relay/server in Germany
$ vad show
$ vad up pl    # Update one hop tunnel to a relay/server in Poland
$ vad
$ vad up se    # Update one hop tunnel to a relay/server in Sweden
$ vad
```

Build a 2 hop (multihop) tunnel:

```sh
$ vad up de pl   # 2 hops: Multihop(de, pl)
$ vad
```

Build a 3 hop tunnel:

```sh
$ vad add           # We need one more peer for the first tunnel
$ vad up de pl se   # 3 hops: Tunnel(de) -> Multihop(pl, se)
$ vad
```

Build a 3 hop circuit with the default path selection algorithm:

```sh
$ vad up default   # You need three different providers!
                   # Each hop will have a different provider.
                   # The first and last hop will have a different AS.
                   # Currently, it does not implement a full AS path inference algorithm!
$ vad
```

Update relay list:

```sh
$ vad update
```

Show account information:

```sh
$ vad info
```

Rotate WireGuard keys for all mapped peers:

```sh
$ vad down
$ vad info
$ vad rotate
$ vad info
$ vad
```

Rotate WireGuard keys for all mapped peers while the VPN is active:

```sh
$ vad up         # Remembers the configuration from `vad up de pl se` and builds a 3 hop tunnel
$ vad info
$ vad rotate     # If the VPN was active, rotate will automatically call `vad up` to use the new keys
$ vad info
$ vad
```

Move your sshd into the physical namespace on `vad up`:

```sh
$ vad down   # `post_up`, `post_down`, `pre_up` and `pre_down` will not be called for a partial down and up
$ # Add the following to your `/etc/vad/1-physical.yaml`:
[...]
post_up:
- mkdir -p /etc/systemd/system/sshd.service.d
- printf "[Service]\nNetworkNamespacePath=/var/run/netns/physical" > /etc/systemd/system/sshd.service.d/override.conf
- systemctl daemon-reload
- systemctl restart sshd
post_down:
- systemctl revert sshd
- systemctl restart sshd
[...]
$ vad up
```

You want to use other `*_up` and/or `*_down` commands for work?
Copy your current configuration `/etc/vad/1-physical.yaml` to `/etc/vad/work.yaml`.

```sh
$ vad down                                     # Necessary, otherwise the commands will not be executed, because the circuit already exists!
$ vad up -c /etc/vad/work.yaml --dns atmpg eu  # Connect to a random relay in the European Union. See `vad up --help` for `--dns` flags.
$ vad up                                       # Connect to another random relay in the European Union with the same nameserver.
```

Onion Service (**HIGHLY EXPERIMENTAL!**):

```sh
# Proxy:
$ vad proxy                             # Must be reachable from the Internet on UDP port 6666.
                                        # This proxy combines STUN, ICE and a signal channel into one service.

# Service:
$ vad up -t service_start -b physical dk
$ vad -v start -i <ip> -p <port> -k <public-key> python -m http.server 8000 --bind ::
                                        # Copy the start command from the output of `vad proxy.
                                        # It will start a webserver in the current directory.
                                        # The service will be started in the network namespace `service` that is linked to `service_start`.
                                        # It will use the circuit that was created by `vad up`.
                                        # Outputs an URL where the service is reachable.

# Client:
$ vad up -t client_start -b physical se
$ vad -v connect <URL>                  # Copy the connect command from the output of `vad start`.
                                        # It will connect to the service and start chromium (http://[fc00::1]:8000) inside of the `client` namespace.
                                        # It will use the circuit that was created by `vad up`.
```

Reset:

```sh
$ vad reset
$ # Corresponds to the following commands:
$ # vad delete --all
$ # vad uninstall
$ # vad down
$ # vad clear # delete account-related information from the state file
```

## TODOs

* [ ] Find a way to make city codes unique
* [ ] Add type hinting for command line arguments.
* [ ] Integration testing with Vagrant
* [ ] Add some documentation comments
* [ ] Test if dependencies are installed while launching
* [ ] Replace `wg` with `python-iproute2` netlink interface
* [ ] Implement a AS path inference algorithm and automatically create circuits as necessary.

## Ideas

* Use anonymous namespaces (except for physical); only optionally name the namespaces to allow easy access with `ip netns exec <namespace>`.
  Is this even possible? When in between namespaces do not have any process running and only have an active WireGuard interface?

## Related projects

* <https://github.com/mullvad/mullvadvpn-app>
* <https://github.com/chutz/mullvad-netns>
* <https://github.com/DanielG/dxld-mullvad>
* <https://github.com/jamesmcm/vopono>
* <https://github.com/chrisbouchard/namespaced-wireguard-vpn>
* <https://github.com/nurupo/pia-wg-netns-vpn>
* <https://github.com/amonakov/vpn-netns>
* <https://github.com/pekman/openvpn-netns>
* <https://github.com/phvr/mullvad-wg/blob/master/mullvad>
* <https://mullvad.net/media/files/mullvad-wg.sh>
