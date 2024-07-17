# Vad

An alternative experimental command line interface (CLI) for Mullvad that is based on network namespaces, supports up to ten hops and does not need a daemon.
It aims to be very user friendly.
It is based on [this](https://www.wireguard.com/netns#sample-script) script.
It also supports features that make Onion Services possible, even without port-forwarding, but these are **HIGHLY EXPERIMENTAL!**

Other VPN providers or self-hosted VPNs are not directly supported, but you can integrate them manually.
You will need two things for this:

1. A WireGuard server list from the provider, in which each server has at least one IP address and a public key.
   But it's a bit more complicated than that, because at the moment the structure of the server list has to match that of Mullvad.
   But if you have such a list, you can put it under `/etc/vad/<provider>.json`.
1. You need to generate a key pair and upload the public key to the provider to get access via their (web) interface.
   You also need to know your default IP address and the provider's DNS server. Then you can add the following
   to the `peers` in `/etc/vad/config.yaml`:

    ```yaml
    - dns: <dns address>
      ipv4: <ipv4 address or null>
      ipv6: <ipv6 address or null>
      private_key: <key>
      provider: <provider>
    ```

With network namespaces all programs from users, except programs that run with root privileges, are forced to use the VPN interface to connect to the Internet, without complex `iptable` rules.
An so-called "kill switch" is already integrated.
After an `vad up`, if the VPN does not work anymore, no traffic will go out of the normal interfaces.
The hop configuration can also be rebuild without traffic leaks.
The physical devices will stay inaccessible until an `vad down`.

## Assumptions

1. Only works under Linux (requires network namespaces);
1. You have a Mullvad account;
1. You only want to use WireGuard servers (not OpenVPN);
1. You don't have a network interface with the names `vad0` in your root namespace;
1. You don't have other network namespaces with the name `physical` or `vad*`; and;
1. You don't want to use Socks Proxies for Multihop.

## Dependencies (TODO)

1. `sudo`,
1. `kill`, `killall`
1. `timeout`,
1. `wg`,
1. `ip`,
1. `iw` (for wifi),
1. `sysctl`,
1. `python-yaml`,
1. `python-dbus`,
1. `python-requests`,
1. `python-termcolor`,
1. `python-prettytable` (for up, info and list command),
1. `resolvconf`,
1. `pass` (optionally).

## Install Dependencies (TODO)

Arch Linux based:

```
# pacman -Syu --needed python-requests python-termcolor python-yaml python-prettytable python-numpy sudo iw wpa_supplicant dhcpcd openresolv wireguard-tools
```

Debian based:

```
# apt install python3-requests python3-termcolor python3-yaml python3-prettytable python3-numpy sudo psmisc wireguard-tools iproute2 iw wpasupplicant dhcpcd5 procps
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

If you do not have a configuration file under `/etc/vad/config.yaml`, `vad init` will create a default configuration for you.
At the moment it assumes that you are using NetworkManager with systemd, in the future it may automatically find a suitable configuration for you.
The configuration will look like this:

```yaml
post_down:
- systemctl revert wpa_supplicant
- systemctl revert NetworkManager
- systemctl start NetworkManager
post_up:
- mkdir -p /etc/systemd/system/wpa_supplicant.service.d
- printf '[Service]\nNetworkNamespacePath=/var/run/netns/physical' > /etc/systemd/system/wpa_supplicant.service.d/override.conf
- mkdir -p /etc/systemd/system/NetworkManager.service.d
- printf '[Service]\nNetworkNamespacePath=/var/run/netns/physical' > /etc/systemd/system/NetworkManager.service.d/override.conf
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
$ vad
```

The up command can be called multiple times:

```sh
$ vad up de    # Build one hop tunnel to a server in Germany
$ vad show
$ vad up pl    # Update one hop tunnel to a server in Poland
$ vad
$ vad up se    # Update one hop tunnel to a server in Sweden
$ vad
```

Build a 2 hop (multihop) tunnel:

```sh
$ vad up de pl   # 2 hops: Multihop(de, pl)
$ vad
```

Build a 3 hop tunnel:

```sh
$ vad add           #  We need one more peer for the first tunnel
$ vad up de pl se   # 3 hops: Tunnel(de) -> Multihop(pl, se)
$ vad
```

Build a 3 hop circuit with the default path selection algorithm:

```sh
$ vad up default   # The first and last hop has not the same provider
$ vad
```

Update server list:

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
$ # Add the following to your `/etc/vad/config.yaml`:
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

You want to use other `*_up` and/or `*_down` commands; and a differnt hop configuration for work?
Copy your current configuration `/etc/vad/config.yaml` to `/etc/vad/work.yaml`.

```sh
$ vad -c /etc/vad/work.yaml up --dns atmpg eu  # Connect to a random server in the European Union. See `vad up --help` for `--dns` flags.
$ vad -c /etc/vad/work.yaml up                 # Connect to another random server in the European Union with the same nameserver.
```

Onion Service (**HIGHLY EXPERIMENTAL!**):

```sh
# Proxy:
$ vad proxy                             # Must be reachable from the Internet on UDP port 6666.

# Service:
$ vad start python -m http.server 8000  # Starts a webserver in the current directory.
                                        # The service will be started in the network namespace `service`.
                                        # Outputs an URL where the service is reachable, e.g. aaah6aaaahcn5vsceb5ntdbls2u3xl2dtqb4uywj2dbgfcjels5guvmovfffyodu.onion.vpn
                                        # The URL contains the address and port of the proxy, as well as the service port (8000).

# Client:
$ vad connect <URL>                     # Uses the URL from the service.
                                        # Afterwards the client can access the service at http://[fc00::1]:8000
```

Reset:

```sh
$ vad reset
$ # Corresponds to the following commands, but additonally deletes account-related information from the configuration file.
$ # vad delete --all
$ # vad uninstall
$ # vad down
```

## TODOs

* [ ] Add default configuration during `vad init` if no configuration exists.
* [ ] Currently the configuration file under `/etc/vad/config.yaml` is not only read but also written to, to store state information.
  From the perspective of the user this is unexpected behaviour and it would be better to split configuration from state.
  The state information could live in `/var/run/vad/state`.
* [ ] Support adding external peers with `vad add`
* [ ] Add `--static-exit` to up command. It will remember the exit after an up and use until down.
* [ ] Integration testing with Vagrant
* [ ] Add some documentation comments
* [ ] Implement a configuration class and api request class
* [ ] Test if dependencies are installed while launching
* [ ] Use type hinting in conjunction with `mypy`.
  Instead of making python more statically typed it is a better idea to reimplement it in a statically typed language.
* [ ] Replace `wg` with `python-iproute2` netlink interface

## Ideas

* Use anonymous namespaces (except for physical); only optionally name the namespaces to allow easy access with `ip netns exec <namespace>`.
* Add `--pick-different-as` and remove `--uniform-by-country`.
  This flag will pick a different country and provider pair for each hop after user provided filter criteria.
  It will hopefully prevent picking the same (virtual/overlay) autonomous system (AS) for each hop.
  We use the term AS a bit loosely here.
  We basically want to prevent that two hops are under the control of one entity.
  This is not that useful at the moment because Mullvad is the only supported provider and is its own overlay AS.
* It would be possible not to break TCP connections if two conditions are met.
  First, you need a static ip address for the exit hop, that is already possible, you can just set a hostname as last hop.
  Second, the WireGuard interface `mullvad0` in the root namespace must not be deleted with an partial down and up; and the ip address must remain the same.
  The latter will possibly work with `wg syncconf`.
* Only delete namespaces/interfaces and change the configuration with a partial down and up if necessary.

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
