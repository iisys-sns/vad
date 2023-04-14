# Vad

An alternative experimental command line interface (CLI) for Mullvad that is based on network namespaces and supports up to ten hops.
It aims to be very user friendly.
It is based on [this](https://www.wireguard.com/netns#sample-script) script.
Even if ten hops are supported, only three may be useful in terms of performance and privacy.
Normally, most VPN users will use only one hop.
If you use more than one hop, you may be more easily identified by a passive external attacker watching incoming and outgoing traffic of an intermediate hop, due to traffic correlation.

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

## Example

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
$ vad init -a       # We need one more device for the first tunnel
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

Rotate WireGuard keys for all mapped devices:

```sh
$ vad down
$ vad info
$ vad rotate
$ vad info
$ vad
```

Rotate WireGuard keys for all mapped devices while the VPN is active:

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
$ # Add the following to your `/etc/mullavd/config.yaml`:
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
$ # Add the following to your `/etc/mullavd/config.yaml`:
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

Portforwarding and game servers (TODO Does not work at the moment but gives a preview how it could work):

```sh
$ vad port 8888           # Forwards a random port (e.g. 55055) from your current exit to the local port 8888.
<exit-ip>:55055 -> 8888   # Normally you can not access this port with your exit ip address, but it will add nat rules with `iptables` so you can.
                          # One port is allocated in your account, for your current exit device. This port 55055 will stay the same as long as it is not deleted.

$ vad info
$ vad down        # Will delete added nat rules.
$ vad up          # Portforwardings do not survive a down and up, because the exit server could change.
                  # But the port is still allocated in your account.

$ vad list <country>      # Normally you want a game server close to the people who will use it.
                          # And it should have a semi static ip address and a static port.
                          # The ip address from your ISP will normally change daily.
                          # This is a good alternative if you just want to start a server temporarly or from time to time
                          # and do not want to deal with a changing address and ports.
                          # Choice one hostname from the list. In our experience the ip address does not change (that often).
$ vad up <hostname>                   # Use the hostname.
$ vad port <game-server-port>         # This will allocate a differnt port from your account (e.g. 60606),
<hostname-exit-ip>:60606 -> <game-server-port>
                                      # if you do not have ports left it will ask you which port you want to delete from your mapped devices.
                                      # Now you can edit the configuration and add `*_pre` and `*_post` commands so everythings starts automatically.
                                      # At least you need to add `vad port <game-server-port>` in `post_up`.
$ vad down                            # Not running
# vad up                              # Running game server. Have fun :)
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

* [ ] Terminology is a bit confusing at the moment, e.g. we use "device" for linux interfaces and Mullvad devices. (Rename "devices" to "peers")
* [ ] Add command `vad add` instead of `vad init -a`
* [ ] Support adding external devices with `vad add`
* [ ] Remove `vad dev`. Replace with `setns()` and `mount`.
* [ ] Add `--static-exit` to up command. It will remember the exit after an up and use until it down.
* [ ] Integration testing with Vagrant
* [ ] Add some documentation comments
* [ ] Implement a configuration class and api request class
* [ ] Test if dependencies are installed while launching
* [ ] Fix double configuration due to `wpa_supplicant`
* [ ] Always pick the device with the most number of ports as exit where the city code matches
* [ ] Add `--static-exit-peer` to up command (useful if specific ports are mapped to this device)
* [ ] Add commands to easily manage port forwarding (`iptables -t nat`): request and forward to local port (automatically add port to exit server if possible).
  ```sh
  $ vad port 22      # map one port from the exit server to the local port 22
  $ vad port 22 443  # map two ports of the exit server to the local ports 22 and 443
  ```
  If this device has no port on this exit server: ask the user which port to delete and add it to the exit.
  If no ports are available on the account anymore, nothing we can do about it, just ask the user to delete one port on another computer.
  It would be possible delete a port from a device not mapped to this computer, but it depends on the use case.
  For now we won't delete that port automatically.
  Add a `--volatile` flag to this command to automatically delete this port on down or partial down.
  Ports mapping may not survive a up/down or partial up/down, because the exit can change!
  If the port mappings already exist this command is a no-op.
* [ ] Use type hinting in conjunction with `mypy`.
  Instead of making python more statically typed it is a better idea to reimplement it in a statically typed language.

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
