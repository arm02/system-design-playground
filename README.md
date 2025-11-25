# System Design Note-talker
##### Write by: Adrian Milano

## Microservices
### Event-Driven Architecture
**Event-Driven Architecture (EDA)** adalah pola arsitektur di mana layanan-layanan (services) dalam sistem berkomunikasi dengan cara memancarkan "Event" (Peristiwa), bukan dengan saling memanggil secara langsung (Direct Request).

**1. Analogi: "Telepon" vs. "Grup WhatsApp"**
- REST API (Request-Response) = Menelepon
  - Service A menelepon Service B: "Halo, tolong update stok dong. Aku tungguin ya di telepon sampai kamu selesai."
  - Masalah: Kalau Service B sibuk atau mati, Service A ikutan macet (nunggu/timeout). Mereka Coupled (saling terikat erat).
- Event-Driven Architecture = Chat di Grup WhatsApp
  - Service A mengirim pesan ke Grup: "Guys, ada ORDER BARU nih (Event)!" Lalu Service A langsung lanjut kerja hal lain.
  - Service B (Gudang) baca pesan itu -> Potong stok.
  - Service C (Email) baca pesan itu -> Kirim invoice.
  - Kelebihan: Service A tidak peduli siapa yang baca, kapan dibaca, atau apakah Service B sedang mati. Yang penting dia sudah lapor. Mereka Decoupled (tidak terikat).
 
**2. Diagram Visual**
```mermaid
graph LR
    subgraph PRODUCER
        User((User)) -->|Checkout| OrderAPI[🛒 Order Service]
        OrderAPI -->|1. Create Order| DB1[(Order DB)]
    end

    subgraph EVENT_BUS
        OrderAPI -- 2. Publish Event:<br/>'OrderCreated' --> Broker{{Event Bus / Broker<br/>RabbitMQ / Kafka}}
    end

    subgraph CONSUMERS
        Broker -- 3. Push Event --> Inv[Inventory Service]
        Broker -- 3. Push Event --> Notif[Notification Service]
        Broker -- 3. Push Event --> Analytics[Analytics Service]
    end

    subgraph ACTIONS
        Inv -->|Update Stock| DB2[(Inventory DB)]
        Notif -->|Send Email| User
        Analytics -->|Update Dashboard| DB3[(Data Warehouse)]
    end
```

**3. Komponen Utama EDA**
1. **Event Producer** (Penerbit):
   - Komponen yang mendeteksi kejadian.
   - Contoh: Saat user klik "Bayar", Order Service menjadi Producer yang menerbitkan event `ORDER_PAID`.
2. **Event Router / Broker** (Perantara):
   - Infrastruktur "pipa" atau "jalur" tempat event mengalir.
   - Tools populer: RabbitMQ, Apache Kafka, AWS SNS/SQS, Google Pub/Sub.
3. **Event Consumer** (Penerima):
   - Komponen yang "mendengarkan" (subscribe) event tertentu dan bereaksi.
   - Contoh: Inventory Service mendengarkan event `ORDER_PAID` untuk mengurangi stok.
  
**4. Mengapa Event Driven Architecture ?**
1. **Decoupling** (Pemisahan Ketergantungan):
   - Jika `Notification Service` error/mati, `Order Service` TIDAK ikut error. User tetap bisa belanja. Emailnya tertunda saja (ada di antrean broker), nanti dikirim pas service nyala lagi.
2. **Scalability** (Skalabilitas):
   - Kalau traffic belanja naik gila-gilaan, kita bisa memperbanyak (scale up) server `Order Service` saja tanpa perlu memperbesar server `Analytics Service` saat itu juga.
3. **Extensibility** (Kemudahan Pengembangan):
   - Besok bos minta fitur baru: "Setiap ada order, kirim data ke tim Marketing."
   - Kita tinggal buat service baru yang dengar event `ORDER_CREATED`. Kita TIDAK PERLU mengedit kodingan `Order Service` sama sekali. Aman dari bug regresi.
  
**5. Tantangan**
1. **Complexity**: Melacak flow jadi susah. "Ini stok berkurang gara-gara event yang mana ya?" Debugging lebih sulit daripada sistem monolith biasa.
2. **Eventual Consistency**: Data tidak sinkron detik itu juga. User sudah bayar, tapi mungkin stok di gudang baru berkurang 2 detik kemudian. Aplikasi harus didesain untuk mentoleransi jeda ini.
