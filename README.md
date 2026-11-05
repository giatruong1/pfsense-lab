# Acme Homelab: Segmented Company Network

A miniature company network built on KVM/QEMU (libvirt, virt-manager) on a CachyOS host.
A pfSense firewall connects three isolated networks and enforces the rules between them.

## Phase 1 status: complete (2026-11-04)

## Network design

| pfSense NIC | libvirt network | Name | Subnet | Firewall IP | DHCP |
|---|---|---|---|---|---|
| em0 | default (NAT) | WAN | 192.168.122.0/24 | DHCP from host | - |
| em1 | net-corp | CORP (LAN) | 10.0.10.0/24 | 10.0.10.1 | .100-.200 |
| em3 | net-servers | SERVERS (OPT1) | 10.0.20.0/24 | 10.0.20.1 | static only |
| em2 | net-guest | GUEST (OPT2) | 10.0.30.0/24 | 10.0.30.1 | .100-.200 |

The three internal libvirt networks have no DHCP and no forwarding. Traffic between them can only pass through pfSense.

## Firewall rules (order matters, first match wins)

- **GUEST:** pass TCP/UDP to 10.0.30.1 port 53 (DNS), then block to alias `RFC1918` (10/8, 172.16/12, 192.168/16), then pass to any.
- **SERVERS:** block to alias `GUEST_NET` (10.0.30.0/24), then pass to any.
- **CORP:** default allow.
- **WAN:** "Block RFC1918 Private Networks" is off, because the KVM NAT hands out a 192.168.122.x address.

## Test results

| Source | Test | Expected | Result | Evidence |
|---|---|---|---|---|
| CORP | internet, ping 10.0.20.1 | works | pass | screenshots/corp-test.png |
| GUEST | internet | works | pass | screenshots/guest-test.png |
| GUEST | ping 10.0.10.1, 10.0.20.1 | blocked | pass | screenshots/guest-test.png |
| GUEST | `nmap -Pn 10.0.20.1` | 1000 filtered ports | pass | screenshots/nmap-guest.png |
| SERVERS | internet | works | pass | screenshots/servers-test.png |
| SERVERS | ping 10.0.30.1 | blocked | pass | screenshots/servers-test.png |

Method: one Debian 13 VM (client1) was moved between the three libvirt networks and given a matching static address for each test.


## Problems I hit and how I fixed them

| Problem | Cause | Fix |
|---|---|---|
| pfSense installer wrote nothing to disk ("not a bootable disk") | disk was not ticked in the ZFS device list | press Space on `vtbd0` so it shows `[*]` before OK |
| pfSense WAN got no DHCP lease | host UFW firewall dropped DHCP from `virbr0` | `ufw allow in on virbr0`, plus `ufw route allow in/out on virbr0` |
| Assigned the LAN IP to the wrong interface (WAN got 10.0.10.1) | typed menu number 1 instead of 2 in console option 2 | reset WAN to DHCP, then set LAN with interface 2 |
| `em2`/`em3` swapped vs the plan | virt-manager added the NICs in a different order | match by MAC address, not by em number |
| GUEST block rule had no effect (0 hits) | "Allow GUEST to internet" was above the block rule, and changes were not applied | drag rules into order, click Save and Apply Changes |
| SERVERS to GUEST was not blocked | same rule-order problem | move Block above Pass, Save, Apply |
| Typed CIDR (10.0.30.0/24) rejected as destination | the mask dropdown would not unlock | created a `GUEST_NET` alias and used that |
| Debian installer could not reach the mirror, no desktop installed | CORP DNS and static IP not set up yet | installed `task-xfce-desktop` after the network worked |

