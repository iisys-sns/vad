# Vad

An alternative experimental command line interface (CLI) for Mullvad that is based on network namespaces, supports up to ten hops and does not need a daemon.
It aims to be very user friendly.
It is based on [this](https://www.wireguard.com/netns#sample-script) script.
It is unclear how much privacy you can gain if use more than one hop.
But you will, for sure, lose performance.

With network namespaces all programs from users, except programs that run with root rights, are forced to use the VPN interface to connect to the Internet, without complex iptable rules.
An so-called "kill switch" is already integrated.
After an `vad up`, if the VPN does not work anymore, no traffic will go out of the normal interfaces.
The hop configuration can also be rebuild without traffic leaks.
The physical devices will stay inaccessible until an `vad down`.

## Assumptions

1. Only works under Linux (requires network namespaces);
1. You have a Mullvad account;
1. You only want to use WireGuard servers (not OpenVPN);
1. You don't have network interfaces with the names `vad[0-9]`;
1. You don't have WireGuard configuration files with the names `/etc/wireguard/vad[0-9]`;
1. You don't have other network namespaces with the name `physical` or `vad[1-9]`;
1. Uses `wpa_supplicant` to configure wlan devices (if you use e.g. NetworkManager you need to duplicate the configuration); and;
1. You don't want to use Socks Proxies for Multihop.

## Dependencies

1. `sudo`,
1. `kill`, `killall`
1. `timeout`,
1. `wg`,
1. `ip`,
1. `iw` (for wifi),
1. `wpa_supplicant` (for wifi),
1. `dhcpcd`,
1. `sysctl`,
1. `python-yaml`,
1. `python-dbus`,
1. `python-requests`,
1. `python-termcolor`,
1. `python-prettytable` (for up, info and list command),
1. `resolvconf`,
1. `pass` (optionally).

## Install Dependencies

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

If you are not connected via an ethernet cable and have only a wlan device add the following configuration file before use. This is a known issue.

```
/etc/wpa_supplicant/wpa_supplicant.conf
---------------------------------------
ctrl_interface=/run/wpa_supplicant
update_config=1

network={
	ssid="<YOUR SSID>"
	psk="<YOUR PSK>"
}
```

If you do not want do configure it manually and; have NetworkManager; currently connected to an wifi network; and; `wpa_supplicant` is used, you can execute this command.
Be aware that this is only a workaround.

```sh
vad up -i
```

If you have any problems, use `vad down` it will rollback all changes from `vad up`.

First use:

```sh
$ vad init   # Will ask for your account number and saves it
$ vad up
```

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
$ vad init -a       # We need one more peer for the first tunnel
$ vad up de pl se   # 3 hops: Tunnel(de) -> Multihop(pl, se)
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
- echo -n "[Service]\nNetworkNamespacePath=/var/run/netns/physical" > /etc/systemd/system/sshd.service.d/override.conf
- systemctl daemon-reload
- systemctl restart sshd
post_down:
- systemctl revert sshd
- systemctl restart sshd
[...]
$ vad up
```

Sometimes NetworkManager interferes with the `/etc/resolv.conf` configuration, to disable it on `vad up`:

```sh
$ vad down
$ # Add the following to your `/etc/vad/config.yaml`:
[...]
pre_up:
- systemctl stop NetworkManager
post_down:
- systemctl start NetworkManager
[...]
$ vad up
```

You want to use other `*_up` and/or `*_down` commands; and a differnt hop configuration for work?
Copy your current configuration `/etc/vad/config.yaml` to `/etc/vad/work.yaml`.

```sh
$ vad -c /etc/vad/work.yaml up --dns atmpg eu  # Connect to a random server in the European Union. See `vad up --help` for `--dns` flags.
$ vad -c /etc/vad/work.yaml up                 # Connect to another random server in the European Union with the same nameserver.
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

* [ ] Currently the configuration file under `/etc/vad/config.yaml` is not only read but also written to, to store state information.
  From the perspective of the user this is unexpected behaviour and it would be better to split configuration from state.
  The state information could live in `/var/run/vad/state`.
* [ ] Add command `vad add` instead of `vad init -a`
* [ ] Support adding external peers with `vad add`
* [ ] Add `--static-exit` to up command. It will remember the exit after an up and use until down.
* [ ] Integration testing with Vagrant
* [ ] Add some documentation comments
* [ ] Implement a configuration class and api request class
* [ ] Test if dependencies are installed while launching
* [ ] A workaround for doubling wlan configuratoin exists now, but it needs to be revisied in the future.
  It would be better to move NetworkManager directly into the physical namespace. From testing this is possible and it sees the devices,
  but will not manage them (keyword: strictly unmanged), for whatever reason.
* [ ] Use type hinting in conjunction with `mypy`.
  Instead of making python more statically typed it is a better idea to reimplement it in a statically typed language.
* [X] The implemented API is deprecated. There is [API documentation](https://api.mullvad.net/app/documentation/#/paths/~1v1~1submit-voucher/post),
  but some of the needed functionallity is deprecated. We need at least:
    * [X] Get relays ([here](https://github.com/mullvad/mullvadvpn-app/blob/main/mullvad-api/src/relay_list.rs))
    * [X] Login ([here](https://github.com/mullvad/mullvadvpn-app/blob/main/mullvad-api/src/access.rs))
    * [X] Get info about account ([here](https://github.com/mullvad/mullvadvpn-app/blob/main/mullvad-api/src/lib.rs))
    * [X] Add/Get/Replace Devices/WireGuard public keys ([here](https://github.com/mullvad/mullvadvpn-app/blob/main/mullvad-api/src/device.rs))
  The most up to date information can be found [here](https://github.com/mullvad/mullvadvpn-app/tree/main/mullvad-api).

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
