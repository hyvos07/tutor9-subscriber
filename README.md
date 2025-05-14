# 🦀 Advanced Programming - Event-Driven Programming

**Nama**  : Daniel Liman <br>
**NPM**   : 2306220753 <br>
**Kelas** : Pemrograman Lanjut A


## Reflection

> What is AMQP?

**AMQP** atau **Advanced Message Queuing Protocol** adalah messaging protocol standar untuk message-oriented middleware (MOM) yang dirancang agar operable di berbagai platform dan bahasa pemrograman. Dengan AMQP, aplikasi dapat saling bertukar pesan secara aman dan terstruktur, tanpa harus mengetahui detail implementasi satu sama lain.

Secara garis besar, AMQP bekerja dengan konsep berikut:

1. **Producer**: Pengirim pesan yang mengirimkan data ke broker.
2. **Broker (Server)**: Perantara yang menampung, mengelola, dan mendistribusikan pesan sesuai aturan (routing) yang telah ditetapkan.
3. **Consumer**: Penerima pesan yang mengambil dan memproses pesan dari broker.

<br>

> What does `guest:guest@localhost:5672` mean? What is the first `guest`, the second `guest`, and `localhost:5672` is for?

`guest:guest@localhost:5672` adalah URI connection string untuk AMQP yang kita pakai.

1. `guest` pertama: Nama pengguna atau username untuk autentikasi dengan broker dari AMQP. Karena kita menggunakan RabbitMQ, default username yang disediakan adalah `guest`.

2. `guest` kedua: Password dari pengguna tadi. Default password dari akun `guest` di RabbitMQ adalah `guest` juga.

3. `localhost`: Ini adalah tempat atau tujuan URI dimana broker AMQP berjalan, yang dimana pada kasus ini adalah di `localhost` atau mesin lokal kita.

4. `5672`: Ini adalah default port yang digunakan oleh AMQP untuk koneksi yang tidak memakai TLS connections. Jika nantinya diperlukan koneksi yang memakai TLS atau SSL, maka harus memakai port `5671`.

Kesimpulannya, `guest:guest@localhost:5672` adalah format URI yang akan dipakai untuk melakukan koneksi pada broker AMQP dari RabbitMQ yang berjalan pada `localhost` di port `5672`, dengan username dan password `guest`. Format ini mirip dengan bentuk URI lainnya yang biasa ditemukan di beberapa aplikasi lainnya, misalnya database.

## Simulation Slow Subscriber

<picture>
    <img src="img/slow.png">
</picture>

*"Why the total number of queue is as such?"*

Pada gambar di atas, dapat dilihat bahwa dengan ditambahkannya `thread::sleep(ten_millis);` pada proses yang ada di subscriber, ia akan memproses event yang diterima nya dengan sedikit lebih lambat. Hal ini pula yang membuat tidak semua message yang disalurkan lewat RabbitMQ dapat langsung diproses semuanya dalam waktu yang singkat. Antisipasi kasus ini sudah dicover dengan pembuatan **queue** untuk message-message berisi event yang belum dapat diproses pada RabbitMQ. Queue ini di-manage dengan bantuan library rust `CrosstownBus`, yang ada di sisi publisher maupun subscriber.

Dengan adanya queue ini, walaupun subscriber harus memproses event lebih lama dari waktu yang diperlukan untuk mengirim event yang baru, publisher dapat mengirim event baru tanpa harus menunggu proses yang ada di subscriber selesai terlebih dahulu. Queue pada di gambar yang ada di atas mencapai 30 messages yang menunggu di queue untuk dikirim dan diproses, karena saya menjalankan publisher berkali-kali tanpa menunggu proses di subscriber selesai.

## Running at Least Three Subscribers

<picture>
    <img src="img/four.png">
</picture>
<picture>
    <img src="img/multi.png">
</picture>

Setelah saya mencoba untuk memakai 4 consumer/subscriber untuk memproses event yang dikirim oleh publisher dengan rate yang hampir mirip, dapat dilihat pada gambar dashboard RabbitMQ di atas bahwa tidak terjadi lagi penumpukan event pada queue yang menunggu untuk diproses. Hal ini terjadi karena RabbitMQ secara otomatis mendistribusikan message-message event dari queue tersebut kepada subscriber-subscriber yang aktif. Dengan begitu, setiap subscriber secara bergantian mengambil message yang ada di queue dan memprosesnya secara individual, tanpa terganggu oleh proses yang terjadi di subscriber yang lain. Karena hal tersebut, tidak terjadi penumpukan di queue dan semua messages yang masuk dapat diproses lebih cepat.

Fenomena ini menunjukkan kelebihan dalam kemampuan skalabilitas horizontal dari sistem queue milik RabbitMQ, di mana kita dapat mereduksi load yang dikerjakan oleh aplikasi dengan menambah lebih banyak subscriber, sehingga meningkatkan throughput dan mengurangi waktu dalam pemrosesan message. Kelebihan ini dapat mengurangi pengubahan source code yang ada untuk meningkatkan performa dari aplikasi yang kita miliki.

### Final Reflection

Pada kode yang sudah kita miliki sekarang, pada subscriber masih membuka koneksi ke RabbitMQ message secara satu persatu tanpa adanya error handling yang cukup aman. Bagian kode seperti di bawah bisa ditambahkan agar lebih aman:

```rs
fn main() {
    let listener =
        CrosstownBus::new_queue_listener("amqp://guest:guest@localhost:5672".to_owned()).unwrap();
    
    match listener.listen(
        "user_created".to_owned(),
        UserCreatedHandler {},
        crosstown_bus::QueueProperties {
            auto_delete: false,
            durable: false,
            use_dead_letter: true,
        },
    ) {
        Ok(_) => {
            println!("Subscriber started successfully");
            loop {}
        }
        Err(e) => {
            eprintln!("Failed to start subscriber: {}", e);
            std::process::exit(1);
        }
    }
}
```

Selain itu, bisa juga diimplementasikan connection pool di publisher untuk mengirim messages ke RabbitMQ lebih aman tanpa harus khawatir jika ada proses yang gagal di salah satu koneksi yang dibuat. Perbaikan ini juga cocok apabila nantinya publisher akan memakai multi-threading.

```rs
fn main() {
    let mut pool = PublisherPool::new("amqp://guest:guest@localhost:5672".to_owned(), 3)
        .expect("Failed to create publisher pool"); // Buat 3 publisher connection

    // ... dst
}
```