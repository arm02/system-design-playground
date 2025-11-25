# **Message Broker**
##### Write by: Adrian Milano

## **Concepts**

**Dead Letter Queue (DLQ)**
- Apa itu: Sebuah antrean khusus yang dirancang untuk menyimpan pesan-pesan yang gagal diproses setelah beberapa kali percobaan retry (coba ulang).
- Tujuan:
    - Isolasi: Mencegah pesan yang "beracun" (poison pill messages) memblokir seluruh antrean utama.
    - Debugging: Memberikan tempat terpusat bagi developer untuk menganalisis mengapa pesan gagal, tanpa perlu menghentikan sistem.
- Cara Kerja: Setelah Consumer gagal memproses pesan N kali, Broker secara otomatis mengalihkannya ke antrean DLQ.

**Idempotency**
- Apa itu: Sifat di mana melakukan operasi yang sama berkali-kali menghasilkan efek akhir yang sama seolah-olah hanya dilakukan sekali.
- Kaitan MQ: Hampir semua Broker menjamin pengiriman At-Least-Once. Artinya, pesan bisa terkirim dua kali (duplicate). Consumer harus didesain Idempotent (misalnya, menggunakan Idempotency Key) untuk menangani duplicate tersebut tanpa menimbulkan efek samping ganda (misalnya, menagih kartu kredit dua kali).

**At-Least-Once vs. Exactly-Once**
- At-Least-Once (Paling Umum): Broker menjamin bahwa pesan akan dikirim ke Consumer setidaknya satu kali (bisa lebih). Ini adalah default yang lebih mudah dicapai di Kafka/RabbitMQ. Solusinya membutuhkan Consumer yang Idempotent.
- Exactly-Once (The Holy Grail): Broker menjamin bahwa pesan akan diproses tepat satu kali, bahkan saat terjadi kegagalan. Ini sangat kompleks dan membutuhkan jaminan di sisi Broker dan Stream Processor (misalnya, Kafka Streams/Flink Transactional Producer). Ini biasanya tidak diperlukan kecuali dalam sistem keuangan yang sangat ketat.

**Consumer Group**
- Apa itu: Sekelompok Consumer yang secara kolektif berlangganan satu Topic (di Kafka) atau Queue (di RabbitMQ).
- Tujuan:
    - Horizontal Scaling: Memungkinkan Anda menjalankan banyak instance Consumer untuk memproses message secara paralel, sehingga meningkatkan throughput.
    - Load Balancing: Setiap pesan di Topic (atau partisi di Kafka) dijamin hanya akan diproses oleh satu Consumer di dalam Group tersebut.
- Kaitan Kafka: Di Kafka, Consumer Group mengelola offset (posisi baca) bersama. Jika sebuah Topic memiliki 4 partitions, dan ada 4 Consumer dalam satu Group, setiap Consumer akan bertanggung jawab atas 1 partition saja.