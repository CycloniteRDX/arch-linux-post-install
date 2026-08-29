# 04 — Establish the security and network baseline

## Goal

Install and enable firewalld, place NetworkManager connections in a
conservative default zone, remove the predefined SSH exception, and record
which processes accept network traffic before the graphical workstation adds
more services.

This chapter makes the following changes:

- installs `firewalld` from the official repositories;
- enables `firewalld.service` at boot;
- uses firewalld's nftables backend;
- makes `public` the default zone for NetworkManager connections that do not
  select another zone;
- removes the predefined `ssh` service from the public zone;
- disables intra-zone forwarding in the public zone if the shipped zone
  enables it;
- keeps outbound traffic and replies to established connections working;
- audits listening TCP and UDP sockets before and after the change.

It does not enable the SSH server, open application ports, configure a trusted
home zone, install a graphical firewall interface, add IP blocklists, alter
DNS, apply broad sysctl hardening, or replace NetworkManager.

## Prerequisites

- Chapters 01 through 03 are complete.
- NetworkManager provides a working internet connection and DNS.
- The full system is up to date.
- `neon` has working local TTY access and password-protected `sudo`.
- The verified Arch installation USB and LUKS passphrase remain available.

Run the commands as `neon`. This procedure is intended for a local TTY, not a
remote shell: changing firewall rules over an SSH session could disconnect the
only available administration path.

## Resulting policy

| Area | Canonical result |
| --- | --- |
| Network manager | NetworkManager only |
| Firewall manager | firewalld only |
| Packet-filter backend | nftables, managed by firewalld |
| Default zone | `public` |
| Allowed public services | `dhcpv6-client` only |
| Explicit ports | None |
| SSH server | Disabled and blocked by the public zone |
| Intra-zone forwarding | Disabled in `public` |
| IP forwarding and masquerading | Disabled |
| Outbound connections | Allowed |
| Denied-packet logging | Left at the firewalld default, `off` |

The firewall is a safety boundary, not a replacement for updates, minimal
services, correct bind addresses, authentication, or application sandboxing.

## Audit the starting network state

Confirm that NetworkManager is the active network manager:

```bash
systemctl is-enabled NetworkManager.service
systemctl is-active NetworkManager.service
nmcli general status
nmcli device status
nmcli -f NAME,TYPE,DEVICE connection show --active
```

NetworkManager must be enabled and active. At least one expected wired or Wi-Fi
connection must be active. Stop if another network manager or independent DHCP
client is controlling the same interface.

Check whether firewall managers are already installed or enabled:

```bash
pacman -Q firewalld nftables ufw 2>&1
systemctl is-enabled firewalld.service nftables.service ufw.service 2>&1
systemctl is-active firewalld.service nftables.service ufw.service 2>&1
```

On the fresh canonical system, firewalld and UFW are absent and their units are
not found. If another firewall is already active, stop and identify its rules;
do not layer this procedure over an unexplained packet filter.

Inspect IP forwarding without changing it:

```bash
sysctl net.ipv4.ip_forward net.ipv6.conf.all.forwarding
```

Both values must be `0`. This laptop is not acting as a router.

## Record listening sockets before enabling the firewall

List listening TCP and UDP sockets together with their owning processes:

```bash
sudo ss -lntup
```

Interpret local addresses carefully:

| Local address | Exposure |
| --- | --- |
| `127.0.0.1` or `::1` | Loopback only; not directly reachable from another machine. |
| A specific LAN address | Reachable through the corresponding interface unless filtered. |
| `0.0.0.0` | Listening on every IPv4 interface. |
| `*` or `::` | Potentially listening on every applicable interface. |

UDP sockets can be less intuitive than TCP listeners, so identify the process
and purpose before treating an unfamiliar row as harmless. A firewall can
block packets to a listening service, but unnecessary services should still be
disabled or bound only to loopback.

Confirm specifically that the SSH server is not enabled or running:

```bash
systemctl is-enabled sshd.service
systemctl is-active sshd.service
```

Both results must be `disabled` and `inactive`. The `openssh` package installed
in chapter 02 provides the `ssh` client used for GitHub connections, but it
does not enable `sshd.service`.

## Install firewalld

Read new entries on the [Arch Linux news page](https://archlinux.org/news/),
then install firewalld through a complete upgrade transaction:

```bash
sudo pacman -Syu firewalld
```

Read the transaction and every package-hook message. Then inspect unresolved
package configuration files:

```bash
sudo pacdiff --output
pacman -Q firewalld
```

Resolve any listed `.pacnew`, `.pacsave`, or `.pacorig` file before
continuing.

Do not install `firewall-config` or `firewall-applet` at this stage. The command
line is sufficient for the initial policy, and graphical integration has not
yet been installed.

## Enable the firewall

Enable and start the daemon:

```bash
sudo systemctl enable --now firewalld.service
```

Verify the daemon and its initial configuration:

```bash
sudo firewall-cmd --state
systemctl is-enabled firewalld.service
systemctl is-active firewalld.service
sudo firewall-cmd --get-default-zone
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --zone=public --list-all
```

Firewalld must report `running`. The default is normally `public`, and
NetworkManager should bind the active network interface to that zone.

The shipped public zone commonly allows both `ssh` and `dhcpv6-client`.
Installing firewalld does not start `sshd`, but leaving the SSH firewall
exception would make a future accidental server activation immediately
reachable. The next section removes that exception.

## Apply the permanent public-zone baseline

Set `public` as the default for every connection without an explicit zone:

```bash
sudo firewall-cmd --set-default-zone=public
```

This particular command changes both runtime and permanent configuration.

Check whether the permanent public zone contains the SSH service:

```bash
sudo firewall-cmd --permanent --zone=public --query-service=ssh
```

If the result is `yes`, remove it:

```bash
sudo firewall-cmd --permanent --zone=public --remove-service=ssh
```

If the query already reports `no`, do not add or remove anything for SSH.

Check whether intra-zone forwarding is enabled:

```bash
sudo firewall-cmd --permanent --zone=public --query-forward
```

If the result is `yes`, disable it:

```bash
sudo firewall-cmd --permanent --zone=public --remove-forward
```

This laptop does not route traffic between public interfaces. IP forwarding
also remains disabled at the kernel level.

Validate the permanent configuration before applying it:

```bash
sudo firewall-cmd --check-config
```

No output and a successful exit indicate that the XML and semantics are
valid. Apply the permanent state to the running firewall:

```bash
sudo firewall-cmd --reload
```

A normal reload preserves connection-tracking state. Do not use
`--complete-reload`; it is intended for severe firewall problems and can
terminate active connections.

## Verify the active and permanent zones

Inspect both representations after the reload:

```bash
sudo firewall-cmd --get-default-zone
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --zone=public --list-all
sudo firewall-cmd --permanent --zone=public --list-all
sudo firewall-cmd --get-log-denied
```

The runtime and permanent public zones must agree:

- target is `default`;
- the active network interface is assigned to `public`;
- services contains `dhcpv6-client` and not `ssh`;
- ports, protocols, forward ports, source ports, and rich rules are empty;
- intra-zone forwarding and masquerading are disabled;
- denied-packet logging is `off`.

The `default` zone target rejects unmatched incoming traffic while permitting
required ICMP handling. `dhcpv6-client` permits DHCPv6 replies needed by a
client; it does not expose an interactive login or general-purpose server.

Do not remove IPv6 support or block all ICMP merely to make the service list
look empty. IPv6 neighbor discovery, path MTU discovery, and other control
traffic are required for correct networking.

## Confirm NetworkManager integration

Display the active connection profile again:

```bash
nmcli -f NAME,TYPE,DEVICE connection show --active
```

Use its exact name in the following command:

```bash
nmcli -g connection.zone connection show "<ACTIVE_PROFILE>"
```

An empty result is correct: NetworkManager then assigns the interface to
firewalld's default `public` zone. An explicit `public` result is also valid.

If the result names `home`, `work`, `trusted`, or another unexpected zone, stop
and inspect that zone before continuing. On a fresh profile, clear the
unexpected assignment and reactivate the connection:

```bash
sudo nmcli connection modify "<ACTIVE_PROFILE>" connection.zone ""
sudo nmcli connection up "<ACTIVE_PROFILE>"
```

Then confirm the active zone again:

```bash
sudo firewall-cmd --get-active-zones
```

Do not bind the interface manually with `firewall-cmd --add-interface`.
NetworkManager owns the connection and notifies firewalld when its interface
appears, disappears, or changes name.

Later, a deliberately trusted Wi-Fi profile may be assigned to a separate
zone, but the initial baseline treats every unknown, home, university, and
mobile-hotspot connection as public.

## Verify nftables ownership

Firewalld uses nftables as its default backend. Inspect the configured backend
and the kernel tables:

```bash
grep -E '^(FirewallBackend|DefaultZone|LogDenied)' /etc/firewalld/firewalld.conf
sudo nft list tables
```

`FirewallBackend=nftables` and a firewalld-owned nftables table must be
present. Do not enable `nftables.service` or maintain an independent
`/etc/nftables.conf` alongside this baseline. Firewalld is the only manager of
the workstation's host firewall rules.

Container engines and virtualization tools can add their own nftables rules.
If Docker, Podman, libvirt, or network namespaces are introduced later, audit
their interaction with firewalld and its `StrictForwardPorts` setting before
publishing any port.

## Verify ordinary network use

Confirm that DNS and outbound HTTPS still work:

```bash
curl --fail --head https://archlinux.org/
```

Using Git over SSH is also an outbound client connection and does not require
the public `ssh` service. If GitHub authentication has already been configured,
this optional check must still succeed:

```bash
ssh -T git@github.com
```

GitHub should report successful authentication and explain that it does not
provide shell access. An authentication failure is a Git/SSH credential issue,
not a reason to open port 22 in the laptop's firewall.

## Audit the final exposure

Repeat the socket and service checks:

```bash
sudo ss -lntup
systemctl is-enabled sshd.service
systemctl is-active sshd.service
systemctl is-enabled nftables.service 2>&1
systemctl is-active nftables.service 2>&1
sysctl net.ipv4.ip_forward net.ipv6.conf.all.forwarding
systemctl --failed
sudo journalctl -b -u firewalld.service -p warning --no-pager
```

Confirm that:

- every network-facing listener has a known purpose;
- `sshd.service` remains disabled and inactive;
- the separate `nftables.service` is disabled and inactive;
- IPv4 and IPv6 forwarding remain `0`;
- no system unit is failed;
- firewalld has no unexplained warnings or errors in the current boot.

Enabling a firewall does not remove listeners. If an unexpected process is
bound to `0.0.0.0`, `::`, or a LAN address, identify its package and unit,
then disable or reconfigure it rather than relying only on packet filtering.

## Adding services later

No application exception is needed now. Later chapters may intentionally add
services such as mDNS, printing, KDE Connect, media sharing, or a local
development server.

Before opening anything:

1. Identify the exact daemon, protocol, port, and required source network.
2. Confirm that authentication and encryption are appropriate.
3. Prefer a predefined firewalld service to a raw port when one matches.
4. Test the change in runtime configuration first.
5. Make it permanent only after validation.
6. Record the reason and remove the exception when it is no longer needed.

Do not copy a generic list of desktop ports into the public zone. A package
installation is not authorization to expose its service.

## Recovery and rollback

If the firewall prevents necessary networking while local TTY access still
works, inspect its state first:

```bash
sudo firewall-cmd --state
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --zone=public --list-all
sudo journalctl -b -u firewalld.service --no-pager
```

To temporarily stop the firewall for diagnosis without changing its boot
setting:

```bash
sudo systemctl stop firewalld.service
```

Restore the service immediately after identifying or correcting the problem:

```bash
sudo systemctl start firewalld.service
```

Stopping the firewall removes its rules because firewalld's default
`CleanupOnExit` policy is enabled. Do not leave the laptop connected to an
untrusted network in that state.

To restore the vendor public-zone defaults deliberately:

```bash
sudo firewall-cmd --permanent --zone=public --load-zone-defaults
sudo firewall-cmd --reload
```

The vendor defaults may re-enable the SSH exception, so inspect the resulting
zone before accepting that rollback.

## Completion checkpoint

```bash
sudo firewall-cmd --state
sudo firewall-cmd --get-default-zone
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --zone=public --list-all
sudo firewall-cmd --permanent --zone=public --list-all
sudo ss -lntup
systemctl is-enabled firewalld.service
systemctl is-active firewalld.service
systemctl is-enabled sshd.service
systemctl is-active sshd.service
sysctl net.ipv4.ip_forward net.ipv6.conf.all.forwarding
systemctl --failed
```

The chapter is complete when:

- firewalld is enabled, active, and reports `running`;
- NetworkManager's active interface uses the public zone;
- runtime and permanent public-zone configurations agree;
- `dhcpv6-client` is the only allowed public service;
- no explicit port, protocol, forwarding rule, masquerade, or rich rule exists;
- firewalld owns the nftables rules and `nftables.service` is not active;
- SSH client connections work while `sshd.service` remains disabled, inactive,
  and blocked from the public zone;
- IP forwarding remains disabled;
- every listening socket has been identified;
- outbound DNS and HTTPS still work;
- no system unit is failed.

## Sources

- [ArchWiki: Security](https://wiki.archlinux.org/title/Security)
- [ArchWiki: firewalld](https://wiki.archlinux.org/title/Firewalld)
- [ArchWiki: NetworkManager](https://wiki.archlinux.org/title/NetworkManager)
- [ArchWiki: OpenSSH](https://wiki.archlinux.org/title/OpenSSH)
- [Arch manual: firewall-cmd(1)](https://man.archlinux.org/man/firewall-cmd.1)
- [Arch manual: firewalld.conf(5)](https://man.archlinux.org/man/firewalld.conf.5)
- [Arch manual: firewalld.zones(5)](https://man.archlinux.org/man/firewalld.zones.5)
- [Arch manual: ss(8)](https://man.archlinux.org/man/ss.8)
- [Arch package: firewalld](https://archlinux.org/packages/extra/any/firewalld/)
- [NetworkManager: firewalld zone integration](https://firewalld.org/documentation/zone/connections-interfaces-and-sources.html)

## Next step

Continue with chapter 05 to install the minimal AMD graphics, Niri, Kitty,
portal, polkit, PipeWire, and XWayland foundation while preserving the working
TTY recovery path.
