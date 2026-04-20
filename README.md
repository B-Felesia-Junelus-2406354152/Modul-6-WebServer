## Reflection Commit 1 - Handle-connection, check response

Pembangunan server single-threaded pada tahap ini memberikan wawasan mendasar tentang bagaimana aplikasi web berinteraksi pada lapisan jaringan. Kode ini tidak lagi sekadar mencatat adanya koneksi, melainkan secara aktif mengekstrak dan membaca muatan data dari koneksi tersebut.

Kode yang digunakan:
```
use std::{
    io::{prelude::*, BufReader},
    net::{TcpListener, TcpStream},
};
fn main() {
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();
    for stream in listener.incoming() {
        let stream = stream.unwrap();
        handle_connection(stream);
    }
}

fn handle_connection(mut stream: TcpStream) {
    let buf_reader = BufReader::new(&mut stream);
    let http_request: Vec<_> = buf_reader
        .lines()
        .map(|result| result.unwrap())
        .take_while(|line| !line.is_empty())
        .collect();
    println!("Request: {:#?}", http_request);
}

```

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

## Reflection Commit 2 - Returning HTML
Pada tahap ini, fungsi handle_connection dimodifikasi agar server tidak hanya membaca data masuk, tetapi juga membalasnya dengan mengirimkan sebuah file HTML. Pembaruan kode berfokus pada cara merakit struktur pesan balasan (HTTP Response) yang sesuai dengan standar protokol web.

```
fn handle_connection(mut stream: TcpStream) {
    let buf_reader = BufReader::new(&mut stream);
    let http_request: Vec<_> = buf_reader
        .lines()
        .map(|result| result.unwrap())
        .take_while(|line| !line.is_empty())
        .collect();

    let status_line = "HTTP/1.1 200 OK";
    let contents = fs::read_to_string("hello.html").unwrap();
    let length = contents.len();

    let response =
        format!("{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}");
        
    stream.write_all(response.as_bytes()).unwrap();
}
```

1. `let status_line = "HTTP/1.1 200 OK";` Baris ini memberi tahu browser bahwa permintaan telah berhasil diproses (200 OK) dan dikirim menggunakan protokol HTTP/1.1.

2. `let contents = fs::read_to_string("hello.html").unwrap();` Baris ini menggunakan library file system (fs) dari Rust untuk membuka dan membaca seluruh teks yang ada di dalam file `hello.html`. Hasil pembacaan tersebut disimpan ke dalam variabel `contents`. Pemanggilan `.unwrap()` digunakan untuk memastikan bahwa jika file gagal dibaca, program akan langsung berhenti.

3. `let length = contents.len();` Berfungsi untuk menghitung panjang atau ukuran dari teks HTML tersebut. Nilai ini dibutuhkan untuk mengisi header Content-Length. Header ini penting agar browser tahu persis seberapa besar data dokumen yang harus ia unduh.

4. `let response = format!("{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}");`  Fungsi `format!` digunakan untuk merangkai semua komponen menjadi satu teks pesan HTTP yang utuh. Formatnya sangat ketat, dimulai dengan status line, diikuti oleh header (Content-Length), lalu pemisah wajib berupa dua baris kosong (\r\n\r\n), dan terakhir diisi oleh isi dokumen HTML itu sendiri (contents).

5. `stream.write_all(response.as_bytes()).unwrap();`
Setelah pesan respons selesai dirakit, teks tersebut diubah menjadi format byte (`.as_bytes()`) dan dikirimkan kembali ke browser melalui jalur koneksi TCP (`stream.write_all`).

![Commit 2 screen capture](/assets/images/commit2.png)

Kemudian output yang dihasikan oleh terminal adalah sebagai berikut.  
```
PS D:\Kuliah\projek\modul6\hello> cargo run
   Compiling hello v0.1.0 (D:\Kuliah\projek\modul6\hello)
warning: unused variable: `http_request`
   |
   |
17 |     let http_request: Vec<_> = buf_reader
   |         ^^^^^^^^^^^^ help: if this is intentional, prefix it with an underscore: `_http_request`
   |
   = note: `#[warn(unused_variables)]` (part of `#[warn(unused)]`) on by default

warning: `hello` (bin "hello") generated 1 warning (run `cargo fix --bin "hello" -p hello` to apply 1 suggestion)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.55s     Running `target\debug\hello.exe`

```

Output terminal tersebut menunjukkan bahwa program berhasil dikompilasi dan berjalan dengan normal (terlihat dari pesan `Finished` dan `Running`), namun kompilator Rust memberikan sebuah pesan peringatan (warning). Peringatan bertuliskan `"unused variable: http_request"` ini muncul karena kita sudah membuat dan menyimpan data ke dalam variabel tersebut, tetapi kita tidak pernah memanggil atau menggunakannya lagi di baris kode selanjutnya.

