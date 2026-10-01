# OpenWrt on Cudy TR1200 v1

[Russian version](README.md#russian-version)

An installation and practical configuration guide for the **Cudy TR1200 v1** travel router, based on a tested OpenWrt 24.10.8 setup.

The resulting configuration provides:

- an AmneziaWG VPN network with a kill switch;
- a separate direct-access Wi-Fi network;
- Wi-Fi client mode for hotel, airport, and home uplinks;
- automatic preference for Wi-Fi uplink over Ethernet WAN;
- a physical VPN on/off switch;
- white/red VPN status indication;
- backup, upgrade, diagnostics, and recovery notes.

> [!WARNING]
> Flashing third-party firmware can void the warranty or make the router unusable. This guide is only for **Cudy TR1200 hardware v1**. Verify the model and hardware revision on the label before continuing. Never interrupt power during flashing.

## 1. Hardware and firmware

| Item | Value |
|---|---|
| Device | Cudy TR1200 v1 |
| Target | `ramips/mt76x8` |
| Tested release | OpenWrt 24.10.8 (`r29233-443ec4032a`) |
| Official sysupgrade image | `openwrt-24.10.8-ramips-mt76x8-cudy_tr1200-v1-squashfs-sysupgrade.bin` |
| SHA-256 | `6c0a8d3844759b4f54cfee6c675e8e969be3d3a9e3e4cd06c319c411a5a47efb` |

The checksum above matches the official OpenWrt 24.10.8 download directory.

Official resources:

- [OpenWrt device page](https://openwrt.org/toh/cudy/tr1200)
- [OpenWrt 24.10.8 images for ramips/mt76x8](https://downloads.openwrt.org/releases/24.10.8/targets/ramips/mt76x8/)
- [Cudy TR1200 v1 downloads](https://www.cudy.com/en-us/pages/download-center/tr1200-1-0)
- [Cudy OpenWrt intermediate firmware information](https://www.cudy.com/en-us/blogs/faq/openwrt-software-download)

OpenWrt releases change over time. Prefer a currently supported stable release after checking device notes and compatibility with any kernel modules you need.

## 2. Files and image types

Two different images are required for a first installation:

1. **Cudy-signed intermediate image** — accepted by the stock Cudy web interface. In the tested package it was named `TR1200-OpenWRT-Flash.bin`.
2. **Official OpenWrt sysupgrade image** — installed after the intermediate firmware has booted.

Do not upload the regular sysupgrade image directly to the stock Cudy interface. Obtain the intermediate image from Cudy for the exact hardware revision; do not use an image intended for another model or flash variant.

If OpenWrt is already installed, use only a compatible sysupgrade image and follow the normal upgrade procedure. The intermediate image is not used for routine upgrades.

## 3. First installation

### Preparation

1. Confirm that the label says **TR1200 v1**.
2. Download the correct Cudy-signed intermediate firmware from Cudy.
3. Download the official OpenWrt sysupgrade image for `cudy_tr1200-v1`.
4. Verify the sysupgrade checksum.
5. Connect a computer to the router by Ethernet.
6. Save the current Cudy configuration if you may need it later.
7. Use stable power and do not flash over an unreliable wireless connection.

Linux/macOS checksum command:

```sh
sha256sum openwrt-24.10.8-ramips-mt76x8-cudy_tr1200-v1-squashfs-sysupgrade.bin
```

Windows PowerShell:

```powershell
Get-FileHash .\openwrt-24.10.8-ramips-mt76x8-cudy_tr1200-v1-squashfs-sysupgrade.bin -Algorithm SHA256
```

### Stage A: stock Cudy firmware to intermediate OpenWrt

1. Open the stock Cudy administration interface.
2. Go to the firmware upgrade page.
3. Select the Cudy-signed `TR1200-OpenWRT-Flash.bin` image.
4. Start the upgrade and wait at least several minutes.
5. Do not expect the old Cudy page to report success: the router is replacing that interface with OpenWrt.
6. Reconnect by Ethernet and open `http://<OPENWRT_DEFAULT_ADDRESS>`.

Wi-Fi may be disabled after the first OpenWrt boot, so Ethernet is strongly recommended.

### Stage B: intermediate build to official OpenWrt

1. In LuCI, open **System → Backup / Flash Firmware**.
2. Upload the official `...cudy_tr1200-v1-squashfs-sysupgrade.bin` image.
3. Do **not** keep settings when moving from the intermediate build to official OpenWrt.
4. Confirm the upgrade and wait for the router to reboot.
5. Reconnect to `http://<OPENWRT_DEFAULT_ADDRESS>` and immediately set a strong root password.

If image validation fails, stop and re-check the model, revision, image name, and current intermediate firmware. Do not force an upgrade unless a current device-specific OpenWrt instruction explicitly requires it and you understand the risk.

## 4. Recommended travel-router layout

The tested setup uses two local networks:

| Network | Example subnet | Purpose | If VPN fails |
|---|---:|---|---|
| `lan` / VPN Wi-Fi | `<VPN_LAN_CIDR>` | Normal client traffic through AmneziaWG | Internet is blocked |
| `direct` / Direct Wi-Fi | `<DIRECT_LAN_CIDR>` | Captive portals and direct diagnostics | Continues through WAN |

Physical uplinks:

- `wwan`: Wi-Fi station/client, preferred uplink, metric `100`;
- Ethernet WAN: backup uplink, metric `200`.

VPN traffic uses the AmneziaWG interface. Direct clients are routed through a separate policy-routing table to the physical uplink.

```text
                    Internet
                       |
           +-----------+-----------+
           |                       |
     Wi-Fi uplink              Ethernet WAN
      metric 100                metric 200
           |                       |
           +--------- wan zone ----+
                       |
             +---------+---------+
             |                   |
       AmneziaWG VPN        direct route
             |               table 100
       Cudy-VPN Wi-Fi      Direct Wi-Fi
       kill switch on      no VPN tunnel
```

Use your own neutral SSIDs and strong unique passwords. Do not copy names or credentials from examples found online.

## 5. Kill switch

The essential firewall rule is simple:

- allow forwarding from `lan` to the VPN firewall zone;
- do **not** allow forwarding from `lan` directly to `wan`;
- allow `direct` to `wan`;
- do not forward `direct` into the VPN zone.

Test it from a client connected to the VPN Wi-Fi:

1. Enable the VPN and confirm the public IP belongs to the VPN path.
2. Run `ifdown VPN` on the router.
3. Internet access on the VPN Wi-Fi must stop.
4. The Direct Wi-Fi should still work if the physical uplink is available.
5. Run `ifup VPN` and verify recovery.

If the VPN Wi-Fi still has Internet access while the VPN is down, the kill switch is not working. Check for an accidental `lan → wan` forwarding.

## 6. Direct network and changing uplinks

The Direct network is useful for hotel captive portals and troubleshooting. A policy rule sends traffic from the Direct subnet to routing table `100`, which contains a default route through the physical uplink.

Useful checks:

```sh
ip rule
ip route show table 100
ubus call network.interface.wwan status
ip -4 route show default
```

Do not hard-code a hotel or home gateway. The gateway normally changes when the router joins another network. Update table `100` from the DHCP-provided default route or automate it with a tested netifd hotplug script.

Before completing a captive portal login:

1. connect a phone or laptop to the Direct Wi-Fi;
2. join the external Wi-Fi using `radio1` as a station assigned to `wwan` and firewall zone `wan`;
3. open an unencrypted HTTP page to trigger the portal;
4. complete authentication;
5. enable and test the VPN network.

## 7. Physical VPN switch

On the tested hardware, the side switch generates `BTN_0` hotplug events. Create `/etc/hotplug.d/button/99-slider-vpn`:

```sh
#!/bin/sh

[ "$BUTTON" = "BTN_0" ] || exit 0

case "$ACTION" in
    pressed)
        logger -t slider-vpn "VPN ON"
        ifup VPN
        ;;
    released)
        logger -t slider-vpn "VPN OFF"
        ifdown VPN
        ;;
esac
```

Then make it executable:

```sh
chmod +x /etc/hotplug.d/button/99-slider-vpn
```

Check events with:

```sh
logread -f | grep -E 'slider-vpn|BTN_0|button'
```

If the mechanical direction is reversed on your unit, swap `ifup VPN` and `ifdown VPN`.

## 8. VPN status LEDs

Observed LED paths:

- `/sys/class/leds/white:status`
- `/sys/class/leds/red:power`

The tested behavior was:

- steady white: VPN is up and a test packet passes through it;
- blinking white: interface is up, but the traffic check fails;
- steady red: VPN is down.

An example monitor is available in [`scripts/vpn-led-monitor`](scripts/vpn-led-monitor). LED names can differ after platform changes, so check `ls /sys/class/leds/` before installing it. A public DNS address is used only as a connectivity probe; replace it if needed.

## 9. Resource limits

The TR1200 v1 has limited RAM and flash. On the tested setup:

- internal writable overlay was approximately 9.6 MiB;
- about 7 MiB remained free after the essential setup;
- approximately 55 MiB RAM remained available under normal load;
- Tailscale worked but consumed too much memory for comfortable operation;
- swap/zram was unavailable in the tested kernel configuration;
- USB extroot worked, but was unnecessary after removing heavy services.

Recommendations:

- check `free -m` and `df -h /overlay` before installing packages;
- do not run a blind mass `opkg upgrade`;
- use sysupgrade for firmware releases;
- install only necessary services;
- keep kernel modules matched to the exact running kernel.

## 10. Backup and upgrade

Create a backup before any significant change:

```sh
umask 077
sysupgrade -k -b /tmp/cudy-tr1200-backup.tar.gz
sha256sum /tmp/cudy-tr1200-backup.tar.gz
```

Download it immediately because `/tmp` is stored in RAM and is erased at reboot.

> [!CAUTION]
> A backup can contain Wi-Fi passwords, VPN private keys, preshared keys, and other secrets. Never commit it to GitHub or attach it to a public issue.

Before sysupgrade:

```sh
ubus call system board
df -h /overlay
free -m
sysupgrade -k -b /tmp/pre-upgrade-backup.tar.gz
```

Confirm image compatibility with the exact model, review release notes, and check whether AmneziaWG packages exist for the new kernel. Test both Wi-Fi networks and the kill switch after upgrading.

## 11. Diagnostics

```sh
date
uptime
free -m
df -h /overlay
ubus call system board
ifstatus VPN
ubus call network.interface.wwan status
ip addr
ip rule
ip route
ip route show table 100
logread | tail -n 120
dmesg | tail -n 120
```

Before sharing output, remove SSIDs, MAC addresses, public IP addresses, hostnames, serial numbers, and all keys or credentials.

Quick interpretation:

- Direct works, VPN Wi-Fi does not: inspect VPN state, endpoint reachability, routes, and logs.
- Neither network works: inspect the physical uplink and DHCP gateway.
- Direct fails after changing hotels: inspect table `100` for an old gateway.
- VPN Wi-Fi works with VPN off: fix the kill switch immediately.
- Memory falls below a safe margin: stop or remove heavy daemons.

## 12. Failsafe and recovery

OpenWrt failsafe can repair a broken network/firewall configuration or reset the root password:

1. connect by Ethernet;
2. trigger failsafe during early boot according to the current OpenWrt device instructions;
3. configure a static client address if DHCP is unavailable;
4. connect to the failsafe address;
5. run `mount_root` and repair only the necessary files;
6. reboot.

Example:

```sh
mount_root
passwd
vi /etc/config/network
vi /etc/config/firewall
reboot -f
```

Do not invent a TFTP/reset sequence. Bootloader recovery is device-specific. For return to Cudy firmware, follow Cudy's current recovery guide; TR1200 has model-specific port requirements.

## 13. What was done — step by step

This is the complete implementation sequence. It can also be used as a safe order for reproducing the setup.

| Step | Action | Reason |
|---:|---|---|
| 1 | Verified model, region, and hardware revision | Images for different devices or revisions are not interchangeable |
| 2 | Saved the stock configuration and recovery resources | A return path should exist before flashing |
| 3 | Installed the Cudy-signed intermediate image | The stock UI does not accept a normal OpenWrt sysupgrade image |
| 4 | Replaced the intermediate build with official OpenWrt | This provides a standard system and official upgrades |
| 5 | Set the root password and verified Ethernet access | Wi-Fi can be disabled on a fresh OpenWrt installation |
| 6 | Changed the local subnet | This avoids common hotel and upstream network conflicts |
| 7 | Configured `wwan` as a Wi-Fi client | The router receives Internet access from an external AP |
| 8 | Kept Ethernet WAN as a backup uplink | A cable can provide an alternative connection |
| 9 | Assigned different uplink metrics | Wi-Fi is preferred and Ethernet remains a fallback |
| 10 | Installed compatible AmneziaWG components and created `VPN` | The main client network gained a VPN route |
| 11 | Created a separate `direct` interface and AP | Captive portals and diagnostics can bypass the VPN |
| 12 | Added a routing table for Direct | Direct clients do not follow the VPN default route |
| 13 | Removed direct `lan → wan` forwarding | This creates a real kill switch |
| 14 | Connected the side switch to `ifup/ifdown VPN` | The tunnel can be controlled without LuCI |
| 15 | Added an LED monitor | VPN status is visible without another device |
| 16 | Verified backup contents | Custom scripts must survive sysupgrade |
| 17 | Tested USB extroot, swap/zram, and Tailscale | This established the practical RAM, flash, and kernel limits |
| 18 | Removed unnecessary heavy components and returned to internal overlay | The final system became simpler and more reliable |

After every major step, RAM, flash, interfaces, routes, firewall, and logs were checked. Applying changes incrementally makes failures much easier to isolate.

## 14. Detailed network setup

### 14.1. Placeholders and interface roles

This public guide uses placeholders instead of observed addresses:

| Placeholder | Replace with |
|---|---|
| `<OPENWRT_DEFAULT_ADDRESS>` | The fresh OpenWrt address documented for the release |
| `<VPN_ROUTER_ADDRESS>` | Router address on the main VPN LAN |
| `<VPN_LAN_CIDR>` | Main client subnet |
| `<DIRECT_ROUTER_ADDRESS>` | Router address on the Direct LAN |
| `<DIRECT_LAN_CIDR>` | Direct client subnet |
| `<UPLINK_GATEWAY>` | Gateway supplied by the current uplink DHCP server |
| `<VPN_TEST_HOST>` | A host selected for testing traffic through the VPN |

Logical interface layout:

| Interface | Protocol | Device/role | Firewall zone |
|---|---|---|---|
| `lan` | static | bridge and main local AP | `lan` |
| `direct` | static | second local AP | `direct` |
| `wwan` | DHCP client | external Wi-Fi station | `wan` |
| `wan` | DHCP client | Ethernet WAN | `wan` |
| `VPN` | `amneziawg` | VPN tunnel | `awg` |

The main LAN subnet was changed before configuring the other components. After applying a new router address, the old LuCI tab stops responding. Renew the client's DHCP lease and reopen LuCI at `<VPN_ROUTER_ADDRESS>`.

### 14.2. Wi-Fi uplink in LuCI

1. Open **Network → Wireless**.
2. Click **Scan** on the radio selected for uplink duty.
3. Select the external network and click **Join Network**.
4. Enter its password without storing it in public notes or screenshots.
5. Assign the connection to a new logical interface named `wwan`.
6. Configure `wwan` as a DHCP client in firewall zone `wan`.
7. Apply changes and wait for DHCP parameters.
8. Verify the result:

```sh
ubus call network.interface.wwan status
ip link show
ip route
iwinfo 2>/dev/null
```

Before sharing this output, remove SSIDs, BSSID/MAC values, addresses, hostnames, and lease data.

### 14.3. Uplink preference

The tested design assigned a lower metric to Wi-Fi and a higher metric to Ethernet WAN. Exact numbers are not important; ensure that:

- the metrics differ;
- the VPN endpoint is reached through a working physical uplink;
- the tunnel never tries to reach its own endpoint through itself;
- routes are recalculated after an uplink change.

```sh
ip route
ip route get <VPN_ENDPOINT_ADDRESS>
```

Do not publish the endpoint value with command output.

## 15. AmneziaWG and the VPN LAN

### 15.1. Importing a profile safely

The `VPN` interface needs settings from an exported AmneziaWG profile: interface address, peer, endpoint, allowed IPs, and protocol-specific parameters. Private and preshared keys belong only on the router. They must not enter Git, unprotected backups, screenshots, or support logs.

Procedure:

1. Find an AmneziaWG build compatible with the exact OpenWrt release and kernel.
2. Install its userspace package and matching kernel module.
3. Create an interface named `VPN`.
4. Transfer settings from your own profile.
5. Assign the interface to a separate firewall zone named `awg`.
6. Allow forwarding only from the main LAN into `awg`.
7. Bring the interface up and verify handshake, routes, and data transfer.

There is intentionally no universal package-install command here: repository and package names depend on the OpenWrt and AmneziaWG versions. A kernel module built for another kernel can make the router unstable.

### 15.2. VPN checks

```sh
ifstatus VPN
ifup VPN
ifdown VPN
ip route
ping -I VPN -c 3 <VPN_TEST_HOST>
logread | grep -Ei 'amnezia|wireguard|VPN|netifd' | tail -n 100
```

`up=true` does not prove that data passes through the tunnel. The LED monitor and manual diagnostics therefore perform a separate test through `VPN`.

### 15.3. When the tunnel does not start

1. Test the Direct network first. If it also fails, fix the uplink before the VPN.
2. Verify system time and timezone.
3. Check the physical route to the endpoint.
4. Confirm that the kernel module matches `uname -r`.
5. Restart the interface with `ifdown VPN; sleep 2; ifup VPN`.
6. Inspect `logread` and `dmesg` after sanitizing identifiers.
7. If server details or keys changed, export a fresh profile instead of randomly editing the old one.

## 16. Direct network, policy routing, and captive portals

### 16.1. Why a second network is needed

When all router traffic defaults to VPN, a hotel login page may be unreachable before captive-portal authentication. The Direct network solves this without weakening protection for clients on the main LAN.

The Direct setup consists of:

- a separate interface, subnet, and DHCP server;
- a second AP;
- firewall zone `direct`;
- forwarding from `direct` to `wan`;
- a policy rule matching the source network or incoming interface;
- a separate routing table with a default route through the physical uplink.

### 16.2. Policy-routing checks

```sh
ip rule
ip route show table 100
ip -4 route show default
```

Expected logic without real addresses:

```text
from <DIRECT_LAN_CIDR> lookup 100
table 100 default via <UPLINK_GATEWAY> dev <UPLINK_DEVICE>
```

The first implementation used a static gateway in table `100`. It worked on one upstream network but became stale after travel. A travel router should derive this gateway from the current DHCP route.

Temporary replacement after verifying the gateway:

```sh
ip route replace table 100 default via <UPLINK_GATEWAY> dev <UPLINK_DEVICE>
```

A netifd hotplug script can automate this, but it must be tested while connecting and disconnecting both uplinks. It must not leave a stale route or compete with a static LuCI entry.

### 16.3. Hotel workflow

1. Connect a client to Direct Wi-Fi.
2. Scan on the uplink radio in LuCI.
3. Join the hotel network through `wwan`.
4. Confirm that DHCP parameters were received.
5. Verify and, if necessary, update table `100`.
6. Open a plain HTTP page to trigger the captive portal.
7. Complete authentication and test Direct Internet access.
8. Enable and test the VPN LAN.
9. Retest the kill switch.

## 17. Final firewall matrix

| Source | Destination | Allowed | Reason |
|---|---|---|---|
| `lan` | `awg` | Yes | Main Internet access through VPN |
| `lan` | `wan` | No | Kill switch |
| `direct` | `wan` | Yes | Captive portal and direct access |
| `direct` | `awg` | No | Direct must not silently enter the VPN |
| `wan` | LuCI/SSH | No | Administration is not exposed upstream |
| internal networks | router | As required | DHCP, DNS, and administration |

The `wan` zone normally uses masquerading. VPN NAT and MTU settings depend on the package and profile. After every firewall change, test the negative case as well: with VPN disabled, the main LAN must lose external access.

## 18. Daily operation

### Normal startup

1. Power on the router and wait for a complete boot.
2. Wait for `wwan` or Ethernet WAN to connect.
3. Move the side switch to VPN ON.
4. Connect clients to the main VPN Wi-Fi.
5. Check the LED; steady white means the tunnel passed its connectivity probe.

Quick check:

```sh
uptime
free -m
df -h /overlay
ifstatus VPN
ubus call network.interface.wwan status
ip route
```

### Internet without VPN

Prefer moving only the required client to Direct Wi-Fi. Turning VPN off globally must not send main-LAN clients directly to WAN; the kill switch should block them.

### Hardware switch test

```sh
logread -f | grep -E 'slider-vpn|BTN_0|button'
```

Add the switch script to `/etc/sysupgrade.conf` so it survives firmware upgrades.

## 19. Running the LED monitor with procd

The monitor file also needs an init script at `/etc/init.d/vpn-led`:

```sh
#!/bin/sh /etc/rc.common

USE_PROCD=1
START=95
STOP=10

start_service() {
    procd_open_instance
    procd_set_param command /usr/bin/vpn-led-monitor
    procd_set_param respawn 3600 5 5
    procd_close_instance
}

stop_service() {
    echo none > /sys/class/leds/white:status/trigger 2>/dev/null
    echo 0 > /sys/class/leds/white:status/brightness 2>/dev/null
}
```

Install and enable it:

```sh
chmod +x /usr/bin/vpn-led-monitor /etc/init.d/vpn-led
/etc/init.d/vpn-led enable
/etc/init.d/vpn-led restart
```

Check the actual names under `/sys/class/leds/` first. Early boot blinking may be platform-controlled; the custom logic starts only after the service runs.

## 20. Experiments and conclusions

### USB extroot

An ext4 USB drive successfully operated as `/overlay`, proving that USB storage, `block-mount`, and ext4 work. Once heavy packages were removed, internal storage was sufficient again. Extroot was removed because a permanently attached drive added another failure point and complicated recovery.

Before returning to internal overlay, critical configs and scripts were compared. Extroot was then disabled in the internal `fstab`; the router booted without USB and VPN, RAM, and writable storage were checked again.

### Swap and zram

The tested kernel did not expose swap support. A USB swap file and zram therefore could not work without another kernel configuration. Creating a large swap file anyway only causes unnecessary writes.

### Tailscale

Tailscale was installed and functioned, including remote access, but `tailscaled` consumed too much of the available RAM. Removing the daemon, firewall additions, and unused dependencies restored a comfortable memory margin.

For this hardware class, a heavy remote-access daemon is better placed on a more capable LAN device. Keep only essential services on the TR1200.

## 21. Backup and restore details

Inspect the backup list first:

```sh
sysupgrade -l
```

It should include the main UCI configuration and custom files:

```text
/etc/config/network
/etc/config/wireless
/etc/config/firewall
/etc/hotplug.d/button/99-slider-vpn
/etc/init.d/vpn-led
/usr/bin/vpn-led-monitor
```

Add custom paths to `/etc/sysupgrade.conf`. `sysupgrade -k -b` records installed-package information, but the archive is not a universal binary package backup. AmneziaWG and other kernel modules may need reinstalling after a clean flash.

Restore only to a compatible release, then verify:

1. required packages and kernel modules;
2. both Wi-Fi networks and DHCP;
3. `wwan`, routes, and table `100`;
4. VPN handshake and traffic;
5. the kill switch;
6. the side switch and LED service.

## 22. OpenWrt upgrades

Before upgrading:

1. Create and download a fresh backup from `/tmp`.
2. Save an installed-package inventory.
3. Check free flash and RAM.
4. Confirm that the image is for TR1200 v1.
5. Verify SHA-256 against the official source.
6. Confirm AmneziaWG availability for the new kernel.
7. Use Ethernet where possible.
8. Do not upgrade with unstable power.

After upgrading:

```sh
ubus call system board
uname -r
df -h /overlay
free -m
ifstatus VPN
ip rule
ip route
ip route show table 100
logread | tail -n 100
```

Do not run a mass `opkg upgrade`. Kernel packages are tied to the exact running kernel; use sysupgrade for release transitions.

## 23. Windows routing problems

During setup, an incorrect Windows default route made connectivity appear broken even though OpenWrt worked correctly. Diagnose with:

```powershell
ipconfig /all
route print
Get-NetRoute -AddressFamily IPv4 |
    Sort-Object DestinationPrefix, RouteMetric
Get-NetIPConfiguration
```

Troubleshooting order:

1. Confirm that the client received an address and gateway from the selected router network.
2. Test `<VPN_ROUTER_ADDRESS>` or `<DIRECT_ROUTER_ADDRESS>`.
3. Test external connectivity without DNS.
4. Test name resolution separately.
5. Look for a stale persistent default route through another adapter.

Never delete every default route blindly. Identify the wrong entry by `InterfaceIndex`, `NextHop`, and metric, then remove only that route with `Remove-NetRoute`.

## 24. Symptom-based troubleshooting

| Symptom | Likely cause | Check first |
|---|---|---|
| VPN Wi-Fi has no Internet; Direct works | VPN, endpoint, or route | `ifstatus VPN`, endpoint route, `logread` |
| Neither network works | Physical uplink | `wwan` status, DHCP, default route |
| Direct fails after changing networks | Stale gateway in table `100` | `ip rule`, `ip route show table 100` |
| Main LAN works while VPN is off | Broken kill switch | ensure there is no `lan → wan` forwarding |
| White LED blinks | Interface is up but probe fails | traffic through `VPN`, routes, logs |
| LuCI does not open but router responds | `uhttpd` or firewall | `/etc/init.d/uhttpd status` |
| Flash fills quickly | Unneeded packages or overlay data | `df`, inspect `/overlay/upper` with `du` |
| Very little available RAM | Heavy daemon | `free`, `top` |
| VPN disappears after upgrade | Missing compatible module | `uname -r`, package list, `dmesg` |
| Access lost after network edit | UCI/firewall error | OpenWrt failsafe and `mount_root` |

Universal diagnostic set:

```sh
date
uptime
free -m
df -h /overlay
ubus call system board
ifstatus VPN
ubus call network.interface.wwan status
ip link
ip rule
ip route
ip route show table 100
logread | tail -n 120
dmesg | tail -n 120
```

Replace all identifiers with placeholders before publication. Even when a command does not print a private key, logs may contain SSIDs, MAC addresses, endpoints, addresses, and hostnames.

## Security checklist

- Set a strong, unique root password.
- Keep LuCI and SSH closed on `wan`/`wwan`.
- Never publish `/etc/config/network` without inspecting it for private keys.
- Never publish router backups.
- Rotate VPN credentials if they may have leaked.
- Review third-party scripts before running them.
- Keep an offline recovery copy and verified firmware checksums.

## License and trademarks

This documentation is provided without warranty. OpenWrt is a trademark of the OpenWrt project; Cudy and AmneziaWG names belong to their respective owners. No vendor endorsement is implied.
