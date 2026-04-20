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


## Reflection Commit 3 - Validating Request and Selectively Responding

![Commit 3 screen capture](/assets/images/commit3.png)

```
fn handle_connection(mut stream: TcpStream) {
    let buf_reader = BufReader::new(&stream);
    let request_line = buf_reader.lines().next().unwrap().unwrap();

    if request_line == "GET / HTTP/1.1" {
        let status_line = "HTTP/1.1 200 OK";
        let contents = fs::read_to_string("hello.html").unwrap();
        let length = contents.len();

        let response = format!(
            "{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}"
        );

        stream.write_all(response.as_bytes()).unwrap();
    } else {
        let status_line = "HTTP/1.1 404 NOT FOUND";
        let contents = fs::read_to_string("404.html").unwrap();
        let length = contents.len();

        let response = format!(
            "{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}"
        );

        stream.write_all(response.as_bytes()).unwrap();
    }
}
```

Pada tahap ini, kita menambahkan fitur validasi request pada server agar bisa memberikan respons yang spesifik secara selektif. Sebelumnya, server kita selalu mengembalikan `hello.html` tanpa memedulikan halaman apa yang sebenarnya diminta oleh pengguna.

Untuk memberikan respons yang berbeda (halaman sukses atau halaman error), kita memeriksa baris pertama dari permintaan klien (`request_line`). Jika pengguna mengakses halaman utama, nilai `request_line` akan berupa `"GET / HTTP/1.1"`. Kita menggunakan blok logika if-else sederhana untuk memisahkan alur ini. Jika kondisinya cocok (mengakses `/`), kita menyiapkan baris status `"HTTP/1.1 200 OK"` dan memilih file `hello.html` untuk ditampilkan. Namun jika pengguna meminta jalur acak apa pun yang tidak terdaftar (misalnya mengakses `127.0.0.1:7878/bad`), maka server akan menganggap halaman tersebut tidak ada. Program akan beralih ke blok else, menyiapkan baris status `"HTTP/1.1 404 NOT FOUND"`, dan memilih file `404.html`.  

*Setelah Refactoring*
```
fn handle_connection(mut stream: TcpStream) {
    let buf_reader = BufReader::new(&mut stream);
    let request_line = buf_reader.lines().next().unwrap().unwrap();

    let (status_line, filename) = if request_line == "GET / HTTP/1.1" {
        ("HTTP/1.1 200 OK", "hello.html")
    } else {
        ("HTTP/1.1 404 NOT FOUND", "404.html")
    };
    
    let contents = fs::read_to_string(filename).unwrap();
    let length = contents.len();

    let response = format!(
        "{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}"
    );

    stream.write_all(response.as_bytes()).unwrap();
}
```

Ketika pertama kali mengimplementasikan logika if-else ini, kode untuk membaca file, menghitung panjang teks, menyatukan header, dan mengirimkan balasan ke stream ditulis sebanyak dua kali (satu di dalam if, satu lagi di dalam else). Pendekatan ini melanggar prinsip desain Don't Repeat Yourself (DRY).

Refactoring sangat krusial dilakukan untuk menghindari kode yang repetitif. Dengan melakukan refactoring, kita menyederhanakan blok if-else sehingga hanya bertugas untuk menentukan dua variabel utama, yaitu `status_line` dan `filename`. Setelah variabel tersebut ditentukan, proses membaca file, merakit format HTTP, dan menuliskannya ke TCP stream cukup dituliskan satu kali saja di luar blok kondisi. Hasilnya, kode menjadi jauh lebih bersih, ringkas, dan lebih mudah dikembangkan jika ke depannya kita ingin menambahkan routing ke banyak halaman baru.

## Reflection Commit 4 - Simulation Slow Response

Pada tahap ini, kita melakukan modifikasi untuk mensimulasikan skenario respons yang lambat (slow request) untuk melihat kelemahan utama dari arsitektur single-threaded web server.

```
fn handle_connection(mut stream: TcpStream) {
    let buf_reader = BufReader::new(&mut stream);
    let request_line = buf_reader.lines().next().unwrap().unwrap();

    let (status_line, filename) = match &request_line[..] {
        "GET / HTTP/1.1" => ("HTTP/1.1 200 OK", "hello.html"),
        "GET /sleep HTTP/1.1" => {
            thread::sleep(Duration::from_secs(10));
            ("HTTP/1.1 200 OK", "hello.html")
        }
        _ => ("HTTP/1.1 404 NOT FOUND", "404.html"),
    };

    let contents = fs::read_to_string(filename).unwrap();
    let length = contents.len();

    let response = format!(
        "{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}"
    );

    stream.write_all(response.as_bytes()).unwrap();
}
```

Logika routing yang sebelumnya menggunakan blok if-else kini diganti dengan match. Struktur match jauh lebih bersih dan efisien di Rust saat kita harus menangani banyak kondisi rute. Kemudian kita juga menambahkan rute `/sleep`. 

```
"GET /sleep HTTP/1.1" => {
    thread::sleep(Duration::from_secs(10));
    ("HTTP/1.1 200 OK", "hello.html")
}
```

Blok kode tersebut akan menambahkan rute baru (/sleep). Ketika rute ini diakses, kita memanggil `thread::sleep` untuk membekukan eksekusi program selama 10 detik sebelum akhirnya merakit balasan status 200 OK dan mengirimkan `hello.html`. Ini berfungsi sebagai simulasi saat server sedang mengerjakan tugas komputasi berat, mengunduh data besar, atau menunggu kueri database yang lambat.

Lalu blok kode `_ => ("HTTP/1.1 404 NOT FOUND", "404.html"),` akan bertindak sebagai penangkap default untuk semua rute lain yang tidak terdefinisi di atasnya, langsung mengembalikan respons 404 NOT FOUND.

Kemudian ketika kita menjalankan kode tersebut, kita mencoba untuk mengakses `127.0.0.1:7878/sleep` di Tab 1 dan mengakses `127.0.0.1:7878` di Tab 2 secara bersamaan. Saat dicoba, Tab 2 akan mengalami loading yang lama dan baru menampilkan halaman setelah Tab 1 selesai memuat.

Hal ini terjadi karena program server yang kita buat berjalan secara single-threaded, artinya ia hanya memiliki satu jalur eksekusi instruksi dari awal hingga akhir. Ketika Tab 1 mengakses `/sleep`, permintaan tersebut masuk ke fungsi `handle_connection`. Kemudian program akan mengeksekusi `thread::sleep(Duration::from_secs(10))`, yang menghentikan total seluruh eksekusi thread utama tersebut selama 10 detik. Kemudian pada saat yang hampir bersamaan, Tab 2 mengirimkan permintaan koneksi. Sistem operasi komputer menerima koneksi TCP dari Tab 2 dan menaruhnya di antrean (queue). Namun, karena thread utama aplikasi Rust kita sedang membeku memproses Tab 1, program tidak bisa melanjutkan iterasi pada loop `for stream in listener.incoming()` untuk mengambil dan memproses stream milik Tab 2. Akibatnya, permintaan Tab 2 sepenuhnya tertahan (blocked) hingga proses `sleep ` Tab 1 selesai, dokumen HTML dikirim ke Tab 1, dan fungsi `handle_connection` pertama selesai dieksekusi.

## Reflection Commit 5 - Multithreaded Server

Pada tahap ini, kita berhasil menyelesaikan masalah antrean lambat (seperti rute /sleep pada Commit 4) dengan mengubah server dari single-threaded menjadi multithreaded menggunakan arsitektur ThreadPool. Hal ini dilakukan dengan mengubah sedikit file `main.rs` dan menambahkan file `lib.rs`.

```
// main.rs
use hello::ThreadPool;

fn main() {
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();
    let pool = ThreadPool::new(4);

    for stream in listener.incoming() {
        let stream = stream.unwrap();

        pool.execute(|| {
            handle_connection(stream);
        });
    }
}
```

```
// lib.rs
use std::{
    sync::{mpsc, Arc, Mutex},
    thread::{self, JoinHandle},
};

pub struct ThreadPool {
    workers: Vec<Worker>,
    sender: mpsc::Sender<Job>,
}

type Job = Box<dyn FnOnce() + Send + 'static>;

impl ThreadPool {
    pub fn new(size: usize) -> ThreadPool {
        assert!(size > 0);

        let (sender, receiver) = mpsc::channel();
        let receiver = Arc::new(Mutex::new(receiver));

        let mut workers = Vec::with_capacity(size);

        for id in 0..size {
            workers.push(Worker::new(id, Arc::clone(&receiver)));
        }

        ThreadPool { workers, sender }
    }

    pub fn execute<F>(&self, f: F)
    where
        F: FnOnce() + Send + 'static,
    {
        let job = Box::new(f);

        self.sender.send(job).unwrap();
    }
}

struct Worker {
    id: usize,
    thread: thread::JoinHandle<()>,
}

impl Worker {
    fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Job>>>) -> Worker {
        let thread = thread::spawn(move || loop {
            let job = receiver.lock().unwrap().recv().unwrap();
            println!("Worker {id} got a job; executing.");
            job();
        });

        Worker { id, thread }
    }
}
```

Konsep utama dari ThreadPool adalah kita menyiapkan sekelompok "pekerja" (workers) di awal. Ketika ada permintaan masuk, server tidak mengerjakannya sendiri, melainkan melemparkan pekerjaan tersebut ke dalam antrean untuk dikerjakan oleh pekerja yang sedang menganggur.

Pada `src/main.rs`, potongan kode `let pool = ThreadPool::new(4);` membuat sebuah pool (kelompok) yang berisi 4 pekerja (thread). Kemudian setiap kali ada koneksi dari browser (stream), `pool.execute(|| { handle_connection(stream); });` akan membungkus fungsi `handle_connection` di dalam sebuah closure `|| { ... }`. Closure ini merupakan sebuah "paket tugas" yang diserahkan ke pool.

Lalu cara kerjanya dapat kita lihat pada `src/lib.rs`,

1. `type Job = Box<dyn FnOnce() + Send + 'static>;` mendefinisikan pekerjaan yang merupakan closure yang dikirim dari `main.rs`. Kita menggunakan Box untuk mengalokasikan memori secara dinamis, dan Send untuk memastikan tugas ini aman dikirim ke thread lain.

2. `let (sender, receiver) = mpsc::channel();`
Instruksi ini menginisialisasi saluran komunikasi Multiple Producer, Single Consumer (mpsc) yang berfungsi untuk mendistribusikan beban kerja dari ThreadPool selaku entitas pengirim kepada para pekerja selaku entitas penerima.

3. `let receiver = Arc::new(Mutex::new(receiver));`
Konstruksi Arc yang membungkus Mutex berfungsi untuk mendistribusikan hak akses saluran antrean tugas secara aman ke seluruh pekerja, sekaligus menjamin eksklusivitas agar hanya satu pekerja yang dapat mengambil tugas pada satu waktu tanpa memicu bentrokan data (data race).

4. `workers.push(Worker::new(id, Arc::clone(&receiver)));`
Proses ini menginisialisasi setiap pekerja dengan ID unik dan salinan referensi aman (Arc::clone) menuju saluran penerima, lalu mengumpulkannya ke dalam pool agar seluruh pekerja terhubung pada antrean tugas yang sama.

5. Fungsi `execute` pada `ThreadPool`
Saat ada tugas baru dari `main.rs`, fungsi ini akan mengambil tugas tersebut lalu mengeksekusi `self.sender.send(job).unwrap()`. Ini artinya, pool melempar tugas tersebut ke dalam saluran komunikasi agar ditangkap oleh pekerja.

6. Proses Pekerja di `Worker::new`
Di dalam pembuatan Worker, kita menjalankan thread baru dengan `thread::spawn`. Di dalamnya terdapat loop `{ ... }` (perulangan tanpa henti). `let job = receiver.lock().unwrap().recv().unwrap();` akan memerintahkan pekerja untuk mengunci saluran penerima secara eksklusif dan menunda eksekusinya (bersiaga) hingga sebuah tugas baru berhasil ditangkap untuk diproses. Lalu pada instruksi `job();`, pekerja mengeksekusi tugas tersebut. Setelah selesai, pekerja akan kembali ke awal loop dan bersiap menerima tugas berikutnya.

## Bonus Reflection  
Pada tahap ini, dilakukan peningkatan kualitas fungsi inisialisasi `ThreadPool` dengan mengganti metode `new` menjadi `build`. Perubahan ini secara mendasar mengubah cara program menangani potensi kegagalan saat alokasi sumber daya.

Fungsi new sebelumnya menggunakan instruksi `assert!(size > 0);`. Jika fungsi ini dipanggil dengan parameter `size` bernilai 0, makro `assert!` akan memicu panic, yang menyebabkan program terhenti secara mendadak dan paksa saat itu juga. Dalam arsitektur perangkat lunak yang robust, sebuah library atau komponen internal (worker pool) tidak seharusnya memiliki otoritas untuk mematikan keseluruhan program secara sepihak tanpa memberikan kesempatan bagi sistem utama untuk melakukan mitigasi.

Fungsi `build` menyelesaikan masalah tersebut dengan memanfaatkan tipe data `Result<ThreadPool, PoolCreationError>` sebagai nilai kembalian. Alih-alih melakukan panic secara internal, fungsi `build` mengevaluasi parameter `size`. Jika valid, ia mengembalikan objek pool yang dibungkus dalam varian `Ok()`. Jika tidak valid `(size == 0)`, ia mengembalikan varian `Err()` yang berisi struktur error kustom (`PoolCreationError`).