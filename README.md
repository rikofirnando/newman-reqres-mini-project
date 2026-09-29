# Implementasi mini project Newman ReqRes di Jenkins

> [!info] Konteks
> Tes manual dan cron di `app2` sudah berjalan. Node/Newman berada di `/home/rikofirnando/.nvm/versions/node/v24.18.0/bin/`. Jenkins agent sebelumnya memakai label `jenkins-agent-01`; pastikan label itu menunjuk mesin yang memiliki Node/Newman tersebut. Panduan ini menggunakan **Pipeline from SCM** sehingga collection dan Jenkinsfile diambil dari Git pada setiap build.

## 1. Apa yang berubah dari cron?

| Cron Linux | Jenkins Pipeline |
|---|---|
| Jadwal pada `crontab -e` | Jadwal pada Jenkinsfile `triggers { cron(...) }` |
| Key di `$HOME/.config/newman/reqres.env` | Key di Jenkins **Secret text** credential |
| Script berjalan dari folder `/home/rikofirnando/Testing/...` | Collection dibaca dari **workspace** hasil checkout Git |
| Hasil di `cron.log` dan `logs/` | Output di **Console Output**, hasil assertion di **Test Result** |

Jenkins job ini tidak memakai file `reqres.env` ataupun cron Linux. Anda bisa mempertahankan cron untuk perbandingan, tetapi dua jadwal akan sama-sama memanggil ReqRes bila keduanya aktif.

## 2. Siapkan repository

Di repository Git yang akan digunakan oleh job baru, buat struktur ini:

```text
repo-anda/
├── Jenkinsfile
└── postman/
    └── reqres.collection.json
```

- Salin `Jenkinsfile` yang disertakan bersama panduan ini ke **root repository**.
- Salin collection dari project mini Anda ke `postman/reqres.collection.json`. Collection harus versi yang sudah diperbaiki dengan `{{api_key}}` dan `{{base_url}}`.
- **Jangan** commit `.config/newman/reqres.env`, `logs/`, laporan JSON, atau API key.
- Commit dan push kedua file ke branch latihan yang akan dibaca job Jenkins.

Contoh jika terminal Anda sedang berada di root repo dan file mini project ada di `~/Testing/newman-reqres-mini-project`:

```bash
mkdir -p postman
cp "$HOME/Testing/newman-reqres-mini-project/reqres.collection.json" postman/reqres.collection.json
# Salin Jenkinsfile dari paket ini ke root repo, lalu tinjau perubahan:
git status --short
git add Jenkinsfile postman/reqres.collection.json
git commit -m "Add scheduled Newman ReqRes pipeline"
git push
```

Jika belum ingin melakukan commit/push, Anda dapat membuat job Pipeline dengan script ditempel di UI dan mengganti `checkout scm` dengan konfigurasi `git` yang sesuai. Untuk panduan ini, gunakan **Pipeline script from SCM** agar `checkout scm` dan branch jelas.

## 3. Pastikan agent dapat menjalankan Newman

Di Jenkins, buka **Manage Jenkins → Nodes**, lalu periksa node berlabel `jenkins-agent-01`. Label pada Jenkinsfile harus cocok dengan node yang akan menjalankan tes.

Path `/home/rikofirnando/.nvm/versions/node/v24.18.0/bin/` harus tersedia **di mesin agent**, bukan hanya di controller. Jika agent berada di mesin lain, pasang Node/Newman pada agent itu dan sesuaikan dua baris `export PATH` di Jenkinsfile. Stage **Check Newman** menampilkan versi Node/Newman dan akan berhenti jika collection tidak ada. Jenkins agent juga harus dapat menjangkau `https://reqres.in`.

> [!note] Kenapa PATH ditulis dua kali?
> Setiap `sh` Jenkins memulai shell baru. PATH yang di-`export` dalam stage **Check Newman** tidak otomatis terbawa ke stage **Run API Tests**. Inilah isu yang mirip dengan cron: terminal interaktif memuat nvm, tetapi proses otomatis belum tentu memuatnya.

## 4. Simpan API key sebagai Jenkins Credential

1. Buka Jenkins → **Manage Jenkins → Credentials**.
2. Pilih domain **(global)** atau folder job yang sesuai → **Add Credentials**.
3. **Kind:** `Secret text`.
4. **Secret:** tempel API key ReqRes **yang sudah diganti** (jangan tempel di Jenkinsfile).
5. **ID:** `reqres-api-key` (harus sama persis dengan Jenkinsfile).
6. **Description:** `ReqRes API key for Newman learning` → **Create**.

`withCredentials` mengikat Secret text ke variabel `REQRES_API_KEY` hanya selama stage pengujian. Jenkins akan berusaha menyamarkannya di Console Output; jangan mencetak key dengan `echo`, `env`, atau mode debug `set -x`. Credential pada agent tetap perlu digunakan hanya dalam job/branch yang Anda percaya.

## 5. Buat job Jenkins

1. Dashboard Jenkins → **New Item**.
2. Nama misalnya `Learn Newman ReqRes` → pilih **Pipeline** → **OK**.
3. Pada bagian **Pipeline**, pilih **Definition: Pipeline script from SCM**.
4. **SCM: Git**, isi **Repository URL** dan Git credential bila repository privat.
5. **Branches to build:** isi branch latihan yang sudah berisi Jenkinsfile dan collection, misalnya `*/feature/multibranch-demo` atau `*/main` sesuai branch yang benar-benar Anda push.
6. **Script Path:** `Jenkinsfile` → **Save**.

Job Pipeline biasa lebih mudah untuk latihan pertama. Sesudah berhasil, barulah pindahkan ke Multibranch bila Anda ingin tiap branch punya pipeline sendiri.

## 6. Jalankan manual di Jenkins

Klik **Build Now**, lalu buka build → **Console Output**. Urutan yang diharapkan:

1. **Checkout** mengambil repo dan branch.
2. **Check Newman** menampilkan Node `v24.18.0` (atau versi yang dipakai agent), versi Newman, dan menemukan `postman/reqres.collection.json`.
3. **Run API Tests** memanggil delapan request.
4. **Test Result** menampilkan 17 assertion yang lulus; build hijau **SUCCESS**.

Pipeline menghasilkan `reports/newman.xml` lalu mempublikasikannya lewat step `junit`. Jika Newman gagal sebelum XML dibuat, `post` melewati publikasi laporan dan Console Output menunjukkan penyebabnya. Newman punya reporter JUnit dan Jenkins dapat membaca XML JUnit.

## 7. Aktifkan jadwal Jenkins setelah build manual sukses

Di Jenkinsfile, hapus komentar dari baris:

```groovy
triggers { cron('H 8 * * *') }
```

Lalu commit dan push. Jalankan **Build Now sekali lagi** supaya Jenkins membaca revisi Jenkinsfile. `H 8 * * *` berarti Jenkins memilih menit yang stabil di antara **08:00–08:59** setiap hari untuk menyebarkan beban job. Jika perlu tepat pukul **08:00**, gunakan `0 8 * * *`; Jenkins biasanya menyarankan `H` ketika banyak job mulai bersamaan. Jadwal mengikuti zona waktu **Jenkins controller**. Cek zona waktu di controller sebelum menyebutnya 08.00 WIB.

Untuk uji jadwal, Anda dapat sementara memakai `H/5 * * * *` (sekitar tiap 5 menit), commit/push, dan tunggu build otomatis. **Kembalikan ke jadwal harian** sesudah terbukti. Pada build otomatis, Console Output biasanya menunjukkan pemicu timer.

> [!warning] Hindari dua scheduler saat latihan
> Jika cron Linux di `app2` masih `* * * * *`, ubah atau nonaktifkan baris tersebut sebelum menguji timer Jenkins agar tidak terjadi panggilan API setiap menit dari dua tempat.

## 8. Troubleshooting

| Gejala | Pemeriksaan |
|---|---|
| Job menunggu `jenkins-agent-01` | Node offline atau label berbeda; cek **Manage Jenkins → Nodes** dan sesuaikan Jenkinsfile. |
| `node: command not found` / `newman: command not found` | Path nvm harus ada di **agent**. Periksa dua baris `export PATH` pada Jenkinsfile. |
| `test -f postman/reqres.collection.json` gagal | File belum di-commit/push ke branch yang dibangun atau nama/path berbeda. |
| `Credentials ... could not be found` | ID harus tepat `reqres-api-key`, jenis **Secret text**, serta scope dapat diakses job. |
| Git checkout gagal | Cek URL repo, Git credential, branch specifier, dan SSH host key jika memakai SSH. |
| Newman menerima 401/403 | Periksa key ReqRes yang aktif dan akses jaringan agent. Jangan tampilkan key di log. |
| Build tidak berjalan otomatis | Periksa Jenkinsfile terbaru telah terbaca, sintaks `triggers`, dan zona waktu controller. |

## 9. Ringkasan konsep

Cron Linux menjalankan file lokal pada jam tertentu. Jenkins menjalankan pipeline dari repo di agent, mengambil credential dari Jenkins, dan menampilkan hasil pengujian per build. `Build Now` adalah tes pertama; timer Jenkins baru diaktifkan setelah pipeline manual sukses.

## Referensi resmi

- [Jenkins: Using credentials](https://www.jenkins.io/doc/book/using/using-credentials/)
- [Jenkins: Pipeline syntax dan triggers](https://www.jenkins.io/doc/book/pipeline/syntax/)
- [Jenkins: Recording tests and artifacts](https://www.jenkins.io/doc/pipeline/tour/tests-and-artifacts/)
- [Postman: Install and run Newman](https://learning.postman.com/docs/reference/newman-cli/installing-running-newman/)
