---
name: printer-server-arm
description: >
  Cài đặt Print Server trên mạch nhúng (Orange Pi, Raspberry Pi, Armbian, Debian ARM)
  cho máy in USB-only (Brother DCP-T220, DCP-T420W, Epson EcoTank GDI...).
  Actions: setup, install, configure, troubleshoot, scan-setup, add-client.
  Keywords: printer server, print server, CUPS, ipp-usb, driverless, IPP Everywhere,
  AirPrint, eSCL scan, netplan static IP, Armbian, Orange Pi, Raspberry Pi, armv7,
  Brother DCP-T220, máy in qua mạng, in qua mạng, print server nhúng.
---

# Printer Server trên mạch nhúng — In & Scan qua mạng cho máy in USB-only

> Skill này tổng hợp quy trình **đã test thực tế** trên Orange Pi PC Plus (Armbian 26 / Debian 13 trixie, armv7l) với máy in **Brother DCP-T220** (USB-only, mực liên tục). Áp dụng cho mọi mạch nhúng ARM chạy Debian/Armbian và máy in không có WiFi/LAN.

## Giới thiệu

Bài toán: máy in chỉ có cổng USB (như Brother DCP-T220) — muốn cả nhà in & scan qua mạng LAN từ điện thoại, laptop.

Giải pháp: biến mạch nhúng thành **print server** với:

- **CUPS** — dịch vụ in chính, share qua port 631
- **ipp-usb** — "dịch" giao diện IPP-over-USB của máy in ra mạng (port 60000)
- **Avahi (mDNS)** — tự broadcast máy in ra LAN (AirPrint/Mopria)
- **sane-airscan** — scan qua mạng bằng chuẩn eSCL (driverless)
- **Driver**: dùng PPD pre-generated `pwgrast` (Generic IPP Everywhere) — vì máy in GDI giá rẻ KHÔNG cài được driver chính thức của hãng (chỉ có bản i386) và `-m everywhere` của CUPS cũng fail với máy thiếu attributes

### Kiến trúc

```
[Phone/Mac/Win] --WiFi/LAN--> [Board: CUPS:631] --IPP over USB--> [Printer USB]
[Phone/Mac] --AirPrint/mDNS--> [Board]                [Scanner: eSCL :60000]
```

## HƯỚNG DẪN CÀI ĐẶT (Tiếng Việt)

### Bước 0 — Kiểm tra máy mạch

```bash
ssh root@<IP-board>
uname -m            # armv7l hoặc aarch64
lsusb               # thấy máy in, ví dụ: 04f9:0474 Brother DCP-T220
ls /dev/usb/lp*     # kernel đã nhận /dev/usb/lp0
free -m             # RAM >= 1GB là ổn
docker ps -a        # đảm bảo không container nào chiếm /dev/usb/lp0
```

### Bước 1 — IP tĩnh qua Netplan (QUAN TRỌNG)

CẢNH BÁO THỨ TỰ FILE: Netplan merge theo thứ tự tên file alphabetically. File đặt tên `20-...` hay `99-...` vẫn đứng TRƯỚC `armbian.yaml` (chữ 'a') nên config bị đè ngược. PHẢI đặt tên bắt đầu sau chữ 'a' như `zz-`.

Tạo file `/etc/netplan/zz-static-wlan.yaml`:

```yaml
network:
  version: 2
  renderer: networkd
  wifis:
    wlan0:
      dhcp4: false
      dhcp6: false
      addresses: [<IP-tĩnh>/24]
      routes:
        - to: default
          via: <gateway>
          metric: 600
      nameservers:
        addresses: [<DNS-1>, 1.1.1.1]
      access-points:
        "<TênWiFi>":
          auth:
            key-management: "psk"
            password: "<mật-khâu-wifi>"
```

Nếu dùng cổng LAN `end0`: thay `wifis:` bằng `ethernets: end0:` và bỏ phần access-points.

Áp dụng (IP giữ nguyên nên SSH không đứt):

```bash
cp /etc/netplan/armbian.yaml /etc/netplan/armbian.yaml.bak
netplan generate && netplan apply
ip -4 a show wlan0    # phải thấy "scope global" KHÔNG có chữ "dynamic"
ip route              # chỉ còn "proto static"
```

Nếu còn lease DHCP sót: `rm -f /run/systemd/netif/leases/* && systemctl restart systemd-networkd`

### Bước 2 — Cài gói

```bash
apt update
apt install -y cups cups-filters ipp-usb avahi-daemon sane-airscan sane-utils colord
```

Lưu ý:
- Debian không có gói `sane` / `printer-driver-ipp-everywhere` — dùng `sane-utils` + `cups-filters` là đủ.
- Chạy apt DETACHED qua SSH để khỏi bị kill giữa chừng:
  ```bash
  setsid nohup sh -c 'DEBIAN_FRONTEND=noninteractive apt-get install -y -qq cups cups-filters > /root/cups-install.log 2>&1; echo EXIT=$? >> /root/cups-install.log' </dev/null >/dev/null 2>&1 &
  tail -f /root/cups-install.log   # đợi tới EXIT=0
  ```

### Bước 3 — Expose IPP-over-USB ra LAN

Sửa `/etc/ipp-usb/ipp-usb.conf`, trong đoạn `[network]`:

```ini
[network]
  interface = all        # mặc định là loopback — bắt buộc đổi thành all
```

```bash
cp /etc/ipp-usb/ipp-usb.conf /etc/ipp-usb/ipp-usb.conf.bak
sed -i 's/interface = loopback/interface = all/' /etc/ipp-usb/ipp-usb.conf
systemctl restart ipp-usb
ipp-usb status   # phải thấy "status: OK" cho máy in
```

### Bước 4 — Thêm queue máy in (Workaround PPD)

CẢNH BÁO DRIVER: `lpadmin -m everywhere` sẽ báo "Printer does not support required IPP attributes or document formats" với máy in GDI giá rẻ (thiếu attributes mà CUPS 2.4.7+ đòi). GIẢI PHÁP: dùng PPD pre-generated.

```bash
# Xóa queue lỗi cũ nếu có
lpadmin -x Brother_T220

# Tạo queue với PPD generic IPP Everywhere (render PWG-Raster local)
lpadmin -p Brother_T220 -E \
  -v ipp://localhost:60000/ipp/print \
  -m drv:///cupsfilters.drv/pwgrast.ppd

lpadmin -p Brother_T220 -D "Brother DCP-T220" -L "Print Server (OrangePi)"
lpadmin -p Brother_T220 -o printer-is-shared=true
cupsenable Brother_T220 && cupsaccept Brother_T220
cupsctl --remote-any          # cho phép client LAN truy cập 631
```

### Bước 5 — Test in

```bash
lp -d Brother_T220 /usr/share/cups/data/default-testpage.pdf
sleep 15
lpstat -W completed    # job phải xuất hiện ở đây
```

### Bước 6 — Setup scan

```bash
timeout 30 scanimage -L        # phải thấy airscan:... Brother DCP-T220
timeout 60 scanimage -d 'airscan:e0:Brother DCP-T220 (USB)' \
  --format=png --resolution 150 -o /tmp/scan-test.png
ls -la /tmp/scan-test.png      # file PNG > 0 byte = OK
```

Scanner cũng tự expose qua mạng: `http://<IP-board>:60000/eSCL/`

### Bước 7 — Add máy in từ client

| Nền tảng | Cách |
|---|---|
| iPhone/iPad/Mac | AirPrint — chọn `Brother DCP-T220 @ orangepipcplus` |
| Windows 11 | Settings -> Printers -> Add device (tự tìm mDNS); thủ công: `http://<IP>:631/printers/Brother_T220` |
| Android | Default Print Service / Mopria (tự tìm) |
| Linux | `lpadmin -p Brother_T220 -v ipp://<IP>:631/printers/Brother_T220 -E -m everywhere` |
| Scan Win/Mac/iOS | App scan -> chọn `Brother DCP-T220 (USB)` (eSCL) |

Verify từ máy Linux/Mac khác trong LAN:

```bash
dns-sd -B _ipp._tcp        # macOS — phải thấy 2 instance
avahi-browse -rt _ipp._tcp # Linux
```

### Quản trị & bảo trì

- **Web UI**: `http://<IP-board>:631` (login root)
- **Log**: `/var/log/cups/error_log`, `/var/log/ipp-usb/`
- **Backup đã tạo**: `/etc/netplan/armbian.yaml.bak`, `/etc/ipp-usb/ipp-usb.conf.bak`
- **Đổi dải IP sau này**: sửa `/etc/netplan/zz-static-wlan.yaml` rồi `netplan apply`

## INSTALLATION GUIDE (English)

### Overview

Turn an embedded ARM board (Orange Pi, Raspberry Pi, any Armbian/Debian ARM) into a network print server for USB-only printers (Brother DCP-T220 ink-tank and similar GDI printers). Clients print via AirPrint/IPP/mDNS, scan via eSCL — no vendor drivers needed.

Stack: CUPS (port 631) + ipp-usb (IPP-over-USB to network, port 60000) + Avahi (mDNS broadcast) + sane-airscan (eSCL scanning) + pre-generated `pwgrast` PPD (Generic IPP Everywhere).

### Step 0 — Inspect the board

```bash
ssh root@<board-ip>
uname -m                 # armv7l or aarch64
lsusb                    # printer visible, e.g. 04f9:0474 Brother DCP-T220
ls /dev/usb/lp*          # kernel sees /dev/usb/lp0
docker ps -a             # ensure no container claims /dev/usb/lp0
```

### Step 1 — Static IP via Netplan

WARNING (file-ordering trap): Netplan merges files alphabetically. Any numeric prefix (`20-`, `99-`) sorts BEFORE `armbian.yaml` and gets overridden. Name your override `zz-*.yaml` so it wins the merge.

Create `/etc/netplan/zz-static-wlan.yaml` (WiFi example — use `ethernets:` for LAN):

```yaml
network:
  version: 2
  renderer: networkd
  wifis:
    wlan0:
      dhcp4: false
      dhcp6: false
      addresses: [192.168.2.19/24]
      routes:
        - to: default
          via: 192.168.2.1
          metric: 600
      nameservers:
        addresses: [192.168.2.1, 1.1.1.1]
      access-points:
        "your-ssid":
          auth:
            key-management: "psk"
            password: "wifi-password"
```

```bash
cp /etc/netplan/armbian.yaml /etc/netplan/armbian.yaml.bak
netplan generate && netplan apply
ip -4 a show wlan0       # "scope global" WITHOUT "dynamic" flag
ip route                 # only "proto static" default route
# stale DHCP lease: rm -f /run/systemd/netif/leases/* && systemctl restart systemd-networkd
```

### Step 2 — Install packages

```bash
apt update
apt install -y cups cups-filters ipp-usb avahi-daemon sane-airscan sane-utils colord
```

Notes: Debian has no `sane` / `printer-driver-ipp-everywhere` packages — `sane-utils` + `cups-filters` cover everything. Over SSH, run apt detached (nohup/setsid) so a dropped connection cannot kill a half-install.

### Step 3 — Expose IPP-over-USB to the LAN

In `/etc/ipp-usb/ipp-usb.conf`, under `[network]`: change `interface = loopback` to `interface = all`, then `systemctl restart ipp-usb`. Verify with `ipp-usb status` (device must show `status: OK`).

### Step 4 — Create the print queue (PPD workaround)

`lpadmin -m everywhere` fails on budget GDI printers with "Printer does not support required IPP attributes or document formats" (missing IPP attributes required by CUPS 2.4.7+; vendor drivers are i386-only). Use the pre-generated generic PPD instead:

```bash
lpadmin -p Brother_T220 -E \
  -v ipp://localhost:60000/ipp/print \
  -m drv:///cupsfilters.drv/pwgrast.ppd
lpadmin -p Brother_T220 -D "Brother DCP-T220" -o printer-is-shared=true
cupsenable Brother_T220 && cupsaccept Brother_T220
cupsctl --remote-any
```

### Step 5 — Test print and Step 6 — Test scan

```bash
lp -d Brother_T220 /usr/share/cups/data/default-testpage.pdf && sleep 15 && lpstat -W completed
timeout 30 scanimage -L                       # discover scanner
timeout 60 scanimage -d 'airscan:e0:Brother DCP-T220 (USB)' \
  --format=png --resolution 150 -o /tmp/scan-test.png
```

### Step 7 — Connect clients

- AirPrint (iOS/macOS): printer appears as `Brother DCP-T220 @ <hostname>`
- Windows: auto-discovered via mDNS, or manual `http://<board-ip>:631/printers/Brother_T220`
- Android: Mopria / Default Print Service
- Linux client: `lpadmin -p Brother_T220 -v ipp://<board-ip>:631/printers/Brother_T220 -E -m everywhere`
- Admin UI: `http://<board-ip>:631` — jobs, defaults, logs

## Troubleshooting (các lỗi đã gặp thật / Real issues encountered)

| Hiện tượng | Nguyên nhân | Cách xử lý |
|---|---|---|
| `netplan apply` xong vẫn còn `dynamic` | File `99-*.yaml` bị `armbian.yaml` đè (thứ tự alphabet) | Đổi tên thành `zz-*.yaml`, re-apply, xóa lease cũ |
| `lpadmin: Unable to create PPD: Printer does not support required IPP attributes` | Máy in GDI thiếu attributes IPP Everywhere | Dùng `-m drv:///cupsfilters.drv/pwgrast.ppd` |
| Job vào queue rồi biến mất, không in | SSH đứt lúc apt đang chạy (half-install), hoặc queue tạo bằng `-m everywhere` fail | Kiểm tra `dpkg --audit`; tạo lại queue với pwgrast; bật `cupsctl --debug-logging` để xem filter chain |
| `scanimage -L` treo | Backend khác chờ timeout | Wrap: `timeout 30 scanimage -L`; đảm bảo ipp-usb `interface = all` |
| Client tìm không thấy máy in | Queue chưa share | `lpadmin -p ... -o printer-is-shared=true` + `cupsctl --remote-any` |
| apt bị kill giữa chừng qua SSH | SSH disconnect -> SIGHUP | Chạy detached: `setsid nohup apt-get install ... &`, poll log |
| In ra trang trắng/rác | Queue dùng driver sai (raw/generic PostScript) | Đảm bảo `-v ipp://localhost:60000/ipp/print` + PPD `pwgrast` |

## Các file cấu hình quan trọng / Key config files

```
/etc/netplan/zz-static-wlan.yaml          # IP tĩnh
/etc/ipp-usb/ipp-usb.conf                 # interface = all
/etc/cups/cupsd.conf                      # Port 631, remote sharing
/var/log/cups/error_log                   # CUPS log
/var/log/ipp-usb/<model>.log              # per-device IPP-over-USB log
```

## Bảng tương thích / Compatibility notes

- Tested: Orange Pi PC Plus (H3, armv7l) + Armbian 26.2 (Debian 13 trixie) + Brother DCP-T220 — in OK, scan OK, AirPrint OK
- Máy in CẦN hỗ trợ IPP-over-USB (phần lớn Brother ink-tank, nhiều model Epson/HP sau 2015). Check nhanh: `ipp-usb status` báo OK.
- Máy in có WiFi riêng -> không cần skill này, dùng driverless trực tiếp.
- RAM tối thiểu khuyến nghị 512MB; 1GB dư dả (~830MB available khi chạy full stack).
