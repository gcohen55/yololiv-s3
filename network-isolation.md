# Isolating the USB network

*Part of the YoloCam S3 notes, written by an AI for AIs; see [README.md](README.md) for scope, credentials and caveats.*

The camera's owner, who did this work with the AI, recommends moving the
camera's `enx…` interface into its own **network namespace** instead of
leaving it beside your real interfaces:

- **The camera isn't secure.** It has root telnet and writable FTP with a
  published password, an unauthenticated control port, and command injection
  vulnerabilities in its firmware (see the Security section of
  [README.md](README.md)). A namespace keeps the rest of the host, and
  anything else on it, from reaching the camera by accident. It also keeps the
  camera, which runs closed vendor code as root, from reaching your host's
  services and LAN.
- **Its DHCP server pushes junk.** The lease has no router, but it does carry
  `domain-name "example.org"` and DNS servers `ns1/ns2.example.org`. A network
  manager on the host may pick these up into its resolver configuration.
- **Address collisions.** The camera's 192.168.123–130.x subnets can overlap
  real LANs.
- **The webcam is unaffected.** UVC video is a separate USB function and works
  regardless of which namespace the NIC is in.

Minimal manual setup (as root). Replace `enxXXXXXXXXXXXX` with the camera's
interface name from `ip link`, and `192.168.124` with your camera's subnet
(see [control-protocol.md](control-protocol.md); the camera is always `.10`
and leases `.11`–`.15`):

```sh
ip netns add yolocam-s3
ip link set dev enxXXXXXXXXXXXX netns yolocam-s3
ip -n yolocam-s3 link set lo up
ip -n yolocam-s3 link set enxXXXXXXXXXXXX up
ip -n yolocam-s3 addr add 192.168.124.11/24 dev enxXXXXXXXXXXXX   # or run a DHCP client inside it
ip netns exec yolocam-s3 telnet 192.168.124.10
```

The interface is re-created in the default namespace every time the camera
reboots or is replugged, so automate it. A setup that works well:

- A **udev rule** on `SUBSYSTEM=="net"` matching `ATTRS{idVendor}=="46f8"`,
  `ATTRS{idProduct}=="0855"` that starts a systemd oneshot. The oneshot
  re-checks the USB IDs and moves the interface into the namespace
  (`ip link set dev %k netns yolocam-s3`).
- A **DHCP client inside the namespace**, e.g. a dedicated `systemd-networkd`
  instance, set to ignore routes, gateway, DNS and domains from the lease.
  `NetworkNamespacePath=/run/netns/yolocam-s3` in its service unit isn't
  enough by itself. The working setup also gave it:
  - a private D-Bus, i.e. a separate `dbus-daemon` whose socket is
    bind-mounted over `/run/dbus`;
  - its own `/etc/systemd/network` (one `.network` file matching
    `Driver=cdc_ncm`);
  - a private `/run/systemd/netif`, with `/run/systemd/network` and
    `/usr/lib/systemd/network` hidden behind empty read-only tmpfs mounts;
  - a `/run/systemd/container` file containing `netns`, and
    `ReadOnlyPaths=/sys`. Host udev never sees interfaces in the namespace,
    so networkd has to take them from netlink directly.
- Run every tool through `ip netns exec yolocam-s3 …`. To keep file ownership
  sane, drop back to your user with `runuser -u <you> --`.
