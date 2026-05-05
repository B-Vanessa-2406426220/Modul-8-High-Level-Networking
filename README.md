## Reflection

> What are the key differences between unary, server streaming, and bi-directional streaming RPC (Remote Procedure Call) methods, and in what scenarios would each be most suitable?

**Unary** adalah yang paling sederhana, di mana client mengirim 1 request dan menunggu satu response dari server, persis seperti cara kerja API REST pada umumnya. Model ini cocok untuk tindakan transaksional atau operasi sederahan seperti login, proses pembayaran, atau fetch data tunggal. Sementara itu, **Server Streaming** memungkinkan server untuk membalas satu request dari client dengan banyak response. Ini sangat berguna ketika kita perlu mengirimkan data dalam jumlah besar atau pembaruan sistem secara real-time, seperti mengunduh file besar atau memantau log sistem. Di sisi lain, **Bi-directional Streaming** adalah model paling canggih di mana client dan server dapat saling mengirim pesan secara bersamaan kapan saja tanpa harus menunggu satu sama lain. Fleksibilitas ini menjadikannya pilihan utama untuk aplikasi yang interaktif (komunikasi dua arah yang intensif dan real-time) seperti chatting, game multiplayer, atau kolaborasi dokumen.

> What are the potential security considerations involved in implementing a gRPC service in Rust, particularly regarding authentication, authorization, and data encryption?

Aspek keamanan dalam gRPC mencakup tiga pilar utama: autentikasi, otorisasi, dan enkripsi. Untuk **autentikasi**, standar industri yang digunakan adalah TLS atau mTLS untuk memverifikasi identitas client dan server sekaligus mengenkripsi koneksi, mengamankan jalur komunikasi. Selain itu, penggunaan JWT (JSON Web Token) yang disisipkan dalam metadata gRPC sangat umum digunakan untuk mengelola sesi pengguna. Di Rust, library tonic mempermudah hal ini melalui fitur interceptor yang bertindak sebagai penjaga gerbang, memeriksa token sebelum permintaan diproses oleh logika utama.

Setelah identitas terverifikasi, **otorisasi** memastikan bahwa pengguna tersebut memang memiliki hak akses (role) untuk memanggil fungsi tertentu. Praktik terbaiknya adalah memusatkan logika ini pada satu lapisan middleware agar kode tetap bersih dan mudah dikelola (menghindari logika otorisasi yang tersebar). Terakhir, mengenai **enkripsi data**, gRPC secara default menggunakan HTTP/2 yang mendukung enkripsi kuat. Namun, untuk data yang sangat sensitif seperti informasi kartu kredit, disarankan untuk melakukan enkripsi tambahan di tingkat aplikasi sebelum data tersebut dikirimkan.

> What are the potential challenges or issues that may arise when handling bidirectional streaming in Rust gRPC, especially in scenarios like chat applications? 

Mengimplementasikan aliran data dua arah di Rust membawa tantangan tersendiri, terutama karena aturan ketat mengenai Manajemen Konkurensi. Karena Rust sangat peduli dengan kepemilikan data (ownership), berbagi status (seperti daftar pengguna aktif dalam aplikasi chat) di antara banyak tugas asinkron memerlukan alat bantu seperti Arc dan Mutex, yang jika tidak dirancang dengan baik, dapat menyebabkan kebuntuan sistem (deadlock).

Selain itu, Penanganan error di tengah-tengah aliran data menjadi krusial. Jika salah satu pihak terputus, server harus secara eksplisit melakukan pembersihan sumber daya agar tidak terjadi kebocoran memori karena tidak ada garbage collector. Masalah Backpressure juga sering muncul, di mana client mengirim data lebih cepat daripada yang bisa diproses server. Hal ini biasanya diatasi dengan membatasi kapasitas antrian (buffer) pada saluran komunikasi agar sistem tidak tumbang karena beban berlebih.

> What are the advantages and disadvantages of using the tokio_stream::wrappers::ReceiverStream for streaming responses in Rust gRPC services?

Penggunaan tokio_stream::wrappers::ReceiverStream dalam layanan gRPC berbasis Rust menawarkan keunggulan utama berupa integrasi yang sangat mudah dengan ekosistem Tokio, di mana pengembang cukup membungkus mpsc::Receiver untuk mengubahnya menjadi sebuah Stream yang kompatibel dengan gRPC. Pendekatan ini mendukung prinsip decoupling, yang memisahkan logika bisnis (sisi pengirim data) dengan logika streaming gRPC, serta memungkinkan skenario multi-producer di mana banyak tugas dapat mengirim pesan ke satu saluran yang sama secara bersamaan. Selain itu, fitur backpressure sudah tersedia secara bawaan melalui bounded channel, yang secara otomatis membatasi laju pengiriman data agar tidak membebani sistem.

Namun, terdapat beberapa kekurangan yang perlu dipertimbangkan, seperti adanya overhead tambahan akibat alokasi memori dan context switching pada saluran jika dibandingkan dengan metode stream langsung. Pengembang juga dihadapkan pada dilema antara menggunakan bounded channel yang dapat menyebabkan pemblokiran atau unbounded channel yang berisiko memicu masalah kekurangan memori (Out of Memory). Dari sisi teknis, penyebaran error melalui saluran ini cukup terbatas karena ReceiverStream hanya mengirim nilai, sehingga error harus dibungkus secara manual ke dalam format Result<T, Status>. Terakhir, jika pihak pengirim terputus atau drop sebelum waktunya, stream akan berakhir begitu saja tanpa memberikan sinyal error yang jelas kepada sisi client.

> In what ways could the Rust gRPC code be structured to facilitate code reuse and modularity, promoting maintainability and extensibility over time? 

Untuk membangun sistem yang mudah dikembangkan dalam jangka panjang, struktur kode harus dirancang secara modular. Sangat direkomendasikan untuk memisahkan logika bisnis utama dari implementasi teknis gRPC dengan menggunakan Traits. Dengan cara ini, kita bisa menguji logika aplikasi tanpa harus menjalankan server gRPC yang sebenarnya.

Selain itu, pembagian file yang jelas antara definisi kontrak (.proto), logika server, dan logika client akan sangat membantu tim dalam memahami alur program. Prinsip Dependency Injection juga perlu diterapkan, di mana koneksi database atau konfigurasi sistem dimasukkan ke dalam modul service alih-alih ditulis secara kaku di dalamnya.

> In the MyPaymentService implementation, what additional steps might be necessary to handle more complex payment processing logic? 

1. Validasi Input:  Cek kelengkapan data (nomor kartu valid, amount > 0, currency didukung).
2. Idempotency Key: Mencegah double charge jika request dikirim ulang (misal saat timeout).
3. Integrasi Payment Gateway: Panggil API eksternal secara async.
4. Database Transaction: Simpan status pembayaran dengan ACID transaction untuk konsistensi data.
5. Retry & Timeout: Menangani kegagalan sementara dari payment ketiga (seperti bank atau gateway pembayaran) dengan mekanisme Retry otomatis menggunakan jeda waktu yang semakin meningkat (exponential backoff).
6. Audit Log: Mencatat setiap perubahan status untuk keperluan pelacakan dan debugging.
7. Notifikasi: Kirim notifikasi ke service lain (email, notif push) setelah pembayaran selesai via event/message queue.

> What impact does the adoption of gRPC as a communication protocol have on the overall architecture and design of distributed systems, particularly in terms of interoperability with other technologies and platforms? 

Adopsi gRPC membawa dampak signifikan terhadap desain arsitektur sistem, terutama melalui pendekatan contract-first development di mana file .proto berfungsi sebagai kontrak yang jelas antar layanan untuk mengurangi miskomunikasi tim. Keamanan data juga lebih terjamin karena sifatnya yang strongly typed, sehingga kesalahan skema dapat dideteksi sejak compile time. Dari sisi performa, kombinasi binary encoding dan multiplexing pada HTTP/2 secara drastis mengurangi latensi serta penggunaan bandwidth. Selain itu, dukungan polyglot memungkinkan integrasi antar berbagai bahasa pemrograman seperti Go, Python, Java, dan Rust secara otomatis melalui pembuatan kode dari skema yang sama.

Namun, terdapat tantangan interoperabilitas yang perlu diperhatikan, seperti belum adanya dukungan native dari peramban web sehingga memerlukan proxy tambahan seperti Envoy atau gRPC-Web untuk kebutuhan frontend. Proses debugging juga menjadi lebih menantang karena format biner tidak dapat dibaca langsung oleh manusia seperti JSON, sehingga membutuhkan alat bantu khusus seperti grpcurl. Di tengah dominasi ekosistem REST, integrasi dengan API pihak ketiga sering kali memerlukan adapter atau gateway tambahan. Terakhir, tim pengembang harus menghadapi learning curve untuk memahami konsep Protobuf dan mekanisme streaming yang berbeda dari model REST konvensional.

> What are the advantages and disadvantages of using HTTP/2, the underlying protocol for gRPC, compared to HTTP/1.1 or HTTP/1.1 with WebSocket for REST APIs?

### Kelebihan
- Multiplexing: HTTP/2 memungkinkan pengiriman banyak pesan secara bersamaan dalam satu koneksi TCP tunggal tanpa harus menunggu antrean. Hal ini memecahkan masalah head-of-line blocking yang sering menghambat performa pada HTTP/1.1.

- Header Compression (HPACK): Berbeda dengan HTTP/1.1 yang mengirimkan header dalam bentuk teks mentah yang redundan, HTTP/2 menggunakan kompresi HPACK. Ini secara signifikan mengurangi beban data yang dikirim, membuat komunikasi antar-layanan menjadi jauh lebih ringan dan cepat.

- Binary Framing (Kecepatan Pemrosesan): Pesan dalam HTTP/2 diubah menjadi format biner yang lebih efisien untuk diproses oleh mesin. Penggunaan format biner ini (bersama Protobuf) adalah alasan mengapa gRPC jauh lebih berperforma dibanding REST API yang masih berbasis teks (JSON/XML).

- Full-Duplex Streaming (Real-Time Native): Protokol ini mendukung aliran data dua arah secara native. Server bisa mendorong data ke client (server push) tanpa menunggu permintaan, memberikan keunggulan responsivitas dibandingkan WebSocket yang harus melakukan upgrade koneksi secara manual.

- Flow Control & Prioritization: HTTP/2 memiliki kontrol aliran yang memungkinkan client mengatur prioritas stream. Data penting dapat didahulukan agar tidak tertutup oleh data berukuran besar, memastikan kelancaran transaksi yang bersifat kritis.

### Kekurangan
- Browser Compatibility (Keterbatasan Akses): Meskipun efisien, fitur framing biner HTTP/2 belum didukung penuh secara native oleh browser, sehingga memerlukan proxy tambahan seperti Envoy atau gRPC-Web untuk sisi frontend.

- Debugging Complexity: Karena format datanya biner, proses pencarian kesalahan tidak semudah HTTP/1.1. Format data biner tidak dapat dibaca langsung lewat Network Tab browser, sehingga membutuhkan alat khusus seperti grpcurl atau Postman untuk inspeksi data.

- Learning Curve: Implementasi HTTP/2 melalui gRPC menuntut pemahaman lebih dalam mengenai konsep asinkron, streaming, dan pengelolaan skema data (Protobuf), yang jauh lebih kompleks dibandingkan model request-response sederhana pada REST.


HTTP/2 unggul dalam performa dan efisiensi, tapi HTTP/1.1 masih relevan untuk kesederhanaan dan kompatibilitas browser. WebSocket cocok untuk real-time ringan tanpa perlu schema ketat.

> How does the request-response model of REST APIs contrast with the bidirectional streaming capabilities of gRPC in terms of real-time communication and responsiveness? 

REST API menggunakan model Request-Response di mana client selalu menjadi pihak yang memulai pembicaraan. Seperti mengirim SMS, di mana kita harus mengirim pesan terlebih dahulu untuk mendapatkan balasan. Untuk aplikasi real-time, REST sering kali terpaksa menggunakan teknik polling yang tidak efisien. Setiap request membuka koneksi baru yang menyebabkan overhead tinggi, namun REST lebih sederhana dan mudah dipahami.

Sebaliknya, gRPC Bidirectional Streaming lebih menyerupai panggilan telepon. Setelah koneksi tersambung, kedua belah pihak (server dan client) bisa berbicara saling kirim kapan saja tanpa perlu menunggu giliran. Koneksi dibuka sekali, data mengalir terus menyebabkan latensi yang dihasilkan sangat rendah, menjadikannya senjata utama untuk chat, live tracking, game real-time, atau alat kolaborasi dokumen yang membutuhkan respons seketika. Memiliki kompleksitas yang lebih tinggi dalam implementasi dan error handling.

> What are the implications of the schema-based approach of gRPC, using Protocol Buffers, compared to the more flexible, schema-less nature of JSON in REST API payloads?

Penggunaan Protocol Buffers (Protobuf) dalam gRPC unggul di sisi performa dan kontrak data. Protobuf mampu mendeteksi kesalahan tipe data secara otomatis saat proses kompilasi, sehingga meminimalkan bug saat aplikasi berjalan. Data yang dikirimkan pun jauh lebih efisien karena menggunakan pengkodean biner yang ukurannya 3 hingga 10 kali lebih kecil dan lebih cepat diproses dibandingkan JSON. Selain itu, file .proto berfungsi sebagai dokumentasi otomatis yang mendukung kompatibilitas jangka panjang (backward/forward compatibility), memungkinkan perubahan skema tanpa merusak sistem yang sudah ada. Namun, pendekatan ini memiliki keterbatasan karena datanya tidak dapat dibaca langsung oleh manusia sehingga debugging menjadi lebih sulit, membutuhkan alat tambahan seperti protoc, serta kurang fleksibel untuk struktur data yang sangat dinamis.

Di sisi lain, JSON dalam REST API memiliki fleksibilitas tinggi dan kemudahan penggunaan yang bersifat schema-less. Format teksnya yang human-readable membuat proses pencarian kesalahan melalui browser atau perintah curl menjadi sederhana, ditambah dengan universal support di hampir semua platform dan bahasa pemrograman. Meskipun sangat fleksibel untuk mengirim struktur data yang belum terdefinisi, JSON memiliki kelemahan karena tidak memiliki penegakan skema secara bawaan, yang sering kali memicu inkonsistensi data antar layanan. Selain itu, penggunaan format teks membuat JSON lebih boros bandwidth dan memiliki performa pemrosesan yang lebih lambat dibandingkan gRPC, terutama saat menangani volume data yang besar dalam sistem terdistribusi.