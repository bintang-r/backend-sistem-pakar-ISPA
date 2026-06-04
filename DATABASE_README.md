# Skema Database Sistem Pakar ISPA

Berikut adalah daftar tabel (Model Django) yang digunakan dalam backend aplikasi ini beserta penjelasannya, relasi antar tabel, dan pemahaman operasional cara kerjanya.

## 1. Authentication (Manajemen Pengguna)

### `User`
Tabel turunan dari `AbstractUser` bawaan Django, digunakan untuk menyimpan data autentikasi pengguna dan peran pengguna (Role).
- **Atribut**: `role` (Admin/User/Health Worker), `full_name`, `profile_picture`.
- **Relasi**:
  - `1:N` ke `Consultation` (Setiap user dapat memiliki banyak histori konsultasi).
  - `1:N` ke `Message` (sebagai Sender atau Receiver).
  - `1:1` ke `Testimonial` (Satu user, satu testimonial).
- **Cara Kerja**: Saat pasien mendaftar di web, datanya masuk ke tabel ini dengan `role` otomatis "user". Admin dapat mengubah role pasien menjadi "health_worker" jika mereka adalah staf medis, sehingga mereka mendapatkan akses ke Dashboard Pakar dan Fitur Chat.

---

## 2. Parameter Utama Sistem Pakar (Master Data)

### `Symptom` (Gejala)
Menyimpan daftar seluruh gejala penyakit ISPA yang bisa dialami oleh pasien.
- **Atribut**: `code` (S01, S02, dst), `name`, `description`.
- **Relasi**:
  - `1:N` ke `RuleSymptom` (Satu gejala bisa menjadi bagian dari banyak aturan IF-THEN).
  - `1:N` ke `CertaintyFactor` (Satu gejala memiliki banyak nilai pakar tergantung penyakitnya).
  - `1:N` ke `ConsultationDetail` (Satu gejala dapat dialami dalam banyak sesi konsultasi pasien).
- **Cara Kerja**: Tabel ini merupakan parameter master. Saat sistem menampilkan form kuesioner diagnosis kepada pengguna, form tersebut di-_generate_ berdasarkan list gejala dari tabel ini secara dinamis.

### `Disease` (Penyakit)
Menyimpan jenis-jenis penyakit ISPA beserta solusi penanganannya.
- **Atribut**: `code` (P01, P02, dst), `name`, `category`, `description`, `recommendation`, `treatment_solutions`, `recovery_steps`.
- **Relasi**:
  - `1:N` ke `Rule` (Satu penyakit memiliki beberapa aturan IF-THEN).
  - `1:N` ke `CertaintyFactor` (Satu penyakit berelasi dengan banyak nilai pakar berdasarkan gejalanya).
- **Cara Kerja**: Hasil akhir (*Final Diagnosis*) dari proses hitung CF akan selalu merujuk pada tabel ini. Rekomendasi obat atau saran kesembuhan yang muncul pada halaman hasil pasien juga diambil langsung dari data atribut tabel ini.

---

## 3. Knowledge Base (Basis Pengetahuan CF & Rule)

### `Rule`
Mendefinisikan kode aturan IF-THEN untuk penyakit tertentu.
- **Atribut**: `code` (R01, R02).
- **Relasi**:
  - `N:1` ke `Disease` (Aturan ini merujuk pada diagnosis penyakit apa).
  - `1:N` ke `RuleSymptom` (Detail dari gejala-gejala IF-nya).
- **Cara Kerja**: Tabel ini berperan sebagai *knowledge base* logika _Forward Chaining_. Sistem membaca tabel ini untuk mengecek, "Jika pasien mengalami Gejala A, B, C, maka Rule P01 aktif (terpenuhi)". 

### `RuleSymptom`
Tabel _Many-to-Many_ (perantara) yang menghubungkan antara `Rule` dengan `Symptom`. Berfungsi sebagai kumpulan kondisi (gejala) yang mengaktifkan aturan tertentu.
- **Relasi**:
  - `N:1` ke `Rule`.
  - `N:1` ke `Symptom`.
- **Cara Kerja**: Menyimpan konfigurasi detail dari tiap aturan. Misalnya untuk `Rule R01` (Tonsilitis), tabel ini mencatat bahwa gejala S01 (Sakit Tenggorokan) dan S02 (Demam) wajib ada agar R01 bisa dianggap terpenuhi secara logika.

### `CertaintyFactor` (Bobot Pakar)
Tabel matriks yang menyimpan bobot kepercayaan/keyakinan (CF Expert) untuk setiap kombinasi Gejala terhadap suatu Penyakit.
- **Atribut**: `expert_cf` (Nilai dari 0 sampai 1).
- **Relasi**:
  - `N:1` ke `Disease`.
  - `N:1` ke `Symptom`.
- **Cara Kerja**: Ini adalah "Otak" perhitungan probabilitas aplikasi. Setiap kali pasien memilih gejala, sistem akan mengalikan nilai *CF User* pasien dengan nilai `expert_cf` dari tabel ini. Tabel inilah yang paling sering diutak-atik (di-tweak) oleh dokter/pakar di halaman admin untuk mengubah keakuratan hasil prediksi sistem.

### `DatasetRow` (Data Latih / Master Dataset)
Menyimpan rekam medis/kasus historis yang digunakan sebagai data latih untuk mencocokkan kemiripan kasus konsultasi pasien baru.
- **Atribut**: `age`, flag *binary* gejala (0 atau 1) seperti `batuk_kering`, `demam`, dll, dan nilai akhir `diagnosis`.
- **Cara Kerja**: Tabel ini akan mempelajari data kasus diagnosis yang pernah ada. Saat pasien baru melakukan konsultasi, algoritma (baik K-NN/Cosine) akan mencocokkan *pattern* gejala pasien dengan seluruh baris historis di tabel ini, lalu mencari *DatasetRow* mana yang paling mirip dan menampilkan riwayat terdekatnya (Dataset Matcher).

---

## 4. Transaksi & Interaksi

### `Consultation`
Menyimpan histori / sesi konsultasi saat seorang pasien (User) melakukan pemeriksaan sistem pakar.
- **Atribut**: `consultation_date`, `final_diagnosis`, `confidence_result` (persentase CF akhir), `age`.
- **Relasi**:
  - `N:1` ke `User` (Pasien yang berkonsultasi).
  - `1:N` ke `ConsultationDetail` (Daftar gejala yang dipilih pasien).
- **Cara Kerja**: Setiap kali pasien menekan tombol "Submit Diagnosis", satu baris `Consultation` baru dibuat. Tabel ini merangkum hasil akhirnya (Penyakit apa, Persentase CF berapa). Tabel inilah yang ditampilkan di halaman "Riwayat Konsultasi" pasien dan di *Dashboard Analytical* admin.

### `ConsultationDetail`
Menyimpan gejala-gejala yang diderita pasien pada suatu sesi konsultasi beserta bobot dari user (CF User).
- **Atribut**: `user_cf` (Bobot keyakinan dari pasien terhadap gejala tersebut, misalnya 0.8 untuk "Sangat Yakin").
- **Relasi**:
  - `N:1` ke `Consultation`.
  - `N:1` ke `Symptom`.
- **Cara Kerja**: Saat pasien memilih "Ya, Sangat Yakin" untuk gejala "Batuk Berdahak" pada saat form kuesioner, baris baru direkam di tabel ini. Tabel ini digunakan sebagai jejak log (*audit trail*) jika suatu saat dokter/admin ingin mengevaluasi *mengapa* sistem mendiagnosis pasien ini dengan penyakit A.

### `Message`
Tabel untuk menyimpan data perpesanan atau obrolan (*chatting*) antar pengguna (misal antara Pasien dan Health Worker).
- **Atribut**: `content`, `timestamp`, `is_read`.
- **Relasi**:
  - `N:1` ke `User` (Sebagai `sender`).
  - `N:1` ke `User` (Sebagai `receiver`).
- **Cara Kerja**: Bekerja layaknya aplikasi *instant messaging*. Sistem membaca dua parameter (`sender` dan `receiver`) untuk merender *chat bubble* kanan atau kiri di UI. Tabel ini memfasilitasi tele-konsultasi pasca-diagnosis antara pasien dan perawat.

### `Testimonial`
Menyimpan penilaian (*rating*) dan ulasan pengguna terkait platform atau hasil konsultasi.
- **Atribut**: `rating`, `content`, `created_at`.
- **Relasi**:
  - `1:1` ke `User` (Hanya satu testimoni untuk setiap user).
- **Cara Kerja**: Ditampilkan secara dinamis di *Landing Page* (Homepage) aplikasi sebagai bagian dari profil (*Social Proof*). Data diambil dan difilter untuk memunculkan review terbaik dari pengguna sungguhan.

---

## 5. Master Data Pakar

### `HealthExpert`
Menyimpan profil pakar medis (seperti dokter) yang dirujuk atau terlibat dalam penentuan nilai basis pengetahuan pada sistem.
- **Atribut**: `name`, `profession`, `workplace`, `notes`.
- **Cara Kerja**: Hanya tabel referensi informasi pakar, tidak secara langsung dilibatkan dalam logika kode kalkulasi CF. Biasanya digunakan di halaman "Tentang Kami" untuk memperlihatkan validitas/sumber ilmiah dari mana tabel `CertaintyFactor` aplikasi ini berasal (misal: "Basis aturan dirancang berdasarkan riset Dokter X dari RS Y").
