<a id="russian-version"></a>

# OpenWrt на Cudy TR1200 v1

[English below](#english-version)

Руководство по установке и практической настройке **Cudy TR1200 v1**, составленное по результатам работы с OpenWrt 24.10.8.

Итоговая конфигурация включает:

- сеть через AmneziaWG с kill switch;
- отдельную Wi-Fi-сеть с прямым выходом в интернет;
- подключение роутера к гостиничному, домашнему или публичному Wi-Fi;
- приоритет Wi-Fi uplink над резервным Ethernet WAN;
- управление VPN боковым переключателем;
- белую/красную индикацию состояния VPN;
- резервное копирование, обновление, диагностику и восстановление.

> [!WARNING]
> Сторонняя прошивка может лишить гарантии или вывести устройство из строя. Инструкция предназначена только для **Cudy TR1200 hardware v1**. Перед началом проверьте модель и аппаратную ревизию на наклейке. Никогда не отключайте питание во время прошивки.

## 1. Устройство и прошивка

| Параметр | Значение |
|---|---|
| Модель | Cudy TR1200 v1 |
| Target | `ramips/mt76x8` |
| Проверенная версия | OpenWrt 24.10.8 (`r29233-443ec4032a`) |
| Официальный sysupgrade-образ | `openwrt-24.10.8-ramips-mt76x8-cudy_tr1200-v1-squashfs-sysupgrade.bin` |
| SHA-256 | `6c0a8d3844759b4f54cfee6c675e8e969be3d3a9e3e4cd06c319c411a5a47efb` |

Указанная контрольная сумма совпадает с официальным каталогом OpenWrt 24.10.8.

Официальные ресурсы:

- [страница устройства в OpenWrt](https://openwrt.org/toh/cudy/tr1200);
- [образы OpenWrt 24.10.8 для ramips/mt76x8](https://downloads.openwrt.org/releases/24.10.8/targets/ramips/mt76x8/);
- [загрузки Cudy TR1200 v1](https://www.cudy.com/en-us/pages/download-center/tr1200-1-0);
- [информация Cudy о промежуточных OpenWrt-прошивках](https://www.cudy.com/en-us/blogs/faq/openwrt-software-download).

Версии OpenWrt меняются. Для новой установки предпочтительна актуальная поддерживаемая стабильная версия после проверки страницы устройства и совместимости нужных модулей ядра.

## 2. Какие образы нужны

Для первой установки используются два разных файла:

1. **Подписанный Cudy промежуточный образ** — принимается штатным веб-интерфейсом Cudy. В проверенном комплекте он назывался `TR1200-OpenWRT-Flash.bin`.
2. **Официальный sysupgrade-образ OpenWrt** — устанавливается после загрузки промежуточной прошивки.

Не загружайте обычный sysupgrade-образ прямо в штатный интерфейс Cudy. Получите у Cudy промежуточную прошивку именно для своей аппаратной ревизии. Образ от другой модели или варианта flash использовать нельзя.

Если OpenWrt уже установлен, для обычного обновления применяется только совместимый sysupgrade-образ. Промежуточная прошивка повторно не нужна.

## 3. Первая установка

### Подготовка

1. Убедитесь, что на наклейке указано **TR1200 v1**.
2. Скачайте у Cudy правильную подписанную промежуточную прошивку.
3. Скачайте официальный sysupgrade-образ для `cudy_tr1200-v1`.
4. Проверьте контрольную сумму sysupgrade-образа.
5. Соедините компьютер и роутер Ethernet-кабелем.
6. Сохраните настройки Cudy, если они могут понадобиться.
7. Используйте стабильное питание и не прошивайте через ненадёжный Wi-Fi.

Linux/macOS:

```sh
sha256sum openwrt-24.10.8-ramips-mt76x8-cudy_tr1200-v1-squashfs-sysupgrade.bin
```

Windows PowerShell:

```powershell
Get-FileHash .\openwrt-24.10.8-ramips-mt76x8-cudy_tr1200-v1-squashfs-sysupgrade.bin -Algorithm SHA256
```

### Этап A: с заводской Cudy на промежуточную OpenWrt

1. Откройте штатную панель управления Cudy.
2. Перейдите на страницу обновления прошивки.
3. Выберите подписанный Cudy файл `TR1200-OpenWRT-Flash.bin`.
4. Запустите обновление и подождите несколько минут.
5. Старая страница Cudy может не показать успешное завершение: вместо неё уже загружается OpenWrt.
6. Переподключитесь по Ethernet и откройте `http://192.168.1.1`.

После первой загрузки OpenWrt беспроводная сеть может быть выключена, поэтому используйте Ethernet.

### Этап B: с промежуточной сборки на официальную OpenWrt

1. В LuCI откройте **System → Backup / Flash Firmware**.
2. Загрузите официальный файл `...cudy_tr1200-v1-squashfs-sysupgrade.bin`.
3. При переходе с промежуточной сборки на официальную **не сохраняйте настройки**.
4. Подтвердите обновление и дождитесь перезагрузки.
5. Снова откройте `http://192.168.1.1` и сразу задайте надёжный пароль root.

Если проверка образа не проходит, остановитесь и ещё раз проверьте модель, ревизию, имя файла и версию промежуточной прошивки. Не используйте принудительное обновление, если этого прямо не требует актуальная инструкция OpenWrt для данного устройства и вы не понимаете риск.

## 4. Рекомендуемая схема travel-router

Проверенная конфигурация использует две локальные сети:

| Сеть | Пример подсети | Назначение | При падении VPN |
|---|---:|---|---|
| `lan` / VPN Wi-Fi | `192.168.8.0/24` | Обычный трафик клиентов через AmneziaWG | Интернет блокируется |
| `direct` / Direct Wi-Fi | `192.168.9.0/24` | Captive portal и диагностика без VPN | Работает через WAN |

Физические uplink-интерфейсы:

- `wwan`: Wi-Fi-клиент, основной uplink, metric `100`;
- Ethernet WAN: резервный uplink, metric `200`.

VPN-клиенты используют AmneziaWG. Клиенты Direct отдельным policy routing направляются напрямую в физический uplink.

```text
                    Интернет
                        |
            +-----------+-----------+
            |                       |
      Wi-Fi uplink              Ethernet WAN
       metric 100                metric 200
            |                       |
            +--------- zone wan ----+
                        |
              +---------+---------+
              |                   |
        AmneziaWG VPN       прямой маршрут
              |               table 100
        VPN Wi-Fi             Direct Wi-Fi
       kill switch             без VPN
```

Используйте собственные нейтральные SSID и уникальные сложные пароли. Не копируйте названия и учётные данные из чужих примеров.

## 5. Kill switch

Главная логика firewall:

- разрешить forwarding из `lan` в firewall-зону VPN;
- **не разрешать** прямой forwarding из `lan` в `wan`;
- разрешить `direct → wan`;
- не направлять `direct` в VPN-зону.

Проверка с клиента VPN-сети:

1. Включите VPN и убедитесь, что внешний IP относится к VPN-маршруту.
2. Выполните на роутере `ifdown VPN`.
3. Интернет в VPN Wi-Fi должен исчезнуть.
4. Direct Wi-Fi должен продолжить работать при исправном uplink.
5. Выполните `ifup VPN` и проверьте восстановление.

Если VPN Wi-Fi продолжает выходить в интернет при выключенном туннеле, kill switch не работает. Ищите ошибочное правило `lan → wan`.

## 6. Direct и смена внешней сети

Direct нужен для captive portal в гостинице и диагностики. Policy rule направляет трафик Direct-подсети в таблицу `100`, где находится default route через физический uplink.

Проверка:

```sh
ip rule
ip route show table 100
ubus call network.interface.wwan status
ip -4 route show default
```

Не фиксируйте домашний или гостиничный gateway в конфигурации. При подключении к другой сети адрес шлюза обычно меняется. Обновляйте таблицу `100` по default route, полученному через DHCP, либо автоматизируйте это проверенным netifd hotplug-скриптом.

Для captive portal:

1. подключите телефон или ноутбук к Direct Wi-Fi;
2. подключите `radio1` как Station к внешнему Wi-Fi, назначив интерфейс `wwan` и firewall-зону `wan`;
3. откройте обычную HTTP-страницу, чтобы вызвать портал;
4. пройдите авторизацию;
5. включите и проверьте VPN-сеть.

## 7. Боковой переключатель VPN

На проверенном устройстве боковой переключатель создаёт hotplug-события `BTN_0`. Создайте `/etc/hotplug.d/button/99-slider-vpn`:

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

Сделайте файл исполняемым:

```sh
chmod +x /etc/hotplug.d/button/99-slider-vpn
```

Проверьте события:

```sh
logread -f | grep -E 'slider-vpn|BTN_0|button'
```

Если направление переключателя на вашем экземпляре обратное, поменяйте местами `ifup VPN` и `ifdown VPN`.

## 8. LED-индикация VPN

На проверенном устройстве использовались:

- `/sys/class/leds/white:status`;
- `/sys/class/leds/red:power`.

Логика:

- белый горит постоянно — VPN поднят, контрольный трафик проходит;
- белый мигает — интерфейс поднят, но проверка трафика не проходит;
- красный горит постоянно — VPN выключен.

Пример монитора находится в [`scripts/vpn-led-monitor`](scripts/vpn-led-monitor). Имена LED могут измениться, поэтому сначала выполните `ls /sys/class/leds/`. Публичный DNS-адрес используется только для проверки связи и при необходимости может быть заменён.

## 9. Ограничения ресурсов

У TR1200 v1 мало RAM и flash. В проверенной конфигурации:

- внутренний writable overlay составлял около 9,6 МиБ;
- после основной настройки оставалось около 7 МиБ;
- обычно было доступно около 55 МиБ RAM;
- Tailscale работал, но расходовал слишком много памяти;
- swap/zram не поддерживался проверенной конфигурацией ядра;
- USB extroot работал, но после удаления тяжёлых сервисов оказался не нужен.

Рекомендации:

- перед установкой пакетов проверяйте `free -m` и `df -h /overlay`;
- не выполняйте массовый `opkg upgrade`;
- переходите между версиями прошивки через sysupgrade;
- устанавливайте только необходимые сервисы;
- используйте kernel modules строго от запущенной версии ядра.

## 10. Backup и обновление

Перед существенными изменениями создавайте backup:

```sh
umask 077
sysupgrade -k -b /tmp/cudy-tr1200-backup.tar.gz
sha256sum /tmp/cudy-tr1200-backup.tar.gz
```

Сразу скачайте архив: `/tmp` находится в RAM и очищается при перезагрузке.

> [!CAUTION]
> Backup может содержать пароли Wi-Fi, приватные и preshared VPN-ключи и другие секреты. Никогда не публикуйте его на GitHub и не прикладывайте к открытым issue.

Перед sysupgrade:

```sh
ubus call system board
df -h /overlay
free -m
sysupgrade -k -b /tmp/pre-upgrade-backup.tar.gz
```

Проверьте точное соответствие образа модели, изучите release notes и убедитесь, что AmneziaWG доступен для нового ядра. После обновления протестируйте обе Wi-Fi-сети и kill switch.

## 11. Диагностика

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

Перед публикацией вывода удалите SSID, MAC-адреса, публичные IP, hostname, серийные номера, ключи и учётные данные.

Быстрая логика поиска проблемы:

- Direct работает, VPN Wi-Fi нет — проверяйте VPN, доступность endpoint, маршруты и логи;
- не работает ни одна сеть — проверяйте физический uplink и DHCP gateway;
- Direct сломался после смены гостиницы — ищите старый gateway в table `100`;
- VPN Wi-Fi работает при выключенном VPN — немедленно исправьте kill switch;
- остаётся слишком мало RAM — остановите или удалите тяжёлые процессы.

## 12. Failsafe и восстановление

Failsafe OpenWrt позволяет исправить network/firewall или сменить забытый пароль root:

1. подключитесь по Ethernet;
2. войдите в failsafe в начале загрузки согласно актуальной инструкции OpenWrt;
3. если DHCP выключен, задайте компьютеру статический адрес;
4. подключитесь к failsafe-адресу роутера;
5. выполните `mount_root` и исправьте только нужные файлы;
6. перезагрузите устройство.

Пример:

```sh
mount_root
passwd
vi /etc/config/network
vi /etc/config/firewall
reboot -f
```

Не придумывайте последовательность TFTP/Reset: восстановление через bootloader зависит от конкретного устройства. Для возврата на Cudy firmware используйте актуальную инструкцию Cudy; у TR1200 есть особые требования к используемому Ethernet-порту.

## Проверка безопасности

- Задайте сложный уникальный пароль root.
- Не открывайте LuCI и SSH в `wan`/`wwan`.
- Не публикуйте `/etc/config/network` без проверки на приватные ключи.
- Никогда не публикуйте backup роутера.
- Перевыпустите VPN-ключи при подозрении на утечку.
- Читайте сторонние скрипты перед запуском.
- Храните офлайн-копию для восстановления и проверенные контрольные суммы.

## Лицензия и товарные знаки

Документация предоставляется без гарантий. OpenWrt является товарным знаком проекта OpenWrt; названия Cudy и AmneziaWG принадлежат соответствующим владельцам. Руководство не означает одобрения со стороны производителей.

---

<a id="english-version"></a>

# OpenWrt on Cudy TR1200 v1

[Russian version above](#russian-version)

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
