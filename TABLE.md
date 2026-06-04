# Daftar Tabel, Kolom, dan Relasi Database

Berikut adalah ringkasan daftar tabel (Model Django) yang digunakan pada backend beserta daftar kolom (arti dalam bahasa Indonesia) dan relasi antar tabelnya.

| Nama Tabel | Deskripsi Singkat | Daftar Kolom Utama & Artinya | Relasi (Foreign Key / One-to-One / Many-to-Many) |
| --- | --- | --- | --- |
| **`User`** | Menyimpan data autentikasi & role (Admin/User/Health Worker). | - `role`: Peran pengguna<br>- `full_name`: Nama lengkap<br>- `profile_picture`: Foto profil | - `1:N` ke `Consultation` (sebagai pasien)<br>- `1:N` ke `Message` (sebagai pengirim/penerima)<br>- `1:1` ke `Testimonial` |
| **`Symptom`** | Master data gejala penyakit ISPA. | - `code`: Kode gejala (misal: S01)<br>- `name`: Nama gejala<br>- `description`: Penjelasan gejala | - `1:N` ke `RuleSymptom`<br>- `1:N` ke `CertaintyFactor`<br>- `1:N` ke `ConsultationDetail` |
| **`Disease`** | Master data jenis penyakit ISPA & solusinya. | - `code`: Kode penyakit (misal: P01)<br>- `name`: Nama penyakit<br>- `category`: Kategori penyakit<br>- `description`: Deskripsi<br>- `recommendation`: Rekomendasi medis<br>- `treatment_solutions`: Solusi pengobatan<br>- `recovery_steps`: Langkah pemulihan | - `1:N` ke `Rule`<br>- `1:N` ke `CertaintyFactor` |
| **`Rule`** | Aturan IF-THEN (Forward Chaining) untuk penyakit tertentu. | - `code`: Kode aturan (misal: R01) | - `N:1` ke `Disease`<br>- `1:N` ke `RuleSymptom` |
| **`RuleSymptom`** | Tabel perantara antara Rule dan Symptom (kondisi IF). | - `rule_id`: ID Aturan terkait<br>- `symptom_id`: ID Gejala terkait | - `N:1` ke `Rule`<br>- `N:1` ke `Symptom` |
| **`CertaintyFactor`** | Matriks bobot pakar (CF Expert) untuk gejala dan penyakit. | - `expert_cf`: Nilai probabilitas/bobot kepastian dari pakar (0 s.d 1) | - `N:1` ke `Disease`<br>- `N:1` ke `Symptom` |
| **`DatasetRow`** | Data latih rekam medis/kasus diagnosis historis. | - `age`: Usia pasien historis<br>- *[flag gejala]*: Nilai 0/1 jika gejala dialami<br>- `diagnosis`: Hasil penyakit akhir | *(Tidak memiliki relasi langsung, digunakan untuk K-NN/Cosine)* |
| **`Consultation`** | Histori/sesi konsultasi pasien dengan hasil akhirnya. | - `consultation_date`: Waktu melakukan konsultasi<br>- `final_diagnosis`: Hasil diagnosis akhir penyakit<br>- `confidence_result`: Persentase hasil keyakinan<br>- `age`: Usia pasien saat itu | - `N:1` ke `User`<br>- `1:N` ke `ConsultationDetail` |
| **`ConsultationDetail`** | Detail gejala yang dipilih pasien pada satu sesi konsultasi. | - `user_cf`: Bobot kepastian/keyakinan dari pasien (misal: "Sangat Yakin") | - `N:1` ke `Consultation`<br>- `N:1` ke `Symptom` |
| **`Message`** | Data obrolan (chat) antar pengguna (Pasien & Pakar). | - `content`: Isi pesan obrolan<br>- `timestamp`: Waktu pesan dikirim<br>- `is_read`: Status apakah pesan sudah dibaca | - `N:1` ke `User` (Sender)<br>- `N:1` ke `User` (Receiver) |
| **`Testimonial`** | Rating dan ulasan dari pasien terkait platform. | - `rating`: Penilaian (bintang)<br>- `content`: Isi ulasan / review<br>- `created_at`: Waktu ulasan dibuat | - `1:1` ke `User` |
| **`HealthExpert`** | Profil pakar medis/dokter referensi CF. | - `name`: Nama lengkap pakar<br>- `profession`: Pekerjaan/profesi pakar<br>- `workplace`: Tempat bekerja medis<br>- `notes`: Catatan referensi/sumber | *(Tabel referensi mandiri, tidak berelasi langsung dengan inti logic)* |

## Diagram Relasi Sederhana

```mermaid
erDiagram
    USER ||--o{ CONSULTATION : melakukan
    USER ||--o{ MESSAGE : mengirim_menerima
    USER ||--o| TESTIMONIAL : memberi

    DISEASE ||--o{ RULE : memiliki
    DISEASE ||--o{ CERTAINTY_FACTOR : berelasi_dengan
    
    SYMPTOM ||--o{ RULE_SYMPTOM : menjadi_syarat
    SYMPTOM ||--o{ CERTAINTY_FACTOR : berelasi_dengan
    SYMPTOM ||--o{ CONSULTATION_DETAIL : dialami

    RULE ||--o{ RULE_SYMPTOM : terdiri_dari

    CONSULTATION ||--o{ CONSULTATION_DETAIL : memiliki_detail
```
