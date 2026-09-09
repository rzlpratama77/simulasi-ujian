# Panduan Lengkap Study Points EX240 — Real Use Case Red Hat 3scale

> **Basis:** materi HTML DO240 v2.11 di folder ini dan dokumentasi resmi Red Hat 3scale 2.11.  
> **Tujuan:** menjelaskan setiap study point secara langkah demi langkah tanpa `lab start`, `lab finish`, wrapper, atau script bantuan RHLS/DO240.  
> **Gaya latihan:** lingkungan OpenShift dan 3scale sungguhan dengan backend API milik sendiri.

---

## 1. Batasan Versi dan Cara Membaca Panduan

Panduan ini mengikuti tampilan dan istilah 3scale 2.11 yang dipakai materi lokal. Pada versi yang berbeda, nama tombol kecil dapat berubah, tetapi konsep dan alur utamanya sama.

Contoh real use case menggunakan perusahaan fiktif **Nusantara Retail**:

| Komponen | Nilai contoh |
|---|---|
| Namespace 3scale | `3scale` |
| Tenant | `retail` |
| Admin Portal | `https://retail-admin.apps.example.com` |
| Developer Portal | `https://retail.apps.example.com` |
| Product | `orders_product` |
| Backend | `orders_backend` |
| Private Base URL | `http://orders-api.orders.svc.cluster.local:8080` |
| Public path backend | `/orders-api` |
| Plan gratis | `orders_free` |
| Plan berbayar | `orders_pro` |
| Developer account | `partner-a` |
| Application | `partner-a-orders-app` |

Ganti semua domain, nama object, namespace, dan credential sesuai lingkungan Anda.

### Peta study points

| Objective | Bagian panduan |
|---|---|
| Access control | Bagian 2 |
| Accounts | Bagian 3 |
| Analytics | Bagian 4 |
| API authentication | Bagian 5 |
| API documentation | Bagian 6 |
| Billing | Bagian 7 |
| Developer portal | Bagian 8 |
| Service discovery | Bagian 9 |
| Product definition | Bagian 10 |
| Service integration | Bagian 11 |
| Latihan mandiri | Bagian 12 |

### 1.1 Istilah yang jangan tertukar

- **Product** adalah API publik yang dilihat konsumen dan mempunyai route APIcast.
- **Backend** adalah API privat/upstream yang sebenarnya.
- **Developer account** adalah organisasi/customer yang memakai API.
- **Application** mewakili satu client/workload milik developer; credential melekat pada application.
- **Application plan** menentukan limit dan harga sebuah application.
- **Provider/admin user** adalah orang yang mengelola Admin Portal, bukan konsumen API.
- **Developer Portal** adalah situs untuk dokumentasi, signup, plan, dan credential developer.
- **APIcast** adalah API gateway 3scale.
- **Staging** dipakai untuk pengujian konfigurasi; **production** untuk trafik resmi.

### 1.2 Orientasi GUI Admin Portal

Di bagian atas Admin Portal terdapat pemilih konteks. Konteks yang paling sering dipakai:

- **Products**: product, backend, mapping rule, plan, policy, gateway, ActiveDocs, analytics.
- **Audience**: developer account, application lintas product, portal, group, billing, invoice.
- **Account Settings**: provider user, invitation anggota tim, token, detail organisasi, notifikasi.

Pola navigasinya:

1. Pilih konteks dari menu/drop-down bagian atas.
2. Pilih product atau backend bila diminta.
3. Gunakan menu sisi kiri.
4. Tombol membuat object biasanya berada di kanan atas atau kanan daftar.
5. Tombol penyimpanan biasanya berada di bagian bawah form.
6. Perubahan integration/policy belum aktif sampai dipromosikan.

### 1.3 Persiapan CLI tanpa script DO240

Toolbox dapat dijalankan dari image resmi. Jangan memasukkan token ke repository atau shell history bersama kode aplikasi.

```bash
podman login registry.redhat.io
podman pull registry.redhat.io/3scale-amp2/toolbox-rhel8:3scale2.11
```

Buat token di **Account Settings → Personal → Tokens → Add Access Token**. Untuk administrasi lengkap, pilih scope yang diperlukan dan `Read & Write`. Simpan token di password manager atau secret store.

Toolbox menyimpan remote pada file `$HOME/.3scalerc.yaml`. Untuk container, gunakan file konfigurasi host yang di-mount dan tunjuk file itu secara eksplisit dengan `--config-file`; cara ini lebih aman dan lebih jelas daripada menebak home directory image:

```bash
mkdir -p "$HOME/.config/3scale-toolbox"
install -m 600 /dev/null "$HOME/.config/3scale-toolbox/config.yaml"
podman run --rm \
  -v "$HOME/.config/3scale-toolbox/config.yaml:/config/.3scalerc.yaml:Z" \
  registry.redhat.io/3scale-amp2/toolbox-rhel8:3scale2.11 \
  3scale --config-file=/config/.3scalerc.yaml \
  remote add retail-tenant -k \
  https://ACCESS_TOKEN@retail-admin.apps.example.com
```

Lindungi file tersebut karena berisi token. Jangan commit file ke Git atau menaruhnya di image yang akan didistribusikan.

Contoh fungsi shell lokal agar perintah lebih pendek:

```bash
three_scale() {
  podman run --rm \
    -v "$HOME/.config/3scale-toolbox/config.yaml:/config/.3scalerc.yaml:Z" \
    registry.redhat.io/3scale-amp2/toolbox-rhel8:3scale2.11 \
    3scale --config-file=/config/.3scalerc.yaml -k "$@"
}
```

Ini fungsi lokal biasa, bukan script RHLS. Dalam contoh berikut, perintah ditulis sebagai `3scale`; sesuaikan dengan binary, alias, atau fungsi yang Anda gunakan.

---

# 2. Access Control

## 2.1 Create and maintain application plans

### Tujuan

Membuat paket akses berbeda untuk API yang sama. Misalnya Free untuk percobaan dan Pro untuk customer berbayar.

### Membuat application plan melalui GUI

1. Login ke Admin Portal.
2. Pada pemilih konteks bagian atas, klik **Products**.
3. Klik product **Orders Product** atau `orders_product`.
4. Pada menu kiri, buka **Applications → Application Plans**.
5. Klik tombol **Create Application Plan** di kanan halaman.
6. Isi:
   - **Name**: `Orders Free`.
   - **System name**: `orders_free`.
   - **Applications require approval?**: nonaktif untuk aktivasi otomatis, aktif bila admin harus meninjau setiap application.
   - **Trial period**: misalnya `14` hari jika diperlukan.
   - **Setup fee** dan **Cost per month**: `0` untuk plan gratis.
7. Klik **Create Application Plan**.
8. Kembali ke daftar plan. Plan baru biasanya hidden/draft.
9. Pada baris plan, buka menu tindakan lalu klik **Publish** agar developer dapat memilihnya.
10. Bila plan harus dipilih otomatis saat signup, gunakan drop-down **Default Plan** dan pilih `Orders Free`.

Ulangi untuk plan Pro:

| Field | Nilai contoh |
|---|---|
| Name | `Orders Pro` |
| System name | `orders_pro` |
| Approval required | Ya |
| Trial | 7 hari |
| Setup fee | 10.00 |
| Cost per month | 49.00 |

### Membuat melalui Toolbox

```bash
3scale application-plan create --publish \
  --approval-required=false \
  --trial-period-days=14 \
  --setup-fee=0 \
  --cost-per-month=0 \
  -t orders_free \
  retail-tenant orders_product "Orders Free"

3scale application-plan create --publish \
  --approval-required=true \
  --trial-period-days=7 \
  --setup-fee=10.00 \
  --cost-per-month=49.00 \
  -t orders_pro \
  retail-tenant orders_product "Orders Pro"
```

Verifikasi:

```bash
3scale application-plan list retail-tenant orders_product
```

### Memelihara plan

- **Edit**: klik nama plan, ubah field, lalu klik **Update Application Plan**.
- **Hide**: mencegah signup baru tanpa memutus application yang sudah berada pada plan.
- **Publish**: membuat plan tersedia bagi developer.
- **Default**: plan otomatis yang dipakai bila workflow tidak memilih plan lain.
- **Delete**: lakukan hanya jika tidak ada application aktif dan kebijakan retensi memperbolehkan.

### Verifikasi nyata

1. Buka Developer Portal pada mode incognito.
2. Pastikan hanya plan published yang muncul.
3. Buat application uji.
4. Pastikan application terhubung ke product dan plan yang benar.
5. Untuk plan approval-required, pastikan state menunggu approval sebelum credential dapat digunakan.

## 2.2 Configure limits and pricing rules

### Membuat usage limit

1. **Products → Orders Product → Applications → Application Plans**.
2. Klik `Orders Free`.
3. Scroll ke **Metrics, Methods, Limits & Pricing Rules**.
4. Pilih metric atau method yang akan dibatasi, misalnya **Hits** atau `Create Order`.
5. Klik **Limits** atau link jumlah limit pada baris tersebut.
6. Klik **New usage limit**.
7. Pilih period: `minute`, `hour`, `day`, `week`, `month`, `eternity`, sesuai kebutuhan UI.
8. Isi **Max. value**, misalnya `60` per minute.
9. Klik **Create usage limit**.
10. Bila halaman plan mempunyai tombol **Update Application Plan**, klik tombol itu juga.

Contoh kebijakan yang sehat:

| Plan | Metric/method | Limit |
|---|---|---|
| Free | Hits | 60/menit dan 10.000/bulan |
| Pro | Hits | 600/menit dan 1.000.000/bulan |
| Free | Create Order | 100/hari |

Limit pada **product level** menghitung seluruh trafik product. Limit pada **backend level** hanya menghitung trafik backend tersebut. Pastikan memilih level yang diminta.

### Mengonfigurasi pricing rules

1. Buka plan berbayar `Orders Pro`.
2. Pastikan **Setup fee** dan **Cost per month** telah diisi.
3. Pada bagian **Metrics, Methods, Limits & Pricing Rules**, cari metric yang ditagih.
4. Klik **Pricing (0)** atau jumlah pricing rule pada baris metric.
5. Klik **New pricing rule**.
6. Buat rentang kontigu, misalnya:

| From | To | Cost per unit |
|---:|---:|---:|
| 1 | 10.000 | 0.010 |
| 10.001 | 100.000 | 0.007 |
| 100.001 | kosong | 0.004 |

7. Klik **Create pricing rule** untuk setiap rentang.
8. Pastikan rentang tidak tumpang tindih dan tidak mempunyai gap yang tidak disengaja.

### Catatan penting

- Metric `Hits` default menangkap path `/`, sehingga pricing pada Hits berlaku untuk semua endpoint yang cocok.
- Harga melekat pada application plan, bukan langsung pada product.
- Setup fee dikenakan sekali; monthly fee dapat diprorata; pricing rule adalah variable cost.
- Uji rate limit dengan application khusus, bukan credential production customer.

---

# 3. Accounts

## 3.1 Create and update user accounts

Ada dua jenis user yang sering tertukar.

### A. Membuat developer account/customer

1. Pilih **Audience** dari menu atas.
2. Menu kiri: **Accounts → Listing**.
3. Klik **Create** atau **Create Account** di kanan atas.
4. Isi organization name, username admin account, email, dan data wajib lain.
5. Klik **Create**.
6. Buka detail account yang baru.
7. Periksa state account dan user admin di dalam account.
8. Aktifkan account/user bila workflow signup Anda memerlukan activation manual.

Untuk memperbarui, buka account dari **Accounts → Listing**, klik **Edit** di dekat Account Details, ubah data, lalu **Update Account**.

### B. Membuat provider/admin user

1. Pilih **Account Settings** pada menu atas.
2. Buka **Users → Invitations**.
3. Klik **Invite a New Team Member**.
4. Masukkan email perusahaan.
5. Klik **Send Invitation**.
6. User membuka email, mengklik link, mengisi username/password, lalu login.
7. User baru menjadi `member` tanpa izin signifikan.
8. Admin membuka **Users → Listing**, klik user, lalu memilih role/permission yang diperlukan.

Gunakan role `admin` hanya untuk orang yang benar-benar membutuhkan full access. Untuk tim analytics atau billing, gunakan `member` dengan permission sempit.

## 3.2 Configure user credentials

### Credential developer application

1. Pilih **Audience → Applications → Listing**.
2. Klik application.
3. Cari panel **API Credentials**.
4. Untuk API key, salin `user_key` melalui kanal aman.
5. Untuk App_ID/App_Key, catat `Application ID` dan daftar application keys.
6. Gunakan **Add Random Key** untuk key yang dihasilkan sistem atau **Add Custom Key** jika kebijakan mengharuskan nilai tertentu.
7. Maksimum lima key dapat terkait dengan satu application pada pola App_ID/App_Key.

Jangan mengirim credential melalui tiket publik atau chat tanpa proteksi. Simpan pada secret manager milik aplikasi client.

### Credential provider user

1. User membuka **Account Settings → Personal**.
2. Gunakan halaman profile/password bila perlu mengubah password.
3. Gunakan **Personal → Tokens** untuk management API.
4. Cabut token yang tidak lagi digunakan.
5. Bila SSO Admin Portal digunakan, kelola credential utama di identity provider, bukan membagikan password lokal.

### Rotasi App_Key tanpa downtime

1. Tambahkan random key kedua pada application.
2. Masukkan key kedua ke secret manager client.
3. Roll out client memakai key kedua.
4. Verifikasi request sukses dan analytics masuk ke application yang sama.
5. Hapus key pertama dari 3scale.
6. Pastikan request dengan key lama gagal.

## 3.3 Send user invitations

### Undangan anggota tim provider

1. **Account Settings → Users → Invitations**.
2. Klik **Invite a New Team Member**.
3. Isi email dan kirim.
4. Pastikan SMTP tenant/cluster sudah dikonfigurasi; pada real use case tidak ada email interceptor RHLS.
5. Minta penerima mengecek inbox dan spam.
6. Setelah registrasi, admin mengatur permission dari **Users → Listing**.

### Undangan developer pada account yang ada

1. **Audience → Accounts → Listing**.
2. Klik developer account tujuan.
3. Buka tab/section **Users**.
4. Klik **Invite user** atau **Add user**.
5. Masukkan email.
6. Penerima menyelesaikan signup dan menjadi user di dalam developer account tersebut.

### Troubleshooting invitation

- Undangan tidak masuk: periksa konfigurasi SMTP, log pod System, sender domain, DNS/SPF/DKIM, dan spam quarantine.
- Link expired: batalkan/abaikan invitation lama lalu kirim ulang.
- Email sudah digunakan: cari user/account lama sebelum membuat duplikat.

## 3.4 Maintain group memberships

Group pada objective ini biasanya dipakai untuk membatasi konten Developer Portal.

### Membuat group dan memberikan akses section

1. Pilih **Audience**.
2. Buka **Developer Portal → Groups**.
3. Klik **Create Group**.
4. Isi nama, misalnya `gold-partners`, lalu buat.
5. Buka **Developer Portal → Content**.
6. Buat atau pilih section, misalnya `Partner Documentation`.
7. Nonaktifkan public access pada section.
8. Pada permission section/group, beri group `gold-partners` akses ke section.
9. Publish section dan page di bawahnya.

### Menambah account ke group

1. **Audience → Accounts → Listing**.
2. Klik account, misalnya `partner-a`.
3. Klik **Group Memberships**, **Group Permissions**, atau link seperti `0 Group Memberships` pada detail account.
4. Centang `gold-partners`.
5. Klik **Save**.

### Verifikasi

- Incognito tanpa login: page private tidak dapat dibaca.
- Developer yang bukan member: tetap ditolak.
- Developer member group: page dapat dibuka.
- Setelah membership dihapus: akses hilang.

## 3.5 Configure usage rules and information gathered from users

Bagian ini terdiri dari **Usage Rules** untuk workflow signup/account dan **Field Definitions** untuk data yang dikumpulkan.

### Mengatur Usage Rules

1. Pilih **Audience** dari menu atas.
2. Pada sidebar kiri buka **Accounts → Usage Rules**. Pada sebagian build jalurnya ditampilkan sebagai **Accounts → Settings → Usage Rules**.
3. Pada bagian **User Account Management Zone**, tentukan apakah user boleh mengedit detail yang pernah dikirim, mengganti password, dan mengelola profilnya sendiri.
4. Pada bagian **Signup**, tentukan:
   - **Developers are allowed to sign up themselves**: aktif untuk self-service signup dari Developer Portal; nonaktif bila semua account harus dibuat/diundang admin.
   - **Account approval required**: aktif bila admin harus memeriksa developer sebelum account diaktifkan.
5. Pada bagian **Users**, aktifkan **Strong passwords** jika password lokal harus memenuhi kebijakan kompleksitas yang ditampilkan UI.
6. Jika Account Plans atau Service Plans diaktifkan, tentukan apakah developer boleh **Change plan directly** atau perubahan memerlukan request/approval.
7. Klik **Save/Update** di bagian bawah halaman.
8. Buka Developer Portal pada private/incognito window dan jalankan signup end-to-end untuk membuktikan aturan bekerja.

Pilih model dengan sengaja:

| Model | Self signup | Account approval | Default plan |
|---|---|---|---|
| Self-service penuh | Aktif | Tidak | Diatur |
| Self-service terkontrol | Aktif | Ya | Diatur atau dipilih user |
| Invite-only/B2B | Tidak | Sesuai workflow internal | Diatur admin |

### Menambah informasi yang dikumpulkan saat signup

1. Pilih **Audience**.
2. Buka **Accounts → Field Definitions**. Pada sebagian tampilan labelnya **Account → Field Definitions**.
3. Tentukan object target: Account, User, atau Application.
4. Klik **Create** atau **New Field Definition**.
5. Isi:
   - **Label**: nama yang dibaca user, misalnya `Company Registration Number`.
   - **System name**: stabil dan tanpa spasi, misalnya `company_registration_number`.
   - **Required**: aktif bila signup tidak boleh lanjut tanpa nilai.
   - **Read only**: aktif bila hanya provider yang boleh mengubah.
   - **Hidden**: gunakan untuk data internal yang tidak boleh tampil kepada developer.
6. Simpan.
7. Buka signup Developer Portal dan pastikan field tampil pada lokasi yang benar.

Contoh desain:

| Object | Field | Aturan |
|---|---|---|
| Account | Legal company name | Required |
| Account | Tax/VAT ID | Required hanya untuk billing tertentu |
| User | Phone number | Optional |
| Application | Production callback URL | Required |
| Account | Partner tier | Hidden/read-only, diisi admin |

Jangan mengumpulkan data pribadi yang tidak dibutuhkan. Tetapkan retensi, akses, dan tujuan pemrosesan sesuai kebijakan perusahaan.

### Terms yang melengkapi usage rules

1. **Audience → Developer Portal → Content** atau bagian signup/legal terms.
2. Edit partial `signup_licence` untuk syarat saat pendaftaran.
3. Edit `new_application_licence` untuk syarat saat membuat application.
4. Jika service plans aktif, edit `service_subscription_licence` untuk syarat subscription.
5. Publish partial/page.
6. Uji alur signup baru dan pastikan checkbox/persetujuan tampil sebelum submit.

Aturan operasional lain dibuat melalui application plan: application approval, trial, usage limit, pricing, dan published/hidden state. Bedakan **account approval** pada Usage Rules dari **application approval** pada application plan.

## 3.6 Maintain application states and plans

### Mengubah plan application

1. **Audience → Applications → Listing** atau **Products → Product → Applications → Listing**.
2. Cari application berdasarkan nama, account, application ID, atau credential.
3. Klik application.
4. Pada bagian plan, klik **Change** atau **Change Plan**.
5. Pilih plan baru.
6. Tinjau dampak limit, approval, dan billing.
7. Klik **Change Plan** dan konfirmasi.
8. Uji credential setelah perubahan.

Untuk banyak account, gunakan daftar dan bulk action hanya setelah memvalidasi filter agar tidak memindahkan customer yang salah.

### Suspend dan resume

1. Buka summary application.
2. Di dekat field **State**, klik ikon/tombol **Suspend**.
3. Konfirmasi.
4. Uji request; seluruh credential application harus ditolak setelah cache gateway diperbarui.
5. Setelah masalah selesai, kembali ke lokasi yang sama dan klik **Resume/Unsuspend**.
6. Uji ulang.

### Approval

- Application pada plan approval-required masuk state menunggu persetujuan.
- Buka application dari listing, tinjau account dan data field, lalu klik approve/accept jika layak.
- Jangan mengubah state tanpa dokumentasi alasan, terutama untuk customer berbayar.

---

# 4. Analytics

## 4.1 Monitor usage, averages, and top applications

### Memastikan data dapat dikumpulkan

1. Mapping rule harus cocok dengan verb dan path request.
2. Mapping rule harus menaikkan method/metric yang benar.
3. Konfigurasi harus dipromosikan ke environment yang menerima trafik.
4. Request harus melewati APIcast, bukan langsung ke backend.

### Melihat traffic product

1. **Products → Orders Product**.
2. Menu kiri: **Analytics → Traffic**.
3. Pilih metric pada drop-down, misalnya `Hits` atau `Create Order`.
4. Pilih rentang waktu: 24 jam, 7 hari, 30 hari, atau custom range.
5. Pilih resolusi per jam/per hari bila tersedia.
6. Bandingkan angka dengan log gateway/backend.

### Daily/hourly averages dan top applications

1. Dari area **Analytics**, buka **Daily Averages** atau **Hourly Averages** jika tersedia pada menu versi Anda.
2. Pilih metric dan rentang waktu yang sama agar perbandingan valid.
3. Buka **Top Applications**.
4. Identifikasi application dengan volume tertinggi.
5. Klik application untuk melihat pemilik, plan, credential, dan traffic khusus application.

Alternatif melihat satu application:

1. **Audience → Applications → Listing**.
2. Klik application.
3. Buka tab/section usage/analytics.
4. Pastikan developer hanya melihat analytics application miliknya dari Developer Portal.

### Apa yang harus dianalisis

- Tren naik/turun, bukan hanya angka sesaat.
- Apakah satu application mendominasi trafik.
- Apakah pemakaian mendekati limit plan.
- Apakah metric endpoint penting naik sesuai bisnis.
- Perbedaan staging vs production.
- Zona waktu dan keterlambatan agregasi.

## 4.2 Analyze alerts, traffic, and integration errors

### Alerts

1. Buka product → **Analytics → Alerts** jika menu tersedia.
2. Tinjau application, metric, persentase limit, dan waktu kejadian.
3. Cocokkan alert dengan plan application.
4. Untuk notifikasi provider, buka **Account Settings → Personal → Notification Preferences**.
5. Aktifkan kategori **API usage alerts** bagi user yang bertanggung jawab.

Alert usage biasanya berkaitan dengan ambang limit. Jangan menganggap alert selalu outage; cek actual request dan application plan.

### Traffic

1. Buka **Analytics → Traffic**.
2. Pilih metric yang tepat.
3. Bandingkan interval sebelum, selama, dan setelah kejadian.
4. Drill down ke Top Applications.
5. Bandingkan response code atau metric custom bila tersedia.

### Integration Errors

1. Product → **Analytics → Integration Errors**.
2. Periksa waktu, endpoint, credential, metric/system name, dan pesan error.
3. Kesalahan umum:
   - authentication parameters missing;
   - application/key tidak ditemukan;
   - no mapping rule matched;
   - metric/method system name salah;
   - backend timeout/connection refused;
   - TLS/DNS upstream gagal.
4. Cocokkan dengan mapping rules di **Integration → Methods & Metrics**.
5. Cocokkan dengan gateway route dan private base URL.
6. Kirim satu request terkontrol dengan `curl -v`.
7. Pastikan request valid menghasilkan status yang diharapkan dan data analytics mulai muncul.

Jika integrasi memakai authorize/report API langsung, authorize valid diharapkan sukses dan report diterima secara asynchronous. Untuk APIcast, fokus pada mapping rule, credential, gateway log, dan backend connectivity.

---

# 5. API Authentication

## 5.1 Understand API credentials

| Pola | Credential | Kelebihan | Risiko/penggunaan |
|---|---|---|---|
| API Key | `user_key` | Paling sederhana | ID dan secret menjadi satu; cocok untuk risiko rendah/testing |
| App_ID/App_Key | `app_id` + `app_key` | Key dapat dirotasi tanpa mengganti application ID | Client harus mengelola dua nilai |
| OpenID Connect | Access token JWT | Identity/authorization terpusat, token berumur terbatas | Memerlukan IdP dan sinkronisasi client yang benar |

Credential mengidentifikasi **application**, sehingga analytics, limit, dan billing dapat dihitung per client. Product menentukan pola authentication untuk semua application pada product tersebut.

Negative test wajib:

- tanpa credential;
- credential salah;
- application suspended;
- credential pola lama setelah authentication mode berubah;
- credential valid tetapi limit terlampaui.

## 5.2 Configure access tokens

Access token di sini adalah token untuk **3scale management APIs**, bukan credential konsumen API.

1. Login sebagai provider user.
2. **Account Settings → Personal → Tokens**.
3. Klik **Add Access Token**.
4. Isi nama yang menunjukkan tujuan, misalnya `ci-product-deployer`.
5. Pilih scope minimum:
   - Account Management API untuk product/account/application dan APIcast configuration.
   - Analytics API untuk membaca analytics.
   - Billing API untuk invoice/billing.
   - Policy Registry API bila mengelola policy registry.
6. Pilih **Read Only** atau **Read & Write**.
7. Klik **Create Access Token**.
8. Salin token saat ditampilkan; biasanya tidak ditampilkan lagi.
9. Simpan dalam OpenShift Secret, Vault, atau secret manager CI.

Contoh secret untuk automation internal:

```bash
oc -n platform-automation create secret generic threescale-ci-token \
  --from-literal=token='REDACTED_ACCESS_TOKEN' \
  --from-literal=admin-url='https://retail-admin.apps.example.com'
```

Jangan menaruh token langsung pada manifest Git. Tetapkan owner, expiry/rotation schedule, dan proses revoke.

Bedakan:

- **Access token** dimiliki provider user dan mengakses management APIs.
- **Service token** terkait product dan dipakai Service Management API.
- **Application credential** dipakai konsumen untuk memanggil API business.

## 5.3 Provide access control using API key

### Konfigurasi

1. **Products → Orders Product → Integration → Settings**.
2. Scroll ke **Authentication**.
3. Pilih **API Key (user_key)**.
4. Tentukan lokasi credential: query parameter atau header jika opsi tersedia.
5. Klik **Update Product**.
6. Buka **Integration → Configuration**.
7. Klik **Promote to Staging APIcast**.

### Buat application dan uji

1. **Applications → Listing → Create Application**.
2. Pilih developer account, product, dan application plan.
3. Isi nama application lalu **Create Application**.
4. Salin `user_key` dari API Credentials.

```bash
curl -i \
  "https://orders-product-retail-apicast-staging.apps.example.com/orders-api/orders?user_key=USER_KEY"
```

Uji tanpa key:

```bash
curl -i \
  "https://orders-product-retail-apicast-staging.apps.example.com/orders-api/orders"
```

Request kedua harus ditolak. Setelah lolos staging, promosikan versi staging yang sama ke production.

## 5.4 Provide access control using App_ID and App_Key Pair

1. Product → **Integration → Settings → Authentication**.
2. Pilih **App_ID and App_Key Pair**.
3. Atur nama parameter bila organisasi memakai standar lain.
4. Klik **Update Product**.
5. Promosikan ke staging.
6. Buka application → **API Credentials**.
7. Catat Application ID dan application key.

```bash
curl -i \
  "https://orders-product-retail-apicast-staging.apps.example.com/orders-api/orders?app_id=APP_ID&app_key=APP_KEY"
```

Jika credential dikirim melalui header sesuai setting:

```bash
curl -i \
  -H 'app_id: APP_ID' \
  -H 'app_key: APP_KEY' \
  "https://orders-product-retail-apicast-staging.apps.example.com/orders-api/orders"
```

Uji `user_key` lama; setelah mode berubah, request itu harus gagal. Untuk rotasi, tambahkan key kedua, migrasikan client, lalu hapus key pertama.

## 5.5 Provide access control using OpenID Connect

Contoh menggunakan Red Hat SSO/Keycloak. Dalam real use case, DNS dan sertifikat IdP harus dipercaya oleh APIcast dan Zync.

### A. Siapkan client integrasi Zync di IdP

1. Login ke Keycloak Admin Console.
2. Pilih realm produksi, misalnya `retail`.
3. Klik **Clients → Create client**.
4. Buat confidential client `zync-client` dengan service account aktif.
5. Buka **Service account roles**.
6. Pilih client roles **realm-management**.
7. Tambahkan role **manage-clients**.
8. Buka **Credentials** dan salin client secret ke secret manager.

Zync memerlukan hak membuat/memperbarui client IdP yang mewakili application 3scale.

### B. Pastikan trust CA

Untuk CA publik, image biasanya sudah percaya. Untuk private CA:

1. Simpan CA chain dalam ConfigMap atau Secret yang dikelola platform.
2. Mount ke trust bundle deployment Zync sesuai prosedur operator/version Anda.
3. Restart/rollout Zync secara terkendali.
4. Pastikan pod Zync kembali `Running` dan log tidak menunjukkan TLS error.

Jangan menyalin patch lab secara buta; gunakan CR/operator configuration yang didukung versi produksi dan dokumentasikan perubahan.

### C. Konfigurasikan product

1. **Products → Orders Product → Integration → Settings**.
2. Pada **Authentication**, pilih **OpenID Connect**.
3. Isi issuer URL:

```text
https://zync-client:CLIENT_SECRET@sso.example.com/auth/realms/retail
```

4. Simpan dengan **Update Product**.
5. Promosikan ke staging.
6. Buat application baru pada product OIDC.
7. Tunggu Zync membuat client di Keycloak.
8. Di Keycloak, buka client yang Client ID-nya sama dengan credential application.
9. Sesuaikan access type, redirect URI, web origins, scope, dan flow untuk jenis client sebenarnya. Jangan memakai wildcard `*` di production kecuali benar-benar dibutuhkan dan disetujui.

### D. Ambil token dan uji

Gunakan flow yang sesuai. Contoh client credentials untuk machine-to-machine bila client confidential dan service account diaktifkan:

```bash
TOKEN=$(curl -sS -X POST \
  https://sso.example.com/auth/realms/retail/protocol/openid-connect/token \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d 'grant_type=client_credentials' \
  -d 'client_id=APPLICATION_CLIENT_ID' \
  -d 'client_secret=APPLICATION_CLIENT_SECRET' \
  | jq -r '.access_token')

curl -i \
  -H "Authorization: Bearer $TOKEN" \
  https://orders-product-retail-apicast-staging.apps.example.com/orders-api/orders
```

### E. Troubleshooting OIDC

- Client tidak dibuat: cek Zync, CA trust, issuer credential, dan `manage-clients`.
- `401`: cek expiry, issuer, audience/client ID, signature, dan clock skew.
- Browser gagal tetapi curl sukses: cek CORS, redirect URI, dan web origins.
- Jangan log token JWT lengkap; decode hanya di workstation aman untuk diagnosis.

---

# 6. API Documentation

## 6.1 Configure ActiveDocs using OpenAPI standards

### A. Siapkan dokumen OpenAPI yang valid

Minimal pastikan mempunyai:

```yaml
openapi: 3.0.3
info:
  title: Orders API
  version: 1.0.0
servers:
  - url: http://orders-api.orders.svc.cluster.local:8080
paths:
  /orders:
    get:
      operationId: listOrders
      responses:
        "200":
          description: OK
```

Validasi dokumen dengan tool OpenAPI organisasi. `servers.url` untuk import provisioning harus menunjuk upstream yang dapat dijangkau APIcast. Jangan memasukkan secret atau contoh data sensitif.

### B. Membuat ActiveDocs melalui GUI

1. Untuk spec tenant-wide: **Audience → Developer Portal → ActiveDocs**.
2. Untuk spec milik product: **Products → Orders Product → ActiveDocs**.
3. Klik **Create a new spec**.
4. Isi Name, System name, Description, dan pilih product bila form meminta.
5. Paste isi OpenAPI pada editor atau upload sesuai kemampuan UI.
6. Aktifkan **Published** bila siap ditampilkan.
7. Klik **Create Spec** atau **Save**.

### C. Import OpenAPI melalui Toolbox tanpa wrapper DO240

Jika file berada di host, jalankan Toolbox binary langsung atau mount file ke container Anda:

```bash
3scale import openapi \
  --target_system_name=orders_product \
  --destination=retail-tenant \
  ./orders-openapi.yaml
```

Import dapat membuat/memperbarui product, backend, methods, mapping rules, dan ActiveDocs dari dokumen. Karena operasinya luas, export/backup konfigurasi lebih dulu dan uji pada tenant nonproduction.

### D. Aktifkan CORS untuk interactive console

1. Product → **Integration → Policies**.
2. Klik **Add policy**.
3. Pilih **CORS Request Handling**.
4. Pindahkan CORS ke atas/sebelum **3scale APIcast** dengan tombol panah/drag control.
5. Buka policy dan isi `allow_origin` dengan domain Admin Portal serta Developer Portal yang benar.
6. Batasi method/header sesuai kebutuhan.
7. Klik **Update Policy**.
8. Klik **Update Policy Chain**.
9. Promosikan ke staging.

### E. Verifikasi ActiveDocs

1. Product → **ActiveDocs**.
2. Klik spec.
3. Buka operasi `GET /orders`.
4. Klik **Try it out**.
5. Masukkan credential application uji.
6. Klik **Execute**.
7. Pastikan request URL menuju APIcast, response berhasil, dan browser tidak menampilkan CORS error.
8. Setelah valid, publish spec dan tampilkan pada page Documentation Developer Portal.

---

# 7. Billing

> Uji billing pertama kali dengan charging nonaktif atau payment gateway sandbox. Jangan menguji kartu nyata pada tenant latihan.

## 7.1 Configure credit card policies

Credit Card Policies pada 3scale adalah URL halaman kebijakan Legal Terms, Privacy, dan Refunds; ini berbeda dari konfigurasi payment gateway.

1. Buat tiga page di **Audience → Developer Portal → Content**:
   - `/legal/credit-card-terms`;
   - `/legal/privacy`;
   - `/legal/refunds`.
2. Isi konten yang telah disetujui Legal/Compliance.
3. Publish ketiga page.
4. Buka **Audience → Billing → Credit Card Policies**.
5. Isi path:
   - **Path to Legal Terms page**;
   - **Path to Privacy page**;
   - **Path to Refund page**.
6. Simpan.
7. Tambahkan link pada portal bila template tidak menampilkannya otomatis:

```liquid
<a href="{{ urls.credit_card_terms }}">Legal Terms</a>
<a href="{{ urls.credit_card_privacy }}">Privacy</a>
<a href="{{ urls.credit_card_refunds }}">Refund Policy</a>
```

8. Uji setiap link dari flow pembayaran Developer Portal.

Untuk payment gateway, buka **Audience → Billing → Settings → Charging & Gateway**, pilih integrasi yang didukung versi Anda, masukkan credential sandbox, pilih currency, lalu baru aktifkan charging setelah end-to-end test berhasil.

## 7.2 Configure billing periods

1. **Audience → Billing → Settings → Charging & Gateway**.
2. Pilih mode:
   - **Prepaid**: fixed/setup fee periode berjalan ditagih di awal; variable usage ditagih bulan berikutnya.
   - **Postpaid**: fixed dan variable fee ditagih bulan berikutnya.
3. Pilih **Currency** yang didukung payment gateway.
4. Pilih format **Billing periods for invoice IDs**:
   - monthly: `YYYY-MM-XXXXXXXX`;
   - yearly: `YYYY-XXXXXXXX`.
5. Isi invoice footnote dan text VAT 0% bila diperlukan.
6. Klik **Save/Update**.

Penting: pilihan monthly/yearly di invoice ID hanya mengubah format identifier, bukan siklus billing. Automated billing 3scale tetap bulanan.

## 7.3 Update users billing state

Billing/charging dapat diatur per developer account.

1. **Audience → Accounts → Listing**.
2. Klik account customer.
3. Cari bagian **Billing** pada account details.
4. Gunakan **Enable/Disable billing** untuk menentukan apakah invoice otomatis dibuat sesuai konfigurasi.
5. Gunakan **Enable/Disable charging** untuk menentukan apakah payment gateway otomatis menagih account tersebut.
6. Konfirmasi perubahan.
7. Catat alasan dan tiket perubahan.

Gunakan kasus:

- Customer sedang dispute: charging dinonaktifkan sementara, billing tetap aktif agar usage tercatat.
- Customer internal: billing/charging dapat dinonaktifkan sesuai kebijakan.
- Customer kembali aktif: aktifkan kembali dan periksa invoice tertunda sebelum charge.

## 7.4 Configure billing details and invoice items

### Billing details account

1. **Audience → Accounts → Listing → pilih account**.
2. Klik **Edit** pada Account/Billing Details.
3. Isi legal name, billing address, billing email, tax/VAT ID, VAT rate, dan VAT code sesuai kebutuhan.
4. Simpan.
5. Pastikan field wajib juga didefinisikan di **Accounts → Field Definitions** jika developer harus mengisinya.

### Invoice items

1. **Audience → Billing → Invoices**.
2. Klik identifier invoice.
3. Bila invoice masih **Open**, cari area **Line Items**.
4. Klik **Add line item**.
5. Isi description, quantity/cost/value sesuai form.
6. Simpan dan tinjau subtotal, tax, dan total.

Hanya invoice yang masih dapat diedit yang seharusnya menerima perubahan. Jangan mengubah invoice finalized/paid tanpa mengikuti proses koreksi keuangan perusahaan.

## 7.5 Issue invoices

1. **Audience → Billing → Invoices**.
2. Filter state `Open`.
3. Klik invoice.
4. Periksa account, periode, currency, line items, tax, due date, dan total.
5. Klik **Issue invoice**.
6. Konfirmasi.
7. State berubah menuju `Pending`/menunggu pembayaran sesuai workflow.
8. Pastikan customer menerima email dan dapat melihat invoice pada Developer Portal.

## 7.6 Charge invoices

Prasyarat: charging global aktif, payment gateway dikonfigurasi, account charging aktif, dan metode pembayaran customer valid.

1. Buka invoice `Pending` atau `Unpaid` yang due date-nya telah tiba.
2. Klik **Charge** atau tindakan pembayaran yang tersedia.
3. Konfirmasi.
4. Periksa hasil:
   - berhasil → `Paid`;
   - gagal → `Unpaid`, lalu sistem dapat retry;
   - tiga kegagalan → `Failed` menurut alur materi.
5. Cocokkan transaction ID dengan dashboard Stripe/Braintree.
6. Jangan memasukkan data kartu mentah ke log/tiket.

## 7.7 Cancel invoices

1. **Audience → Billing → Invoices**.
2. Buka invoice yang belum paid dan memang harus dibatalkan.
3. Klik **Cancel invoice**.
4. Baca dialog konfirmasi; pastikan identifier dan account benar.
5. Klik konfirmasi.
6. Verifikasi state `Canceled`.
7. Catat alasan, approver, dan tindakan lanjutan pada sistem keuangan.

Cancellation bukan refund. Bila transaksi sudah paid, ikuti proses refund payment gateway dan accounting; jangan hanya mengubah tampilan invoice.

## 7.8 Ringkasan state invoice

| State | Arti |
|---|---|
| Open | Masih dapat diperbarui |
| Finalized | Semua charge periode sudah dimasukkan |
| Pending | Sudah diterbitkan, menunggu pembayaran |
| Unpaid | Percobaan pembayaran gagal dan dapat dicoba ulang |
| Paid | Pembayaran berhasil |
| Failed | Retry berhenti setelah kegagalan berulang |
| Canceled | Dibatalkan administrator |

---

# 8. Developer Portal

## 8.1 Customize developer portal

### Ubah nama dan logo

1. **Account Settings → Account Details → Edit**.
2. Ubah organization name lalu **Update Account**.
3. Pilih **Audience → Developer Portal → Logo**.
4. Upload logo yang ukurannya sudah dioptimalkan.
5. Buka **Developer Portal → Content**.
6. Di tree sebelah kanan/kiri editor, buka **Layouts → Main layout**.
7. Tambahkan:

```liquid
<a class="navbar-brand" href="/">
  <img src="{{ provider.logo_url }}"
       alt="{{ provider.name }}"
       height="32">
  {{ provider.name }}
</a>
```

8. Klik **Publish**, bukan hanya Save Draft.

### Ubah CSS

1. Dari tree Content, klik `default.css` atau stylesheet theme.
2. Ubah warna/font dengan tetap menjaga contrast dan responsive layout.
3. Klik **Publish**.
4. Refresh portal dengan cache browser dinonaktifkan saat testing.

### Buat page dan menu

1. **Developer Portal → Content**.
2. Pilih section induk.
3. Klik **New Page** di kanan atas.
4. Isi Title dan Path, misalnya `Status` dan `/status`.
5. Buka **Advanced Options**, aktifkan **Liquid enabled** bila memakai variable/tag Liquid.
6. Klik **Create Page**.
7. Isi HTML/Markdown lalu **Publish**.
8. Buka **Partials → submenu**.
9. Tambahkan:

```liquid
<li class="{% if request.path == '/status' %}active{% endif %}">
  <a href="/status">Status</a>
</li>
```

10. Publish partial dan uji logged-in serta anonymous view.

### Prinsip Liquid

- `{{ variable }}` mencetak nilai.
- `{% instruction %}` menjalankan logika seperti `if`, `for`, atau `include`.
- Lihat **Developer Portal → Docs → Liquid Reference** untuk object yang tersedia.
- Jangan menaruh secret dalam template; semua output portal dianggap dapat dilihat client.

## 8.2 Configure portal settings

### Domain dan access

1. **Audience → Developer Portal → Domains & Access**.
2. Tinjau Developer Portal domain.
3. Untuk go-live, klik **Open your Portal to the world** atau ubah access code sesuai proses UI.
4. Konfirmasi hanya setelah page legal, signup, email, dan plan siap.

### Signup dan activation

1. Buka settings/feature visibility Developer Portal.
2. Tentukan apakah signup tersedia.
3. Tentukan apakah account/application membutuhkan approval.
4. Konfigurasikan field definitions dan legal terms.
5. Uji signup menggunakan alamat email nyata yang dapat menerima pesan.
6. Dari **Audience → Accounts → Listing**, approve/activate account bila dibutuhkan.

### Redirects, visibility, dan restricted content

1. **Developer Portal → Redirects** untuk memetakan URL lama ke URL baru.
2. **Developer Portal → Feature Visibility** untuk mengaktifkan/menonaktifkan fitur portal.
3. **Developer Portal → Groups** untuk group konten privat.
4. Pastikan page/section/layout/partial yang berubah sudah published.

### Checklist go-live portal

- HTTPS valid dan domain resolvable.
- Logo, navigation, mobile layout, dan accessibility diuji.
- Signup/login/reset password dan email bekerja.
- Plan yang benar terlihat.
- ActiveDocs dapat mengeksekusi request melalui CORS.
- Page legal/privacy/refund tersedia.
- Konten private tidak bocor kepada anonymous user.

---

# 9. Service Discovery

## 9.1 Add services from an underlying OpenShift cluster

Contoh service sudah berjalan di namespace `orders`:

```bash
oc -n orders get svc orders-api
oc -n orders get endpoints orders-api
```

### A. Tambahkan metadata discovery

```bash
oc -n orders label service orders-api \
  discovery.3scale.net=true

oc -n orders annotate service orders-api \
  discovery.3scale.net/scheme=http

oc -n orders annotate service orders-api \
  discovery.3scale.net/port=8080
```

Tiga metadata wajib:

| Jenis | Key | Contoh |
|---|---|---|
| Label | `discovery.3scale.net` | `true` |
| Annotation | `discovery.3scale.net/scheme` | `http` |
| Annotation | `discovery.3scale.net/port` | `8080` |

Optional annotation dapat menunjukkan path atau lokasi OpenAPI description bila didukung.

### B. Berikan RBAC minimum

Service account System yang melakukan discovery adalah `amp` di namespace 3scale. Beri `view` hanya pada namespace aplikasi:

```bash
oc -n orders policy add-role-to-user view \
  system:serviceaccount:3scale:amp
```

Verifikasi:

```bash
oc auth can-i get services -n orders \
  --as=system:serviceaccount:3scale:amp
```

### C. Import dari GUI

1. Login Admin Portal.
2. Pilih **Products**.
3. Klik **New Product** atau **Create Product**.
4. Pilih tab/opsi **Import from OpenShift**.
5. Pilih namespace `orders`.
6. Pilih service `orders-api`.
7. Tinjau scheme, port, product name, backend name, dan private URL.
8. Klik **Create Product**.
9. Buka product hasil import dan periksa backend association.
10. Buat plan dan application; import service tidak otomatis memberi credential client.
11. Pastikan mapping rules tersedia, promosikan ke staging, lalu uji.

### Troubleshooting

- Namespace/service tidak muncul: cek label, annotation, RBAC, dan service discovery enabled.
- `502 Bad Gateway`: cek endpoint, targetPort/port, scheme, DNS, dan NetworkPolicy.
- `404/No Mapping Rule`: buat mapping rule lalu promote.
- ActiveDocs tidak update: cek description path dan OpenAPI endpoint service.

---

# 10. Product Definition

## 10.1 Configure methods and metrics

### Memahami perbedaan

- **Metric** adalah counter umum, misalnya hits, bytes, atau response-201.
- **Method** biasanya child dari metric dan merepresentasikan operasi bisnis, misalnya `List Orders` atau `Create Order`.
- **Mapping rule** menghubungkan HTTP verb/path ke method atau metric.

### Membuat metric

1. **Products → Orders Product → Integration → Methods & Metrics**.
2. Klik **New metric**.
3. Isi Friendly name, System name, Unit, Description.
4. Klik **Create Metric**.

CLI:

```bash
3scale metric create retail-tenant orders_product successful_orders \
  --unit=1
```

### Membuat method

1. Pada halaman Methods & Metrics, cari metric induk, biasanya `Hits`.
2. Klik **New method** atau ikon tambah di bawah metric.
3. Isi:
   - Friendly name: `Create Order`;
   - System name: `create_order`;
   - Description.
4. Simpan.

### Membuat mapping rules

1. Pada method/metric, klik **Add a mapping rule**.
2. Isi:

| Verb | Pattern | Increment |
|---|---|---|
| GET | `/orders-api/orders` | `list_orders` +1 |
| POST | `/orders-api/orders` | `create_order` +1 |
| GET | `/orders-api/orders/{id}` atau pola versi UI | `get_order` +1 |

3. Pastikan rule lebih spesifik tidak tertutup rule umum.
4. Perhatikan default mapping `/` ke Hits dapat menghitung seluruh request.
5. Simpan.
6. Promosikan ke staging.
7. Kirim satu request per endpoint.
8. Periksa analytics setiap method.

### Desain yang baik

- System name stabil karena dipakai analytics/API.
- Jangan mengganti system name sembarangan setelah integrasi digunakan.
- Bedakan counter path/request dengan counter response; response-specific metric biasanya membutuhkan Custom Metrics policy.
- Hindari double counting yang tidak disengaja dari mapping rules tumpang tindih.

---

# 11. Service Integration

## 11.1 Configure API gateway to point to private API

### A. Buat backend

1. Pilih **Products**, lalu buka tab/list **Backends**.
2. Klik **Create Backend**.
3. Isi:
   - Name: `Orders Backend`;
   - System name: `orders_backend`;
   - Private Base URL: `http://orders-api.orders.svc.cluster.local:8080`.
4. Klik **Create Backend**.

Private Base URL adalah base/root upstream. Jangan menambahkan endpoint `/orders` bila path itu seharusnya tetap berasal dari request.

### B. Buat product

1. **Products → New Product**.
2. Isi Name `Orders Product`, System name `orders_product`.
3. Klik **Create Product**.

CLI alternatif:

```bash
3scale service create retail-tenant orders_product
```

### C. Kaitkan backend

1. Product → **Integration → Backends**.
2. Klik **Add Backend**.
3. Pilih `orders_backend`.
4. Isi public path `/orders-api`.
5. Klik **Add to Product**.

Translasi yang diharapkan:

```text
Public  https://gateway.example.com/orders-api/orders/123
Private http://orders-api.orders.svc.cluster.local:8080/orders/123
```

APIcast melepas mounting path `/orders-api`, lalu menggabungkan sisa path dengan Private Base URL.

### D. Connectivity test dari cluster

Sebelum menyalahkan APIcast, uji service dan endpoint:

```bash
oc -n orders get svc,endpoints orders-api
oc -n 3scale run connectivity-test \
  --image=registry.access.redhat.com/ubi9/ubi-minimal \
  --restart=Never --command -- \
  curl -sS http://orders-api.orders.svc.cluster.local:8080/health
oc -n 3scale logs connectivity-test
oc -n 3scale delete pod connectivity-test
```

Gunakan image yang diizinkan organisasi dan hapus pod uji setelah selesai.

## 11.2 Promote API gateway to staging and production

### Melalui GUI

1. Product → **Integration → Configuration**.
2. Tinjau sandbox/latest configuration: backend URL, mapping rules, authentication, policy.
3. Klik **Promote v.N to Staging APIcast**.
4. Salin staging public base URL dan contoh curl.
5. Uji positive dan negative case.
6. Periksa integration errors dan analytics.
7. Setelah staging lulus, kembali ke Configuration.
8. Klik **Promote v.N to Production APIcast** atau promote staging version ke production.
9. Konfirmasi nomor versi yang dipromosikan sama dengan versi yang diuji.
10. Uji production menggunakan application khusus smoke test.

### Melalui Toolbox

Deploy latest/sandbox ke staging:

```bash
3scale proxy-config deploy retail-tenant orders_product
```

Lihat staging configuration:

```bash
3scale proxy-config show retail-tenant orders_product sandbox -o json | jq
```

Promosikan latest staging configuration ke production:

```bash
3scale proxy-config promote retail-tenant orders_product
```

Sebelum menjalankannya, gunakan `proxy-config show`/help untuk memastikan staging memuat versi yang baru diuji. Jangan mempromosikan ulang hanya karena nomor versi terlihat paling besar.

### Checklist sebelum production

- DNS/TLS production valid.
- Backend dapat dijangkau dari APIcast.
- Mapping rules tidak terlalu luas.
- Authentication dan credential negative test berhasil.
- Rate limit sesuai plan.
- CORS hanya mengizinkan origin yang diperlukan.
- Policy chain benar.
- Monitoring dan rollback version diketahui.

## 11.3 Create and maintain custom service policies

### A. Standard policy dari GUI

1. Product → **Integration → Policies**.
2. Klik **Add policy**.
3. Pilih policy, misalnya:
   - Routing;
   - Header Modification;
   - IP Check;
   - Anonymous Access;
   - JWT Claim Check;
   - Custom Metrics;
   - Retry;
   - Maintenance Mode;
   - CORS Request Handling.
4. Klik policy pada chain untuk mengisi configuration.
5. Klik **Update Policy**.
6. Gunakan tombol panah/drag untuk mengatur urutan.
7. Pastikan **3scale APIcast** tetap ada.
8. Klik **Update Policy Chain**.
9. Promote ke staging dan uji.

Urutan penting karena request melewati policy dari atas ke bawah. Contoh:

```text
CORS Request Handling
Routing
Custom Metrics
3scale APIcast
```

### B. Export, edit, dan import policy chain

```bash
3scale policies export retail-tenant orders_product \
  > orders-product-policy-backup.yml
```

Edit copy, bukan backup. Contoh Maintenance Mode:

```yaml
---
- name: maintenance_mode
  version: builtin
  configuration: {}
  enabled: true
- name: apicast
  version: builtin
  configuration: {}
  enabled: true
```

```bash
3scale policies import retail-tenant orders_product \
  -f orders-product-policy-new.yml
```

Import mengganti seluruh chain. Jika policy yang ada tidak disalin, policy itu hilang. Setelah import, verifikasi chain, promote staging, lalu uji.

### C. Custom policy buatan organisasi

Untuk benar-benar membuat policy code sendiri:

1. Tentukan phase NGINX/APIcast yang digunakan dan kontrak configuration schema.
2. Implementasikan policy Lua mengikuti APIcast policy framework versi yang digunakan.
3. Tambahkan unit/integration tests.
4. Build image APIcast kustom secara reproducible dan scan vulnerability.
5. Push ke registry internal.
6. Deploy pada environment nonproduction.
7. Registrasikan `CustomPolicyDefinition`/mekanisme policy registry yang didukung versi platform agar policy muncul di Admin Portal.
8. Tambahkan policy ke chain product.
9. Uji urutan, failure mode, latency, memory, dan rollback.
10. Promote hanya setelah gateway image dan product configuration sama-sama tervalidasi.

Jangan memasang custom code langsung ke pod berjalan. Perubahan harus berasal dari image/CR yang dapat direproduksi dan bertahan setelah restart.

### D. Maintenance policy lifecycle

1. Export chain sebagai backup.
2. Tambahkan Maintenance Mode.
3. Update chain dan promote staging.
4. Verifikasi status `503` dan body yang disepakati.
5. Promote production pada change window.
6. Setelah maintenance, hapus policy.
7. Update dan promote lagi.
8. Verifikasi service pulih.

---

# 12. Latihan End-to-End Tanpa RHLS

Gunakan checklist berikut pada cluster milik sendiri:

1. Deploy API `orders-api` dan Service pada namespace `orders`.
2. Uji `/health` langsung dari pod di namespace 3scale.
3. Buat backend dan product.
4. Kaitkan backend pada `/orders-api`.
5. Buat methods `list_orders`, `get_order`, `create_order` dan mapping rules.
6. Buat plan Free dan Pro.
7. Tambahkan limit dan pricing rule.
8. Buat developer account dan application.
9. Uji API key, lalu migrasikan lab pribadi ke App_ID/App_Key.
10. Promote ke staging dan lakukan positive/negative tests.
11. Periksa Traffic, Top Applications, Alerts, dan Integration Errors.
12. Import OpenAPI dan uji ActiveDocs.
13. Kustomisasi Developer Portal dan buat restricted section berbasis group.
14. Konfigurasikan billing menggunakan payment gateway sandbox.
15. Issue dan charge invoice uji; jangan gunakan kartu nyata.
16. Export product/policy configuration sebagai backup.
17. Promote versi teruji ke production.

## 12.1 Bukti yang sebaiknya disimpan

- Output object/list dari Toolbox tanpa secret.
- Screenshot plan, limit, pricing, account state, dan invoice state.
- `curl -i` positive dan negative test.
- Nomor proxy configuration staging/production.
- Export policy chain dan product configuration.
- Screenshot analytics setelah traffic uji.
- Catatan rollback dan perubahan.

## 12.2 Kesalahan yang paling sering terjadi

- Product dianggap sama dengan backend.
- Private Base URL memasukkan endpoint terlalu panjang.
- Backend sudah dikaitkan tetapi mapping rule belum dibuat.
- Perubahan disimpan tetapi belum dipromosikan.
- Versi yang belum diuji dipromosikan ke production.
- Usage limit dibuat pada metric atau level yang salah.
- Pricing rules overlap/gap.
- Policy `apicast` hilang saat import.
- CORS berada setelah APIcast.
- Access token diberi semua scope padahal hanya perlu read-only analytics.
- Credential ditulis ke Git atau screenshot.
- Service discovery gagal karena RBAC `3scale:amp` belum diberikan.
- Page/layout portal masih Draft.
- Billing ID format dianggap mengubah billing cycle.
- Cancel invoice dianggap sama dengan refund.

---

# 13. Referensi

Materi utama lokal:

- `ch01*.html`: arsitektur, instalasi, Toolbox.
- `ch02*.html`: product/backend, plans, accounts, versioning, discovery, tenants.
- `ch03*.html`: APIcast dan policies.
- `ch04*.html`: provider users, credentials, API key, App_ID/App_Key, OIDC.
- `ch05*.html`: Developer Portal, Liquid, OpenAPI, ActiveDocs.
- `ch06*.html`: analytics dan monitoring.
- `ch07*.html`: billing dan pricing.

Dokumentasi resmi tambahan yang dipakai untuk objective yang tidak dijabarkan penuh di HTML kursus:

- Red Hat 3scale 2.11 Admin Portal Guide — Billing Settings:  
  `https://docs.redhat.com/en/documentation/red_hat_3scale_api_management/2.11/html/admin_portal_guide/configure-billing`
- Red Hat 3scale 2.11 Creating the Developer Portal — Credit Card Policies:  
  `https://docs.redhat.com/en/documentation/red_hat_3scale_api_management/2.11/html-single/creating_the_developer_portal/creating_the_developer_portal`
- Red Hat 3scale 2.11 Admin Portal Guide — Analytics:  
  `https://docs.redhat.com/en/documentation/red_hat_3scale_api_management/2.11/html/admin_portal_guide/analytics`
- Red Hat 3scale 2.11 Admin Portal Guide — Suspending Applications:  
  `https://docs.redhat.com/documentation/en-us/red_hat_3scale_api_management/2.11/html/admin_portal_guide/suspend-application`
- Red Hat 3scale 2.11 Creating the Developer Portal — Signup Flows:  
  `https://docs.redhat.com/en/documentation/red_hat_3scale_api_management/2.11/html/creating_the_developer_portal/signup-flows`
- 3scale Toolbox — lokasi config dan referensi command:  
  `https://github.com/3scale/3scale_toolbox`
