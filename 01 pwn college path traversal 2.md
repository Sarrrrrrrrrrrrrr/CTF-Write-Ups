# Path Traversal 2 — pwn.college

**Platform:** pwn.college
**Path:** Web Security
**Topik:** Path Traversal via Filter Bypass

> Walkthrough edukatif. Jawaban akhir sengaja tidak dicantumkan — kamu solve sendiri dengan mengikuti workflow.

---

## Deskripsi Challenge

Level ini adalah lanjutan dari path traversal dasar. Di level sebelumnya, developer belum sadar sama sekali kalau `../` bisa dipakai buat naik direktori. Di level ini, developer **udah nyoba nge-blok** traversal, tapi caranya bego dan justru nunjukin kesalahpahaman mendasar tentang gimana filter string bekerja.

---

## Thought Process

Ini urutan cara mikir gw waktu solve, step by step.

### 1. Recon direktori dulu

Sebelum ngapa-ngapain, cek dulu isi direktori yang aplikasi layanin. Aplikasinya nge-serve file dari `/challenge/files/`, jadi ke sana dulu:

```bash
ls /challenge/files/
```

Hasilnya ada 2 hal:
- Folder `fortunes/`
- File `index.html`

### 2. Baca hint di index.html

Buka `index.html`, baca isinya. Ada hint yang ngarahin ke `fortunes`. Di CTF, kalau aplikasi ngasih hint eksplisit ke sesuatu, biasanya bukan dekorasi doang — patut dicek.

### 3. Cek isi `fortunes/`

```bash
ls /challenge/files/fortunes/
```

Isinya 3 file yang kelihatannya biasa, ga ada yang mencurigakan atau langsung berguna sebagai jawaban. Awalnya kelihatan buntu.

**Tapi finding ini penting** — walaupun ga langsung ngasih jawaban, sekarang gw punya **nama file/folder valid** di dalem `/challenge/files/`. Nanti ini krusial banget buat payload.

### 4. Baca source code aplikasinya

Aplikasi Flask-nya bisa dibaca langsung:

```bash
cat /challenge/server
```

Atau curl kalau di-serve sebagai endpoint. Ini kode-nya:

```python
#!/usr/bin/exec-suid -- /usr/bin/python3 -I
import flask
import os
app = flask.Flask(__name__)

@app.route("/shared", methods=["GET"])
@app.route("/shared/<path:path>", methods=["GET"])
def challenge(path="index.html"):
    requested_path = app.root_path + "/files/" + path.strip("/.")
    print(f"DEBUG: {requested_path=}")
    try:
        return open(requested_path).read()
    except PermissionError:
        flask.abort(403, requested_path)
    except FileNotFoundError:
        flask.abort(404, f"No {requested_path} from directory {os.getcwd()}")
    except Exception as e:
        flask.abort(500, requested_path + ":" + str(e))

app.secret_key = os.urandom(8)
app.config["SERVER_NAME"] = f"challenge.localhost:80"
app.run("challenge.localhost", 80)
```

### 5. Analisis kode — cari filter-nya

Dua observasi kunci:

**Observasi #1**: Semua akses harus lewat route `/shared/`. Ga ada route lain. Kalau request langsung ke `/x/../flag` tanpa `/shared/`, aplikasi ga bakal jawab (404 route not found).

**Observasi #2**: Baris ini:

```python
requested_path = app.root_path + "/files/" + path.strip("/.")
```

Filter-nya cuma `path.strip("/.")`. Dev-nya kelihatannya mikir strip bakal ngapus semua `/` dan `.` dari string. Tapi:

### 6. Cara kerja `.strip()` yang sebenarnya

`.strip()` cuma nyentuh **ujung depan dan ujung belakang** string. Bagian tengah aman.

Tes cepat di Python:
```python
>>> "abc/../def".strip("/.")
'abc/../def'
```

Ga berubah — karena ujung depan `a` (bukan `/` atau `.`), ujung belakang `f` (bukan `/` atau `.`), jadi strip skip semuanya.

Kesimpulan: **kalau `../` di taro di depan atau belakang string, kena strip. Kalau di tengah, lolos.**

### 7. Reasoning ke strategi bypass

Dari filter yang cuma menyentuh ujung, strategi jelas:

- Depan payload: karakter aman (bukan `/`, bukan `.`) → prefix
- Tengah payload: `../` sebanyak yang dibutuhkan → traversal
- Belakang payload: nama file target (bukan `/`, bukan `.`) → target

Format:
```
/shared/<prefix_aman>/<../ N kali>/<target>
```

### 8. Pilih prefix yang tepat

Ini bagian yang bikin gw sempat stuck. Payload kayak `/shared/x/../../../flag` **kelihatan bener**, tapi 404 terus.

Sebabnya: OS resolve path step by step. Kalau folder `x` ga ada di `/challenge/files/`, resolusi gagal di step pertama — sebelum sempat proses `../`.

Ini kenapa **Step 1-3 tadi penting**. Karena gw udah tau `fortunes/` beneran ada di `/challenge/files/`, gw bisa pake `fortunes` sebagai prefix aman, bukan random string.

### 9. Susun payload

Hitung berapa `../` yang dibutuhkan dari `/challenge/files/fortunes/`:

- Naik 1 → `/challenge/files/`
- Naik 2 → `/challenge/`
- Naik 3 → `/` (root)

Sesuaikan sama posisi file target.

### 10. Test payload

```bash
curl --path-as-is http://challenge.localhost:80/shared/fortunes/[../ N kali]/<target>
```

Flag `--path-as-is` penting — tanpa itu, curl bakal nge-resolve `../` di sisi client sebelum kirim ke server. Payload jadi ga jalan.

---

## Source Code Aplikasi

Udah di-quote di bagian Thought Process step 4 di atas. Baris kunci yang jadi vulnerability:

```python
requested_path = app.root_path + "/files/" + path.strip("/.")
```

---

## Alternatif: URL Encoding

Kalau `--path-as-is` ga cukup atau ada middleware yang tetep nge-resolve `../`, bisa pake URL encoding:

```
%2f = /
%2e = .
```

Contoh:

```bash
curl 'http://challenge.localhost:80/shared/fortunes/..%2f..%2f..%2f<target>'
```

Server tetep bakal decode jadi `/` sebelum masuk ke filter, jadi payload-nya masih jalan tapi ga di-collapse di sisi client.

---

## Troubleshooting

**Error 404 "No such file or directory"**:
- Cek pesan error — kalau path udah kebentuk bener (ada `../` di dalemnya) tapi file ga ketemu, kemungkinan `<prefix>` yang dipake ga ada di `/challenge/files/`. Ganti sama file/folder yang beneran ada (contoh: `fortunes` atau `index.html`).

**Error 403 Permission Denied**:
- Bagus, artinya path udah nyampe ke file target. Tapi user biasa ga boleh baca. Karena aplikasi Flask jalan dengan privilege khusus (`#!/usr/bin/exec-suid`), request lewat curl harusnya tetep berhasil.

**Filter kayak ga jalan**:
- Verifikasi filter emang bermasalah dengan tes manual di Python:
  ```python
  print("aaa/../../bbb".strip("/."))
  ```
  Hasilnya harus `aaa/../../bbb` — utuh, ga berubah.

---

## Insight Kunci

1. **Recon dulu, exploit belakangan.** Sebelum nyusun payload, cek dulu isi direktori aplikasi. Nama file/folder valid bakal jadi prefix aman di payload. Tanpa recon, mudah stuck di 404 karena OS gagal resolve path yang lewat folder ga eksis.

2. **`.strip()` cuma menyentuh ujung string.** Ini kesalahpahaman umum. Developer sering ngira dia ngapus karakter di seluruh string, padahal cuma di depan sama belakang aja.

3. **Filter mentah ga cukup.** Path validation yang bener harus dilakukan setelah normalisasi (`os.path.abspath()` / `os.path.realpath()`), bukan cuma sanitasi string mentah.

4. **`--path-as-is` di curl.** Wajib dipake buat testing path traversal — tanpa ini, curl auto-normalize URL di sisi client dan payload jadi ga jalan.

5. **Route Flask `<path:path>`.** Berbeda dari `<string:path>` — dia terima slash. Ini yang bikin bisa nyusun payload multi-level.

6. **Solusi bener buat developer:**
   ```python
   import os
   base = "/challenge/files/"
   requested = os.path.realpath(base + path)
   if not requested.startswith(base):
       flask.abort(403)
   ```
   Cek path setelah normalisasi, bukan sebelum. Ini yang lupa dilakukan sama developer di challenge ini.

---

## Referensi

- [OWASP Path Traversal](https://owasp.org/www-community/attacks/Path_Traversal)
- [Python `.strip()` documentation](https://docs.python.org/3/library/stdtypes.html#str.strip)
- [Flask URL converters](https://flask.palletsprojects.com/en/latest/api/#url-route-registrations)
