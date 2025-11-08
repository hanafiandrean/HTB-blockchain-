Write‑Up— NotADemocraticElection (CTF Blockchain)
Tujuan dokumen: menjelaskan dari nol sampai dapat flag. 
1) Gambaran Singkat (Apa yang kita lakukan?)
Ini challenge smart contract bertema ‘kekuatan uang menentukan pemenang’. Ada dua pihak: ALF dan CIM. Tugas kita: membuat kandidat ‘CIM’ jadi pemenang (bytes3 = 0x43494d), sesuai checker di kontrak Setup. Setelah status solved = true, kita klaim flag via menu TCP terpisah.
•	Kontrak target diakses lewat RPC HTTP (JSON‑RPC).
•	Flag diambil lewat service TCP (menu) — bukan lewat RPC.
•	Agar bisa deposit/vote, kita butuh saldo (ETH) untuk biaya + nilai deposit.
Real‑case Analogy (kenapa ini penting?)
Bayangkan pemilu yang sebenarnya: kalau sistemnya cacat (siapa bayar paling banyak menang), demokrasi rusak. Di dunia nyata, ini mirip vote‑buying / pay‑to‑play di governance token DeFi. Challenge ini melatih kita memahami alur interaksi on‑chain (funding, call fungsi, event log) dan integrasi off‑chain service (menu TCP) yang melakukan verifikasi sebelum mengeluarkan flag.
2) Endpoint & Akun yang Terlibat
RPC (HTTP, JSON‑RPC): http://83.136.255.235:36473
Menu TCP (nc): 83.136.255.235:42915
Menu TCP memberikan informasi penting (‘Connection information’) seperti private key solver, address solver, alamat Setup & Target khusus session‑mu. Jadi, selalu ambil data terbaru dari opsi (1).
printf "1\r\n" | nc -v 83.136.255.235 42915
Perintah di atas memilih menu ‘1’ untuk menampilkan info koneksi. Output yang contoh kita lihat:
Private key     :  0x67f9d8dc31675e...a19dfd
Address         :  0x85259126F1862083d02FC8A1853fafe44Da6B1A0
Target contract :  0x70514fa75836f0544F9A777913799Bd43DB70448
Setup contract  :  0x2573F4b37129cF205E46BA05A2E93aF461bb0AAe
Arti data:
•	Private key + Address: akun ‘solver’ yang dipakai checker back‑end untuk validasi saat klaim flag.
•	Target contract: alamat NotADemocraticElection yang harus kita interaksi (deposit + vote).
•	Setup contract: kontrak checker; solved jika winner() == 0x43494d (‘CIM’).
3) Persiapan CLI (Foundry + netcat)
•	Pastikan 'forge' dan 'cast' ada di PATH.
export RPC_URL=http://83.136.255.235:36473
export PK_SOLVER=0x67f9d8dc31675ec75197049d73ac90d9058b11c664e4f961d6b11bfcd0a19dfd
export SOLVER=$(cast wallet address --private-key "$PK_SOLVER")
Ini menyetel variabel lingkungan agar semua perintah ‘cast’ mengarah ke RPC yang benar dan memakai key solver yang diberikan service.
4) Danai Akun Solver (supaya bisa Deposit + Gas)
Biasanya RPC menyediakan akun ‘funder’ yang ‘unlocked’. Kita cek akun yang tersedia:
cast rpc eth_accounts -r "$RPC_URL"
Pilih salah satu sebagai FUNDER (mis. 0xD4DD…6744). Kirim 1200 ETH ke SOLVER (untuk deposit 1000 ETH + biaya gas).
export FUNDER=0xd4dd339a15c992c59d6eb76f552f13593ea66744
cast send "$SOLVER" --from "$FUNDER" --unlocked -r "$RPC_URL" --value 1200ether --gas-limit 21000
Tujuan perintah: melakukan transfer ETH sederhana dari FUNDER ke SOLVER. — ‘--unlocked’ artinya node mengizinkan pengiriman tanpa private key lokal (akun dibuka di node). — ‘--gas-limit 21000’ karena ini TX transfer biasa.
cast balance "$SOLVER" -r "$RPC_URL"
Cek saldo solver untuk memastikan cukup sebelum lanjut.
5) Verifikasi Alamat & Status Kontrak
export SETUP=0x2573F4b37129cF205E46BA05A2E93aF461bb0AAe
export TARGET=0x70514fa75836f0544F9A777913799Bd43DB70448
cast code  "$SETUP"  -r "$RPC_URL" | head -c 2      # harus '0x'
cast call  "$SETUP"  "TARGET()(address)" -r "$RPC_URL"  # harus sama dengan $TARGET
cast call  "$SETUP"  "isSolved()(bool)" -r "$RPC_URL"   # awalnya false
Arti: ‘cast code’ memastikan alamat berisi bytecode. ‘cast call’ membaca state view function dari kontrak.
6) Deposit 1000 ETH ke Kontrak Target
cast send "$TARGET" "depositVoteCollateral(string,string)" "Sato" "shiNakamoto"   -r "$RPC_URL" --private-key "$PK_SOLVER" --value 1000ether --gas-limit 300000
Fungsi ini menyimpan kolateral besar (1000 ETH). Dua string ‘Sato’ dan ‘shiNakamoto’ hanyalah parameter metadata yang disyaratkan fungsi (misal sebagai identitas/komentar). Intinya: syarat ‘uang menentukan suara’.
Event log yang muncul meng-catat siapa yang deposit dan berapa nilainya. Ini membantu audit/transparansi on‑chain.
7) Vote untuk Kandidat 'CIM' (0x43494d)
# 'CIM' dalam ASCII = 0x43 0x49 0x4D => bytes3 0x43494d
cast send "$TARGET" "vote(bytes3,string,string)" 0x43494d "Sato" "shiNakamoto"   -r "$RPC_URL" --private-key "$PK_SOLVER" --gas-limit 300000
Fungsi vote() menulis pilihan kandidat. Tantangan ini mengecek pemenang via winner(). Kita harus memastikan winner() == 0x43494d agar Setup.isSolved() menjadi true.
cast call "$TARGET" "winner()(bytes3)" -r "$RPC_URL"   # hasil: 0x43494d
cast call "$SETUP"  "isSolved()(bool)"  -r "$RPC_URL"  # hasil: true
8) Klaim Flag via Menu TCP
Setelah solved = true, gunakan menu TCP. Biasanya urutan input yang diterima: pilih opsi (3), masukkan address solver, lalu alamat Setup.
( printf "3\r\n"; sleep 0.2; printf "%s\r\n" "$SOLVER"; sleep 0.2; printf "%s\r\n" "$SETUP"; ) | nc -v 83.136.255.235 42915
Output yang benar akan menampilkan flag seperti:
HTB{h4sh_c0ll1s10n_t0_br1ng_b4ck_d3m0cr4cy}
Kalau ‘Conditions not satisfied!’, berarti salah alamat, salah urutan, atau state on‑chain belum sesuai (winner bukan CIM / saldo kurang / deposit belum).
9) Error Umum & Cara Mengatasi
•	Insufficient funds for gas * price + value — Akun pengirim tidak punya cukup ETH untuk biaya gas + nilai transfer/deposit. Solusi: top‑up dulu (funding).
•	Already deposited — Kamu memanggil deposit dua kali dari address yang sama. Solusi: lanjut langsung ke vote(), atau pakai address lain bila desainnya per‑address.
•	invalid value '' for '[TO]' — Variabel alamat kosong (contoh $TARGET belum ter‑set). Solusi: echo variabel, pastikan tidak kosong sebelum cast send.
•	RPC vs Menu: Port 36473 (HTTP RPC) berbeda dengan 42915 (TCP menu). Jangan ‘cast’ ke 42915, dan jangan ‘netcat’ ke 36473.
•	Address Campur — Jangan pakai alamat hasil deploy di lokal (127.0.0.1) untuk remote RPC, gunakan alamat yang diberikan oleh menu (opsi 1).
10) Kenapa 'CIM' bisa jadi winner?
Berdasarkan interaksi dan event, kontrak menentukan pemenang berdasar deposit dan input kandidat. Kita melakukan deposit besar (1000 ETH) dan vote untuk kandidat CIM (0x43494d). Kontrak lalu menetapkan winner() = CIM. Checker Setup hanya mengecek kondisi ini agar isSolved() = true.
Intinya: kita tidak eksploitasi bug rumit; kita mengikuti ‘aturan curang’ sistem dengan menyediakan deposit besar agar pilihan kita yang menang. Inilah sindiran ke ‘Not A Democratic Election’.
11) Ringkasan Perintah & Tujuan
printf "1\r\n" | nc -v host 42915
→ Ambil info koneksi (private key solver, alamat Setup/Target).
cast rpc eth_accounts
→ Lihat akun unlocked di node (calon FUNDER).
cast send "$SOLVER" --from "$FUNDER" --unlocked --value 1200ether
→ Danai solver untuk deposit + gas.
cast code "$SETUP"
→ Pastikan alamat kontrak valid (berisi bytecode).
cast call "$SETUP" "TARGET()(address)"
→ Konfirmasi alamat target dari Setup.
cast call "$SETUP" "isSolved()(bool)"
→ Cek status solved sebelum/sesudah aksi.
cast send "$TARGET" "depositVoteCollateral(...)" --value 1000ether
→ Deposit kolateral agar suara ‘bernilai’.
cast send "$TARGET" "vote(bytes3,...)" 0x43494d
→ Pilih kandidat CIM (0x43494d).
cast call "$TARGET" "winner()(bytes3)"
→ Verifikasi pemenang di kontrak target.
(printf "3"; ... ) | nc host 42915
→ Klaim flag (opsi 3, kirim SOLVER dan SETUP).
12) Catatan Keamanan & Praktik Baik
•	Jangan pakai private key personal di lingkungan CTF publik. Gunakan key dari challenge saja.
•	Bedakan ‘local testnet’ dengan ‘remote RPC’ — alamat kontrak berbeda.
•	Selalu verifikasi state di chain sebelum klaim flag (winner & isSolved).
•	Simpan command & TX hash untuk audit (reproducible write‑up).
Penutup
Dengan alur di atas, kamu menyiapkan environment, mendanai akun solver, mengeksekusi deposit dan vote yang tepat, memverifikasi state di on‑chain, lalu mengeksekusi klaim flag via menu TCP. Hasil akhirnya: flag keluar. Jika ada bagian yang ingin diperdalam (mis. decoding event log, ABI, atau menulis script otomatisasi), bilang — bisa kita tambah versi scripting penuh.
<img width="432" height="622" alt="image" src="https://github.com/user-attachments/assets/212d5e5e-43d9-46cf-8734-5ec50b719b97" />
