# NeoSIAK — Dokumentasi API Lengkap

**Universitas Pancasila — Sistem Informasi Akademik (NeoSIAK)**

> Dokumen ini adalah dokumentasi **lengkap dan konsolidasi** dari seluruh endpoint API yang **benar-benar terimplementasi** di codebase NeoSIAK (`routes/api.php`), disusun langsung dari kode sumber per 23 Juni 2026.

| | |
|---|---|
| **Tech Stack** | Laravel 10 + Laravel Passport (OAuth2) |
| **Base Path** | `/api` |
| **Base URL (production)** | `https://neosiak.univpancasila.ac.id/api` |
| **Total Endpoint** | 14 |
| **Format Data** | JSON |

---

## Daftar Isi

1. [Ringkasan Endpoint](#1-ringkasan-endpoint)
2. [Metode Autentikasi](#2-metode-autentikasi)
3. [Format Response](#3-format-response)
4. [Endpoint — Legacy / Internal (tanpa OAuth2)](#4-endpoint--legacy--internal-tanpa-oauth2)
5. [Endpoint — User (OAuth2 Bearer Token)](#5-endpoint--user-oauth2-bearer-token)
6. [Endpoint — API v1 (OAuth2 Scope-based)](#6-endpoint--api-v1-oauth2-scope-based)
7. [OAuth2 — Cara Mendapatkan Access Token](#7-oauth2--cara-mendapatkan-access-token)
8. [Referensi Kode Error & HTTP Status](#8-referensi-kode-error--http-status)
9. [Rate Limiting](#9-rate-limiting)
10. [Catatan Keamanan & Known Issues](#10-catatan-keamanan--known-issues)

---

## 1. Ringkasan Endpoint

| # | Method | Path | Controller | Auth |
|---|---|---|---|---|
| 1 | GET | `/api/master/mahasiswa` | `Api\MahasiswaController@getByNim` | ❌ Tidak ada (public) |
| 2 | GET | `/api/master/mahasiswa/transkrip-nilai` | `Api\MahasiswaController@rekapNilai` | API Key (query param `apiKey`) |
| 3 | POST | `/api/uploadProfilePhoto` | `Api\MahasiswaController@uploadProfilePhoto` | ❌ Tidak ada (public) |
| 4 | POST | `/api/student/blok-status` | `Api\StudentStatusController@index` | Checksum (body param `checksum`) |
| 5 | POST | `/api/mahasiswa-baru` | `Api\MahasiswaBaruController@store` | API Key (header `X-API-KEY`) |
| 6 | GET | `/api/user` | `Api\UserController@getUserInfo` | OAuth2 Bearer (`auth:api`) |
| 7 | GET | `/api/user/fasttrack` | `Api\UserController@getUserFasttrack` | OAuth2 Bearer (`auth:api`) |
| 8 | GET | `/api/v1/master/mahasiswa/list` | `Api\V1\MahasiswaApiController@list` | OAuth2 Scope `read-mahasiswa` |
| 9 | GET | `/api/v1/master/mahasiswa/search` | `Api\V1\MahasiswaApiController@search` | OAuth2 Scope `read-mahasiswa` |
| 10 | GET | `/api/v1/stats/mahasiswa` | `Api\V1\MahasiswaApiController@stats` | OAuth2 Scope `read-mahasiswa` |
| 11 | GET | `/api/v1/master/alumni/list` | `Api\V1\AlumniApiController@list` | OAuth2 Scope `read-alumni` |
| 12 | GET | `/api/v1/master/alumni/{nim}` | `Api\V1\AlumniApiController@detail` | OAuth2 Scope `read-alumni` |
| 13 | GET | `/api/v1/master/prodi/list` | `Api\V1\ProdiApiController@list` | OAuth2 Scope `read-prodi` |
| 14 | GET | `/api/v1/master/fakultas/list` | `Api\V1\ProdiApiController@fakultasList` | OAuth2 Scope `read-prodi` |

Semua endpoint bersifat **read-only (GET)**, kecuali 3 endpoint POST: `uploadProfilePhoto`, `student/blok-status`, dan `mahasiswa-baru`.

---

## 2. Metode Autentikasi

NeoSIAK API **tidak** memakai satu metode auth yang konsisten — bergantung kapan endpoint dibuat. Ada **5 mekanisme** yang berjalan paralel:

| Mekanisme | Dipakai Pada | Cara Kerja |
|---|---|---|
| **Tidak ada (public)** | `/master/mahasiswa`, `/uploadProfilePhoto` | Tidak ada pengecekan apapun — siapa saja bisa akses. |
| **API Key via query param** | `/master/mahasiswa/transkrip-nilai` | Parameter `apiKey` dibandingkan dengan nilai *hardcoded* di `MahasiswaController.php`. |
| **Checksum via body param** | `/student/blok-status` | Parameter `checksum` dibandingkan dengan string *hardcoded* di `StudentStatusController.php`. |
| **API Key via header** | `/mahasiswa-baru` | Header `X-API-KEY` divalidasi `hash_equals()` terhadap `config('pmb.neosiak_api_key')` (env `NEOSIAK_PMB_API_KEY`), via middleware `pmb.apikey` (`App\Http\Middleware\PmbApiKey`). |
| **OAuth2 Bearer Token (Passport)** | `/user`, `/user/fasttrack`, semua `/v1/*` | Header `Authorization: Bearer {access_token}`. Lihat [§7](#7-oauth2--cara-mendapatkan-access-token). |

> ⚠️ Untuk mekanisme API Key/checksum *hardcoded*, nilai literalnya **tidak dicantumkan** di dokumen ini — lihat source code terkait atau hubungi tim developer NeoSIAK untuk mendapatkan kredensial yang berlaku. Lihat juga [§10](#10-catatan-keamanan--known-issues).

### 2.1 OAuth2 Scopes yang Tersedia

Didefinisikan di `app/Providers/AuthServiceProvider.php`:

| Scope | Deskripsi | Default Scope |
|---|---|---|
| `get-email` | Akses field email user | ✅ |
| `get-username` | Akses field username user | ✅ |
| `read-mahasiswa` | Baca data mahasiswa (lookup, list, search, stats) | — |
| `read-alumni` | Baca data alumni (mahasiswa status Lulus) | — |
| `read-prodi` | Baca data master prodi & fakultas | — |

- **Token expiry:** Access Token 15 hari, Refresh Token 30 hari, Personal Access Token 6 bulan.
- Endpoint `/v1/*` memakai middleware `client:{scope}` (`Laravel\Passport\Http\Middleware\CheckClientCredentials`) — middleware ini mengecek scope token **tanpa mensyaratkan grant type tertentu**, sehingga mendukung baik token dari **Client Credentials Grant** maupun **Authorization Code Grant**, selama tokennya memiliki scope yang sesuai.
- Endpoint `/user` dan `/user/fasttrack` memakai middleware `auth:api` (tanpa scope check tambahan), tapi field `email` di response disembunyikan kecuali token punya scope `get-email`.

---

## 3. Format Response

**Penting:** NeoSIAK memakai **4 konvensi response berbeda** tergantung endpoint mana yang dipanggil (akumulasi dari iterasi development yang berbeda-beda). Perhatikan baik-baik konvensi per endpoint di bagian detail.

### Konvensi A — `SuccessResponseResource` / `FailedResponseResource`
Dipakai oleh: `/master/mahasiswa`, `/master/mahasiswa/transkrip-nilai`, `/uploadProfilePhoto`, `/user`, `/user/fasttrack`.

```json
// Success
{ "status": "success", "code": "200", "message": "Successfully Retrieve Data", "data": { ... } }

// Failed
{ "status": "failed", "code": "404", "message": "Data Not Found", "data": null }
```

### Konvensi B — `ApiResponse` trait
Dipakai oleh: seluruh endpoint `/v1/*`.

```json
// Success
{ "status": "00", "message": "Successfully Retrieve Data", "data": [ ... ] }

// Success + pagination
{ "status": "00", "message": "Successfully Retrieve Data", "data": [ ... ], "meta": { "current_page": 1, "per_page": 50, "total": 245, "last_page": 5 } }

// Not Found (HTTP 404)
{ "status": "10", "message": "Data tidak ditemukan", "data": null }

// Validation Error (HTTP 422)
{ "status": "02", "message": "Parameter \"q\" wajib diisi (minimal 3 karakter)", "errors": null }

// Server Error (HTTP 500)
{ "status": "01", "message": "Pesan error" }
```

| Kode `status` | HTTP | Arti |
|---|---|---|
| `00` | 200 | Sukses |
| `01` | 500 | Server error |
| `02` | 422 | Validation error |
| `10` | 404 | Data tidak ditemukan |

### Konvensi C — Ad-hoc `{message, data}`
Dipakai oleh: `/mahasiswa-baru`.

```json
// Success (201)
{ "message": "Data mahasiswa baru berhasil disimpan", "data": { "npm": "20210001" } }

// Error (4xx/5xx)
{ "message": "Pesan error" }
```

### Konvensi D — Ad-hoc `{status, messages}`
Dipakai oleh: `/student/blok-status`.

```json
// Success (200)
{ "status": "success", "messages": "Successfully" }

// Unauthorized (401)
{ "status": "failed", "messages": "Unauthorized" }
```

---

## 4. Endpoint — Legacy / Internal (tanpa OAuth2)

### 4.1 Lookup Mahasiswa by NIM

```
GET /api/master/mahasiswa
```

Mengambil data dasar satu mahasiswa berdasarkan NIM. **Endpoint ini public, tidak ada autentikasi apapun.**

**Query Parameters:**

| Parameter | Tipe | Wajib | Keterangan |
|---|---|---|---|
| `nim` | string | Ya | NIM mahasiswa |

**Response (200):**
```json
{
  "status": "success",
  "code": "200",
  "message": "Successfully Retrieve Data",
  "data": {
    "nim": "1223210001",
    "nama_mhs": "Budi Santoso",
    "newemail": "budi@student.univpancasila.ac.id",
    "kd_prodi": "55201",
    "prodi": { "kd_prodi": "55201", "prodi": "Informatika", "fakultas": "Teknik", "fakultas_id": 5 },
    "sim": { "nim": "1223210001", "ktp": "3174012301000001" }
  }
}
```

**Response (404):**
```json
{ "status": "failed", "code": 404, "message": "Data Not Found", "data": null }
```

> Field `sim.ktp` (NIK/KTP) ikut terekspos — lihat catatan keamanan di [§10](#10-catatan-keamanan--known-issues).

---

### 4.2 Transkrip Nilai / Rekap Nilai Mahasiswa

```
GET /api/master/mahasiswa/transkrip-nilai
```

Mengambil rekap nilai (transkrip) mahasiswa: total SKS, SKS lulus, IPK, dan detail nilai per mata kuliah (wajib + peminatan). Endpoint ini menjalankan beberapa validasi bisnis sebelum mengembalikan data (status pembayaran, status mahasiswa, kewajiban pengisian EDOM).

**Query Parameters:**

| Parameter | Tipe | Wajib | Keterangan |
|---|---|---|---|
| `apiKey` | string | Ya | API key statis (lihat tim developer) |
| `nim` | string | Ya | NIM mahasiswa |

**Logika validasi (berurutan):**
1. Jika mahasiswa terdaftar program MBKM (`stsmbkm == 1`) → langsung kembalikan rekap nilai.
2. Jika status terakhir mahasiswa (`Statusmhs`, diurutkan `thakd` desc) termasuk `Non Aktif`, `Lulus`, atau `Cuti` → langsung kembalikan rekap nilai.
3. Selain itu, cek blok pembayaran tahun akademik aktif (`StsMhsBlok`) — jika tidak ada record valid (`utsblok`/`uasblok`/`khsblok` ≠ `0` dan `kd_prodi` ≠ `00`) → **404** "Maaf Anda Belum melakukan pembayaran...".
4. Jika prodi mahasiswa wajib EDOM (`btedom == 5`) dan ada form EDOM yang belum diisi (`stsisitot_t2 == 0`) pada tahun akademik EDOM aktif → **500** "Maaf Anda Belum melakukan pengisian EDOM...".
5. Jika semua lolos → kembalikan rekap nilai.

**Response (200):**
```json
{
  "status": "success",
  "code": "200",
  "message": "Successfully Retrieve Data",
  "data": {
    "total_sks": 144,
    "sks_lulus": 140,
    "ipk": "3.45",
    "detail": [
      {
        "nim": "1223210001",
        "nama_mhs": "Budi Santoso",
        "kurikulum": "2020",
        "kd_mk": "IF101",
        "nama_mk": "Algoritma dan Struktur Data",
        "sks": 3,
        "nhuruf": "A",
        "mutusks": 12
      }
    ]
  }
}
```

**Response error:**
```json
// apiKey salah/kosong — HTTP 401
{ "status": "failed", "code": "401", "message": "Unauthorized", "data": null }

// belum bayar — HTTP 404
{ "status": "failed", "code": "404", "message": "Maaf Anda Belum melakukan pembayaran. Silahkan hubungi Prodi masing-masing.", "data": null }

// belum isi EDOM — HTTP 500
{ "status": "failed", "code": "500", "message": "Maaf Anda Belum melakukan pengisian EDOM. Silahkan isi EDOM di Neosiak.", "data": null }
```

---

### 4.3 Upload Foto Profil Mahasiswa

```
POST /api/uploadProfilePhoto
```

Upload foto profil mahasiswa. File diteruskan ke storage server eksternal (`storage.univpancasila.ac.id`) lalu URL hasil upload disimpan ke record mahasiswa. **Endpoint ini public, tidak ada autentikasi apapun.**

**Request (multipart/form-data):**

| Field | Tipe | Wajib | Keterangan |
|---|---|---|---|
| `nim` | string | Ya | NIM mahasiswa target |
| `photo` | file | Ya* | Mimetype `png`/`jpg`/`jpeg`, maksimal 2048 KB |

\* Jika field `photo` tidak dikirim, request **tidak menghasilkan response berarti** (kosong/null) — bukan error, karena kode hanya memproses logic di dalam `if ($request->file('photo'))`.

**Response (200):**
```json
{
  "status": "success",
  "code": "200",
  "message": "Berhasil Unggah Foto",
  "data": { "foto": "https://storage.univpancasila.ac.id/.../thumbnail.jpg" }
}
```

**Response error:**
```json
// gagal upload ke storage eksternal — HTTP 500
{ "status": "failed", "code": "500", "message": "Gagal Unggah Foto", "data": null }

// exception lain (termasuk validasi file gagal) — HTTP 500
{ "status": "failed", "code": "500", "message": "<pesan exception>", "data": null }
```

> Field `nim` tanpa validasi keberadaan — jika NIM tidak ditemukan, `$data` akan `null` dan request bisa menghasilkan error saat `$data->save()`.

---

### 4.4 Update Status Blok Mahasiswa

```
POST /api/student/blok-status
```

Endpoint internal untuk membuka/menutup blok akademik mahasiswa (KRS/UTS/UAS/KHS) berdasarkan event dari sistem lain (misalnya pembayaran). Auth memakai **checksum statis**.

**Request Body:**

| Parameter | Tipe | Wajib | Keterangan |
|---|---|---|---|
| `checksum` | string | Ya | Checksum statis (lihat tim developer) |
| `npm` | string | Ya | NIM mahasiswa (nama param tetap `npm`) |
| `tipe` | string | Ya | `KRS`, `UTS`, `UAS`, `ATR`, atau `ALL` |
| `periode` | string | Ya | Tahun akademik (`th_akademik`), contoh `20251` |
| `cicilan_ke` | string | Tidak | Diterima tapi **belum dipakai** di logic apapun saat ini |
| `virtual_account` | string | Hanya untuk `tipe=ATR` | Nomor VA (dicocokkan ke kolom `nova` tabel `bayar_antara`) |
| `datetime_payment` | string | Hanya untuk `tipe=ATR` | Tanggal/jam pembayaran semester antara |

**Logika per `tipe`:**

| `tipe` | Efek |
|---|---|
| `KRS` | Jika belum ada record `StsMhsBlok` untuk NIM+periode, buat baru (blok default 0). |
| `UTS` | Set `utsblok = 1`. |
| `UAS` | Set `uasblok = 1` dan `khsblok = 1`. |
| `ATR` | Update `BayarAntara` (cari by `nova = virtual_account`) → `stsbayar = 'lunas'`, `tglbayar = datetime_payment`. |
| `ALL` | Kombinasi `KRS` + `UTS` + `UAS` (buka semua blok). |

**Response (200):**
```json
{ "status": "success", "messages": "Successfully" }
```

**Response (401) — checksum salah/kosong:**
```json
{ "status": "failed", "messages": "Unauthorized" }
```

---

### 4.5 Sinkronisasi Mahasiswa Baru (PMB → NeoSIAK)

```
POST /api/mahasiswa-baru
```

Endpoint untuk sinkronisasi data mahasiswa baru dari sistem **PMB Online** ke NeoSIAK. Dilindungi middleware `pmb.apikey`.

**Headers:**

| Header | Wajib | Keterangan |
|---|---|---|
| `X-API-KEY` | Ya | Harus cocok dengan `config('pmb.neosiak_api_key')` (env `NEOSIAK_PMB_API_KEY`) |
| `Content-Type` | Ya | `application/json` |
| `Accept` | Disarankan | `application/json` |

**Request Body:**

```json
{
  "npm": "20210001",
  "profile": {
    "nama": "Budi Santoso",
    "nik": "3174012301000001",
    "nisn": "0012345678",
    "jenis_kelamin": "L",
    "tempat_lahir": "Jakarta",
    "tanggal_lahir": "2000-01-23",
    "alamat": "Jl. Srengseng Sawah No. 1, Jakarta Selatan",
    "kode_pos": "12640",
    "rt": "001",
    "rw": "002",
    "no_hp": "08123456789",
    "no_telp_rumah": "02178901234",
    "email": "budi.santoso@email.com"
  },
  "akademik": {
    "kd_prodi": "55201",
    "th_akademik": "20251",
    "periodemasuk": "20251",
    "agama": "Islam",
    "pin": "1234567",
    "stpid": null
  },
  "sim": {
    "status_menikah": "Belum Menikah",
    "tempat_tinggal": "Kos",
    "biaya_studi": "Orang Tua",
    "beasiswa_masuk_up": false,
    "tinggi_badan": 170,
    "golongan_darah": "A",
    "kewarganegaraan": "WNI"
  },
  "pembayaran": {
    "tipe_pembayaran": "Pembayaran Lunas Semester 1",
    "skema_pembayaran": "Cicilan 3x"
  }
}
```

**Validasi Field:**

| Field | Tipe | Wajib | Aturan |
|---|---|---|---|
| `npm` | string | ✅ | Unik — ditolak jika NIM sudah ada di tabel `mahasiswa` |
| `profile.nama` | string | ✅ | |
| `profile.nik` | string | ✅ | Tepat 16 digit |
| `profile.nisn` | string\|null | ❌ | |
| `profile.jenis_kelamin` | string | ✅ | Hanya `L` atau `P` |
| `profile.tempat_lahir` | string | ✅ | |
| `profile.tanggal_lahir` | string | ✅ | Format `Y-m-d` |
| `profile.alamat`, `kode_pos`, `rt`, `rw`, `no_hp`, `no_telp_rumah` | string\|null | ❌ | |
| `profile.email` | string | ✅ | Format email valid & unik di tabel `users` |
| `akademik.kd_prodi` | string | ✅ | Harus ada di tabel master `prodi`, lookup `kdpst_dikti` |
| `akademik.th_akademik` | string | ✅ | Format `YYYYS`, contoh `20251` |
| `akademik.periodemasuk` | string | ✅ | Format `YYYYS` |
| `akademik.agama` | string\|null | ❌ | Default `Islam` jika tidak dikirim |
| `akademik.pin` | string | ✅ | |
| `akademik.stpid` | string\|null | ❌ | |
| `sim.*` (object, opsional) | — | ❌ | Jika `sim` dikirim: `status_menikah`, `tempat_tinggal`, `biaya_studi`, `tinggi_badan` (integer), `golongan_darah`, `kewarganegaraan` wajib; `beasiswa_masuk_up` opsional |
| `pembayaran.*` (object, opsional) | — | ❌ | Jika `pembayaran` dikirim: `tipe_pembayaran` wajib; `skema_pembayaran` opsional (saat ini **diabaikan**, reserved untuk fitur masa depan) |

**Nilai `pembayaran.tipe_pembayaran` yang dikenali** (menentukan blok UTS/UAS/KHS awal di `stsmhsblok`):

| Nilai | utsblok | uasblok | khsblok |
|---|---|---|---|
| `"Pembayaran Lunas Semester 1"` | 1 | 1 | 1 |
| `"Pembayaran Lunas 8 Semester"` | 1 | 1 | 1 |
| Nilai lain (termasuk typo) | 0 | 0 | 0 |

**Response (201) — Sukses:**
```json
{ "message": "Data mahasiswa baru berhasil disimpan", "data": { "npm": "20210001" } }
```

**Response error:**

| HTTP | Skenario | Contoh `message` |
|---|---|---|
| 401 | API Key tidak valid/tidak ada | `Unauthorized: API Key tidak valid` |
| 422 | NPM sudah terdaftar | `NPM 20210001 sudah terdaftar` |
| 422 | Email sudah dipakai user lain | `Email budi@email.com sudah digunakan` |
| 422 | `kd_prodi` tidak ditemukan | `kd_prodi 99999 tidak ditemukan` |
| 422 | Validasi field gagal | `The profile.nik field must be 16 characters.` |
| 500 | Error server/exception | `Terjadi kesalahan internal` |

**Yang terjadi di balik layar saat sukses:**
1. Insert `users`: `username = npm`, `password = hash(nik)`, role `mahasiswa` di-assign.
2. Insert `mahasiswa`: `status = "Aktif"`, `smt = 1`, `kdpst_dikti` di-lookup dari `kd_prodi`, `email_verified = 0`.
3. Insert `mhs_sim` (jika `sim` atau `pembayaran` dikirim).
4. Insert `stsmhsblok` (jika `pembayaran` dikirim).

> Mahasiswa **belum bisa login** setelah sync — wajib melakukan **Aktivasi Akun** manual di NeoSIAK (NIM + NIK → set email & password baru). Endpoint ini **non-idempotent**: re-sync NIM yang sama akan selalu gagal 422.

---

## 5. Endpoint — User (OAuth2 Bearer Token)

Kedua endpoint berikut memakai middleware `auth:api` (Laravel Passport) — wajib header:
```
Authorization: Bearer {access_token}
Accept: application/json
```

### 5.1 Info User yang Sedang Login

```
GET /api/user
```

**Response (200):**
```json
{
  "status": "success",
  "code": "200",
  "message": "Successfully Retrieve Data",
  "data": {
    "id": "01h...",
    "name": "Budi Santoso",
    "username": "1223210001",
    "email": "budi@student.univpancasila.ac.id",
    "first_name": "Mr. ",
    "photo": "https://storage.univpancasila.ac.id/.../foto.jpg",
    "photo_thumbnail": "https://storage.univpancasila.ac.id/.../thumb.jpg"
  }
}
```

- Field `email` **hanya muncul** jika access token memiliki scope `get-email`.
- Field `first_name` dihitung dari `mahasiswa.sex` (`L` → `"Mr. "`, selain itu → `"Mrs. "`).
- Field `photo` dan `photo_thumbnail` diambil dari `mahasiswa.storage_url` dan `mahasiswa.storage_url_thumbnail` — `null` jika mahasiswa belum pernah upload foto profil (lihat [§4.3](#43-upload-foto-profil-mahasiswa)).
- Field yang selalu disembunyikan: `fakultas_id`, `email_verified_at`, `password`, `remember_token`, `created_at`, `updated_at`, relasi `mahasiswa`.

---

### 5.2 Info User + Data Akademik (Fasttrack)

```
GET /api/user/fasttrack
```

Sama seperti `/user`, tapi diperkaya dengan ringkasan akademik mahasiswa (untuk dashboard/portal).

**Response (200):**
```json
{
  "status": "success",
  "code": "200",
  "message": "Successfully Retrieve Data",
  "data": {
    "id": "01h...",
    "name": "Budi Santoso",
    "username": "1223210001",
    "email": "budi@student.univpancasila.ac.id",
    "kd_prodi": "55201",
    "kdpst_dikti": "57201",
    "jumlah_sks": 120,
    "ipk": 3.45,
    "periodemasuk": "20221",
    "status": "Aktif",
    "statusMahasiswa": [
      { "id": 1, "thakd": "20221", "nim": "1223210001", "nama": "Budi Santoso", "statusmhs": "Aktif" }
    ]
  }
}
```

- `jumlah_sks` dan `ipk` dihitung dari `NilaiMahasiswa` dengan filter `tampil = 'Y'` dan `nhuruf` tidak kosong/`-`.
- `statusMahasiswa` adalah riwayat status mahasiswa per tahun akademik (`Statusmhs`), urut `thakd` ascending.

---

## 6. Endpoint — API v1 (OAuth2 Scope-based)

**Base path:** `/api/v1`

Semua endpoint di bagian ini memakai middleware `client:{scope}` — wajib header:
```
Authorization: Bearer {access_token}
Accept: application/json
```
Token harus memiliki scope yang sesuai (lihat kolom **Scope** masing-masing). Format response mengikuti **Konvensi B** ([§3](#3-format-response)).

### 6.1 List Mahasiswa

```
GET /api/v1/master/mahasiswa/list
Scope: read-mahasiswa
```

**Query Parameters:**

| Parameter | Tipe | Wajib | Default | Keterangan |
|---|---|---|---|---|
| `status` | string | ❌ | semua | `Aktif`, `Lulus`, `Cuti`, `Non Aktif`, dll. |
| `kd_prodi` | string | ❌ | — | Kode program studi |
| `fakultas_id` | int | ❌ | — | ID fakultas |
| `jenjang` | string | ❌ | — | `D3`, `D4`, `S1`, `S2`, `S3` |
| `angkatan` | string | ❌ | — | Periode masuk (`periodemasuk`) |
| `page` | int | ❌ | `1` | Halaman |
| `per_page` | int | ❌ | `50` | Maksimal `200` |

**Response (200):**
```json
{
  "status": "00",
  "message": "Successfully Retrieve Data",
  "data": [
    {
      "nim": "1223210001",
      "nama_mhs": "Budi Santoso",
      "kd_prodi": "55201",
      "nama_prodi": "Informatika",
      "fakultas": "Teknik",
      "fakultas_id": 5,
      "jenjang": "S1",
      "status": "Aktif",
      "sex": "L",
      "email": "budi@student.univpancasila.ac.id",
      "hp": "081234567890",
      "agama": "Islam",
      "periodemasuk": "20221",
      "photo": "https://storage.univpancasila.ac.id/.../thumb.jpg",
      "alamat": "Jl. Contoh No. 1",
      "rt_rw": "001/002",
      "kelurahan": "Srengseng Sawah",
      "kecamatan": "Jagakarsa",
      "kodepos": "12640"
    }
  ],
  "meta": { "current_page": 1, "per_page": 50, "total": 245, "last_page": 5 }
}
```

---

### 6.2 Search Mahasiswa (Autocomplete)

```
GET /api/v1/master/mahasiswa/search
Scope: read-mahasiswa
```

**Query Parameters:**

| Parameter | Tipe | Wajib | Default | Keterangan |
|---|---|---|---|---|
| `q` | string | ✅ | — | Keyword nama (LIKE `%q%`) atau NIM (LIKE `q%`), **minimal 3 karakter** |
| `status` | string | ❌ | semua | Filter status |
| `limit` | int | ❌ | `10` | Maksimal `50` |

**Response (200):**
```json
{
  "status": "00",
  "message": "Successfully Retrieve Data",
  "data": [
    { "nim": "1223210001", "nama_mhs": "Budi Santoso", "kd_prodi": "55201", "nama_prodi": "Informatika", "status": "Aktif" }
  ]
}
```

**Response (422) — `q` kurang dari 3 karakter atau tidak dikirim:**
```json
{ "status": "02", "message": "Parameter \"q\" wajib diisi (minimal 3 karakter)", "errors": null }
```

---

### 6.3 Statistik Mahasiswa

```
GET /api/v1/stats/mahasiswa
Scope: read-mahasiswa
```

Tidak ada query parameter. Mengembalikan agregat jumlah mahasiswa per status, per fakultas, dan per prodi.

**Response (200):**
```json
{
  "status": "00",
  "message": "Successfully Retrieve Data",
  "data": {
    "total_aktif": 8500,
    "total_lulus": 45000,
    "total_cuti": 120,
    "total_non_aktif": 85,
    "per_fakultas": [
      { "fakultas_id": 5, "fakultas": "Teknik", "aktif": 1200, "lulus": 8500, "cuti": 15, "non_aktif": 10, "total": 9725 }
    ],
    "per_prodi": [
      { "kd_prodi": "55201", "prodi": "Informatika", "fakultas": "Teknik", "aktif": 450, "lulus": 3200, "cuti": 5, "non_aktif": 3, "total": 3658 }
    ]
  }
}
```

---

### 6.4 List Alumni

```
GET /api/v1/master/alumni/list
Scope: read-alumni
```

Daftar mahasiswa berstatus **Lulus**, diperkaya data kelulusan dari relasi `statusmhsLulus`.

**Query Parameters:**

| Parameter | Tipe | Wajib | Keterangan |
|---|---|---|---|
| `kd_prodi` | string | ❌ | Filter kode prodi |
| `fakultas_id` | int | ❌ | Filter ID fakultas |
| `tahun_lulus` | string | ❌ | Filter `thakd` kelulusan, contoh `2024/2025` |
| `jenjang` | string | ❌ | Filter jenjang |
| `has_email` | string `"true"` | ❌ | Hanya yang punya email tidak kosong |
| `has_phone` | string `"true"` | ❌ | Hanya yang punya HP tidak kosong |
| `page` | int | ❌ | Default `1` |
| `per_page` | int | ❌ | Default `50`, maksimal `200` |

**Response (200):**
```json
{
  "status": "00",
  "message": "Successfully Retrieve Data",
  "data": [
    {
      "nim": "1223210001",
      "nama_mhs": "Budi Santoso",
      "kd_prodi": "55201",
      "nama_prodi": "Informatika",
      "fakultas": "Teknik",
      "fakultas_id": 5,
      "jenjang": "S1",
      "periodemasuk": "20221",
      "email": "budi@student.univpancasila.ac.id",
      "hp": "081234567890",
      "sex": "L",
      "ipk_final": 3.45,
      "judul_tugas_akhir": "Implementasi Machine Learning untuk Prediksi Kelulusan",
      "thakd_lulus": "2024/2025"
    }
  ],
  "meta": { "current_page": 1, "per_page": 50, "total": 245, "last_page": 5 }
}
```

---

### 6.5 Detail Alumni by NIM

```
GET /api/v1/master/alumni/{nim}
Scope: read-alumni
```

**Path Parameter:** `nim` (string) — hanya mengembalikan data jika `status = 'Lulus'`.

**Response (200):** sama shape dengan item di [§6.4](#64-list-alumni).

**Response (404):**
```json
{ "status": "10", "message": "Alumni dengan NIM tersebut tidak ditemukan", "data": null }
```

---

### 6.6 List Program Studi

```
GET /api/v1/master/prodi/list
Scope: read-prodi
```

**Query Parameters:**

| Parameter | Tipe | Wajib | Keterangan |
|---|---|---|---|
| `jenjang` | string | ❌ | Filter jenjang (`S1`, `S2`, dll.) |
| `fakultas_id` | int | ❌ | Filter ID fakultas |

**Response (200):**
```json
{
  "status": "00",
  "message": "Successfully Retrieve Data",
  "data": [
    { "kd_prodi": "55201", "prodi": "Informatika", "fakultas": "Teknik", "fakultas_id": 5, "jenjang": "S1" }
  ]
}
```

---

### 6.7 List Fakultas

```
GET /api/v1/master/fakultas/list
Scope: read-prodi
```

Tidak ada query parameter. Data berasal dari `GROUP BY fakultas_id, fakultas` pada tabel `prodi` (NeoSIAK tidak punya tabel fakultas terpisah).

**Response (200):**
```json
{
  "status": "00",
  "message": "Successfully Retrieve Data",
  "data": [
    { "fakultas_id": 1, "fakultas": "Farmasi", "jumlah_prodi": 3 },
    { "fakultas_id": 5, "fakultas": "Teknik", "jumlah_prodi": 4 }
  ]
}
```

---

## 7. OAuth2 — Cara Mendapatkan Access Token

Berlaku untuk endpoint `/user`, `/user/fasttrack`, dan seluruh `/v1/*`. NeoSIAK memakai **Laravel Passport** dengan 2 grant type:

### 7.1 Client Credentials Grant (server-to-server, tanpa user)

```http
POST /oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id={client_id}
&client_secret={client_secret}
&scope=read-mahasiswa read-alumni read-prodi
```

**Response:**
```json
{ "token_type": "Bearer", "expires_in": 1296000, "access_token": "eyJ0eXAiOiJKV1Qi..." }
```

### 7.2 Authorization Code Grant (atas nama user yang login)

```
# Step 1 — Redirect user:
GET /oauth/authorize?client_id={id}&redirect_uri={uri}&response_type=code&scope=read-mahasiswa

# Step 2 — User login & approve, redirect balik:
GET {redirect_uri}?code={auth_code}

# Step 3 — Tukar code dengan token:
POST /oauth/token
grant_type=authorization_code
&client_id={id}&client_secret={secret}&redirect_uri={uri}&code={auth_code}
```

**Response:**
```json
{ "token_type": "Bearer", "expires_in": 1296000, "access_token": "eyJ0...", "refresh_token": "def50200..." }
```

### 7.3 Refresh Token

```http
POST /oauth/token
grant_type=refresh_token
&client_id={id}&client_secret={secret}&refresh_token={refresh_token}&scope=read-mahasiswa
```

### 7.4 Memanggil Endpoint API

```http
GET /api/v1/master/prodi/list HTTP/1.1
Host: neosiak.univpancasila.ac.id
Authorization: Bearer {access_token}
Accept: application/json
```

> `client_id`/`client_secret` diterbitkan melalui halaman **OAuth Config** di admin panel NeoSIAK (route `oauth-config.*`) — hubungi admin NeoSIAK untuk pendaftaran client baru. Daftar route OAuth bawaan Passport yang tersedia: `oauth/authorize`, `oauth/token`, `oauth/token/refresh`, `oauth/clients`, `oauth/personal-access-tokens`, `oauth/tokens`, `oauth/scopes`.

---

## 8. Referensi Kode Error & HTTP Status

| HTTP | Kapan Terjadi |
|---|---|
| `200` | Request berhasil |
| `201` | Resource berhasil dibuat (`/mahasiswa-baru`) |
| `401` | Token/API Key/checksum tidak valid, expired, atau tidak disertakan |
| `403` | Token tidak memiliki scope yang diperlukan (`/v1/*`) |
| `404` | Data tidak ditemukan |
| `422` | Validation error |
| `429` | Rate limit terlampaui |
| `500` | Server error |

**Auth error generik dari Passport (di luar JSON konvensi A/B/C/D):**
```json
{ "message": "Unauthenticated." }              // 401
{ "message": "Invalid scope(s) provided." }    // 403
```

---

## 9. Rate Limiting

Seluruh route `/api/*` melewati middleware group `api` Laravel, yang menerapkan `throttle:api` secara default (lihat `app/Http/Kernel.php`). Tidak ada override custom per-route di `routes/api.php` saat ini — seluruh endpoint memakai limit default Laravel.

Saat limit terlampaui:
```
HTTP/1.1 429 Too Many Requests
Retry-After: {detik}
```

---

## 10. Catatan Keamanan & Known Issues

Temuan berikut diambil langsung dari pembacaan source code — relevan untuk siapa pun yang mengelola atau mengaudit API ini:

1. **Endpoint public tanpa proteksi apapun:**
   - `GET /api/master/mahasiswa` — siapa saja bisa lookup data mahasiswa (termasuk NIK/KTP) hanya dengan tahu NIM.
   - `POST /api/uploadProfilePhoto` — siapa saja bisa upload foto ke akun mahasiswa manapun hanya dengan tahu NIM.
2. **Kredensial hardcoded di source code** (bukan di `.env`): API key di `MahasiswaController::rekapNilai`, checksum di `StudentStatusController::index`, dan API key storage eksternal di `MahasiswaController::uploadProfilePhoto`. Disarankan dipindah ke environment variable agar rotasi kredensial tidak butuh deploy ulang kode, dan tidak ter-commit ke git history.
3. **`/api/mahasiswa-baru` bersifat strict & non-idempotent** — re-sync NIM yang sama selalu gagal 422. Tidak ada mekanisme upsert.
4. **Inkonsistensi format response** — 4 konvensi berbeda ([§3](#3-format-response)) menyulitkan client membuat satu generic error handler. Pertimbangkan migrasi bertahap ke satu konvensi (disarankan Konvensi B / `ApiResponse` trait) untuk endpoint baru.
5. **`StudentStatusController::index`** menerima parameter `cicilan_ke` yang **tidak dipakai** di logic manapun saat ini — kemungkinan sisa implementasi yang belum selesai atau sudah deprecated.
6. **`MahasiswaController::uploadProfilePhoto`** mengembalikan response kosong (bukan error) jika field `photo` tidak dikirim dalam request — bisa membingungkan konsumen API yang mengharapkan error eksplisit.
7. **Field sensitif** (`password`, `pwdasl`, `newpassword`, `pin`) **tidak** diekspos melalui endpoint `/v1/*` — sudah memakai API Resource Laravel untuk kontrol field. Namun endpoint legacy `/master/mahasiswa` mengekspos field `sim.ktp` (NIK).

---

*Dokumen ini dihasilkan berdasarkan pembacaan langsung `routes/api.php` dan seluruh controller terkait (`app/Http/Controllers/Api/**`) per 23 Juni 2026. Jika ada endpoint baru ditambahkan setelah tanggal ini, dokumen perlu diperbarui ulang.*
