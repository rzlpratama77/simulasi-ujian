# Simulasi Ujian EX240 — Red Hat Certified Specialist in API Management

> **Berbasis:** DO240 v2.11 / Red Hat 3scale API Management 2.11
> **Format:** Performance-based (hands-on), sama seperti bentuk ujian asli EX240
> **Pendamping:** [RANGKUMAN-DO240-3scale-EX240.md](./RANGKUMAN-DO240-3scale-EX240.md)

---

## ⚠️ Baca Ini Dulu — Tentang Keaslian Soal

**Dokumen ini BUKAN soal asli EX240.** Isi ujian Red Hat bersifat rahasia, dilindungi NDA, dan tidak dipublikasikan. Situs mana pun yang mengklaim menjual "real EX240 dumps" melanggar Red Hat Certification Program Agreement — memakainya berisiko **pembatalan sertifikasi permanen**.

Yang Anda pegang ini adalah **simulasi yang disusun dari dua sumber sah**:
1. **Objektif ujian EX240 yang dipublikasikan Red Hat** di halaman resmi ujian.
2. **Seluruh isi kursus DO240 v2.11** — kursus resmi yang memang dirancang Red Hat untuk mempersiapkan EX240.

Karena EX240 menguji kemampuan yang sama dengan yang dilatih DO240, mengerjakan simulasi ini secara serius di lab nyata akan melatih tepat keterampilan yang diuji — tanpa perlu bocoran.

> **Verifikasi ulang objektif terbaru** di `https://www.redhat.com/en/services/training/ex240-red-hat-certified-specialist-api-management-exam` sebelum ujian. Detail seperti durasi dan passing score dapat berubah; angka di bawah adalah pola umum ujian Red Hat, bukan kutipan resmi.

### Format Ujian Asli (gambaran umum)

| Aspek | Keterangan |
|---|---|
| **Bentuk** | Performance-based / hands-on. Anda mengerjakan tugas nyata di cluster OpenShift + 3scale, **bukan** pilihan ganda. |
| **Durasi** | Sekitar 3 jam (konfirmasi di halaman resmi). |
| **Passing score** | Ujian Red Hat umumnya 210/300 (70%). |
| **Penilaian** | Otomatis, berdasarkan **hasil akhir (state)** — bukan cara Anda mencapainya. GUI atau CLI sama saja nilainya. |
| **Akses** | Dokumentasi resmi 3scale/OpenShift biasanya tersedia. Internet umum, catatan pribadi, dan perangkat lain **tidak**. |
| **Persistensi** | Konfigurasi harus **bertahan setelah reboot**. Jangan andalkan alias atau env var yang tidak tersimpan. |

---

## Peta Blueprint → Soal

| Domain (objektif EX240) | Bobot estimasi | Soal di simulasi ini |
|---|---|---|
| Instalasi 3scale & Toolbox CLI | ~10% | Tugas 1, 2 |
| Mengelola API: product, backend, plan, rate limit | ~20% | Tugas 3, 4, 5 |
| Versioning API | ~8% | Tugas 6, 7 |
| Service discovery | ~7% | Tugas 8 |
| Multi-tenancy | ~7% | Tugas 9 |
| API Gateway (APIcast) & policies | ~15% | Tugas 10, 11 |
| Keamanan API (user, API key, key-pair, OIDC) | ~18% | Tugas 12, 13, 14 |
| Developer Portal & ActiveDocs | ~10% | Tugas 15, 16, 17 |
| Monitoring & analytics | ~8% | Tugas 18, 19 |
| Monetisasi (billing & pricing) | ~7% | Tugas 20 |

---

## Aturan Main Simulasi Ini

1. **Kerjakan di lab nyata.** Membaca kunci jawaban tanpa mengetik = tidak melatih apa pun. EX240 menilai state, bukan hafalan.
2. **Timebox 3 jam** untuk Bagian A (20 tugas). Pakai timer.
3. **Jangan buka Bagian C (kunci jawaban)** sebelum waktu habis.
4. **Verifikasi setiap tugas** sebelum lanjut. Di ujian asli tidak ada partial credit untuk niat baik — hanya hasil.
5. **Promosikan konfigurasi.** Kesalahan paling mahal dan paling sering: lupa `Promote to Staging/Production`. Perubahan yang tidak dipromosikan = tidak dinilai.

### Asumsi Lingkungan

```
Cluster OpenShift  : https://api.ocp4.example.com:6443
Wildcard domain    : apps.ocp4.example.com
User OpenShift     : admin / redhat   (htpasswd_provider)
                     developer / redhat
Project 3scale     : 3scale
Tenant default     : 3scale
Admin Portal       : https://3scale-admin.apps.ocp4.example.com
Master Portal      : https://master.apps.ocp4.example.com
Developer Portal   : https://3scale.apps.ocp4.example.com
Toolbox remote     : 3scale-tenant
```

Password Admin Portal & Master Portal diambil dari secret `system-seed` (lihat Tugas 2).

---
---

# BAGIAN A — 20 Tugas Praktik

> Kerjakan berurutan. Beberapa tugas bergantung pada hasil tugas sebelumnya.

---

## Tugas 1 — Instalasi 3scale API Management (15 poin)

Perusahaan Anda akan menggelar 3scale di cluster OpenShift yang baru.

1. Buat project OpenShift bernama **`3scale`**.
2. Pastikan cluster dapat menarik image dari `registry.redhat.io` dengan membuat secret `docker-registry` bernama **`threescale-registry-auth`**, lalu tautkan ke service account `default` (untuk pull) dan `builder`.
3. Instal operator **Red Hat Integration - 3scale** dari OperatorHub ke namespace `3scale`, gunakan update channel **`threescale-2.11`** dan update approval **Automatic**.
4. Deploy custom resource **`APIManager`** bernama `apimanager-sample` dengan:
   - `wildcardDomain` = `apps.ocp4.example.com`
   - resource requirement dinonaktifkan (agar muat di lab)
5. Verifikasi Admin Portal dapat diakses.

---

## Tugas 2 — Konfigurasi 3scale Toolbox CLI (10 poin)

1. Ambil password `admin` Admin Portal tenant `3scale` dari secret OpenShift.
2. Login ke Admin Portal, buat **access token** bernama `toolbox` dengan **semua scope** (Billing, Account Management, Analytics, Policy Registry) dan permission **Read & Write**.
3. Instal 3scale Toolbox CLI dari container image resmi.
4. Konfigurasikan remote bernama **`3scale-tenant`** yang menunjuk ke Admin Portal tenant `3scale`, dan pastikan remote tersebut **bertahan antar eksekusi** container.
5. Buat alias `3scale` yang **bertahan antar sesi login**.
6. Verifikasi dengan menampilkan daftar remote.

---

## Tugas 3 — Backend, Product, dan Promosi (20 poin)

Tim perpustakaan sudah men-deploy dua microservice di cluster:

```
http://books-api.book-library.svc.cluster.local:8080
http://patrons-api.book-library.svc.cluster.local:8080
```

1. Buat product bernama **`library_product`** menggunakan Toolbox CLI.
2. Buat dua backend:
   | Name | Private Base URL |
   |---|---|
   | `book-backend` | `http://books-api.book-library.svc.cluster.local:8080` |
   | `patron-backend` | `http://patrons-api.book-library.svc.cluster.local:8080` |
3. Asosiasikan kedua backend ke `library_product` dengan path publik:
   - `book-backend` → `/b-api`
   - `patron-backend` → `/p-api`
4. Promosikan konfigurasi ke **staging** dan kemudian ke **production**.
5. Buktikan endpoint `/b-api/books` dan `/p-api/patrons` dapat diakses dari luar cluster melalui APIcast production.

---

## Tugas 4 — Application Plan Berbayar dan Rate Limit (20 poin)

Pada product **`library_product`**:

1. Buat application plan bernama **`east-branch-plan`** (system name `east_branch_plan`) melalui Toolbox CLI dengan ketentuan:
   - Trial period: **15 hari**
   - Setup fee: **1.00**
   - Cost per month: **10.00**
   - Approval **tidak** diperlukan
   - Plan langsung **published**
2. Tetapkan rate limit pada plan tersebut untuk backend `book-backend`: maksimum **5 request per menit**.
3. Jadikan `east-branch-plan` sebagai **default plan** product tersebut.
4. Verifikasi plan muncul di daftar plan product.

---

## Tugas 5 — Application, Kredensial, dan Uji Rate Limit (15 poin)

1. Temukan ID account developer bernama **`john`**.
2. Buat application bernama **`east-branch-app`** milik account `john`, pada product `library_product` dengan plan `east_branch_plan`.
3. Ambil **`user_key`** application tersebut ke variable shell `USER_KEY` **menggunakan Toolbox CLI** (bukan menyalin dari GUI).
4. Kirim request berulang hingga rate limit terlampaui, dan tunjukkan pesan yang dikembalikan APIcast.

---

## Tugas 6 — URL Versioning (20 poin)

API `books-api` di project `applications-versions` merilis **v2** dengan breaking change (field `authorName` diganti objek `author`). Saat ini hanya **v1** yang publik.

Pada product **`applications_versions`**:

1. Buat backend **`books_backend_v2`** yang menunjuk ke `http://books-api.applications-versions.svc.cluster.local:80/api/v2`.
2. Asosiasikan backend tersebut ke product dengan path publik **`/v2`**.
3. Buat metric baru bernama **`hits_v2`** menggunakan Toolbox CLI.
4. Buat mapping rule `GET /v2` yang menaikkan metric `hits_v2`.
5. Promosikan ke staging **menggunakan Toolbox CLI** (bukan GUI).
6. Verifikasi `https://STAGING_DOMAIN/v2/books` mengembalikan struktur `author` yang baru.

---

## Tugas 7 — Request Header Versioning dengan Routing Policy (15 poin)

Product **`applications_versions`** juga harus melayani klien yang mengirim header `x-api-version: v2` ke URL tanpa prefix versi.

1. Tambahkan policy **`Routing`** ke policy chain product sehingga request dengan header `x-api-version` bernilai `v2` diarahkan ke `http://books-api.applications-versions.svc.cluster.local:80/api/v2`.
2. Pastikan urutan policy chain **benar**.
3. Terapkan policy chain tersebut **melalui Toolbox CLI menggunakan file YAML**.
4. Verifikasi dengan `curl` menyertakan header versi.

---

## Tugas 8 — Service Discovery (20 poin)

1. Buat project OpenShift bernama **`library`**.
2. Deploy aplikasi contoh dari `https://github.com/RedHatTraining/DO240-apps.git`, context dir `library/books-api`, build env `NODE_ENV=development`, dengan nama **`books-api`**.
3. Buat service `books-api` **dapat ditemukan (discoverable)** oleh 3scale: skema `http`, port `8080`.
4. Berikan izin yang diperlukan agar 3scale dapat membaca service di project `library`.
5. Impor service tersebut menjadi product + backend 3scale melalui Admin Portal.
6. Buat application plan **`library_basic_plan`** dan application **`library-app`** (account `john`) agar product hasil import dapat diakses.
7. Verifikasi endpoint `/books` melalui staging APIcast.

---

## Tugas 9 — Multi-tenancy (20 poin)

1. Ambil password user `master` dari secret OpenShift dan login ke **Master Portal**.
2. Buat tenant baru dengan data:
   - Username: `do240-user`
   - Email: `do240@redhat.com`
   - Password: `do240`
   - Organization/Group name: `do240`
3. **Aktifkan** user `do240-user`.
4. Sebutkan (dan verifikasi) empat URL yang dihasilkan tenant ini: Admin Portal, Developer Portal, Staging APIcast, Production APIcast.
5. Dari Admin Portal tenant `do240`, undang user baru ke alamat `gls@redhat.com`, ambil link undangan dari email interceptor, dan daftarkan user `gls-user` (password `gls-password`).
6. Berikan `gls-user` izin untuk **melihat dan mengelola developer account & application pada semua API product**, lalu verifikasi.

---

## Tugas 10 — Self-Managed APIcast Gateway (20 poin)

1. Buat project OpenShift bernama **`apicast`**.
2. Instal operator **Red Hat Integration - 3scale APIcast gateway** ke namespace `apicast`.
3. Di tenant `3scale`, buat access token bernama `apicast` dengan scope **`Account Management API`** dan permission **Read Only**.
4. Buat secret OpenShift **`apicast-secret`** yang memuat `AdminPortalURL` berisi token tersebut.
5. Deploy custom resource **`APIcast`** bernama `custom-apicast` yang:
   - menggantikan environment **`staging`**
   - diekspos di host `custom-apicast.apps.ocp4.example.com` dengan TLS
   - dibatasi memory `128Mi`
6. Konfigurasikan product **`gateways_apicast`** agar memakai gateway self-managed tersebut untuk staging, lalu promosikan.
7. Verifikasi endpoint `/echo` melalui host gateway baru.

---

## Tugas 11 — APIcast Policies (20 poin)

Pada product **`gateways_policies`**:

1. Promosikan konfigurasi awal ke staging dan catat `user_key`.
2. Tambahkan policy **`Anonymous Access`** dengan `auth_type` = `user_key` dan isi user key yang Anda catat, sehingga endpoint dapat diakses **tanpa** parameter `user_key`. Pastikan urutannya benar dalam policy chain.
3. Tambahkan policy **`Maintenance Mode`**, promosikan, dan tunjukkan HTTP status code yang dikembalikan.
4. Hapus policy `Maintenance Mode`, promosikan, dan buktikan service pulih.
5. Ekspor policy chain akhir product ke file `policy_chain.yml` menggunakan Toolbox CLI.

---

## Tugas 12 — User Admin Portal & Kontrol Akses (15 poin)

1. Undang user baru ke `member_user@redhat.com`, ambil link dari email interceptor, dan daftarkan sebagai `member_user` (password `gls-password`).
2. Verifikasi bahwa `member_user` **tidak** memiliki akses ke Products maupun Backends.
3. Berikan `member_user` izin **hanya** untuk mengakses dan mem-query analytics **semua API product** — tanpa izin lain.
4. Undang dan daftarkan user kedua bernama `admin_user` (password `gls-password`), lalu berikan role **Admin (full access)**.
5. Verifikasi kedua level akses tersebut.

---

## Tugas 13 — Autentikasi API Key dan App_ID/App_Key Pair (15 poin)

Pada product **`secure_keys`**:

1. Buat application **`secure_keys_app`** untuk account `john` dengan plan `secure_keys_basic`.
2. Ambil `user_key`-nya via Toolbox CLI dan buktikan endpoint `/books` dapat diakses dengan `user_key`.
3. Ubah pola autentikasi product menjadi **App_ID and App_Key Pair**, lalu promosikan.
4. Buktikan bahwa `user_key` **sudah tidak berlaku** — sebutkan pesan errornya.
5. Ambil `application_id` via Toolbox CLI dan lakukan panggilan API yang berhasil menggunakan pasangan `app_id` + `app_key`.

---

## Tugas 14 — Integrasi OIDC dengan Red Hat Single Sign-On (25 poin)

Product **`secure_oauth`** harus mewajibkan JWT dari RHSSO realm `do240`.

1. Buktikan lebih dulu bahwa product masih dapat diakses hanya dengan `user_key`.
2. Pastikan komponen **Zync** mempercayai sertifikat RHSSO (gunakan script yang disediakan lab), dan verifikasi pod `zync-que` kembali `Running`.
3. Di RHSSO realm `do240`, buat client **`zync-client`** dari file JSON yang disediakan, lalu berikan client role **`manage-clients`** dari **Realm Management** pada tab Service Account Roles. Catat **Secret**-nya.
4. Konfigurasikan product `secure_oauth` memakai **OpenID Connect**, dengan OIDC Issuer URL yang benar. Promosikan.
5. Buat application **`sso_app`** pada account `Developer` dengan plan `secure_oauth_basic`. Catat **Client ID**-nya.
6. Ubah client RHSSO hasil sinkronisasi menjadi: Access Type `public`, Valid Redirect URIs `*`, Web Origins `*`.
7. Arahkan aplikasi front end ke client ID tersebut (configmap `book-config` di project `secure-oauth`), lalu paksa pod front end memuat ulang konfigurasi.
8. Aktifkan **CORS** pada product dan pastikan posisinya benar dalam policy chain. Promosikan.
9. Verifikasi login front end sebagai `student` / `redhat`.

---

## Tugas 15 — Developer Portal: Publikasi & Konten Terbatas (20 poin)

1. Publikasikan Developer Portal ke publik ("Open your Portal to the world").
2. Daftarkan user Developer Portal baru dari portal: organization `GLS`, username `portal-user`, email `gls@redhat.com`, password `redhat`. Aktifkan account-nya dari Admin Portal.
3. Buat **section privat** bernama `Restricted section` (tidak public) dan sebuah **page** bernama `Restricted page` di dalamnya dengan path **`/restricted`** dan konten `<h1>Welcome to the Restricted Section</h1>`. Publikasikan.
4. Buktikan halaman tersebut **tidak dapat** diakses anonim.
5. Buat group **`privileged`**, beri akses ke `Restricted section`, dan masukkan account `GLS` ke group tersebut.
6. Buktikan `portal-user` **dapat** mengakses `/restricted`.

---

## Tugas 16 — Kustomisasi Developer Portal (15 poin)

1. Ubah nama organisasi menjadi **`Bob's Boxes`**.
2. Unggah logo dari `~/DO240/labs/portal-customizing/icon.png`.
3. Tampilkan logo tersebut di navbar dengan mengedit layout **`Main layout`** (gunakan variable Liquid yang tepat, tinggi `32px`), lalu publikasikan.
4. Ubah warna latar `body` pada `default.css` menjadi oranye (`rgb(255, 140, 0)`), publikasikan.
5. Buat page baru berjudul **About** dengan path **`/about`**, pastikan **Liquid enabled**.
6. Tambahkan link navigasi ke `/about` pada partial **`submenu`**, dan buat link tersebut mendapat class `active` saat sedang dibuka.

---

## Tugas 17 — Import OpenAPI & ActiveDocs (20 poin)

1. Pada file `~/DO240/labs/portal-openapi/openapi-definition.yaml`, ganti nilai `CHANGE_ME` di bagian `servers` dengan `http://petshelter-api.portal-openapi.svc.cluster.local`.
2. Impor definisi tersebut ke tenant `3scale-tenant` dengan target system name **`portal_openapi`**.
3. Verifikasi bahwa import menghasilkan backend dengan URL yang benar, **menggunakan Toolbox CLI**.
4. Buat application plan **`basic_plan`** dan application **`shelter-app`** (account `john`).
5. Konfigurasikan policy **CORS Request Handling** dengan `allow_origin` yang mengizinkan **Admin Portal dan Developer Portal sekaligus**, tempatkan sebelum policy `3scale APIcast`. Promosikan ke staging lalu production.
6. Uji endpoint `GET /pet/findByStatus` dari ActiveDocs di Admin Portal.
7. Ambil `user_key` application `shelter-app` menggunakan **satu perintah** Toolbox CLI + `jq`.

---

## Tugas 18 — Custom Metrics & Analytics (20 poin)

Product **`monitoring_analytics`** memakai backend `status-api` yang mengembalikan status code 200/201/203 sesuai path `/status/{code}`.

1. Buat metric **`status`** (unit `1`) dan tautkan ke mapping rule `GET /status`.
2. Buat metric **`status-203`** (unit `1`).
3. Tambahkan policy **`Custom Metrics`** yang menaikkan `status-203` **hanya** ketika HTTP status code response adalah `203`. Pastikan posisinya benar dalam policy chain.
4. Promosikan ke staging.
5. Kirim 20 request menggunakan script lab, lalu buktikan di Analytics bahwa `status-203` hanya menghitung response 203, sedangkan `status` menghitung seluruh 20 request.

---

## Tugas 19 — Monitoring Infrastruktur dengan Prometheus & Grafana (20 poin)

1. Tampilkan daftar metrik mentah yang diekspos pod APIcast staging (port metrics-nya berapa?).
2. Instal operator **Prometheus** ke namespace `3scale` dan buat instance `Prometheus` bernama `3scale-monitor` yang memilih podmonitor **dan** rule berlabel `app=3scale-api-management`, memakai service account `prometheus-k8s`.
3. Instal operator **Grafana** ke namespace `3scale`, buat instance `Grafana` yang memilih dashboard berlabel `app=3scale-api-management`, lalu buat `GrafanaDataSource` yang menunjuk ke `http://prometheus-operated:9090`.
4. Aktifkan monitoring pada `APIManager`.
5. Buktikan bahwa resource `podmonitors` dan `prometheusrules` **belum ada sebelumnya** dan **muncul setelah** monitoring diaktifkan.
6. Buka dashboard **Apicast** di Grafana.

---

## Tugas 20 — Billing dan Pricing Rules (20 poin)

**Bagian A — Billing** (product `monetizing_billing`)

1. Ubah mode billing tenant menjadi **PREPAID**.
2. Buat application plan **`monthly_plan`** yang published, tanpa approval, dengan setup fee **30.0** dan cost per month **10.0**.
3. Buat application **`monetizing_billing_app`** untuk account `john` pada plan tersebut.
4. Ubah jadwal proses billing agar berjalan **setiap menit** (patch configmap `system` dan dc `system-sidekiq` dengan file lab yang disediakan).
5. Jalankan script lab untuk memundurkan tanggal berakhirnya trial, lalu verifikasi terbitnya invoice dengan state **Finalized** berisi dua line item.

**Bagian B — Pricing** (product `echo`, plan `custom`)

6. Set setup fee **5.00** dan cost per month **3.00**.
7. Buat tiga pricing rule pada metric **`Hits`**:
   | From | To | Cost per unit |
   |---|---|---|
   | 1 | 120 | 0.5 |
   | 121 | 1121 | 0.3 |
   | 1122 | *(kosong)* | 0.1 |

---
---

# BAGIAN B — 30 Soal Konsep (Pemanasan / Pendinginan)

> EX240 asli **tidak** berisi pilihan ganda. Bagian ini murni alat cek pemahaman sebelum masuk lab.

**B1.** Komponen mana yang tetap melayani trafik saat API Manager down, dan mengapa?

**B2.** Sebutkan URL Admin Portal, Developer Portal, dan Staging APIcast untuk tenant bernama `finance` pada wildcard domain `example.com`.

**B3.** Anda punya 6 product yang semuanya memakai satu API internal yang sama. Berapa backend minimum yang perlu dibuat?

**B4.** Apa isi Private Base URL sebuah backend untuk API di `http://svc.ns.svc.cluster.local:8080/api/orders/list`? Jelaskan alasannya.

**B5.** Perubahan product sudah disimpan tapi klien masih menerima perilaku lama. Apa penyebab paling mungkin?

**B6.** Perintah Toolbox apa yang mempromosikan ke **staging**, dan apa yang mempromosikan ke **production**?

**B7.** Apa perbedaan `3scale service` dan `3scale product` dalam kursus ini, dan mana yang disarankan dipakai?

**B8.** Secret OpenShift apa yang menyimpan password `admin` dan `master` bawaan 3scale? Sebutkan key-nya masing-masing.

**B9.** Sebutkan tiga metadata **wajib** agar sebuah OpenShift Service dapat ditemukan service discovery 3scale.

**B10.** Service account apa yang perlu diberi role `view`, dan di project mana, agar 3scale bisa menemukan service di project lain?

**B11.** Apakah service discovery aktif secara default jika 3scale diinstal via operator?

**B12.** Dalam URL versioning, jika public path product adalah `/v2` dan backend Private Base URL adalah `http://svc/api/v2`, ke URL private apa request `https://public/v2/books` diterjemahkan?

**B13.** Di mana posisi policy `Routing` harus ditempatkan relatif terhadap policy `apicast`, dan mengapa?

**B14.** Policy apa yang menghapus kewajiban `user_key`? Apa risiko keamanannya?

**B15.** Policy apa yang mengembalikan HTTP 503 untuk semua request?

**B16.** Policy mana yang **wajib** ada dalam setiap policy chain?

**B17.** Sebutkan dua field wajib pada manifest `APIcast` yang menentukan (a) environment yang digantikan dan (b) host eksternal.

**B18.** Scope minimum apa yang harus dimiliki access token untuk self-managed APIcast gateway?

**B19.** Berapa maksimum key yang bisa diasosiasikan dengan satu application ID pada pola App_ID/App_Key?

**B20.** Mengapa pola App_ID/App_Key lebih baik dari API key tunggal saat melakukan rotasi kredensial?

**B21.** Komponen 3scale apa yang menyinkronkan application ke RHSSO, dan komponen apa yang menjadwalkannya?

**B22.** Role RHSSO apa yang harus dimiliki service account client Zync?

**B23.** Claim JWT apa yang harus cocok dengan kredensial application 3scale?

**B24.** Front end React gagal menampilkan hasil API meski status 200. Policy apa yang kemungkinan kurang, dan di posisi mana dalam chain?

**B25.** Apa perbedaan **Layout** dan **Partial** di Developer Portal?

**B26.** Tiga langkah apa yang diperlukan untuk membuat halaman Developer Portal hanya terlihat oleh sekelompok developer?

**B27.** Objek 3scale apa saja yang otomatis dibuat oleh `3scale import openapi`?

**B28.** Metric default apa yang dibuat 3scale, path apa yang ditangkapnya, dan apa implikasinya terhadap pricing rule?

**B29.** Field apa pada `APIManager` yang mengaktifkan monitoring, dan resource apa yang muncul setelahnya?

**B30.** Dalam mode **postpaid**, apa yang ditagih pada hari pertama bulan? Setelah berapa kali gagal bayar invoice menjadi `failed`?

---
---

# BAGIAN C — Kunci Jawaban Lengkap (Bagian A)

> ⛔ **Berhenti di sini** kalau Anda belum menyelesaikan Bagian A.

Setiap jawaban memuat: **langkah**, **perintah lengkap**, **verifikasi**, dan **jebakan umum**.

---

## Kunci Tugas 1 — Instalasi 3scale API Management

### Langkah 1.1 — Login & buat project

```bash
[student@workstation ~]$ oc login -u admin -p redhat api.ocp4.example.com:6443
[student@workstation ~]$ oc new-project 3scale
```

Atau via GUI: **Home → Projects → Create Project**, nama `3scale`.

### Langkah 1.2 — Registry authentication

```bash
[student@workstation ~]$ oc create secret docker-registry \
  threescale-registry-auth \
  --docker-server=registry.redhat.io \
  --docker-username=REDHAT_USERNAME \
  --docker-password=REDHAT_TOKEN

[student@workstation ~]$ oc secrets link default \
  threescale-registry-auth --for=pull

[student@workstation ~]$ oc secrets link builder threescale-registry-auth
```

> Username & token diperoleh dari `https://access.redhat.com/terms-based-registry/` → **New Service Account**. Username menyertakan prefix angka, contoh `89238478|do240`.

### Langkah 1.3 — Instal operator

GUI: **Operators → OperatorHub** → cari `3scale` → pilih **Red Hat Integration - 3scale** → **Install**:

| Field | Nilai |
|---|---|
| Update channel | `threescale-2.11` |
| Installed Namespace | `3scale` |
| Update approval | `Automatic` |

### Langkah 1.4 — Deploy APIManager

Pada halaman operator → **Provided APIs → APIManager → Create instance → YAML view**:

```yaml
apiVersion: apps.3scale.net/v1alpha1
kind: APIManager
metadata:
  name: apimanager-sample
  namespace: 3scale
spec:
  wildcardDomain: apps.ocp4.example.com
  resourceRequirementsEnabled: false
```

Setara via CLI:

```bash
[student@workstation ~]$ oc apply -f apimanager.yaml
```

### Verifikasi

```bash
[student@workstation ~]$ oc get pods -n 3scale
[student@workstation ~]$ oc get routes -n 3scale | grep 3scale-admin
```

Buka `https://3scale-admin.apps.ocp4.example.com` — harus tampil halaman login.

### ⚠️ Jebakan

- **`wildcardDomain` salah** → semua route jadi tidak resolvable. Harus domain cluster, di lab ini `apps.ocp4.example.com`.
- **Deploy butuh beberapa menit.** Admin Portal 404/503 di awal itu normal — tunggu, jangan hapus dan buat ulang.
- Pod `ErrImagePull` / `ImagePullBackOff` → secret registry belum dibuat atau belum ditautkan.
- Pod stuck bukan `Running`/`Completed` → resource node kurang; itulah gunanya `resourceRequirementsEnabled: false`.

---

## Kunci Tugas 2 — Konfigurasi 3scale Toolbox CLI

### Langkah 2.1 — Ambil password admin

```bash
[student@workstation ~]$ oc get secret system-seed -n 3scale \
  --template={{.data.ADMIN_PASSWORD}} | base64 -d; echo
```

Bentuk alternatif yang juga sah:

```bash
[student@workstation ~]$ oc get secret system-seed -n 3scale \
  -o json | jq -r .data.ADMIN_PASSWORD | base64 -d; echo
```

### Langkah 2.2 — Buat access token

Admin Portal → **Dashboard → Account Settings → Personal → Tokens → Add Access Token**:

| Field | Nilai |
|---|---|
| Name | `toolbox` |
| Scopes | **All** — Billing API, Account Management API, Analytics API, Policy Registry API |
| Permission | `Read & Write` |

**Salin token sekarang** — tidak ditampilkan lagi setelah halaman ditutup.

### Langkah 2.3 — Instal Toolbox

```bash
[student@workstation ~]$ podman login registry.redhat.io
Username: username_from_service_account
Password: token_from_service_account

[student@workstation ~]$ podman pull \
registry.redhat.io/3scale-amp2/toolbox-rhel8:3scale2.11
```

### Langkah 2.4 — Remote yang persisten

Ini inti soalnya. `podman run` membuat container **baru** tiap eksekusi, jadi remote hilang. Solusinya: buat named container berisi remote, lalu commit jadi image.

```bash
[student@workstation ~]$ podman run --name=toolbox-original \
registry.redhat.io/3scale-amp2/toolbox-rhel8:3scale2.11 3scale remote \
add 3scale-tenant -k https://ACCESS_TOKEN@3scale-admin.apps.ocp4.example.com

[student@workstation ~]$ podman commit toolbox-original toolbox
Getting image source signatures
...output omitted...
Storing signatures
```

> Salah ketik? `podman rm toolbox-original` lalu ulangi.

### Langkah 2.5 — Alias persisten

```bash
[student@workstation ~]$ alias 3scale="podman run toolbox 3scale -k"
```

Agar **bertahan antar sesi**, tambahkan ke `~/.bashrc`:

```bash
[student@workstation ~]$ echo 'alias 3scale="podman run toolbox 3scale -k"' >> ~/.bashrc
```

### Verifikasi

```bash
[student@workstation ~]$ 3scale remote list
3scale-tenant https://3scale-admin.apps.ocp4.example.com access-token
```

### ⚠️ Jebakan

- **Lupa `-k`** → error sertifikat SSL, karena image Toolbox tidak memuat CA lab.
- **Lupa `podman commit`** → remote hilang, setiap perintah gagal "remote not found".
- **Alias hanya di shell aktif** → soal minta persisten; ujian menilai state setelah reboot.

---

## Kunci Tugas 3 — Backend, Product, dan Promosi

### Langkah 3.1 — Product

```bash
[student@workstation ~]$ 3scale service create 3scale-tenant library_product
Service 'library_product' has been created with ID: 1
```

### Langkah 3.2 — Backend (via Admin Portal)

**Backends → Create Backend**, dua kali:

| Name | Private Base URL |
|---|---|
| `book-backend` | `http://books-api.book-library.svc.cluster.local:8080` |
| `patron-backend` | `http://patrons-api.book-library.svc.cluster.local:8080` |

### Langkah 3.3 — Asosiasi backend ke product

**Products → library_product → Integration → Backends → Add Backend**:

- `book-backend` dengan path **`/b-api`**
- `patron-backend` dengan path **`/p-api`**

### Langkah 3.4 — Promosi

**Integration → Configuration**:
1. **Promote v.1 to Staging APIcast**
2. **Promote v.1 to Production APIcast**

### Langkah 3.5 — Verifikasi

```bash
[student@workstation ~]$ API=https://library-product-3scale-apicast-production\
.apps.ocp4.example.com:443
[student@workstation ~]$ USER_KEY=02e852852327972ebc6edfae27b9037a

[student@workstation ~]$ curl -s $API/b-api/books?user_key=$USER_KEY | jq
[
  {
    "title": "Frankenstein",
    "authorName": "Mary Shelley",
    "year": 1818,
    "copies": 10
  },
  ...output omitted...
]

[student@workstation ~]$ curl -s $API/p-api/patrons?user_key=$USER_KEY | jq
[
  {
    "name": "John Smith",
    "numBooks": 3,
    "cardNumber": "00001"
  },
  ...output omitted...
]
```

### ⚠️ Jebakan

- **Private Base URL menyertakan path endpoint** (`/books`) → salah. Backend hanya root URL; endpoint dijangkau lewat path.
- **Path backend duplikat dalam satu product** → ditolak; path harus unik karena itulah dasar routing.
- **Promosi hanya ke staging** padahal soal minta production juga.
- **Lupa port `:8080`** pada Private Base URL.

---

## Kunci Tugas 4 — Application Plan Berbayar dan Rate Limit

### Langkah 4.1 — Plan via Toolbox

Butuh **service ID** lebih dulu:

```bash
[student@workstation ~]$ 3scale service list 3scale-tenant
ID      NAME              SYSTEM_NAME
1       library_product   library_product
```

```bash
[student@workstation ~]$ 3scale application-plan create -p \
  --approval-required=false --cost-per-month=10.00 --setup-fee=1.0 \
  -t "east_branch_plan" --trial-period-days=15 \
  3scale-tenant library_product "east-branch-plan"
Created application plan id: 16. Default: false; Disabled: false
```

**Arti opsi:**

| Opsi | Fungsi |
|---|---|
| `-p` / `--publish` | Plan langsung published (bukan hidden) |
| `-t` | System name plan |
| `--approval-required=false` | Application tidak perlu persetujuan admin |
| `--cost-per-month=10.00` | Fixed fee bulanan |
| `--setup-fee=1.0` | Biaya sekali di awal |
| `--trial-period-days=15` | Masa percobaan |

### Langkah 4.2 — Rate limit 5/menit pada backend

**Products → library_product → Applications → Application Plans → east-branch-plan** → scroll ke **Metrics, Methods, Limits & Pricing Rules** → bagian **Backend Level** → klik `book-backend` → **Limits → New usage limit**:

| Field | Nilai |
|---|---|
| Period | `minute` |
| Max. Value | `5` |

**Create usage limit** → lalu **Update Application plan**.

### Langkah 4.3 — Jadikan default

**Applications → Application Plans** → drop-down **Default Plan** → pilih `east-branch-plan`.

### Verifikasi

```bash
[student@workstation ~]$ 3scale application-plan list 3scale-tenant library_product
ID      NAME               SYSTEM_NAME
16      east-branch-plan   east_branch_plan
```

### ⚠️ Jebakan

- **Limit dipasang di Product Level padahal soal minta Backend Level** (atau sebaliknya). Baca cermat.
- **Lupa klik `Update Application plan`** setelah membuat usage limit.
- Argumen posisi Toolbox: urutannya `TENANT PRODUCT "NAMA PLAN"` — nama plan (dengan spasi) di akhir, system name lewat `-t`.

---

## Kunci Tugas 5 — Application, Kredensial, dan Uji Rate Limit

### Langkah 5.1 — Cari account

```bash
[student@workstation ~]$ 3scale account find 3scale-tenant john
org_name => Developer
id => 3
```

### Langkah 5.2 — Buat application

```bash
[student@workstation ~]$ 3scale application create \
  3scale-tenant 3 library_product east_branch_plan east-branch-app
Created application id: 9
```

> Argumen ke-2 bisa **ID account** (`3`) maupun **nama** (`john`) — keduanya diterima.

### Langkah 5.3 — Ambil user_key via CLI

```bash
[student@workstation ~]$ USER_KEY=$(3scale application show 3scale-tenant 9 \
  -o json | jq -r '.user_key')
[student@workstation ~]$ echo $USER_KEY
bc18aa400edba94148e37a4f632e34b2
```

Pola satu-baris tanpa tahu ID lebih dulu:

```bash
[student@workstation ~]$ USER_KEY=$(3scale application list 3scale-tenant -o json \
  | jq -r '.[] | select(.name == "east-branch-app") | .user_key')
```

### Langkah 5.4 — Uji rate limit

```bash
[student@workstation ~]$ DO240/labs/applications-plans/test_api_plan.sh $USER_KEY
Making 100 requests to curl -s https://...
Request No: 1
...output omitted...
Request No: 100
Finished!

[student@workstation ~]$ curl "https://library-product-3scale-apicast-staging.apps.ocp4.example.com:443/b-api/books?user_key=$USER_KEY"
Usage limit exceeded
```

**Jawaban yang dicari: `Usage limit exceeded`.**

### ⚠️ Jebakan

- Grafik Analytics mengelompokkan data **per jam**, jadi angkanya bisa tampak lebih besar dari limit per menit. Itu bukan bug.
- Rate limit berlaku **per application**, bukan per IP.

---

## Kunci Tugas 6 — URL Versioning

### Langkah 6.1 — Ambil variable via CLI

```bash
[student@workstation ~]$ APP_ID=$(3scale application list 3scale-tenant -o json \
   | jq '.[] | select(.name == "applications_versions_app") | .id')

[student@workstation ~]$ USER_KEY=$(3scale application \
  show 3scale-tenant $APP_ID -o json \
  | jq -r '.user_key')

[student@workstation ~]$ STAGING_DOMAIN=$(3scale proxy-config \
  show 3scale-tenant applications_versions sandbox -o json \
  | jq -r '.content.proxy.staging_domain')

[student@workstation ~]$ echo $STAGING_DOMAIN; echo $USER_KEY
applications-versions-3scale-apicast-staging.apps.ocp4.example.com
987f7abce12aaf49a8c297793bf44893
```

> `sandbox` = environment **staging** pada perintah `proxy-config show`.

### Langkah 6.2 — Backend v2

**Backends → Create Backend**:

| Field | Nilai |
|---|---|
| Name | `books_backend_v2` |
| Private Base URL | `http://books-api.applications-versions.svc.cluster.local:80/api/v2` |

### Langkah 6.3 — Asosiasi dengan path `/v2`

**Products → applications_versions → Integration → Backends → Add Backend** → pilih `books_backend_v2`, path **`/v2`**.

Penerjemahan yang terjadi (`/v2` dihapus lalu digabung ke base URL backend):

| Public URL | Private URL |
|---|---|
| `https://STAGING_DOMAIN/v2/books` | `http://books-api.applications-versions.svc.cluster.local/api/v2/books` |

### Langkah 6.4 — Metric

```bash
[student@workstation ~]$ 3scale metric create \
  3scale-tenant applications_versions hits_v2
Created metric id: 11. Disabled: false
```

### Langkah 6.5 — Mapping rule

**Integration → Mapping Rules → Create Mapping Rule**:

| Field | Nilai |
|---|---|
| Verb | `GET` |
| Pattern | `/v2` |
| Method or Metric to increment | **Metric** → `hits_v2` |

### Langkah 6.6 — Promosi via CLI

```bash
[student@workstation ~]$ 3scale proxy-config deploy \
  3scale-tenant applications_versions
{
  "service_id": 3,
  "endpoint": "https://applications-versions-3scale-apicast-production.apps.ocp4.example.com:443",
  ...output omitted...
```

> `deploy` → **staging**. `promote` → **production**. Ini pasangan yang sering tertukar.

### Verifikasi

```bash
[student@workstation ~]$ curl -s \
  "https://$STAGING_DOMAIN/v2/books?user_key=$USER_KEY" | jq
[
  {
    "title": "Frankenstein",
    "author": {
      "name": "Mary Shelley",
      "birthYear": 1797
    },
    "year": 1818,
    "copies": 10
  },
  ...output omitted...
]
```

### ⚠️ Jebakan

- **`No Mapping Rule matched`** → mapping rule `/v2` belum dibuat atau belum dipromosikan.
- Tanpa mapping rule, backend yang sudah terasosiasi **tetap tidak publik**.
- Mapping rule dan metric adalah dua hal terpisah: mapping rule membuka akses, metric menghitung.

---

## Kunci Tugas 7 — Request Header Versioning dengan Routing Policy

### Langkah 7.1 — File policy chain

```bash
[student@workstation ~]$ cat > policy-chain.yaml <<'EOF'
- name: routing
  version: builtin
  configuration:
    rules:
    - condition:
        combine_op: and
        operations:
        - value_type: plain
          match: header
          value: v2
          op: matches
          header_name: x-api-version
      url: http://books-api.applications-versions.svc.cluster.local:80/api/v2
  enabled: true
- name: apicast
  version: builtin
  configuration: {}
  enabled: true
EOF
```

**Urutan wajib: `routing` DULU, `apicast` SESUDAHNYA.**

### Langkah 7.2 — Import

```bash
[student@workstation ~]$ 3scale policies import \
  3scale-tenant applications_versions -f policy-chain.yaml
```

> Perintah ini membuat konfigurasi baru **dan** mempromosikannya ke staging.

### Alternatif via GUI

**Integration → Policies → Add policy → Routing**, konfigurasi:

| Field | Nilai |
|---|---|
| url | `http://books-api.applications-versions.svc.cluster.local:80/api/v2` |
| combine_op | `and` |
| match | `header` |
| value | `v2` |
| value_type | `Evaluate 'value' as plain text.` |
| op | `==` |
| header_name | `x-api-version` |

Geser `Routing` ke atas `3scale APIcast` → **Update Policy Chain** → promosikan.

### Verifikasi

```bash
[student@workstation ~]$ curl -v -s --header "x-api-version: v2" \
  "https://$STAGING_DOMAIN/books?user_key=$USER_KEY"
...output omitted...
> GET /books?user_key=987f7abce12aaf49a8c297793bf44893 HTTP/1.1
> x-api-version: v2
>
< HTTP/1.1 200 OK
...output omitted...
```

### ⚠️ Jebakan

- **`Routing` di bawah `apicast`** → policy tidak pernah dieksekusi. Ini kesalahan nomor satu pada soal policy.
- `import` **menimpa seluruh chain**. Kalau ada policy lain yang harus dipertahankan, `export` dulu, edit, baru `import`.
- Lupa policy `apicast` di file YAML → chain jadi tidak valid.

---

## Kunci Tugas 8 — Service Discovery

### Langkah 8.1 — Project & deploy

```bash
[student@workstation ~]$ oc login -u=admin -p=redhat \
  --server=https://api.ocp4.example.com:6443

[student@workstation ~]$ oc new-project library

[student@workstation ~]$ oc new-app \
  --name=books-api \
  --context-dir=library/books-api \
  --build-env NODE_ENV=development \
  https://github.com/RedHatTraining/DO240-apps.git
...output omitted...
  --> Success
```

### Langkah 8.2 — Metadata discovery

```bash
[student@workstation ~]$ oc label svc/books-api \
  discovery.3scale.net="true"
service/books-api labeled

[student@workstation ~]$ oc annotate svc/books-api \
  discovery.3scale.net/port="8080"
service/books-api annotated

[student@workstation ~]$ oc annotate svc/books-api \
  discovery.3scale.net/scheme="http"
service/books-api annotated
```

**Tiga metadata wajib:**

| Tipe | Key | Nilai |
|---|---|---|
| **Label** | `discovery.3scale.net` | `true` |
| **Annotation** | `discovery.3scale.net/scheme` | `http` |
| **Annotation** | `discovery.3scale.net/port` | `8080` |

Opsional: `/path`, `/description-path`, `/discovery-version`.

### Langkah 8.3 — Izin

```bash
[student@workstation ~]$ oc policy add-role-to-user \
  view system:serviceaccount:3scale:amp -n library
clusterrole.rbac.authorization.k8s.io/view added: "system:serviceaccount:3scale:amp"
```

Verifikasi discovery aktif (default aktif jika instalasi via operator):

```bash
[student@workstation ~]$ oc describe configmap system -n 3scale
...output omitted...
service_discovery.yml:
  production:
    enabled: <%= cluster_token_file_exists = ... %>
    bearer_token: "<%= File.read(cluster_token_file_path) ... %>"
    authentication_method: service_account
```

Cek dua sisi sekaligus:

```bash
[student@workstation ~]$ ~/DO240-apps/scripts/ls-discoverable.sh
{
  "service-name": "books-api",
  "service-namespace": "library",
  "labels": { "discovery.3scale.net": "true", ...output omitted... },
  "annotations": {
    "discovery.3scale.net/port": "8080",
    "discovery.3scale.net/scheme": "http",
    ...output omitted...
  }
}

[student@workstation ~]$ oc get rolebinding -o wide -A \
  | grep -E 'NAME|ClusterRole/view|3scale/amp'
NAMESPACE  NAME   ROLE               AGE     USERS   GROUPS   SERVICEACCOUNTS
library    view   ClusterRole/view   5d22h                    3scale/amp
```

### Langkah 8.4 — Import

Admin Portal → **Products → Create Product → Import from OpenShift** → Namespace `library`, Name `books-api` → **Create Product**.

### Langkah 8.5 — Plan + application

```bash
[student@workstation ~]$ 3scale application-plan create \
  3scale-tenant library-books-api library_basic_plan
Created application plan id: 13. Default: false; Disabled: false

[student@workstation ~]$ 3scale application create \
  3scale-tenant john library-books-api library_basic_plan library-app
Created application id: 7
```

> System name product hasil import berpola **`NAMESPACE-SERVICENAME`** → `library-books-api`.

### Verifikasi

```bash
[student@workstation ~]$ curl \
  https://library-books-api-3scale-apicast-staging.apps.ocp4.example.com:443/books?user_key=YOUR_USER_KEY
[{"title":"Frankenstein", ...output omitted...
```

### ⚠️ Jebakan

- Service tidak muncul di drop-down Namespace → label/annotation salah, **atau** SA `amp` belum punya role `view`.
- `ls-discoverable.sh` hanya cek **keberadaan** annotation, bukan nilainya. Port salah tetap lolos script tapi menghasilkan **502 Bad Gateway** saat diakses.
- Import saja **tidak cukup** — tanpa plan + application, tidak ada `user_key`, product tidak bisa diakses.

---

## Kunci Tugas 9 — Multi-tenancy

### Langkah 9.1 — Password master

```bash
[student@workstation ~]$ oc get secret system-seed -n 3scale \
  --template={{.data.MASTER_PASSWORD}} | base64 -d
```

Login ke `https://master.apps.ocp4.example.com/` sebagai `master`.

### Langkah 9.2 — Buat tenant

**Dashboard → Audience → Accounts → Create**:

| Field | Nilai |
|---|---|
| Username | `do240-user` |
| Email | `do240@redhat.com` |
| Password | `do240` |
| Password confirmation | `do240` |
| Organization/Group name | `do240` |

> **Di 3scale, tenant = master account.** `Organization/Group name` menjadi subdomain semua URL tenant.

### Langkah 9.3 — Aktifkan user

Klik **2 Users** → baris `do240-user` → **Activate**.

> Tenant baru otomatis punya dua user: `do240-user` (milik Anda) dan `3scale Admin` (dipakai sistem). Keduanya ber-role admin.

### Langkah 9.4 — Empat URL

| Komponen | URL |
|---|---|
| Admin Portal | `https://do240-admin.apps.ocp4.example.com` |
| Developer Portal | `https://do240.apps.ocp4.example.com` |
| Staging APIcast | `https://api-do240-apicast-staging.apps.ocp4.example.com` |
| Production APIcast | `https://api-do240-apicast-production.apps.ocp4.example.com` |

> Pola APIcast di atas berlaku untuk product default `API`. Product lain berpola `PRODUCT-TENANT-apicast-staging.WILDCARD_DOMAIN`.

### Langkah 9.5 — Undang user

Admin Portal `do240` → **Dashboard → Account Settings → Users → Invitations → Invite a New Team Member** → `gls@redhat.com` → **Send**.

**Logout dulu**, lalu ambil email:

```bash
[student@workstation ~]$ ~/DO240-apps/scripts/get-emails.sh
---------- MESSAGE FOLLOWS ----------
...output omitted...
Please sign up by following this link: https://do240-admin.apps.ocp4.example.com/p/signup/c9d30638114c6c9433867c5775689278
...output omitted...
------------ END MESSAGE ------------
```

Buka link → daftar:

| Field | Nilai |
|---|---|
| Username | `gls-user` |
| First name | `GLS` |
| Last name | `Red Hat` |
| Password | `gls-password` |

### Langkah 9.6 — Beri izin

Login sebagai `do240-user` → **Account Settings → Users → Listing → GLS Red Hat** → bagian **Administrative**:

1. Centang **`Create, read, update and delete developer accounts and applications of selected API products`**
2. Muncul opsi baru → pilih **`All current and future API products`**
3. **Update User**

Verifikasi: login sebagai `gls-user` → Dashboard kini menampilkan product `API`.

### ⚠️ Jebakan

- **Lupa Activate** → `do240-user` tidak bisa login sama sekali.
- **Masih login saat membuka link signup** → form signup tidak muncul. Logout dulu.
- User undangan default ber-role **`member` tanpa izin apa pun** — dashboard-nya kosong. Itu perilaku normal, bukan error.
- Master Portal sempat down setelah `lab start` karena pod di-restart untuk email server. Tunggu.

---

## Kunci Tugas 10 — Self-Managed APIcast Gateway

### Langkah 10.1 — Project & operator

```bash
[student@workstation ~]$ oc login \
  -u=admin -p=redhat --server=https://api.ocp4.example.com:6443
[student@workstation ~]$ oc new-project apicast
```

GUI: **Operators → OperatorHub** → cari `APIcast` → **Red Hat Integration - 3scale APIcast gateway** → **Install** → Installed Namespace `apicast`.

### Langkah 10.2 — Access token

Admin Portal tenant `3scale` → **Account Settings → Personal → Tokens → Add Access Token**:

| Field | Nilai |
|---|---|
| Name | `apicast` |
| Scopes | **`Account Management API`** |
| Permission | `Read Only` |

### Langkah 10.3 — Secret

```bash
[student@workstation ~]$ oc create secret generic apicast-secret \
  --from-literal=AdminPortalURL=https://ACCESS_TOKEN@3scale-admin.apps.ocp4.example.com
secret/apicast-secret created
```

### Langkah 10.4 — Custom resource

```yaml
apiVersion: apps.3scale.net/v1alpha1
kind: APIcast
metadata:
  name: custom-apicast
spec:
  adminPortalCredentialsRef:
    name: apicast-secret
  deploymentEnvironment: staging
  exposedHost:
    host: custom-apicast.apps.ocp4.example.com
    tls:
    - {}
  resources:
    limits:
      cpu: '0'
      memory: 128Mi
```

```bash
[student@workstation ~]$ oc apply -f apicast.yaml
apicast.apps.3scale.net/custom-apicast created
```

### Langkah 10.5 — Arahkan product

**Products → gateways_apicast → Integration → Settings** → bagian **DEPLOYMENT** → pilih **`APIcast self-managed`** → **Staging Public Base URL** = `https://custom-apicast.apps.ocp4.example.com` → **Update Product**.

**Integration → Configuration → Promote v.1 to Staging APIcast**.

### Verifikasi

```bash
[student@workstation ~]$ curl "https://custom-apicast.apps.ocp4.example.com:443/echo?user_key=USER_KEY"
{
  "method": "GET",
  "path": "/",
  ...output omitted...
  "headers": {
    "HTTP_VERSION": "HTTP/1.1",
    "HTTP_HOST": "echo-api.3scale.net",
    ...output omitted...
  }
}
```

### ⚠️ Jebakan

- **Scope token salah.** Wajib `Account Management API` — gateway memakainya untuk menarik konfigurasi dari APIManager. Scope lain tidak cukup.
- **`deploymentEnvironment` salah** (`production` padahal soal minta `staging`) → gateway menggantikan environment yang keliru.
- Self-managed gateway **tidak otomatis berlaku ke semua product**. Harus diaktifkan per product di Integration → Settings.
- Operator APIcast membuat route sendiri dari `exposedHost.host`; tidak perlu `oc create route` manual.

---

## Kunci Tugas 11 — APIcast Policies

### Langkah 11.1 — Promosi awal & ambil key

**Products → gateways_policies → Integration → Configuration → Promote v.1 to Staging APIcast**. Salin `user_key` dari contoh curl.

```bash
[student@workstation ~]$ curl \
https://gateways-policies-3scale-apicast-staging.apps.ocp4.example.com:443/echo
Authentication parameters missing
```

### Langkah 11.2 — Anonymous Access

**Integration → Policies → Add policy → Anonymous Access**. Klik policy tersebut:

| Field | Nilai |
|---|---|
| enabled | ✅ |
| auth_type | `user_key` |
| user_key | *(key yang Anda salin)* |

**Update Policy** → geser `Anonymous Access` ke **atas** `3scale APIcast` → **Update Policy Chain** → **Promote v.2 to Staging APIcast**.

```bash
[student@workstation ~]$ curl \
https://gateways-policies-3scale-apicast-staging.apps.ocp4.example.com:443/echo
{
  "method": "GET",
  "path": "/",
  ...output omitted...
}
```

> ⚠️ Risiko: siapa pun dengan URL bisa memanggil API, dengan **izin yang sama** seperti pemilik key yang dikonfigurasi.

### Langkah 11.3 — Maintenance Mode

**Add policy → Maintenance Mode → Update Policy Chain → Promote v.3 to Staging APIcast**.

```bash
[student@workstation ~]$ curl -i \
https://gateways-policies-3scale-apicast-staging.apps.ocp4.example.com:443/echo
HTTP/1.1 503 Service Temporarily Unavailable
...output omitted...
Service Unavailable - Maintenance
```

**Jawaban status code: `503`.**

### Langkah 11.4 — Hapus

Klik **Maintenance Mode** → **Remove** (bawah form) → **Update Policy Chain** → **Promote v.4 to Staging APIcast**.

```bash
[student@workstation ~]$ curl \
https://gateways-policies-3scale-apicast-staging.apps.ocp4.example.com:443/echo
{
  "method": "GET",
  ...output omitted...
}
```

### Langkah 11.5 — Ekspor

```bash
[student@workstation ~]$ 3scale policies export 3scale-tenant \
gateways_policies >> policy_chain.yml

[student@workstation ~]$ cat policy_chain.yml
---
- name: anonymous_access
  version: builtin
  configuration:
    auth_type: user_key
    user_key: some-user-key
  enabled: true
- name: apicast
  version: builtin
  configuration: {}
  enabled: true
```

### ⚠️ Jebakan

- **Setiap perubahan policy butuh promosi baru.** Versi naik tiap kali (v.2, v.3, v.4). Lupa promote = tidak ada efek.
- **`Update Policy` ≠ `Update Policy Chain`.** Yang pertama menyimpan konfigurasi satu policy, yang kedua menyimpan chain-nya. Keduanya perlu diklik.
- Urutan salah → `Anonymous Access` di belakang `apicast` tidak berpengaruh.

---

## Kunci Tugas 12 — User Admin Portal & Kontrol Akses

### Langkah 12.1 — Undang member

**Account Settings → Users → Invitations → Invite a New Team Member** → `member_user@redhat.com`.

```bash
[student@workstation ~]$ ~/DO240-apps/scripts/get-emails.sh
---------- MESSAGE FOLLOWS ----------
From: no-reply@apps.ocp4.example.com
To: member_user@redhat.com
...output omitted...
Please sign up by following this link: https://3scale-admin.apps.ocp4.example.com/p/signup/dbbfcd4fe317fae0e0bdc2187e70da6b
...output omitted...
```

Logout → buka link → daftar `member_user` / `gls-password`.

### Langkah 12.2 — Verifikasi tanpa izin

Login sebagai `member_user`: welcome page menyatakan tidak ada akses ke API mana pun; menu **Products** dan **Backends** tidak dapat dibuka.

### Langkah 12.3 — Beri izin analytics saja

Login sebagai `admin` → **Account Settings → Users → Listing → member_user** → bagian **ADMINISTRATIVE**:

1. Klik **`Access & query analytics of`**
2. Pilih **`All current and future existing API products`**
3. Submit

Verifikasi: login sebagai `member_user` → buka product `API` → **hanya menu Analytics** yang tersedia di sidebar.

### Langkah 12.4 — Buat admin

Undang `user_admin@redhat.com` → ambil link → daftar `admin_user` / `gls-password` → sebagai `admin`, buka **Users → Listing → admin_user** → **ADMINISTRATIVE** → pilih **`Admin (full access)`** → submit.

Verifikasi: `admin_user` punya akses penuh.

### Ringkasan dua tingkat izin

| Tingkat | Contoh izin |
|---|---|
| **Level Admin Portal** | Manage Developer Portal content; Manage customer billing; Update settings di Audience (Accounts, Applications, Billing, Developer Portal, Messages) |
| **Level per-product** | Developer accounts & applications; Analytics; Product & backend configuration; Policies & policy chains |

Izin per-product bisa diberikan ke **semua product (current & future)** atau **subset** saja.

### ⚠️ Jebakan

- Soal minta izin **hanya analytics** — jangan berikan izin lain, penilaian bisa memeriksa izin berlebih.
- Undangan butuh SMTP. Di lab, email ditangkap interceptor; di produksi butuh secret `system-smtp` yang benar.

---

## Kunci Tugas 13 — API Key dan App_ID/App_Key Pair

### Langkah 13.1 — Application

```bash
[student@workstation ~]$ 3scale application list 3scale-tenant \
  --plan=secure_keys_basic --service=secure_keys
ID	NAME	STATE	ENABLED	ACCOUNT_ID	SERVICE_ID	PLAN_ID

[student@workstation ~]$ 3scale application create 3scale-tenant \
  john secure_keys secure_keys_basic secure_keys_app
Created application id: 13
```

### Langkah 13.2 — Uji dengan user_key

```bash
[student@workstation ~]$ API_KEY=$(3scale application show 3scale-tenant 13 \
  -o json | jq -r '.user_key')
[student@workstation ~]$ echo $API_KEY
bc18aa400edba94148e37a4f632e34b2

[student@workstation ~]$ curl \
  "https://secure-keys-3scale-apicast-staging.apps.ocp4.example.com:443/books?user_key=$API_KEY" | jq
...output omitted...
```

### Langkah 13.3 — Ubah ke key-pair

**Products → secure_keys → Integration → Settings** → bagian **Authentication** → pilih:

> **`App_ID and App_Key Pair — The application is identified via the App_ID and authenticated via the App_Key`**

**Update Product** → **Configuration** → **Promote v.2 to Staging APIcast**.

> Nilai `user_key` lama menjadi nilai `app_key` default.

### Langkah 13.4 — Buktikan user_key gagal

```bash
[student@workstation ~]$ curl \
  "https://secure-keys-3scale-apicast-staging.apps.ocp4.example.com:443/books?user_key=$API_KEY"; echo
Authentication parameters missing
```

**Jawaban pesan error: `Authentication parameters missing`.**

### Langkah 13.5 — Panggilan dengan pasangan

```bash
[student@workstation ~]$ API_ID=$(3scale application show 3scale-tenant 13 \
  -o json | jq -r '.application_id')
[student@workstation ~]$ echo $API_ID
eb34878e

[student@workstation ~]$ curl \
  "https://secure-keys-3scale-apicast-staging.apps.ocp4.example.com:443/books?app_key=$API_KEY&app_id=$API_ID"| jq
...output omitted...
```

Bentuk header (jika product dikonfigurasi memakai header):

```bash
[student@workstation ~]$ curl "https://.../" \
  --header 'app_id: 76b49d63' \
  --header 'app_key: 0a5eb0c64bd6748b6e37a7084a273e07'
```

### ⚠️ Jebakan

- **Lupa promosi setelah ganti pola autentikasi** → `user_key` masih jalan, soal dianggap belum selesai.
- Bedakan field JSON: **`user_key`** (jadi app_key) vs **`application_id`** (jadi app_id). Bukan `.id` — `.id` adalah ID internal application (13).
- Maksimum **5 key** per application ID pada pola key-pair.

---

## Kunci Tugas 14 — Integrasi OIDC dengan RHSSO

### Langkah 14.1 — Kondisi awal

```bash
[student@workstation ~]$ 3scale application list 3scale-tenant \
  --service=secure_oauth
ID	NAME	STATE	ENABLED	ACCOUNT_ID	SERVICE_ID	PLAN_ID
7	secure_oauth_app	live	true	3	3	10

[student@workstation ~]$ 3scale application show 3scale-tenant 7 \
 -o json | jq -r '.user_key'
acaa2950a6124cd27a45194c023d2a3d

[student@workstation ~]$ curl \
  "https://secure-oauth-3scale-apicast-staging.apps.ocp4.example.com/books?user_key=acaa2950a6124cd27a45194c023d2a3d" \
  | jq
...output omitted...
```

Berhasil **tanpa** JWT → OIDC belum aktif.

### Langkah 14.2 — CA trust untuk Zync

```bash
[student@workstation ~]$ sh \
  /home/student/DO240/labs/secure-oauth/inject_rhsso_ca.sh
...output omitted...
configmap/zync-new-ca-bundle created
deploymentconfig.apps.openshift.io/zync-que volume updated
deploymentconfig.apps.openshift.io/zync-que updated

[student@workstation ~]$ oc -n 3scale get pods \
  -l threescale_component_element=zync-que
NAME               READY   STATUS    RESTARTS   AGE
zync-que-3-jmsst   1/1     Running   0          8s
```

> Tanpa ini, `zync-que` gagal TLS ke RHSSO dan sinkronisasi client tidak pernah terjadi.

### Langkah 14.3 — Client zync-client

```bash
[student@workstation ~]$ oc -n rhsso get secret \
  credential-keycloak --template={{.data.ADMIN_PASSWORD}} \
  | base64 -d ; echo
7jpffNJgZTG8yA==
```

Buka `https://keycloak-rhsso.apps.ocp4.example.com/auth/admin/master/console/#/realms/do240`, login `admin`.

1. **Clients → Create → Select file** → `/home/student/DO240/labs/secure-oauth/zync-client.json` → **Save**
2. Tab **Service Account Roles** → **Client Roles** → **Realm Management** → pilih **`manage-clients`** → **Add selected**
3. Tab **Credentials** → catat **Secret**

> `manage-clients` adalah izin yang memungkinkan Zync **membuat client lain** di RHSSO. Tanpa itu, log Zync menunjukkan `insufficient_scope` / `Forbidden`.

### Langkah 14.4 — Konfigurasi OIDC pada product

**Products → secure_oauth → Integration → Settings** → **Authentication** → **`OpenID Connect — Use OpenID Connect for any OAuth 2.0 flow.`**

**OpenID Connect Issuer URL** berformat `https://CLIENT-ID:CLIENT-SECRET@RHSSO-URL`:

```
https://zync-client:f6f0..7dd4@keycloak-rhsso.apps.ocp4.example.com/auth/realms/do240
```

**Update Product** → **Configuration** → **Promote v.2 to Staging APIcast**.

### Langkah 14.5 — Application

**Applications → Listing → Create Application** → account `Developer`, product `secure_oauth`, plan `secure_oauth_basic`:

- Name: `sso_app`
- Description: `sso_app`

Catat **Client ID**, misal `7308b6b9`.

### Langkah 14.6 — Ubah client RHSSO

RHSSO → **Clients** → klik client `7308b6b9` (namanya `sso_app`):

| Property | Nilai |
|---|---|
| Access Type | `public` |
| Valid Redirect URIs | `*` |
| Web Origins | `*` |

**Save**.

### Langkah 14.7 — Front end

```bash
[student@workstation ~]$ oc edit cm book-config -n secure-oauth
apiVersion: v1
data:
  REACT_APP_CLIENT_ID: 7308b6b9
...output omitted...

[student@workstation ~]$ oc -n secure-oauth delete pod \
  -l app=books-frontend
pod "books-frontend-v2-f5658556-57vqh" deleted
```

> Configmap tidak otomatis termuat ulang; pod harus dibuat ulang.

### Langkah 14.8 — CORS

**Integration → Policies → Add policy → CORS Request Handling** → geser ke **posisi pertama** chain → konfigurasi `allow_origin` = `*` → **Update Policy** → **Update Policy Chain** → promosikan.

> ⚠️ CORS **wajib pertama**. Kalau tidak, front end gagal.

### Langkah 14.9 — Verifikasi

Buka `http://books-frontend-secure-oauth.apps.ocp4.example.com`, login `student` / `redhat`.

### Troubleshooting yang diuji

| Gejala | Diagnosis |
|---|---|
| Client tidak muncul di RHSSO | `oc logs -l threescale_component_element=zync-que` → cari `insufficient_scope` / `Forbidden` |
| APIcast menolak JWT | Aktifkan `apicast.stagingSpec.logLevel: debug`, lalu `oc logs -l threescale_component_element=staging` → contoh: `'aud' claim is required` |
| Front end blank walau API 200 | CORS kurang atau posisinya salah |

### ⚠️ Jebakan

- Empat hal yang diverifikasi 3scale pada JWT: **ada JWT**, **ditandatangani OIDC provider**, **valid/tidak expired**, **memuat claim wajib (`azp`, `aud`)**.
- Discovery endpoint yang dibaca APIcast: `https://OIDC_PROVIDER_HOST/.well-known/openid-configuration`.
- Setelah OIDC aktif, autentikasi dengan `user_key` saja **ditolak**.

---

## Kunci Tugas 15 — Developer Portal: Publikasi & Konten Terbatas

### Langkah 15.1 — Publikasikan

**Dashboard → Audience → Developer Portal → Content** → klik **Open your Portal to the world** → **Ok** → **Visit Portal**.

### Langkah 15.2 — Daftarkan user

Di portal: **Signup to plan Basic**:

| Field | Nilai |
|---|---|
| Organization/group name | `GLS` |
| Username | `portal-user` |
| Email | `gls@redhat.com` |
| Password | `redhat` |

Admin Portal → **Accounts → Listing** → **Activate** account `portal-user`.

> Signup otomatis membuat **account + user + application**. Buka `GLS → GLS's App` untuk melihatnya.

### Langkah 15.3 — Section & page privat

**Developer Portal → Content** → panah di samping **New Page** → **New Section**:

| Field | Nilai |
|---|---|
| Title | `Restricted section` |
| Public | ❌ tidak dicentang |

**Create Section**. Lalu **New Page**:

| Field | Nilai |
|---|---|
| Title | `Restricted page` |
| Section | `Restricted section` |
| Path | `/restricted` |
| Content | `<h1>Welcome to the Restricted Section</h1>` |

**Create Page** → **Publish**.

### Langkah 15.4 — Verifikasi tertutup

Buka jendela **incognito** (Ctrl+Shift+N) → `https://3scale.apps.ocp4.example.com/restricted` → **`Not found`**.

> Kalau masih login sebagai admin, Anda tetap melihat kontennya. Wajib incognito atau logout.

### Langkah 15.5 — Group

**Developer Portal → Groups → Create Group** → Name `privileged` → **Create Group**.

Klik **Privileged** → pilih `Restricted section` → **Save**.

> Ada bug: section harus dipilih **setelah** group dibuat, lalu group di-update. Membuat group dan memilih section dalam satu langkah tidak tersimpan.

**Accounts → Listing → GLS → 0 Group Memberships** → pilih `privileged` → **Save**.

### Langkah 15.6 — Verifikasi terbuka

Login sebagai `portal-user` / `redhat` di `https://3scale.apps.ocp4.example.com/` → buka `/restricted` → konten tampil.

### Tiga langkah kunci (hafalkan)

1. Section **non-public** + page di dalamnya
2. **Group** yang diberi akses ke section itu
3. **Assign account** ke group

---

## Kunci Tugas 16 — Kustomisasi Developer Portal

### Langkah 16.1 — Nama organisasi

**Account Settings** → di samping `Account Details` klik **Edit** → Organization name `Bob's Boxes` → **Update Account**.

### Langkah 16.2 — Logo

**Audience → Developer Portal → Logo** → unggah `~/DO240/labs/portal-customizing/icon.png`.

### Langkah 16.3 — Main layout

**Developer Portal → Content** → di bawah **Layouts** pilih **`Main layout`**:

```html
...output omitted...
      <nav class="navbar navbar-fixed-top navbar-inverse" role="navigation">
        <div class="container tabbed">
          <div class="navbar-header">
            <button type="button" class="navbar-toggle collapsed" data-toggle="collapse" data-target="#navbar-1">
              <span class="sr-only">Toggle navigation</span>
              <span class="icon-bar"></span>
              <span class="icon-bar"></span>
              <span class="icon-bar"></span>
            </button>
            <a class="navbar-brand" href="/">
              <img src="{{ provider.logo_url }}" height="32px" />
              {{  provider.name }}
            </a>
          </div>
          {% include 'submenu'%}
        </div>
      </nav>
...output omitted...
```

**Publish**.

> Variable Liquid yang dicari: **`{{ provider.logo_url }}`** dan **`{{ provider.name }}`**.

### Langkah 16.4 — CSS

Pilih file **`default.css`**:

```css
...output omitted...
body {
  background-color: darkorange; /* fallback */
  background-color: rgb(255, 140, 0);
  ...output omitted...
}
...output omitted...
```

**Publish**.

### Langkah 16.5 — Page About

Buka file **`Documentation`** → **New Page**:

| Field | Nilai |
|---|---|
| Title | About |
| Path | /about |
| Advanced Options | **Liquid enabled** ✅ |

**Create Page**, isi kontennya:

```html
<h1>About</h1>

<p>Lorem ipsum dolor sit amet, consectetur adipiscing elit. Etiam accumsan est id neque interdum posuere. Ut eleifend et sapien nec pharetra. Vivamus condimentum magna vitae justo rutrum, non vestibulum odio ultricies. Morbi eget sem sit amet erat dapibus molestie. Nullam nec felis et massa tincidunt interdum. Integer imperdiet nisi ac lacus feugiat pretium. Suspendisse accumsan consequat augue, a feugiat metus commodo sed.</p>
```

**Publish**.

### Langkah 16.6 — Navigasi

Buka partial **`submenu`** di bawah **Partials**:

```html
...output omitted...
  {% if current_user %}
    ...output omitted...
        <li><a class="{% if urls.docs.active? %}active{% endif %}" href="/docs">Documentation</a></li>

        <li class="{% if request.path == "/about" %}active{% endif %}">
          <a href="/about">About</a>
        </li>

      </ul>
  ...output omitted...
  {% else %}
    <ul class="nav navbar-nav">
      <li><a href="/docs">Documentation</a></li>
      <li><a href="/#plans">Plans</a></li>
      <li class="{% if request.path == "/about" %}active{% endif %}">
          <a href="/about">About</a>
      </li>
    </ul>
    ...output omitted...
  {% endif %}
</div>
```

**Publish**.

### ⚠️ Jebakan

- Link harus ditambahkan di **kedua cabang** `{% if current_user %}` — user login **dan** anonim. Menambahkan hanya di satu cabang = setengah nilai.
- **Publish, bukan Save.** Save hanya menyimpan draft; portal publik tidak berubah.
- Sintaks Liquid: `{{ }}` **menghasilkan output**, `{% %}` **kontrol struktur tanpa output**.

---

## Kunci Tugas 17 — Import OpenAPI & ActiveDocs

### Langkah 17.1 — Edit definisi

Pada `~/DO240/labs/portal-openapi/openapi-definition.yaml`, bagian `servers`, ganti `CHANGE_ME` menjadi:

```
http://petshelter-api.portal-openapi.svc.cluster.local
```

### Langkah 17.2 — Import

```bash
[student@workstation ~]$ \
  ~/DO240/labs/portal-openapi/3scale_wrapper.sh import openapi \
  --target_system_name=portal_openapi \
  --destination=3scale-tenant \
  DO240/labs/portal-openapi/openapi-definition.yaml
{
  "code": "E_3SCALE",
  "message": "User key must be provided by --default-credentials-userkey optional param",
  "class": "ThreeScaleToolbox::Error"
}
Created service id: 3, name: Pet Shelter
Service proxy updated
destroying all mapping rules
Created PUT /pet$ endpoint
Created POST /pet$ endpoint
Created GET /pet/findByStatus$ endpoint
Created GET /pet/{petId}$ endpoint
Created DELETE /pet/{petId}$ endpoint
```

> Pesan `E_3SCALE` itu **normal dan boleh diabaikan** — hanya muncul karena `--default-credentials-userkey` tidak diberikan.

**Yang otomatis dibuat import:** product (nama dari `title`), backend (nama dari `title`, URL dari `servers`), serta mapping rules & methods (dari `paths`).

### Langkah 17.3 — Verifikasi backend

```bash
[student@workstation ~]$ 3scale proxy-config show \
  3scale-tenant portal_openapi sandbox -o json \
  | jq -r '.content.proxy.api_backend'
http://petshelter-api.portal-openapi.svc.cluster.local:80
```

### Langkah 17.4 — Plan + application

```bash
[student@workstation ~]$ 3scale application-plan create \
  3scale-tenant portal_openapi basic_plan
Created application plan id: 13. Default: false; Disabled: false

[student@workstation ~]$ 3scale application create \
  3scale-tenant john portal_openapi basic_plan shelter-app
Created application id: 7
```

### Langkah 17.5 — CORS untuk dua domain

**Integration → Policies → Add policy → CORS Request Handling**, geser sebelum `3scale APIcast`:

| Field | Nilai |
|---|---|
| allow_origin | `(3scale-admin\|3scale).apps.ocp4.example.com` |

**Update Policy** → **Update Policy Chain** → promosikan ke **staging** lalu **production**.

> Sintaks kurung mengizinkan dua domain sekaligus: Admin Portal (`3scale-admin.apps.ocp4.example.com`) dan Developer Portal (`3scale.apps.ocp4.example.com`).

### Langkah 17.6 — Uji ActiveDocs

Product `portal_openapi` → **ActiveDocs** → klik `portal_openapi` → **GET /pet/findByStatus** → **Try it out**:

- `status` = `available`
- drop-down **First userkey from latest 5 applications** → `shelter-app`
- **Execute**

```json
{
  "0": {
    "photoUrls": ["photo_1_url", "photo_2_url"],
    "name": "spotty",
    "id": 0,
    "category": { "name": "dogs", "id": 6 },
...output omitted...
```

### Langkah 17.7 — user_key satu perintah

```bash
[student@workstation ~]$ 3scale application list \
  --service=portal_openapi 3scale-tenant -o json \
  | jq -r '.[] | select(.name == "shelter-app")|.user_key'
7acb0243fdc7a30882f39dbf95261ab7
```

### ⚠️ Jebakan

- Tanpa CORS, ActiveDocs interaktif gagal karena request berasal dari domain portal, bukan domain API.
- Import **tidak** membuat application plan/application — harus dibuat manual.
- `--target_system_name` menentukan system name **product, backend, dan ActiveDocs** sekaligus; `--destination` menentukan **tenant**. Sering tertukar.

---

## Kunci Tugas 18 — Custom Metrics & Analytics

### Langkah 18.1 — Metric `status` + mapping rule

**Integration → Methods & Metrics → New metric**:

| Field | Nilai |
|---|---|
| Friendly name | `status` |
| System name | `status` |
| Unit | `1` |
| Description | Tracking requests to the status endpoint. |

**Create Metric**. Lalu pada baris metric `status` klik **Add a mapping rule**:

| Field | Nilai |
|---|---|
| Verb | `GET` |
| Pattern | `/status` |
| Method or Metric to increment | **Metric** → `status` |

Setara via CLI:

```bash
[student@workstation ~]$ 3scale metric create 3scale-tenant monitoring_analytics status --unit=1
```

### Langkah 18.2 — Metric `status-203`

**New metric**:

| Field | Nilai |
|---|---|
| Friendly name | `status-203` |
| System name | `status-203` |
| Unit | `1` |
| Description | Tracking of 203 HTTP response codes. |

> Metric ini **sengaja tidak** ditautkan mapping rule — kenaikannya ditangani policy.

### Langkah 18.3 — Policy Custom Metrics

**Integration → Policies → Add policy → Custom Metrics** → geser ke **posisi pertama** → klik untuk konfigurasi:

| Field | Nilai |
|---|---|
| Metric to increment | `status-203` |
| left | `{{status}}` |
| right | `203` |
| op | `matches` |
| left_type | `Evaluate 'left' as liquid` |
| right_type | `Evaluate 'right' as plain text` |

**Update Policy** → **Update Policy Chain**.

> Ekspresi terbaca: `{{status}} matches 203`. Variable built-in **`{{status}}`** memberi status code response. `left` harus **liquid**, `right` harus **plain**.

### Langkah 18.4 — Promosi

**Integration → Configuration → Promote v.1 to Staging APIcast**. Salin `user_key`.

### Langkah 18.5 — Uji

```bash
[student@workstation ~]$ DO240/labs/monitoring-analytics/test_status_api.sh \
 USER_KEY
Making 20 requests
Responses with status code 200: 7
Responses with status code 201: 8
Responses with status code 203: 5
```

**Analytics → Traffic** → drop-down:

| Metric | Nilai yang diharapkan |
|---|---|
| `status-203` | **5** (sama dengan jumlah response 203) |
| `status` | **20** (seluruh request) |

### ⚠️ Jebakan

- **Policy tidak di posisi pertama** → tidak pernah dieksekusi, metric tetap 0.
- **`left_type` dan `right_type` tertukar** → `{{status}}` diperlakukan sebagai teks harfiah, tidak pernah cocok.
- Dua cara menaikkan metric: **mapping rule** (otomatis saat path diakses) vs **policy Custom Metrics** (berbasis status code response). Soal ini menguji keduanya sekaligus.

---

## Kunci Tugas 19 — Monitoring Prometheus & Grafana

### Langkah 19.1 — Metrik mentah APIcast

```bash
[student@workstation ~]$ oc login -u admin https://api.ocp4.example.com:6443
[student@workstation ~]$ oc project 3scale

[student@workstation ~]$ oc rsh svc/apicast-staging
sh-4.4$ curl http://localhost:9421/metrics
...output omitted...
nginx_http_connections{state="accepted"} 129
nginx_http_connections{state="active"} 1
nginx_http_connections{state="handled"} 129
...output omitted...
sh-4.4$ exit
```

**Jawaban port: `9421`.**

### Langkah 19.2 — Prometheus

**Operators → OperatorHub** → `prometheus` → pastikan project `3scale` terpilih → **Prometheus Operator → Install** → Installed Namespace `3scale`.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: 3scale-monitor
  namespace: 3scale
spec:
  serviceAccountName: prometheus-k8s
  podMonitorSelector:
   matchExpressions:
   - key: app
     operator: In
     values:
     - 3scale-api-management
  ruleSelector:
    matchExpressions:
    - key: app
      operator: In
      values:
      - 3scale-api-management
```

```bash
[student@workstation ~]$ oc apply -f \
DO240/labs/monitoring-infrastructure/prometheus-3scale-monitor.yml
prometheus.monitoring.coreos.com/3scale-monitor created
```

> Soal minta memilih **podmonitor DAN rule** → butuh `podMonitorSelector` **dan** `ruleSelector`. Melewatkan salah satu = alert rule tidak terbaca.

### Langkah 19.3 — Grafana

**OperatorHub** → `grafana` → **Grafana Operator → Install** → namespace `3scale`.

```yaml
apiVersion: integreatly.org/v1alpha1
kind: Grafana
metadata:
  name: grafana
  namespace: 3scale
spec:
  ingress:
    enabled: True
  config:
    log:
      mode: "console"
      level: "warn"
    auth:
      disable_login_form: False
      disable_signout_menu: True
    auth.anonymous:
      enabled: True
  dashboardLabelSelector:
  - matchExpressions:
    - key: app
      operator: In
      values:
      - 3scale-api-management
```

```bash
[student@workstation ~]$ oc apply -f \
DO240/labs/monitoring-infrastructure/grafana-3scale.yml
grafana.integreatly.org/grafana created
```

Data source:

```yaml
apiVersion: integreatly.org/v1alpha1
kind: GrafanaDataSource
metadata:
  name: prometheus
  namespace: 3scale
spec:
  name: middleware
  datasources:
    - name: Prometheus
      type: prometheus
      access: proxy
      url: http://prometheus-operated:9090
      isDefault: true
      version: 1
      editable: true
      jsonData:
        timeInterval: "5s"
```

```bash
[student@workstation ~]$ oc apply -f \
DO240/labs/monitoring-infrastructure/grafana-prometheus-datasource.yml
grafanadatasource.integreatly.org/prometheus created
```

### Langkah 19.4–19.5 — Aktifkan monitoring & buktikan

**Sebelum:**

```bash
[student@workstation ~]$ oc get podmonitors
No resources found in 3scale namespace.
[student@workstation ~]$ oc get prometheusrules
No resources found in 3scale namespace.
```

```bash
[student@workstation ~]$ oc edit apimanager apimanager-sample
```

Tambahkan di dalam `spec`:

```yaml
spec:
  ...output omitted...
  monitoring:
    enabled: true
  ...output omitted...
```

**Sesudah:**

```bash
[student@workstation ~]$ oc get podmonitors
...output omitted...
apicast-production   1s
apicast-staging      1s
...output omitted...

[student@workstation ~]$ oc get prometheusrules
...output omitted...
apicast                         51s
backend-listener                52s
...output omitted...
```

### Langkah 19.6 — Grafana

`https://grafana-route-3scale.apps.ocp4.example.com/dashboards` → **3scale** → **Apicast** → pilih **Last 5 minutes**.

> Operator 3scale membuat **tujuh** dashboard. Tiga yang perlu diingat: **Apicast** (error request, pemilih staging/production), **System** (tenant request per detik), **Zync** (Zync request per detik, CPU).

### ⚠️ Jebakan

- **Urutan salah.** Operator + instance harus ada **sebelum** `monitoring.enabled: true`, kalau tidak resource yang dibuat 3scale tidak ada yang memungutnya.
- **Label `app=3scale-api-management` harus persis.** Salah label = Prometheus tidak menemukan apa pun.
- Grafana kosong di awal itu wajar — Prometheus baru mengumpulkan data setelah monitoring aktif. Ganti rentang ke **Last 5 minutes**.

---

## Kunci Tugas 20 — Billing dan Pricing Rules

### Bagian A — Billing

**A.1 — Mode prepaid**

**Audience → Billing → Settings → Charging & Gateway** → **Switch to PREPAID** → **OK**.

**A.2 — Plan berbayar**

```bash
[student@workstation ~]$ 3scale application-plan create --publish \
  --approval-required=false --cost-per-month=10.0 --setup-fee=30.0 \
  3scale-tenant monetizing_billing "monthly_plan"
Created application plan id: 11. Default: false; Disabled: false
```

**A.3 — Application**

```bash
[student@workstation ~]$ 3scale application create 3scale-tenant \
  john monetizing_billing monthly_plan monetizing_billing_app
Created application id: 7
```

**A.4 — Percepat jadwal billing**

```bash
[student@workstation ~]$ oc login -u=admin -p=redhat \
  --server=https://api.ocp4.example.com:6443

[student@workstation ~]$ oc -n 3scale patch cm system --type merge \
  --patch-file /home/student/DO240/labs/monetizing-billing/sidekiq_schedule.yml
configmap/system patched

[student@workstation ~]$ oc -n 3scale patch dc system-sidekiq \
  --patch-file /home/student/DO240/labs/monetizing-billing/system-volumes.yml
deploymentconfig.apps.openshift.io/system-sidekiq patched
```

> ⚠️ File `sidekiq_schedule.yml` lab **sengaja tidak lengkap** — hanya memuat cron job billing. Di produksi, konfigurasi ini akan mematikan jenis job lain.

**A.5 — Majukan trial & verifikasi**

```bash
[student@workstation ~]$ sh \
  /home/student/DO240/labs/monetizing-billing/update_trial_dates.sh
mysql: [Warning] Using a password on the command line interface can be insecure.
```

**Audience → Billing → Invoices** → invoice account `Developer` dengan state **Finalized**. Klik nomornya → **Line Items**:

| Line item | Nilai |
|---|---|
| Fixed fee ('monthly_plan') | maksimum **$10** (diprorata sesuai jarak ke awal bulan) |
| Setup fee ('monthly_plan') | **$30** |

> Invoice terbit **sehari setelah** trial berakhir. Job berjalan tiap menit — refresh kalau daftar masih kosong.

### Bagian B — Pricing Rules

**B.6 — Fixed cost**

**Products → echo → Applications → Application Plans → custom**:

| Field | Nilai |
|---|---|
| Setup fee | `5.00` |
| Cost per month | `3.00` |

**Update Application plan**.

**B.7 — Tiga pricing rule**

Di **Metrics, Methods, Limits & Pricing Rules** → baris metric **`Hits`** → **Pricing (0)** → **new pricing rule** tiga kali:

| From | To | Cost Per Unit |
|---|---|---|
| `1` | `120` | `0.5` |
| `121` | `1121` | `0.3` |
| `1122` | *(kosong)* | `0.1` |

**Create pricing rule** setiap kali.

> Rentang terakhir dengan `To` kosong = **tak terbatas ke atas**.
> Karena `Hits` menangkap semua path (`/`), pricing rule ini berlaku ke **seluruh endpoint** product.

### Konsep billing yang diuji

**Mode:**

| Mode | Perilaku |
|---|---|
| **Prepaid** | Fixed + setup fee ditagih **awal bulan berjalan**; variable cost bulan ini ditagih awal bulan depan |
| **Postpaid** | **Semua** fee ditagih awal bulan berikutnya |

**Siklus state invoice:**

```
Open → Finalized → Pending → Paid
                      ↓
                   Unpaid → (3x gagal) → Failed
Open/Finalized/Pending → Canceled (oleh admin)
```

**Fixed vs variable:**

| Jenis | Sifat |
|---|---|
| **Setup fee** | Sekali di awal langganan. **Tidak** dikenakan lagi saat pindah plan |
| **Cost per month** | Bulanan tetap. **Diprorata** jika langganan di tengah bulan |
| **Pricing rule** | Variable, per rentang request, pada **metric atau method** |

### ⚠️ Jebakan

- **Pricing rule dibuat di application plan, bukan di halaman settings product.**
- Setup fee **tidak** berulang saat pindah plan — pertanyaan jebakan klasik.
- Invoice `Open` **masih bisa** diubah; `Finalized` sudah memuat seluruh charge periode berjalan.
- Verifikasi pricing rule butuh siklus bulanan — di lab tidak bisa dibuktikan langsung. Itu memang disengaja.

---

# BAGIAN D — Kunci Jawaban Soal Konsep (Bagian B)

**B1.** **API Gateway (APIcast).** Gateway menarik informasi routing, user, dan rate limit dari API Manager **secara periodik lalu meng-cache-nya**. Karena itu ia tidak perlu bertanya ke API Manager tiap request (mengurangi latency) dan tetap bisa bekerja mandiri saat API Manager down beberapa menit.

**B2.**
- Admin Portal: `finance-admin.example.com`
- Developer Portal: `finance.example.com`
- Staging APIcast: `api-finance-apicast-staging.example.com`

**B3.** **Satu.** Backend tidak "dimiliki" product; satu backend yang sama dapat diasosiasikan ke banyak product.

**B4.** `http://svc.ns.svc.cluster.local:8080` — **hanya root URL**. Backend menunjuk ke root aplikasi agar satu backend dapat melayani banyak endpoint; path (`/api/orders/list`) ditentukan oleh mapping product/backend dan request klien.

**B5.** **Konfigurasi belum dipromosikan.** Perubahan product tidak live sampai di-**Promote to Staging**, lalu **Promote to Production**. Ikon peringatan pada item Configuration menandakan ada perubahan yang tertahan.

**B6.**
- Staging: `3scale proxy-config deploy TENANT PRODUCT`
- Production: `3scale proxy-config promote TENANT PRODUCT`

**B7.** Keduanya merujuk objek yang sama — **product dan service adalah hal yang sama** di 3scale. Kursus menyarankan memakai super-command **`service`**, karena belum semua fitur dipindahkan ke super-command `product`.

**B8.** Secret **`system-seed`** di namespace `3scale`. Key: **`ADMIN_PASSWORD`** (admin Admin Portal) dan **`MASTER_PASSWORD`** (user master di Master Portal).

**B9.**
1. Label `discovery.3scale.net` = `true`
2. Annotation `discovery.3scale.net/scheme` = `http` atau `https`
3. Annotation `discovery.3scale.net/port` = nomor port

(Opsional: `/path`, `/description-path`, `/discovery-version`.)

**B10.** Service account **`amp`** dari project 3scale, diberi role **`view`** **pada project tempat service berada**:

```bash
oc policy add-role-to-user view system:serviceaccount:3SCALE_PROJECT:amp -n PROJECT_NAME
```

Untuk seluruh cluster: `oc adm policy add-cluster-role-to-user view system:serviceaccount:3SCALE_PROJECT:amp`.

**B11.** **Ya.** Jika 3scale diinstal via operator, service discovery aktif secara default. Verifikasi lewat `oc describe configmap system -n 3scale` pada entry `service_discovery.yml` → `production.enabled`.

**B12.** `http://svc/api/v2/books`. 3scale **menghapus public path `/v2`** dari request, lalu menggabungkan sisanya (`/books`) ke Private Base URL backend.

**B13.** **Sebelum** policy `apicast`. Policy chain dieksekusi berurutan; jika `Routing` berada setelah `apicast`, request sudah diproses dan diteruskan oleh apicast sehingga routing tidak pernah berlaku.

**B14.** **`Anonymous Access`.** Risikonya: siapa pun yang tahu URL dapat memanggil API mana pun milik product tersebut, dan request itu memperoleh **izin yang sama** dengan pemilik user key yang dikonfigurasi dalam policy.

**B15.** **`Maintenance Mode`** — merespons semua request dengan status code dan pesan yang telah dikonfigurasi (default `503 Service Unavailable - Maintenance`).

**B16.** **`3scale APIcast`** (`apicast`). Policy ini merepresentasikan konfigurasi routing 3scale itu sendiri.

**B17.**
- (a) `spec.deploymentEnvironment` — bernilai `staging` atau `production`
- (b) `spec.exposedHost.host` — host route RHOCP eksternal

**B18.** **`Account Management API`.** Gateway memakainya untuk menarik konfigurasi (routing, limit) dari APIManager. Permission `Read Only` sudah memadai.

**B19.** **Lima.**

**B20.** Karena **ID dan secret token terpisah**, Anda bisa membuat beberapa key untuk satu app ID: terbitkan secret baru → perbarui aplikasi → nonaktifkan secret lama. Rotasi berjalan **tanpa gangguan layanan**. Pada API key tunggal, key berperan sekaligus sebagai ID dan secret, sehingga regenerasi langsung memutus akses.

**B21.** **Zync** yang menyinkronkan (berkomunikasi asinkron dengan RHSSO dan membuat API credentials); **Sidekiq** yang menjadwalkan Zync saat application baru dibuat.

**B22.** **`manage-clients`** — client role dari **Realm Management**, diberikan pada tab **Service Account Roles**. Tanpa itu Zync tidak dapat membuat client di RHSSO (log menunjukkan `insufficient_scope` / `Forbidden`).

**B23.** **`azp`** (authorized party) — harus berkorespondensi dengan API credentials application di product. 3scale juga memverifikasi kehadiran claim wajib lain seperti **`aud`** (audience), bahwa token ditandatangani OIDC provider yang dikonfigurasi, dan bahwa token belum expired.

**B24.** Policy **`CORS Request Handling`**, dan harus ditempatkan **paling awal** dalam policy chain. Pastikan juga RHSSO mengimplementasikan CORS (Web Origins pada client).

**B25.** **Layout** mendefinisikan **struktur umum halaman** (header, footer, referensi CSS/JS) yang dipakai banyak page. **Partial** adalah **elemen kecil yang dapat dipakai ulang dan disematkan di dalam page** (misalnya disclaimer atau submenu). Layout membungkus page; partial dimuat oleh page.

**B26.**
1. Buat **section non-public** dan page di dalamnya
2. Buat **group** dan beri akses ke section tersebut
3. **Assign account/user** ke group itu (Accounts → Listing → Group membership)

**B27.**
- **Product** — nama dari atribut `title` file OAS
- **Backend** — nama dari `title`, URL dari bagian `servers`
- **Mapping rules dan methods** — dari bagian `paths`
- **ActiveDocs**

Tidak termasuk: application plan dan application — keduanya harus dibuat manual.

**B28.** Metric **`Hits`**, menangkap path **`/`** (semua path). Implikasinya: **pricing rule yang ditautkan ke `Hits` berlaku untuk seluruh endpoint product**, bukan sebagian.

**B29.** Field **`spec.monitoring.enabled: true`** pada `APIManager`. Setelahnya operator 3scale membuat **podmonitors** (mis. `apicast-staging`, `apicast-production`) dan **prometheusrules** (mis. `apicast`, `backend-listener`), plus tujuh dashboard Grafana.

**B30.** Pada hari pertama bulan, mode **postpaid** menjalankan: menagih **variable cost bulan sebelumnya**, **finalize invoice `open` bulan sebelumnya**, dan **membuat invoice `open` bulan ini sambil menagih fixed cost**. Invoice menjadi **`failed` setelah tiga kali** percobaan pembayaran gagal.

---
---

# BAGIAN E — Lembar Penilaian Mandiri

Isi setelah menyelesaikan Bagian A. Nilai **hanya jika state akhirnya benar dan terverifikasi** — sama seperti ujian asli.

| # | Tugas | Poin | Diperoleh |
|---|---|---|---|
| 1 | Instalasi 3scale | 15 | ___ |
| 2 | Toolbox CLI | 10 | ___ |
| 3 | Backend, Product, Promosi | 20 | ___ |
| 4 | Application Plan & Rate Limit | 20 | ___ |
| 5 | Application & Kredensial | 15 | ___ |
| 6 | URL Versioning | 20 | ___ |
| 7 | Routing Policy | 15 | ___ |
| 8 | Service Discovery | 20 | ___ |
| 9 | Multi-tenancy | 20 | ___ |
| 10 | Self-Managed APIcast | 20 | ___ |
| 11 | APIcast Policies | 20 | ___ |
| 12 | User & Kontrol Akses | 15 | ___ |
| 13 | API Key & Key-Pair | 15 | ___ |
| 14 | OIDC / RHSSO | 25 | ___ |
| 15 | Developer Portal: Restricted | 20 | ___ |
| 16 | Kustomisasi Portal | 15 | ___ |
| 17 | Import OpenAPI | 20 | ___ |
| 18 | Custom Metrics | 20 | ___ |
| 19 | Prometheus & Grafana | 20 | ___ |
| 20 | Billing & Pricing | 20 | ___ |
| | **TOTAL** | **365** | **___** |

**Konversi ke skala 300:** `(perolehan ÷ 365) × 300`

| Skor | Interpretasi |
|---|---|
| ≥ 255 (70%) | Setara passing score ujian Red Hat — siap |
| 220–254 | Nyaris. Ulangi domain yang gagal, lalu tes ulang |
| 180–219 | Perlu latihan lab terarah pada domain lemah |
| < 180 | Ulangi seluruh Guided Exercise DO240 sebelum simulasi lagi |

### Diagnosis per Domain

Jika Anda kehilangan poin di area ini, ulangi bagian rangkuman berikut:

| Tugas gagal | Baca ulang |
|---|---|
| 1, 2 | Bab 1 — Instalasi & Toolbox |
| 3, 4, 5 | Bab 2.1, 2.2 — Backend, Product, Plan |
| 6, 7 | Bab 2.3 — Versioning |
| 8 | Bab 2.4 — Service Discovery |
| 9 | Bab 2.5 — Multi-tenancy |
| 10, 11 | Bab 3 — APIcast & Policies |
| 12, 13, 14 | Bab 4 — Keamanan |
| 15, 16, 17 | Bab 5 — Developer Portal |
| 18, 19 | Bab 6 — Monitoring |
| 20 | Bab 7 — Monetisasi |

---

# BAGIAN F — Strategi Ujian & Kesalahan Fatal

## Alokasi Waktu (3 jam)

| Fase | Durasi | Aktivitas |
|---|---|---|
| **Survei** | 10 menit | Baca **semua** soal dulu. Tandai yang punya dependensi. |
| **Fondasi** | 30 menit | Instalasi, Toolbox, remote, akses portal. Semua bergantung ke sini. |
| **Eksekusi** | 100 menit | Kerjakan dari yang paling Anda kuasai. Lewati yang macet. |
| **Sapuan** | 25 menit | Kembali ke soal yang dilewati. |
| **Verifikasi** | 15 menit | **Promosikan semua**, uji ulang setiap endpoint. |

## Sepuluh Kesalahan yang Paling Mahal

1. **Tidak mempromosikan konfigurasi.** Nomor satu, sejauh ini. Perubahan yang tidak dipromosikan tidak terlihat oleh penilai. Biasakan: ubah → **Promote to Staging** → (jika diminta) **Promote to Production**.
2. **Urutan policy chain salah.** `CORS` dan `Custom Metrics` harus **pertama**; `Routing` dan `Anonymous Access` harus **sebelum** `apicast`.
3. **Lupa `Update Policy Chain`** setelah `Update Policy`. Dua tombol berbeda, keduanya wajib.
4. **Private Base URL berisi path endpoint.** Backend hanya root URL.
5. **Konfigurasi tidak persisten.** Alias di shell aktif saja tidak cukup — tulis ke `~/.bashrc`. Remote Toolbox tanpa `podman commit` akan hilang.
6. **Scope access token salah.** APIcast gateway butuh `Account Management API`; Toolbox butuh scope penuh.
7. **Metadata service discovery tidak lengkap.** Label + dua annotation, ketiganya wajib. Plus role `view` untuk SA `amp`.
8. **Product hasil import tidak bisa diakses** karena application plan + application belum dibuat.
9. **Menukar `deploy` dan `promote`.** `deploy` = staging, `promote` = production.
10. **Panik saat pod belum `Running`.** Deployment 3scale butuh beberapa menit. Kerjakan tugas lain sambil menunggu — jangan hapus dan buat ulang, itu justru memicu error `System Name has already been taken` yang butuh 10+ menit untuk pulih.

## Perintah Verifikasi Cepat

Jalankan di 15 menit terakhir:

```bash
# Semua pod 3scale sehat?
oc get pods -n 3scale

# Semua remote Toolbox terbaca?
3scale remote list

# Daftar product & plan
3scale service list 3scale-tenant
3scale application-plan list 3scale-tenant PRODUCT

# Konfigurasi mana yang aktif di staging?
3scale proxy-config show 3scale-tenant PRODUCT sandbox -o json | jq '.version'

# Policy chain aktual
3scale policies export 3scale-tenant PRODUCT

# Endpoint benar-benar hidup?
curl -i "https://HOST/PATH?user_key=KEY"
```

## Jika Ada Yang Macet

- **502 Bad Gateway** → port/URL backend salah, atau service backend mati. Cek `oc get svc` dan log APIcast.
- **`No Mapping Rule matched`** → mapping rule belum ada untuk path itu, atau belum dipromosikan.
- **`Authentication parameters missing`** → pola autentikasi tidak cocok dengan parameter yang dikirim.
- **`Usage limit exceeded`** → rate limit bekerja (kadang ini justru hasil yang diinginkan soal).
- **`ErrImagePull`** → secret registry belum dibuat/ditautkan.
- **`422 System Name has already been taken`** → resource lama belum benar-benar terhapus. 3scale eventually consistent; tunggu 10 menit atau pakai nama lain.

---

## Sumber Belajar Sah

| Sumber | Kegunaan |
|---|---|
| Halaman resmi EX240 di redhat.com | Objektif ujian terbaru — **selalu cek sebelum ujian** |
| Kursus DO240 (Red Hat Learning Subscription) | Materi resmi persiapan, dengan lab |
| `access.redhat.com/documentation/.../red_hat_3scale_api_management/2.11/` | Dokumentasi produk — biasanya tersedia saat ujian, biasakan menavigasinya |
| `github.com/RedHatTraining/DO240-apps` | Aplikasi & script lab |
| [RANGKUMAN-DO240-3scale-EX240.md](./RANGKUMAN-DO240-3scale-EX240.md) | Rangkuman lengkap kursus ini |

**Yang harus dihindari:** situs "exam dumps" yang mengklaim menjual soal asli. Selain melanggar perjanjian sertifikasi Red Hat dan berisiko pembatalan permanen, isinya biasanya usang dan tidak melatih keterampilan hands-on yang justru dinilai EX240.

---

*Simulasi ini disusun dari objektif ujian EX240 yang dipublikasikan Red Hat dan seluruh isi kursus DO240 v2.11 (revisi `do240-2.11-40390f6`). Bukan soal ujian asli.*
