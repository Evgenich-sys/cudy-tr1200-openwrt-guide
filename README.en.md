# OpenWrt on Cudy TR1200 v1

[??????? ?????? / Russian version](README.md#russian-version)

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
6. Reconnect by Ethernet and open `http://192.168.1.1`.

Wi-Fi may be disabled after the first OpenWrt boot, so Ethernet is strongly recommended.

### Stage B: intermediate build to official OpenWrt

1. In LuCI, open **System → Backup / Flash Firmware**.
2. Upload the official `...cudy_tr1200-v1-squashfs-sysupgrade.bin` image.
3. Do **not** keep settings when moving from the intermediate build to official OpenWrt.
4. Confirm the upgrade and wait for the router to reboot.
5. Reconnect to `http://192.168.1.1` and immediately set a strong root password.

If image validation fails, stop and re-check the model, revision, image name, and current intermediate firmware. Do not force an upgrade unless a current device-specific OpenWrt instruction explicitly requires it and you understand the risk.

## 4. Recommended travel-router layout

The tested setup uses two local networks:

| Network | Example subnet | Purpose | If VPN fails |
|---|---:|---|---|
| `lan` / VPN Wi-Fi | `192.168.8.0/24` | Normal client traffic through AmneziaWG | Internet is blocked |
| `direct` / Direct Wi-Fi | `192.168.9.0/24` | Captive portals and direct diagnostics | Continues through WAN |

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
