# Vad

An alternative experimental command line interface (CLI) for Mullvad that is based on network namespaces and supports up to ten hops.
It is based on the script <https://www.wireguard.com/netns>.
Even if ten hops are supported, only three may be useful in terms of performance and privacy.

## Assumptions

1. Only works under Linux (requires network namespaces);
1. You have a Mullvad account;
1. You only want to use Wireguard servers (not OpenVPN);
1. You don't have a network interface with the name `mullvad0`;
1. You don't have wireguard configuration files with the names `/etc/wireguard/mullvad[0-9]`;
1. You don't have other network namespaces with the name `physical` or `mullvad[1-9]`;
1. The `resolv.conf` is shared between root and physical namespace.
1. Uses `wpa_supplicant` to configure wlan devices (if you use e.g. NetworkManager you need to duplicate the configuration); and;
1. You don't want to use Socks Proxies for Multihop.

## Dependencies

1. `sudo`,
1. `kill`, `killall`
1. `wg`,
1. `ip`,
1. `iw` (for wifi),
1. `wpa_supplicant` (for wifi),
1. `dhcpcd`,
1. `sysctl`,
1. `python-yaml`,
1. `python-requests`,
1. `python-prettytable` (for info and list command),
1. `resolvconf`,
1. `pass` (optionally).

## Install Dependencies

Arch Linux based:

```sh
$ pacman -Syu --needed python-requests python-yaml python-prettytable python-numpy sudo iw wpa_supplicant dhcpcd openresolv wireguard-tools
```

Debian based:

```sh
$ apt install python3-requests python3-yaml python3-prettytable python3-numpy sudo psmisc wireguard-tools iproute2 iw wpasupplicant dhcpcd5 procps
```

## Untested

There could be problems with other configured wireguard/vpn interfaces.

## Example

First use:

```sh
$ vad init   # Will ask for your account number and saves it
$ vad up
```

Show status information:

```sh
$ vad status
```

The up command can be called multiple times:

```sh
$ vad up de    # Build one hop tunnel to a server in Germany
$ vad status
$ vad up pl    # Update one hop tunnel to a server in Poland
$ vad status
$ vad up se    # Update one hop tunnel to a server in Sweden
$ vad status
```

Build a 2 hop (multihop) tunnel:

```sh
$ vad up de pl   # 2 hops: Multihop(de, pl)
$ vad status
```

Build a 3 hop tunnel:

```sh
$ vad init -a       # We need one more device for the first tunnel
$ vad up de pl se   # 3 hops: Tunnel(de) -> Multihop(pl, se)
$ vad status
```

Update server list:

```sh
$ vad update
```

Show account information:

```sh
$ vad info
```

Rotate Wireguard keys for your first configured device:

```sh
$ vad down
$ vad delete 0
$ vad init
($ vad rotate)
```

Rotate Wireguard keys for your first configured device while the vpn is active:

```sh
$ vad up         # Remembers the configuration from `vad up de pl se` and builds a 3 hop tunnel
$ vad delete 0
$ vad init
($ vad rotate)
$ vad up
$ vad status
```

Reset:

```sh
$ vad down
$ vad delete 0   # Repeat for all devices
($ vad reset)
```

## TODOs

* [ ] Add the possibility to use a specific configuration name besides "default";
* [ ] Add `vad reset` command
* [ ] Add documentation for PostUp/PreUp/PostDown/PostUp
* [ ] Always pick the device with the most number of ports as exit
* [ ] Add `--exit-device` to up command (useful if specific ports are mapped to this device)
* [ ] Use "interface" for linux network interfaces and use "device" for a mullvad device
* [ ] Add `vad service`, which automatically starts the vpn on system startup, creates the physical namespace, move new network devices into physical namespace and rotates wireguard keys every 4 days (same as the mullvad app).
* [ ] Add commands to easily manage port forwarding (`iptables -t nat`): request and forward to local port (automatically add port to exit server if possible).
  ```sh
  $ vad port 22      # map one port from the exit server to the local port 22
  $ vad port 22 443  # map two ports of the exit server to the local ports 22 and 443
  ```
  If this device has no port on this exit server: ask the user which port to delete and add it to the exit.
  If no ports are available on the account anymore, nothing we can do about it, just ask the user to delete one port on another computer.
  It would be possible delete a port from a device not mapped to this computer, but it depends on the use case.
  For now we won't delete that port automatically.
  Add a `--volotile` flag to this command to automatically delete this port on down or partial down.
  Ports mapping may not survive a up/down or partial up/down, because the exit can change!
  If the port mappings already exist this command is a no-op.
* [ ] Change API from wwww to mullvad API (makes it possible to update wg keys, so devices don't need to be deleted and recreated).
* [ ] Add device_id and device_name to configuration file
* [ ] Add support for private/external wireguard servers.
  These servers are called "devices".
  These devices would have four additional values: ipv4, ipv6, public_key and hostname.
  Hostname does not need to be a real hostname.
  Basically these devices have only one endpoint.
  This devices could be selected via the hostname in the up command, e.g.
  ```
  $ vad up de dysnomia de
  ```
  The hostname search order needs to change to private-external-devices and then mullvad servers.
  This makes the configuration step more complicated, is this worth it?

## Ideas

* Use anonymous namespaces (except for physical); only optionally name the namespaces to allow easy access with `ip netns exec <namespace>`.
* Add `--pick-different-as` and remove `--uniform-by-country`.
  This flag will pick a different country and provider pair for each hop after user provided filter criteria.
  It will hopefully prevent picking the same (virtual/overlay) autonomous system (AS) for each hop.
  We use the term AS a bit loosely here.
  We basically want to prevent that two hops are under the control of one entity.
  This is not that useful at the moment because Mullvad is the only supported provider and is its own overlay AS.
* A good name for this script may be `vad` which is easy to type and the suffix of Mullvad.
  In the future we may support other providers.
* It would be possible to add an namespace between the first hop and the root namespace.
  If the exit stays static an partial down and up would not break TCP connections.
  It can just not send data for a short amount of time.
  This idea does not seem to be useful at the moment(?).
  What could be the use case?

## Related projects

* <https://github.com/mullvad/mullvadvpn-app>
* <https://github.com/chutz/mullvad-netns>
* <https://github.com/DanielG/dxld-mullvad>
* <https://github.com/jamesmcm/vopono>
* <https://github.com/chrisbouchard/namespaced-wireguard-vpn>
* <https://github.com/nurupo/pia-wg-netns-vpn>
* <https://github.com/amonakov/vpn-netns>
* <https://github.com/pekman/openvpn-netns>
