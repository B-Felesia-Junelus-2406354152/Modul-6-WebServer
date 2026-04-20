## Reflection Commit 1 - Handle-connection, check response

Pembangunan server single-threaded pada tahap ini memberikan wawasan mendasar tentang bagaimana aplikasi web berinteraksi pada lapisan jaringan. Kode ini tidak lagi sekadar mencatat adanya koneksi, melainkan secara aktif mengekstrak dan membaca muatan data dari koneksi tersebut.

Pada fungsi main, program menginisialisasi server dengan `TcpListener::bind("127.0.0.1:7878")`. Hal ini memerintahkan program untuk binding server pada antarmuka localhost (`127.0.0.1`) di port `7878`. Melalui loop `for stream in listener.incoming()`, server akan terus berjalan (memblokir penyelesaian `main`) untuk menerima setiap koneksi TCP yang masuk. Setiap koneksi (``stream`) yang berhasil diekstrak dengan `.unwrap()` kemudian diteruskan ke fungsi `handle_connection`. Pendekatan ini merupakan penerapan praktik clean code sederhana dengan memisahkan logika penerimaan koneksi dari logika pemrosesan data.

Fungsi `handle_connection(mut stream: TcpStream)` bertanggung jawab penuh untuk mengurai data yang dikirim oleh klien (dalam hal ini, browser). Parameter stream dideklarasikan sebagai `mut` (mutable) karena proses membaca dari stream akan mengubah status (posisi baca) dari objek tersebut.

Proses pembacaan menggunakan library Input/Output standar (`std::io`).

1. `let buf_reader = BufReader::new(&mut stream);`: Mengakses data secara langsung byte-demi-byte dari jaringan tidaklah efisien. `BufReader` bertindak sebagai lapisan buffer, mengambil potongan data yang lebih besar ke dalam memori, sehingga memungkinkan program membacanya dengan metode tingkat tinggi, seperti baris demi baris.

2. Ekstraksi Request: Konstruksi `http_request: Vec<_>` mendemonstrasikan kekuatan metode fungsional dalam ekosistem Rust:
 - `.lines()`: Mengubah aliran byte dari `BufReader` menjadi iterator yang menghasilkan string, dipisahkan oleh karakter newline.

 - `.map(|result| result.unwrap())`: Digunakan untuk mengekstraks nilai String dari setiap Result yang dihasilkan.

 - `.take_while(|line| !line.is_empty())`: Digunakan untuk menginstruksikan iterator untuk berhenti mengambil data begitu menemui baris kosong pertama.

 - `.collect()`: Mengumpulkan elemen-elemen dari iterator ke dalam vektor (Vec), menghasilkan sekumpulan string yang merepresentasikan keseluruhan request header.

Setelah dijalankan, dihasilkan output sebagai berikut dalam terminal saya.  
```
Request: [
    "GET / HTTP/1.1",
    "Host: 127.0.0.1:7878",
    "Connection: keep-alive",
    "Cache-Control: max-age=0",
    "sec-ch-ua: \"Google Chrome\";v=\"147\", \"Not.A/Brand\";v=\"8\", \"Chromium\";v=\"147\"",
    "sec-ch-ua-mobile: ?0",
    "sec-ch-ua-platform: \"Windows\"",
    "Upgrade-Insecure-Requests: 1",
    "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36",      
    "Accept: text/html,application/xhtml+xml,application/xml;q=0.9,imag    "Sec-Fetch-Dest: doc    "Sec-Fetch-Dest: document",
    "Accept-Encoding: gzip, deflate, br, zstd",
    "Accept-Language: en-US,en;q=0.9",
    "Cookie: csrftoken=vRGcEQOB4B8iTWc8gcKzkWDZ0hRP43FX",
]
```

Output yang dihasilkan di terminal menunjukkan anatomi lengkap dari HTTP request. Baris pertama, `GET / HTTP/1.1`, adalah perintah instruksi utamanya yang meminta server untuk memuat (GET) halaman root atau beranda (/) menggunakan HTTP versi 1.1. Seluruh baris di bawahnya adalah HTTP Headers yang bertindak sebagai metadata teknis untuk memberikan konteks kepada server, seperti menentukan alamat server yang dituju (`Host`), meminta jaringan tidak langsung diputus setelah merespons (`Connection`), dan memaksa pengambilan data terbaru tanpa cache (`Cache-Control`). Headers ini juga menginformasikan profil spesifik dari klien yang digunakan (Chrome versi 147 di OS Windows via `sec-ch-ua` dan `User-Agent`), mendeklarasikan tipe file dan dokumen pendukung keamanan yang bisa diproses (`Accept`, `Sec-Fetch-*`), menyebutkan algoritma kompresi serta bahasa yang didukung (`Accept-Encoding`, `Accept-Language`), dan menyertakan data sesi yang tersimpan di memori browser berupa token keamanan (`Cookie`).

