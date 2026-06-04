# Penjelasan Metode Sistem Pakar dan Penerapannya

Aplikasi Sistem Pakar Diagnosis ISPA ini menggunakan gabungan beberapa metode kecerdasan buatan (Artificial Intelligence) untuk menghasilkan diagnosis yang akurat. Berikut adalah penjelasan ketiga metode utama dan bagaimana penerapannya secara langsung di dalam kode dan alur aplikasi.

---

## 1. Knowledge Acquisition (Akuisisi Pengetahuan dari Dataset)

**Penjelasan:**
Knowledge Acquisition adalah proses mengumpulkan data dan pengetahuan medis, lalu mengubahnya menjadi format yang bisa dipahami komputer. Alih-alih hanya mengandalkan wawancara pakar, sistem ini dirancang untuk dapat "belajar" dan mengekstrak aturan (Rule) serta nilai kepastian (Certainty Factor Pakar) secara otomatis berdasarkan dataset rekam medis historis.

**Penerapan pada Aplikasi:**
- Semua data historis pasien disimpan di tabel `DatasetRow`.
- Saat fungsi _Training_ dijalankan (melalui endpoint API `/api/rules/train/` atau via command line `recalculate_rules_and_cf()`), sistem akan melakukan proses berikut secara *offline/background*:
  - Mengelompokkan semua kasus berdasarkan penyakit (misal: Faringitis).
  - Menghitung probabilitas (*Expert CF*) untuk setiap gejala. Rumusnya adalah:
    > `expert_cf = (Jumlah Kasus Penyakit X dengan Gejala Y) / (Total Kasus Penyakit X)`
  - Jika `expert_cf` sebuah gejala di atas ambang batas (threshold), gejala tersebut akan disimpan ke dalam tabel `CertaintyFactor` dan dijadikan sebagai aturan baku (Tabel `Rule` & `RuleSymptom`).
- **Keuntungan**: Dokter tidak perlu repot memasukkan angka CF secara manual satu per satu. Sistem yang akan merumuskannya secara otomatis berdasarkan ribuan data rekam medis.

---

## 2. Forward Chaining (Runut Maju)

**Penjelasan:**
Forward Chaining adalah metode inferensi (penarikan kesimpulan) yang bergerak maju, mulai dari mengumpulkan fakta-fakta (gejala) yang ada untuk mencapai suatu kesimpulan akhir (diagnosis penyakit).

**Penerapan pada Aplikasi:**
Metode ini digunakan sebagai **Filter Awal** agar sistem tidak perlu menghitung algoritma CF untuk seluruh penyakit yang ada di database. 
- Saat pasien memasukkan daftar gejala yang dialaminya (Fakta), sistem membandingkan gejala tersebut dengan seluruh aturan (`Rule`) penyakit di database.
- Sebuah penyakit **lolos filter** Forward Chaining jika memenuhi syarat (misalnya minimal 2 gejala dari penyakit tersebut cocok dengan yang dialami pasien, ATAU kecocokan mencapai rasio 50%).
- Penyakit yang lolos filter ini disebut **Kandidat Penyakit**.
- Penyakit yang sama sekali tidak memiliki kecocokan gejala akan diabaikan (di-eliminasi lebih awal) untuk menghemat komputasi dan mencegah diagnosis yang tidak masuk akal (False Positive).

---

## 3. Certainty Factor (Faktor Kepastian MYCIN)

**Penjelasan:**
Certainty Factor (CF) adalah metode untuk mengukur seberapa yakin atau pasti suatu diagnosis ketika berhadapan dengan informasi yang samar atau tidak pasti (misalnya saat pasien menjawab "Mungkin", "Ragu-ragu", "Sangat Yakin" pada kuesioner gejala). 

Metode CF mengukur dua sisi keyakinan:
1. **CF Pakar (Expert CF)**: Nilai probabilitas empiris dari sistem (didapat dari tahap Knowledge Acquisition).
2. **CF Pengguna (User CF)**: Nilai keyakinan subjektif pasien saat berkonsultasi (misal: 0.8 untuk Sangat Yakin, 0.4 untuk Kurang Yakin).

**Penerapan pada Aplikasi:**
Setelah *Forward Chaining* menghasilkan daftar Kandidat Penyakit, metode CF mulai bekerja (dikalkulasi di memori backend):

**Langkah 1: Menghitung CF Current (Gejala Tunggal)**
Untuk setiap gejala yang dialami pasien pada kandidat penyakit tersebut, sistem mengalikan nilai CF User dengan CF Expert:
> `CF(Gejala_A) = CF_User * CF_Expert`

**Langkah 2: Menghitung CF Gabungan (Kombinasi Gejala)**
Sistem kemudian menggabungkan nilai CF dari setiap gejala secara berantai (sekuensial). Rumus standar MYCIN yang diterapkan:
- Jika `CF1` dan `CF2` bernilai positif (pasien yakin mengalami gejala):
  > `CF_Gabungan = CF1 + CF2 * (1 - CF1)`
- Jika bernilai negatif (sistem mendukung adanya _penalty_ untuk gejala yang sangat berlawanan):
  > `CF_Gabungan = CF1 + CF2 * (1 + CF1)`
- Jika berbeda tanda:
  > `CF_Gabungan = (CF1 + CF2) / (1 - min(|CF1|, |CF2|))`

**Langkah 3: Kesimpulan Diagnosis (Final Diagnosis)**
- Hasil akhir berupa persentase. Misalnya, Penyakit Faringitis mendapat CF Gabungan akhir `0.865`. Maka sistem akan memunculkannya ke user sebagai **Tingkat Keyakinan 86.5%**.
- Penyakit dengan persentase CF tertinggi akan ditetapkan sebagai **Final Diagnosis** dan disimpan di tabel `Consultation`.

---

## Alur Singkat (Summary Workflow)

1. **(Offline)** Dataset diolah (Knowledge Acquisition) ➔ Terbentuk *Rule* & nilai *Expert CF*.
2. **(Online)** Pasien mengisi kuesioner gejala.
3. **(Online)** Sistem menyeleksi kandidat penyakit berdasarkan kuesioner tersebut (Forward Chaining).
4. **(Online)** Sistem menghitung nilai akhir keyakinan dari kandidat penyakit yang lolos filter (Certainty Factor).
5. **(Online)** Pasien menerima hasil diagnosis (Penyakit dominan, Solusi pengobatan, & Persentase kepastian).
