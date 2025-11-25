# **Security**
##### Write by: Adrian Milano

**1. JWT Flow (JSON Web Token)**

JWT digunakan untuk Authentication (siapa Anda) dan Authorization (izin Anda).
1. Authentication (Login): Klien mengirim username dan password ke Auth Service.
2. Token Generation: Auth Service memvalidasi kredensial. Jika valid, ia membuat JWT, menandatanganinya menggunakan Secret Key (HMAC/RSA), dan menyimpulkan claims (misalnya, `user_id`, `role`) di dalam payload.
3. Token Return: Token dikirim kembali ke klien (misalnya, disimpan di HTTP-only Cookie atau Local Storage).
4. Authorization (Next Request): Klien menyertakan Token di header `Authorization: Bearer <token>` untuk setiap request ke API Gateway atau Resource Service.
5. Validation: API Gateway atau middleware memverifikasi tanda tangan Token menggunakan Secret Key (tanpa perlu memanggil Auth Service) dan mengekstrak claims untuk mengizinkan atau menolak akses.

**2. Bcrypt / Argon2 (Password Hashing)**

Kedua algoritma ini adalah standar industri untuk hashing password (mengubah password menjadi string yang tidak dapat dibaca) dan mencegah serangan Rainbow Table atau Brute Force.
- Bcrypt: Algoritma yang menggunakan fungsi Key Derivation Function (KDF) dan secara bawaan lambat (slow) dan cost-factor. Ini berarti membutuhkan waktu dan CPU yang signifikan untuk menghitung, sehingga memperlambat serangan brute force (misalnya, membutuhkan 1 detik untuk menghitung $1$ password, sehingga $1$ juta password membutuhkan $1$ juta detik).
- Argon2: Saat ini dianggap sebagai algoritma hashing terkuat dan pemenang Password Hashing Competition. Selain menjadi cost-factor (CPU-intensive), Argon2 juga memory-hard (membutuhkan banyak RAM), yang membuatnya sangat efektif melawan serangan yang menggunakan GPU atau ASIC karena perangkat tersebut memiliki keterbatasan RAM.

Penting: Tidak pernah menggunakan hashing cepat seperti SHA-256 atau MD5 untuk password. Selalu gunakan KDF yang lambat dan teruji seperti Bcrypt atau Argon2.

**3. SQL Injection Prevention (Injeksi SQL)**

SQL Injection terjadi ketika input pengguna yang tidak dimurnikan (unsanitized) disalahgunakan sebagai bagian dari query SQL, berpotensi memaparkan, mengubah, atau menghapus data.
- Pencegahan Utama (The ONLY Answer): Gunakan Prepared Statements (Parameterized Queries).
    - Cara Kerja: Template query (misalnya, SELECT * FROM users WHERE id = $1) dikirim ke database secara terpisah dari nilai input pengguna (nilai 123). Database tidak pernah menginterpretasikan input sebagai bagian dari kode SQL, hanya sebagai data murni.
- Framework Go: ORM (GORM/sqlx) dan paket database/sql secara otomatis menggunakan prepared statements.

**4. XSS & CSRF (Web Vulnerabilities)**

Ini adalah dua serangan web paling umum yang menargetkan client (browser):

A. Cross-Site Scripting (XSS)
- Serangan: Penyerang menyuntikkan (inject) malicious client-side script (biasanya JavaScript) ke halaman web yang dilihat oleh user lain.
- Pencegahan:
    - Output Encoding: Selalu escape (mengubah karakter berbahaya menjadi representasi aman, misalnya < menjadi &lt;) semua input pengguna sebelum ditampilkan di HTML. Template engine modern (React, Vue, Go's html/template) sering melakukannya secara default.
    - Content Security Policy (CSP): Menggunakan HTTP header untuk membatasi sumber yang diizinkan untuk memuat script dan asset lainnya.

B. Cross-Site Request Forgery (CSRF)
- Serangan: Penyerang menipu user yang sudah terautentikasi untuk mengirim request yang tidak disengaja ke situs web Anda (misalnya, transfer uang) dari situs web penyerang.
- Pencegahan:
    - SameSite Cookies: Mengatur cookie otentikasi ke `SameSite=Strict` atau `Lax`.
    - CSRF Tokens: Setiap request yang mengubah status (POST, PUT, DELETE) harus menyertakan CSRF Token unik di header atau body. Server memvalidasi token ini.
    - Custom Headers: Membutuhkan header non-standar (misalnya, `X-Requested-With`) karena browser biasanya membatasi permintaan lintas domain untuk header kustom.

**5. Rate Limiting**
- Pencegahan: Mencegah penyalahgunaan API atau serangan DoS dengan membatasi jumlah request yang diizinkan dari client atau IP tertentu dalam jangka waktu tertentu.
- Implementasi: Dilakukan di API Gateway atau middleware terdepan, sering menggunakan Redis sebagai central counter dengan algoritma Fixed Window atau Leaky Bucket.

**6. HTTPS (Keamanan Transport Layer)**
- Pencegahan: Memastikan semua komunikasi antara client dan server dienkripsi, mencegah eavesdropping (menguping) atau man-in-the-middle attacks saat data melintasi jaringan publik.
- Mekanisme: Menggunakan TLS/SSL (Transport Layer Security).
- Senior Check: Memastikan deployment menggunakan sertifikat TLS yang diperbarui, dan semua lalu lintas HTTP dialihkan secara paksa ke HTTPS.

**7. Environment Management (.env + Vault)**

Manajemen konfigurasi dan secrets (kunci rahasia) secara aman.

- .env (Environment Variables): Digunakan untuk menyimpan konfigurasi non-sensitif yang berubah antar lingkungan (Dev/Staging/Prod), seperti Port Number, Log Level, atau Config Server Address.
    - Penting: File `.env` tidak boleh dikomit ke Git.
- Vault (Secret Store): Digunakan untuk menyimpan secrets sensitif (misalnya, Password Database, Private Keys, API Keys pihak ketiga) secara terpusat, terenkripsi, dan memiliki kontrol akses ketat (Role-Based Access Control).
    - Alur Aman: Aplikasi tidak mengambil password dari .env. Aplikasi hanya mengambil token otentikasi dari .env, menggunakan token tersebut untuk mengakses Vault, dan Vault mengembalikan secret yang dibutuhkan ke aplikasi saat runtime. Ini adalah pola Config Management yang aman.