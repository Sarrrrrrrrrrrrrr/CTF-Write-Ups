# An Unusual Sighting — HTB Walkthrough

**Kategori:** Forensics (Sherlocks)
**Difficulty:** Very Easy
**Platform:** HackTheBox

> Walkthrough edukatif. Jawaban akhir sengaja tidak dicantumkan — kamu solve sendiri dengan mengikuti workflow.

---

## Deskripsi Challenge

> As the preparations come to an end, and The Fray draws near each day, our newly established team has started work on refactoring the new CMS application. However, after some time we noticed that a lot of our work mysteriously has been disappearing! We managed to extract the SSH Logs and the Bash History from our dev server in question. The faction that manages to uncover the perpetrator will have a massive bonus come the competition!
>
> **Note:** Operating Hours of Korp: 0900 - 1900
> **Note 2:** All timestamps are in the format they appear in the logs

**Diberikan:**
- `sshd.log` — log SSH server
- `bash_history` — riwayat command yang dijalankan di server

**Interface:** Netcat service yang bertanya secara berurutan.

---

## Overview

Tim development mengalami kejadian aneh: pekerjaan mereka "hilang" secara misterius. Log SSH dan bash history dari dev server telah diekstrak. Tugas kamu adalah menganalisis log-log itu, cari anomali, identifikasi attacker, dan rekonstruksi timeline serangannya.

**Petunjuk penting:** Jam kerja normal tim adalah **09:00 - 19:00**. Aktivitas di luar jam itu = kemungkinan besar attacker.

---

## Mindset Forensics

Sebelum buka file, ingat prinsip dasar log analysis:

**Cari 5W1H:**
- **WHO** — user/IP mana yang terlibat
- **WHEN** — kapan kejadiannya
- **WHERE** — dari mana asalnya
- **WHAT** — apa yang dilakukan
- **HOW** — teknik yang dipakai

**Cari ANOMALI, bukan semua data:**
- Login di luar jam kerja
- IP yang beda dari biasanya
- Command yang ga sesuai role user
- Failed login banyak sebelum sukses (brute force)
- Timestamp dengan aktivitas mencurigakan

---

## Struktur File

### `sshd.log`

Log SSH server dengan format:

```
[YYYY-MM-DD HH:MM:SS] <event description>
```

Event penting yang muncul:
- `Server listening on X.X.X.X port N` — SSH server startup, ngasih tau IP + port
- `Connection from <IP> port <src_port> on <server_IP> port <server_port>` — koneksi masuk
- `Failed publickey for <user> from <IP>` — coba login pake key tapi gagal
- `Failed password for <user> from <IP>` — coba login pake password tapi gagal
- `Accepted password for <user> from <IP>` — login berhasil
- `Starting session: shell on pts/N for <user>` — session shell dimulai
- `Received disconnect from <IP>` — user disconnect

### `bash_history`

Riwayat command yang diketik user di shell. Biasanya format-nya polos:

```
command1
command2
command3
```

Beberapa versi bash history nyimpen timestamp (dengan format `#<epoch>` di baris sebelum command), sebagian engga.

---

## Workflow Analisis

### Step 1: Baseline — Kenali Pola Normal

Sebelum cari anomali, harus tau dulu pola normal-nya. Scroll file log, perhatiin:

- **IP range mana** yang paling sering muncul (biasanya IP internal / VPN corporate)
- **Username apa** yang sering login (developer name, service account)
- **Jam berapa** login biasa terjadi (harusnya di 0900-1900)
- **Pattern login normal**: `Failed publickey` → `Accepted password` (ini normal — banyak client coba key dulu sebelum fallback ke password)

```bash
grep "Accepted" sshd.log | head -20
```

Bakal ngasih gambaran login sukses secara umum.

### Step 2: Cari IP & Port SSH Server

Baris pertama log biasanya ngasih info server:

```bash
head -5 sshd.log
```

Cari baris `Server listening on <IP> port <PORT>`. Kombinasiin sama IP tujuan dari baris `Connection from ... on <server_IP> port <server_port>` — itu identitas SSH server.

### Step 3: Timeline Semua Login Sukses

```bash
grep "Accepted" sshd.log
```

Bikin timeline kronologis: **kapan**, **siapa**, **dari IP mana**.

### Step 4: Filter Login Mencurigakan

Login di luar jam kerja (09:00 - 19:00) itu red flag utama. Cek:

```bash
grep "Accepted" sshd.log | awk -F'[][]' '{print $2}' | awk '{print $2}'
```

Command di atas nge-extract jam dari setiap login sukses. Cari yang **sebelum 09:00** atau **sesudah 19:00**.

### Step 5: Identifikasi Attacker

Kombinasi yang paling mencurigakan:
- Login **di luar jam kerja**
- Login sebagai **root** (privileged account)
- Dari **IP yang beda** dari IP normal tim
- **Ga ada failed attempt** sebelumnya (kayak udah tau password)

Tulis: timestamp, IP, user, port source.

### Step 6: Timeline Session Attacker

Cari kapan attacker disconnect:

```bash
grep <IP_attacker> sshd.log
```

Bakal keluar semua event terkait attacker — connection, login, session, disconnect. Rekonstruksi durasi session.

### Step 7: Analisis Bash History

Buka `bash_history`, cari command yang dilakukan attacker:
- Command manipulasi file (`rm`, `mv`, `chmod`, dll)
- Command exfiltration (`scp`, `curl`, `wget`, `nc`)
- Command backdoor (edit `.ssh/authorized_keys`, tambah user, dll)
- Command clean-up (`history -c`, `unset HISTFILE`, dll)

---

## Panduan Mapping Pertanyaan

| Topik Pertanyaan | Cari di Mana |
|------------------|--------------|
| IP:PORT SSH Server | `sshd.log` baris `Server listening` + IP tujuan koneksi |
| First successful login | `sshd.log` baris pertama `Accepted password` |
| Fingerprint SSH key attacker | `sshd.log` baris `Failed publickey` — bagian `SHA256:...` |
| IP attacker | Login sukses di luar jam kerja |
| Username yang di-compromise | User di baris login mencurigakan |
| Waktu attacker login/logout | `Accepted password` / `Disconnected from user` untuk IP attacker |
| Command yang dijalankan | `bash_history` |
| Failed login attempts | Hitung baris `Failed password` |

---

## Tips Format Jawaban

- **Timestamp**: biasanya harus persis format yang muncul di log, contoh `YYYY-MM-DD HH:MM:SS`. Jangan cuma tulis jam-nya doang.
- **IP:PORT**: format biasanya `IP:PORT` tanpa spasi.
- **Fingerprint**: bagian setelah `SHA256:` — bisa dengan atau tanpa prefix `SHA256:`, coba dua-duanya.
- **Command**: kadang minta persis dengan spasi/argumen; jangan retype, copy langsung dari file.
- **Angka**: durasi biasanya dalam menit; hitung selisih timestamp.

---

## Command Cheatsheet Analisis Log

```bash
# Baca log dengan pager
less sshd.log

# Cari kata di log (case insensitive)
grep -i "keyword" sshd.log

# Cari multiple keyword sekaligus
grep -E "Accepted|Failed" sshd.log

# Extract semua IP unik yang pernah login sukses
grep "Accepted" sshd.log | awk '{print $11}' | sort -u

# Hitung frekuensi (buat cari IP yang paling sering muncul)
grep "Accepted" sshd.log | awk '{print $11}' | sort | uniq -c | sort -rn

# Cari login di luar jam kerja
grep "Accepted" sshd.log | awk -F'[][]' '{split($2,a," "); split(a[2],t,":"); if (t[1]<9 || t[1]>=19) print}'

# Cari semua event dari IP tertentu
grep "192.168.1.100" sshd.log

# Timeline event kronologis
sort sshd.log > sorted.log
```

---

## Insight Kunci

1. **Operating hours = filter utama.** Kalau challenge ngasih jam kerja, itu tool paling powerful buat sortir aktivitas normal vs mencurigakan.

2. **Failed publickey ≠ attack.** Banyak SSH client otomatis coba semua key yang ada di `~/.ssh/` sebelum fallback ke password. Ini normal, bukan indicator serangan.

3. **Failed password = red flag.** Kalau ada banyak failed password diikuti accepted, itu classic brute force pattern.

4. **Pattern normal**: 1 kali `Failed publickey` diikuti `Accepted password` — normal.
   **Pattern attack**: 3+ kali `Failed password` diikuti `Accepted password` — brute force sukses.
   **Pattern super attack**: Ga ada failed sama sekali langsung `Accepted password` di jam aneh — kredensial curian.

5. **Timeline itu raja.** Rekonstruksi kronologi kejadian sebelum jawab pertanyaan. Timeline ngasih konteks yang bikin jawaban jadi jelas.

6. **Cross-reference file.** Kombinasiin insight dari SSH log sama bash history. SSH log ngasih "siapa & kapan", bash history ngasih "apa yang dilakukan".

---

## Referensi

- [OpenSSH Log Format](https://man.openbsd.org/sshd)
- [Linux Bash History](https://www.gnu.org/software/bash/manual/html_node/Bash-History-Facilities.html)
- [Log Analysis Fundamentals](https://www.sans.org/white-papers/33528/)
