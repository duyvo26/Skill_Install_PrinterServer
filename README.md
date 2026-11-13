# printer-server-arm

Embedded Print Server for USB-only printers — turn an ARM single-board computer (Orange Pi, Raspberry Pi, any Armbian/Debian ARM device) into a network print + scan server, with zero vendor drivers.

*Language / Ngôn ngữ: [English](#english) | [Tiếng Việt](#tiếng-việt)*

---

# English

## What is this?

Many budget printers (e.g. **Brother DCP-T220** ink-tank) have only a USB port — no WiFi, no Ethernet. This project turns a cheap ARM board into a **print server** so every device on your LAN can print and scan over the network:

- **Print** from iPhone/iPad/Mac (AirPrint), Windows (mDNS/IPP), Android (Mopria), Linux (IPP)
- **Scan** from Windows/Mac/mobile via eSCL (driverless)
- **No vendor drivers required** — printers without native Linux ARM drivers still work

### Architecture

```
[Phone/Laptop] --WiFi/LAN--> [ARM Board] --USB--> [Printer]
                                |--- CUPS        :631   (print queue, web admin)
                                |--- ipp-usb     :60000 (IPP-over-USB -> network)
                                |--- Avahi/mDNS  (AirPrint/Mopria broadcast)
                                |--- sane-airscan       (eSCL scanner)
```

### Tested hardware

| Component | Tested with | Status |
|---|---|---|
| Board | Orange Pi PC Plus (Allwinner H3, armv7l) | OK |
| OS | Armbian 26.2 (Debian 13 "trixie", kernel 6.18) | OK |
| Printer | Brother DCP-T220 (USB ink-tank) | Print OK, Scan OK, AirPrint OK |
| Memory | 1 GB RAM (~830 MB available with full stack) | OK |

Requirements: ARM board with USB host port, Debian/Armbian-based OS, printer that supports **IPP-over-USB** (most Brother ink-tank and many Epson/HP models from ~2015+).

## Quick start

Full step-by-step guide: [SKILL.md](SKILL.md)

```bash
# 1. Static IP (see SKILL.md Step 1 — netplan file MUST be named zz-*.yaml)

# 2. Install packages
apt update
apt install -y cups cups-filters ipp-usb avahi-daemon sane-airscan sane-utils colord

# 3. Expose IPP-over-USB to LAN
sed -i 's/interface = loopback/interface = all/' /etc/ipp-usb/ipp-usb.conf
systemctl restart ipp-usb && ipp-usb status   # device must show "status: OK"

# 4. Create print queue (pre-generated PPD workaround)
lpadmin -p Brother_T220 -E \
  -v ipp://localhost:60000/ipp/print \
  -m drv:///cupsfilters.drv/pwgrast.ppd
lpadmin -p Brother_T220 -o printer-is-shared=true
cupsenable Brother_T220 && cupsaccept Brother_T220
cupsctl --remote-any

# 5. Test print
lp -d Brother_T220 /usr/share/cups/data/default-testpage.pdf

# 6. Test scan
timeout 30 scanimage -L
```

### Connect from clients

| Platform | Method |
|---|---|
| iPhone / iPad / macOS | AirPrint — auto-discovered |
| Windows 10/11 | Add printer (mDNS auto) or `http://<board-ip>:631/printers/Brother_T220` |
| Android | Default Print Service / Mopria |
| Linux | `lpadmin -p Brother_T220 -v ipp://<board-ip>:631/printers/Brother_T220 -E -m everywhere` |
| Admin UI | `http://<board-ip>:631` (root login) |
| Scan | Windows Scan / Image Capture / mobile scan app — eSCL device |

## Why not the official vendor driver?

1. Brother (and most vendors) ship Linux printer drivers as **i386 binaries only** — they do not run on ARM (armv7/aarch64).
2. CUPS 2.4.7+'s built-in `lpadmin -m everywhere` fails on budget GDI printers: *"Printer does not support required IPP attributes or document formats"*.
3. **Workaround**: the pre-generated generic PPD (`drv:///cupsfilters.drv/pwgrast.ppd`) renders everything locally to PWG-Raster, which the printer natively accepts over IPP-over-USB.

## Troubleshooting

See the troubleshooting table in [SKILL.md](SKILL.md#troubleshooting-cac-loi-da-gap-that--real-issues-encountered) — covers: netplan file ordering, PPD creation failure, jobs stuck/missing, scanimage hangs, SSH-killed apt installs, and wrong-driver output.

## Files

| File | Purpose |
|---|---|
| `SKILL.md` | Agent-installable skill: bilingual (VN/EN) step-by-step install guide + troubleshooting |
| `README.md` | This file — project introduction for GitHub |

---

# Tiếng Việt

## Dự án này là gì?

Nhiều dòng máy in giá rẻ (ví dụ **Brother DCP-T220** mực liên tục) chỉ có cổng USB — không WiFi, không LAN. Dự án này biến một mạch nhúng ARM giá rẻ thành **máy chủ in** để mọi thiết bị trong mạng LAN đều in và scan qua mạng:

- **In** từ iPhone/iPad/Mac (AirPrint), Windows (mDNS/IPP), Android (Mopria), Linux (IPP)
- **Scan** từ Windows/Mac/điện thoại qua eSCL (driverless)
- **Không cần driver hãng** — máy in không có driver Linux cho ARM vẫn in được

### Kiến trúc

```
[Điện thoại/Laptop] --WiFi/LAN--> [Mạch ARM] --USB--> [Máy in]
                                 |--- CUPS        :631   (hàng đợi in, quản trị web)
                                 |--- ipp-usb     :60000 (IPP-over-USB -> mạng)
                                 |--- Avahi/mDNS  (broadcast AirPrint/Mopria)
                                 |--- sane-airscan       (scanner eSCL)
```

### Phần cứng đã test

| Thành phần | Đã test với | Kết quả |
|---|---|---|
| Mạch | Orange Pi PC Plus (Allwinner H3, armv7l) | OK |
| Hệ điều hành | Armbian 26.2 (Debian 13 "trixie", kernel 6.18) | OK |
| Máy in | Brother DCP-T220 (mực liên tục, USB) | In OK, Scan OK, AirPrint OK |
| RAM | 1 GB (~830 MB còn trống khi chạy đủ stack) | OK |

Yêu cầu: mạch ARM có cổng USB host, OS Debian/Armbian, máy in hỗ trợ **IPP-over-USB** (hầu hết Brother mực liên tục, nhiều model Epson/HP từ ~2015 trở đi).

## Cài đặt nhanh

Hướng dẫn đầy đủ từng bước: [SKILL.md](SKILL.md)

```bash
# 1. IP tĩnh (xem SKILL.md Bước 1 — file netplan PHẢI đặt tên zz-*.yaml)

# 2. Cài gói
apt update
apt install -y cups cups-filters ipp-usb avahi-daemon sane-airscan sane-utils colord

# 3. Expose IPP-over-USB ra LAN
sed -i 's/interface = loopback/interface = all/' /etc/ipp-usb/ipp-usb.conf
systemctl restart ipp-usb && ipp-usb status   # máy in phải hiện "status: OK"

# 4. Tạo queue in (workaround PPD pre-generated)
lpadmin -p Brother_T220 -E \
  -v ipp://localhost:60000/ipp/print \
  -m drv:///cupsfilters.drv/pwgrast.ppd
lpadmin -p Brother_T220 -o printer-is-shared=true
cupsenable Brother_T220 && cupsaccept Brother_T220
cupsctl --remote-any

# 5. Test in
lp -d Brother_T220 /usr/share/cups/data/default-testpage.pdf

# 6. Test scan
timeout 30 scanimage -L
```

### Kết nối từ client

| Nền tảng | Cách |
|---|---|
| iPhone / iPad / macOS | AirPrint — tự động tìm thấy |
| Windows 10/11 | Add printer (tự tìm mDNS) hoặc `http://<IP-board>:631/printers/Brother_T220` |
| Android | Default Print Service / Mopria |
| Linux | `lpadmin -p Brother_T220 -v ipp://<IP-board>:631/printers/Brother_T220 -E -m everywhere` |
| Quản trị | `http://<IP-board>:631` (login root) |
| Scan | Windows Scan / Image Capture / app scan mobile — thiết bị eSCL |

## Tại sao không dùng driver chính thức của hãng?

1. Brother (và phần lớn hãng) chỉ phát hành driver Linux dạng **binary i386** — không chạy trên ARM (armv7/aarch64).
2. `lpadmin -m everywhere` của CUPS 2.4.7+ báo lỗi với máy in GDI giá rẻ: *"Printer does not support required IPP attributes or document formats"*.
3. **Giải pháp**: PPD generic pre-generated (`drv:///cupsfilters.drv/pwgrast.ppd`) render toàn bộ về PWG-Raster ngay trên server — định dạng máy in chấp nhận sẵn qua IPP-over-USB.

## Xử lý sự cố

Xem bảng troubleshooting trong [SKILL.md](SKILL.md) — bao gồm: thứ tự file netplan, lỗi tạo PPD, job kẹt/biến mất, `scanimage` treo, apt bị kill khi SSH đứt, in ra trang trắng do driver sai.

## Các file trong repo

| File | Mục đích |
|---|---|
| `SKILL.md` | Skill cài được vào agent: hướng dẫn song ngữ (VN/EN) + troubleshooting |
| `README.md` | File này — giới thiệu dự án cho GitHub |
