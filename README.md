# Vad

An alternative experimental command line interface (CLI) for Mullvad that is based on network namespaces and supports up to ten hops.
It aims to be very user friendly.
It is based on [this](https://www.wireguard.com/netns) script.
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
1. You don't have a network interface with the name `mullvad0`;
1. You don't have WireGuard configuration files with the names `/etc/wireguard/mullvad[0-9]`;
1. You don't have other network namespaces with the name `physical` or `mullvad[1-9]`;
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
1. `python-termcolor`
1. `python-prettytable` (for info and list command),
1. `resolvconf`,
1. `pass` (optionally).

## Install Dependencies

Arch Linux based:

```sh
# pacman -Syu --needed python-requests python-termcolor python-yaml python-prettytable python-numpy sudo iw wpa_supplicant dhcpcd openresolv wireguard-tools
```

Debian based:

```sh
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
	mesh_fwding=1
}
```

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

Rotate WireGuard keys for all mapped devices:

```sh
$ vad down
$ vad info
$ vad rotate
$ vad info
$ vad status
```

Rotate WireGuard keys for all mapped devices while the VPN is active:

```sh
$ vad up         # Remembers the configuration from `vad up de pl se` and builds a 3 hop tunnel
$ vad info
$ vad rotate     # If the VPN was active, rotate will automatically call `vad up` to use the new keys
$ vad info
$ vad status
```

Move your sshd into the physical namespace on `vad up`:

```sh
$ vad down   # `post_up`, `post_down`, `pre_up` and `pre_down` will not be called for a partial down and up
$ # Add the following to your `/etc/mullavd/config.yaml`:
[...]
default:
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
default:
  pre_up:
  - systemctl stop NetworkManager
  post_down:
  - systemctl start NetworkManager
[...]
$ vad up
```

You want to use other `*_up` and/or `*_down` commands; and a differnt hop configuration for work?

```sh
$ vad -c work up --dns atmpg eu    # Connect to a random server in the European Union and no adult content (p) for work :) See `vad up --help` for `--dns` flags.
$ vad up                           # Connect to another random server in the European Union with the same nameserver.
$ vad down                         # The "work" profile will stay active until `vad down`.
$ vad up                           # Uses the "default" profile again.
```

Switch profile on the fly:

```sh
$ vad -c work up --force           # This will execute `pre_down` and `post_down` commands from "default", activates the "work" profile and executes `pre_up` and `post_up` commands from it.
$ vad -c default up                # If you call it without `--force`, it will warn you that it ignores `-c default` and executes an up in the "work" profile.
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
$ vad -c <game> up <hostname>         # Use the hostname.
$ vad port <game-server-port>         # This will allocate a differnt port from your account (e.g. 60606),
<hostname-exit-ip>:60606 -> <game-server-port>
                                      # if you do not have ports left it will ask you which port you want to delete from your mapped devices.
                                      # Now you can edit the configuration and add `*_pre` and `*_post` commands so everythings starts automatically.
                                      # At least you need to add `vad port <game-server-port>` in `post_up`.
$ vad down                            # Not running
# vad -c <game> up                    # Running game server. Have fun :)
```

Reset:

```sh
$ vad reset
$ # Corresponds to the following commands, but additonally deletes account-related information from the configuration file.
$ # vad delete 0 (repeated for every mapped device)
$ # vad service rm (TODO)
$ # vad down
```

## TODOs

* [ ] Rename `active_section` and `section` to `active_profile` and `profile`
* [ ] Add some documentation comments
* [ ] Test assumptions in `update_command` about unique country and city codes
* [ ] Use typing hinting in conjunction with `mypy`
* [ ] Implement a configuration class and api request class
* [ ] Test if dependencies are installed while launching
* [ ] Execute `vad down` if `vad up` fails on the critical path
* [ ] Fix double configuration of `wpa_supplicant`
* [ ] Always pick the device with the most number of ports as exit where the city code matches
* [ ] Add `--exit-device` to up command (useful if specific ports are mapped to this device)
* [ ] Terminology is a bit confusing at the moment, e.g. we use "device" for linux interfaces and Mullvad devices.
* [ ] Add `vad move/mv`, `vad service add` and `vad service rm`, which automatically starts the VPN on system startup, creates the physical namespace, move new network devices into physical namespace and rotates WireGuard keys every 4 days (same as the mullvad app).
  Look at [example](https://unix.stackexchange.com/questions/460028/automatically-move-physical-network-interfaces-to-namespace).
  Automatically move device to and create if not exists the physical namespace (Test with `udevadm test --action="add" <device>` and enable with `udevadm control --reload`):
  ```
  /etc/udev/rules.d/vad.rules
  ---------------------------
  SUBSYSTEM=="net", ACTION=="add", DEVPATH!="/devices/virtual/*", RUN+="vad mv"
  ```
  Automatically execute key rotation every 4 days:
  ```
  /usr/local/lib/system/vad.rotate.service
  ----------------------------------------
  [Unit]
  Description=Rotate WireGuard keys of all devices

  [Service]
  Type=oneshot
  ExecStart=vad rotate
  ```
  `OnCalendar` of the timer is testable with: `systemd-analyze calendar --iterations=8 '*-*-2/4'`.
  List timers: `systemctl list-timers --all`.
  ```
  /usr/local/lib/systemd/vad.rotate.timer
  ---------------------------------------
  [Unit]
  Description=Rotate WireGuard keys of all devices every 4 days

  [Timer]
  OnCalendar=*-*-2/4
  Persistent=true

  [Install]
  WantedBy=timers.target
  ```
  Automatically execute vad up daily:
  ```
  /usr/local/lib/system/vad.up.service
  ------------------------------------
  [Unit]
  Description=Execute vad up

  [Service]
  Type=oneshot
  ExecStart=vad up
  ```
  ```
  /usr/local/lib/systemd/vad.up.timer
  -----------------------------------
  [Unit]
  Description=Execute vad up daily to connect to other servers

  [Timer]
  OnCalendar=daily
  Persistent=true

  [Install]
  WantedBy=timers.target
  ```
  Automatically start vad on system startup:
  ```
  /usr/local/lib/systemd/vad.service
  ----------------------------------
  [Unit]
  Description=Starts VPN on system startup
  After=syslog.target network.target
  Wants=network.target

  [Service]
  Type=oneshot
  RemainAfterExit=yes
  ExecSearchPath=/usr/local/bin:/usr/bin
  ExecStart=vad up

  [Install]
  WantedBy=multi-user.target
  ```
  To enable execute: `systemctl daemon-reload`, `systemctl enable vad.rotate.timer` and `systemctl enable vad`.
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
* [ ] Add support for private/external WireGuard servers.
  A server is split into a "device" and a "server".
  A device has the following attributes: `private_key`, `ipv4` and `ipv6`.
  A server has has at least the following attributes: `public_key`, `ipv4`, and `ipv6`.
  These devices would have four additional attributes: `endpoint_ipv4`, `endpoint_ipv6`, `endpoint_public_key` and `endpoint_hostname`.
  Hostname does not need to be a real hostname.
  Basically these devices have only one endpoint.
  This devices could be selected via the hostname in the up command, e.g.
  ```
  $ vad up de dysnomia de
  ```
  The hostname search order needs to change to private-external-devices and then Mullvad servers.
  This makes the configuration step more complicated, is it worth it?

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
