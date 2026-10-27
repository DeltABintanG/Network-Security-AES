# Key Distribution dengan KDC dan Komunikasi Terenkripsi AES

Implementasi skenario *key distribution* tiga entitas (**KDC**, **Alice**, **Bob**) dengan enkripsi simetris **AES-128** yang ditulis manual (tanpa library kriptografi). Setelah handshake lima langkah selesai, Alice dan Bob dapat saling berkirim pesan terenkripsi secara dua arah.

## Fitur

- AES-128 (ECB, padding PKCS#7) dan KDC diimplementasikan manual.
- Skenario key distribution lima langkah dengan session key, ticket, dan nonce.
- Tiga entitas berjalan sebagai program terpisah, dapat dijalankan di tiga IP berbeda.
- Chat dua arah: Alice dan Bob dapat bertindak sebagai pengirim maupun penerima.
- Key berformat ASCII, ciphertext ditampilkan dalam hexadecimal.
- Master key dapat diubah lewat menu tanpa menjalankan ulang program.

## Persyaratan

- Python 3.8 atau lebih baru
- Tidak ada dependensi eksternal (hanya modul bawaan: `socket`, `threading`, `hashlib`, `os`, `random`)

## Struktur File

| File | Fungsi |
|---|---|
| `aes.py` | Implementasi AES-128 manual (`ManualAES`): key expansion, SubBytes, ShiftRows, MixColumns, enkripsi/dekripsi blok, padding |
| `common.py` | Konstanta, format pesan, framing socket, pembuatan dan parsing pesan protokol, nonce, session key |
| `kdc.py` | Server KDC: menyimpan master key Alice dan Bob, menerbitkan session key dan ticket |
| `alice.py` | Entitas pemrakarsa: meminta session key ke KDC, mengautentikasi diri ke Bob, lalu chat |
| `bob.py` | Entitas penerima: menerima ticket dari Alice, memverifikasi nonce, lalu chat |

## Protokol

Notasi: `E(K, M)` berarti enkripsi `M` dengan kunci `K`. `Ka` dan `Kb` adalah master key Alice dan Bob yang dibagi dengan KDC, `Ks` adalah session key, `N1` dan `N2` adalah nonce.

```
1. Alice -> KDC : ID_A || ID_B || N1
2. KDC   -> Alice: E(Ka, [ Ks || ID_A || ID_B || N1 || E(Kb, [Ks || ID_A]) ])
3. Alice -> Bob  : E(Kb, [Ks || ID_A])                (ticket)
4. Bob   -> Alice: E(Ks, N2)
5. Alice -> Bob  : E(Ks, f(N2))                       (f(N) = N + 1)
6. Alice <-> Bob : E(Ks, pesan)                       (chat dua arah)
```

Alice memeriksa bahwa `N1` dan `ID_B` pada balasan KDC cocok dengan permintaannya. Bob memeriksa bahwa nilai yang dikirim Alice pada langkah 5 sama dengan `f(N2)`.

### Format Pesan

Setiap pesan dikirim dengan framing berikut:

```
[ TYPE : 1 byte ][ LEN : 4 byte, big-endian ][ PAYLOAD : LEN byte ]
```

| TYPE | Nama | Isi payload |
|---|---|---|
| `0x01` | `T_KDC_REQ` | `ID_A` (8 byte) + `ID_B` (8 byte) + `N1` (8 byte), plaintext |
| `0x02` | `T_KDC_RESP` | Ciphertext 80 byte |
| `0x03` | `T_TICKET` | Ticket, ciphertext 32 byte |
| `0x04` | `T_NONCE_B` | `E(Ks, N2)`, 16 byte |
| `0x05` | `T_NONCE_ACK` | `E(Ks, f(N2))`, 16 byte |
| `0x06` | `T_MSG` | Pesan chat terenkripsi (kelipatan 16 byte) |
| `0x7F` | `T_ERR` | Pesan error (ASCII) |

ID berukuran 8 byte (diisi spasi), nonce berupa 8 digit ASCII.

### Penanganan Key

Master key berupa string ASCII dengan panjang bebas. Key diturunkan menjadi 16 byte dengan mengambil 16 byte pertama dari SHA-256 string tersebut (`hashlib`). Session key dibangkitkan acak sepanjang 16 byte oleh KDC untuk setiap permintaan.

## Cara Menjalankan

Jalankan urutannya: **KDC**, lalu **Bob**, lalu **Alice**.

| Entitas | Port | Perintah |
|---|---|---|
| KDC | 5001 | `python kdc.py` |
| Bob | 6000 | `python bob.py` |
| Alice | (klien) | `python alice.py` |

### Satu perangkat (pengujian cepat)

```
python kdc.py      # Bind IP: Enter
python bob.py      # Bind IP: Enter, pilih 1
python alice.py    # IP KDC dan IP Bob: Enter, pilih 1
```

Untuk mensimulasikan tiga IP berbeda pada satu perangkat Windows, gunakan alamat loopback berbeda:

- KDC: bind `127.0.0.2`
- Bob: bind `127.0.0.3`
- Alice: IP KDC `127.0.0.2`, IP Bob `127.0.0.3`

### Tiga perangkat berbeda

1. Hubungkan ketiga perangkat ke jaringan yang sama dan catat IP masing-masing (`ipconfig` di Windows, `ip a` di Linux).
2. Jalankan `kdc.py` di perangkat A dan `bob.py` di perangkat B (bind IP: Enter).
3. Jalankan `alice.py` di perangkat C. Isi **IP KDC** dengan IP perangkat A dan **IP Bob** dengan IP perangkat B.
4. Izinkan port **5001** (KDC) dan **6000** (Bob) pada firewall perangkat terkait.

Contoh aturan firewall Windows (PowerShell administrator):

```
netsh advfirewall firewall add rule name="KDC" dir=in action=allow protocol=TCP localport=5001
netsh advfirewall firewall add rule name="Bob" dir=in action=allow protocol=TCP localport=6000
```

> Port 5000 sengaja tidak dipakai karena dicadangkan oleh Windows pada sebagian sistem.

## Penggunaan

### Menu KDC

| Pilihan | Fungsi |
|---|---|
| 1 | Ubah master key ALICE |
| 2 | Ubah master key BOB |
| 3 | Lihat entitas terdaftar |
| 0 | Keluar |

### Menu Alice dan Bob

| Pilihan | Fungsi |
|---|---|
| 1 | Alice: mulai handshake dan chat. Bob: menunggu koneksi Alice |
| 2 | Ubah master key |
| 0 | Keluar |

Master key awal: Alice `default_A`, Bob `default_B` (sama dengan yang terdaftar di KDC). Jika key diubah di salah satu pihak, ubah juga di pihak lainnya agar sama; jika berbeda, handshake akan gagal.

### Chat

Setelah handshake berhasil, ketik pesan lalu Enter. Setiap pesan yang dikirim dan diterima menampilkan ciphertext dalam hex beserta plaintext-nya. Ketik `/quit` atau `exit` untuk mengakhiri chat.

## Skenario Pengujian

| Skenario | Hasil yang diharapkan |
|---|---|
| Key KDC dan Alice/Bob sama | Handshake berhasil, chat berjalan dua arah |
| Master key Alice berbeda dengan KDC | Alice gagal mendekripsi balasan KDC |
| Master key Bob berbeda dengan KDC | Bob gagal mendekripsi ticket (`Dekripsi Ticket Gagal`) |
| Key diubah lewat menu lalu handshake diulang | Berhasil tanpa restart program |
| ID tidak terdaftar | KDC menolak (`ID tidak dikenal`) |

## Verifikasi dengan Wireshark

Filter yang dipakai:

```
tcp.port == 5001 || tcp.port == 6000
```

Klik kanan paket, pilih **Follow, TCP Stream**, lalu tampilan **Hex Dump**. Hasil yang diharapkan:

- Langkah 1 berisi `ID_A`, `ID_B`, dan `N1` sebagai teks (tidak dienkripsi sesuai skenario).
- Langkah 2 sampai chat berupa data acak yang sama dengan hex pada log terminal; isi pesan tidak terbaca.

Pada Windows, gunakan interface **Npcap Loopback Adapter** untuk lalu lintas `127.x.x.x`. Lalu lintas ke WSL tidak dapat ditangkap Wireshark Windows; gunakan `pktmon` atau jalankan Wireshark/`tcpdump` di dalam WSL.

## Batasan

- AES berjalan dalam mode ECB tanpa IV: blok plaintext yang sama menghasilkan ciphertext yang sama.
- Tidak ada pemeriksaan integritas (MAC), timestamp, atau perlindungan terhadap serangan replay.
- Penurunan key dari master key memakai `hashlib.sha256` bawaan Python; hanya AES dan protokol KDC yang ditulis manual.
- Program ditujukan untuk keperluan pembelajaran, bukan untuk penggunaan produksi.
