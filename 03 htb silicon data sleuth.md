# Silicon Data Sleuth — HTB Walkthrough

**Kategori:** Forensics (Sherlocks)
**Difficulty:** Easy
**Platform:** HackTheBox

> Walkthrough edukatif. Jawaban akhir sengaja tidak dicantumkan — kamu solve sendiri dengan mengikuti workflow.

---

## Deskripsi Challenge

> Di antara debu dan pasir yang mengelilingi vault, kamu menemukan sebuah PCB berkarat... Kamu mencoba membaca tulisan yang tergores di atasnya, tertulis Open..W...RT, sebuah router! Kamu serahkan ke ahli hardware dan ternyata chip ROM-nya masih utuh! Mereka berhasil membaca data dari silikon yang usang itu dan memberikan sebuah firmware image. Sekarang tugasmu adalah memeriksa firmware itu dan mungkin memulihkan informasi berguna yang penting untuk membuka dan melewati sistem keamanan vault!

**Diberikan:** `chal_router_dump.bin` — dump firmware mentah 16 MB dari router OpenWRT.
**Interface:** Service netcat yang bertanya secara berurutan tentang konfigurasi router.

---

## Overview

Challenge ini adalah latihan analisis firmware. Dump-nya berupa raw ROM image yang berisi bootloader, kernel, factory rootfs, dan writable overlay. Informasi konfigurasi — credential, network setup, firewall rule — semuanya terkubur di dalam filesystem dan archive yang bertingkat, dan tiap pertanyaan netcat menargetkan satu potongan informasi yang berbeda.

---

## Tools yang Dibutuhkan

```bash
sudo apt update
sudo apt install binwalk squashfs-tools python3-pip -y
pip3 install jefferson --break-system-packages
```

| Tool | Kegunaan |
|------|----------|
| `file` | Identifikasi tipe file |
| `binwalk` | Scan signature filesystem/binary yang ter-embed |
| `dd` | Potong byte range mentah dari dump |
| `unsquashfs` | Extract filesystem SquashFS |
| `jefferson` | Extract filesystem JFFS2 |
| `tar` | Extract archive .tgz / .tar.gz |
| `strings` | Ambil text yang bisa dibaca dari binary |

---

## Step 1 — Recon Awal

Mulai dari identifikasi dasar:

```bash
file chal_router_dump.bin
```

Hasil: `data`. Ga ada header yang dikenal — ini binary mentah, bukan format yang ter-wrap.

Scan signature yang ter-embed:

```bash
binwalk chal_router_dump.bin
```

Temuan yang diharapkan (layout khas OpenWRT):

| Offset | Deskripsi |
|--------|-----------|
| `0x17DA0` | String bootloader U-Boot |
| `0x180000` | uImage — kernel MIPS OpenWrt Linux |
| `0x42C2C8` | **SquashFS** filesystem (rootfs read-only) |
| `0x7C0000` | **JFFS2** filesystem (writable overlay) |

Dua filesystem — ini insight kunci:
- **SquashFS** = default pabrik, read-only
- **JFFS2** = overlay writable yang berisi config hasil modifikasi user

Konfigurasi asli yang aktif tersimpan di JFFS2, bukan di SquashFS.

---

## Step 2 — Extract SquashFS Rootfs

Potong partisi SquashFS pake offset dari `binwalk`:

```bash
dd if=chal_router_dump.bin of=squashfs.img bs=1 skip=<offset_decimal>
unsquashfs squashfs.img
```

Eksplor `squashfs-root/`:

```bash
ls squashfs-root/etc/
cat squashfs-root/etc/openwrt_release
cat squashfs-root/etc/shadow
```

Observasi: file `shadow` di sini nampilin **kondisi default pabrik** — field password mungkin kosong. Ini normal. Kalau pertanyaannya soal hash, shadow SquashFS ini bukan jawabannya. Lanjut.

---

## Step 3 — Extract JFFS2 Overlay

```bash
dd if=chal_router_dump.bin of=jffs2.img bs=1 skip=<offset_decimal>
jefferson jffs2.img -d jffs2_out
```

Lihat isinya:

```bash
find jffs2_out -type f | head -30
```

File penting yang harus diperhatikan:

```
jffs2_out/upper/sysupgrade.tgz
```

**`sysupgrade.tgz`** itu archive spesial OpenWRT — dia bundel backup dari semua file config yang udah dimodifikasi user, dibuat waktu router di-setup. Di sinilah konfigurasi asli yang aktif tersimpan.

---

## Step 4 — Extract sysupgrade.tgz

```bash
mkdir sysupgrade_extract
tar -xzf jffs2_out/upper/sysupgrade.tgz -C sysupgrade_extract
```

Sekarang `sysupgrade_extract/` berisi directory tree Linux lengkap. File yang menarik:

```bash
ls sysupgrade_extract/etc/
ls sysupgrade_extract/etc/config/
```

File kunci buat pertanyaan CTF:

| File | Berisi |
|------|--------|
| `etc/shadow` | Akun user + hash password |
| `etc/config/network` | Config interface (LAN, WAN, credential PPPoE) |
| `etc/config/wireless` | SSID WiFi, encryption, key |
| `etc/config/firewall` | Zone firewall dan aturan port forwarding |
| `etc/config/dhcp` | DHCP server, DNS, static lease |
| `etc/config/system` | Hostname, timezone, setting log |
| `etc/dropbear/` | Host key SSH |
| `etc/openwrt_release` | Metadata versi OpenWRT |

---

## Menjawab Pertanyaan

Tiap pertanyaan netcat memetakan ke file config tertentu. Workflow umumnya:

1. Baca pertanyaan. Identifikasi informasi jenis apa yang diminta (auth? network? wireless? firewall?).
2. Buka file config yang cocok pake `cat`.
3. Cari baris `option` atau `list` yang relevan.
4. Submit nilainya.

### Panduan Mapping Pertanyaan

| Topik Pertanyaan | Cek di Mana |
|------------------|-------------|
| Versi OpenWRT | `etc/openwrt_release` — `DISTRIB_RELEASE` |
| Hash password root | `etc/shadow` — baris yang dimulai `root:` |
| Credential PPPoE | `etc/config/network` — interface `wan`, `option username` / `option password` |
| SSID WiFi / key / encryption | `etc/config/wireless` — section `wifi-iface` |
| Port forwarding | `etc/config/firewall` — blok `config redirect` |
| DHCP / DNS | `etc/config/dhcp` |
| Static host / lease | `etc/config/dhcp` |
| Fingerprint host key SSH | `etc/dropbear/` |

### Tips Format Jawaban

- Beberapa jawaban minta **seluruh baris** persis (misal entry shadow). Yang lain cuma nilai di dalam quote-nya aja.
- Jawaban numerik mungkin harus **diurutkan numerik** dan **dipisah koma tanpa spasi**. Baca contoh format di pertanyaan baik-baik.
- String versi kadang diminta dalam format `X.YY.Z` meskipun string di file pake suffix beda.

---

## Script Reproduksi Lengkap

```bash
#!/bin/bash
# Workflow ekstraksi Silicon Data Sleuth
set -e

# Install dependencies
sudo apt update
sudo apt install binwalk squashfs-tools python3-pip -y
pip3 install jefferson --break-system-packages

# Scan
echo "=== Scan firmware ==="
binwalk chal_router_dump.bin

# Extract SquashFS (sesuaikan offset dari output binwalk)
SQUASHFS_OFFSET=<dari_binwalk>
dd if=chal_router_dump.bin of=squashfs.img bs=1 skip=$SQUASHFS_OFFSET
unsquashfs squashfs.img

# Extract JFFS2 (sesuaikan offset dari output binwalk)
JFFS2_OFFSET=<dari_binwalk>
dd if=chal_router_dump.bin of=jffs2.img bs=1 skip=$JFFS2_OFFSET
jefferson jffs2.img -d jffs2_out

# Extract bundle config user
mkdir sysupgrade_extract
tar -xzf jffs2_out/upper/sysupgrade.tgz -C sysupgrade_extract

# Eksplor
echo "=== Eksplor config yang udah di-extract ==="
ls sysupgrade_extract/etc/config/
```

---

## Insight Kunci

1. **Layout firmware itu berlapis.** Factory rootfs (SquashFS) sifatnya read-only dan berisi default. Konfigurasi user yang beneran ada di writable overlay (JFFS2).

2. **Pola `sysupgrade.tgz`.** Device berbasis OpenWRT biasa nyimpen archive backup config di overlay JFFS2 dengan nama `sysupgrade.tgz`. Selalu cari archive di dalam partisi overlay.

3. **Ekstraksi bertingkat itu normal.** Challenge ini butuh tiga lapis ekstraksi:
   ```
   chal_router_dump.bin
   └── partisi JFFS2
        └── upper/sysupgrade.tgz
             └── etc/{shadow, config/*}
   ```

4. **Cocokin tool sama format-nya.** Filesystem yang beda butuh extractor yang beda:
   - SquashFS → `unsquashfs`
   - JFFS2 → `jefferson`
   - `.tgz` → `tar -xzf`
   - Byte range mentah → `dd`

5. **Extension `.bin` itu ga bermakna.** Selalu jalanin `file` dan `binwalk` dulu — jangan pernah asumsi format dari extension-nya doang.

6. **Config OpenWRT itu seragam.** Semua device OpenWRT nyimpen runtime config di `/etc/config/` dalam bentuk file UCI yang readable. Sekali paham layout file-nya, ekstraksi informasi dari dump OpenWRT manapun bakal ngikutin pola yang sama.

---

## Referensi

- [Dokumentasi OpenWRT](https://openwrt.org/docs/start)
- [Binwalk GitHub](https://github.com/ReFirmLabs/binwalk)
- [Jefferson (extractor JFFS2)](https://github.com/sviehb/jefferson)
- [Sistem UCI OpenWRT](https://openwrt.org/docs/guide-user/base-system/uci)
