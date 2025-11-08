# 🗳️ NotADemocraticElection — Humanized Write-Up (Foundry/Cast)

![Foundry](https://img.shields.io/badge/Tooling-Foundry%20%7C%20forge%20%26%20cast-6E56CF)
![RPC](https://img.shields.io/badge/RPC-JSON--RPC-blue)
![ChainID](https://img.shields.io/badge/Chain%20ID-31337-0A7EA4)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy%20to%20Medium-22C55E)
![Status](https://img.shields.io/badge/Status-Solved-brightgreen)

> **Goal singkat**: bikin pemenang jadi **CIM** (bytes3 `"CIM"` = `0x43494d`), sehingga `Setup.isSolved()` → **true**; lalu klaim **flag** via menu TCP.

---

## 📚 Table of Contents
- [Konsep Singkat](#konsep-singkat)
- [Arsitektur Tantangan](#arsitektur-tantangan)
- [Prasyarat](#prasyarat)
- [Informasi Koneksi (Menu TCP)](#informasi-koneksi-menu-tcp)
- [Setup Environment](#setup-environment)
- [Langkah Penyelesaian](#langkah-penyelesaian)
- [Verifikasi](#verifikasi)
- [Klaim Flag](#klaim-flag)
- [Diagram Alur ASCII](#diagram-alur-ascii)
- [Troubleshooting](#troubleshooting)
- [Kenapa Ini Bekerja?](#kenapa-ini-bekerja)
- [Skrip Otomasi](#skrip-otomasi)
- [Catatan & Tips](#catatan--tips)

---

## 🧠 Konsep Singkat
Di dunia ini, “demokrasi” hanya nama. Untuk bisa memilih, kamu **wajib deposit 1000 ETH**. Pemenangnya pun dipaksa jadi **CIM**. Tugasmu: **deposit → vote "CIM" → verifikasi solved → ambil flag**.

---

## 🏗️ Arsitektur Tantangan

- **RPC JSON-RPC (HTTP)**: `http://83.136.255.235:36473`  
  Dipakai oleh **cast/forge** (Foundry).
- **Menu TCP (plain text)**: `83.136.255.235:42915`  
  Dipakai untuk **ambil info instance** & **klaim flag**.

**Kontrak:**
- `Target` = NotADemocraticElection (tempat deposit & vote)
- `Setup`  = checker, expose `TARGET()` & `isSolved()`

**Kandidat:**
- **CIM** = `bytes3("CIM")` = `0x43494d` ✅ (harus jadi winner)

---

## 🧰 Prasyarat
- Foundry (forge, cast) terpasang & PATH benar.
- `jq` (untuk parsing JSON sederhana).
- `nc`/`netcat` untuk akses menu TCP.

**Cek Foundry:**
```bash
which forge
which cast
# idealnya -> ~/.config/.foundry/bin/forge & cast
```

> Jika muncul error “ZOE … unknown option (--rpc-url)”, PATH-mu salah (bukan Foundry). Perbaiki:
```bash
echo 'export PATH="$HOME/.config/.foundry/bin:$PATH"' >> ~/.zshenv
source ~/.zshenv
```

---

## 🔑 Informasi Koneksi (Menu TCP)
Ambil data instance spesifikmu dari menu:
```bash
printf "1
" | nc -v 83.136.255.235 42915
```
Kamu akan mendapat:
- `Private key`  → **PK_SOLVER**
- `Address`      → **SOLVER**
- `Target`       → **TARGET**
- `Setup`        → **SETUP**

> Data ini **unik per instance**. Jangan campur dengan instance lain, kalau tidak akan muncul **“Conditions not satisfied!”** saat klaim.

---

## ⚙️ Setup Environment
```bash
# RPC server untuk JSON-RPC
export RPC_URL=http://83.136.255.235:36473

# Dari menu "1 - Connection information"
export PK_SOLVER=0x...           # private key solver yang diberikan menu
export SOLVER=$(cast wallet address --private-key "$PK_SOLVER")
export TARGET=0x...              # target dari menu
export SETUP=0x...               # setup dari menu

# (opsional) akun donor di node untuk mendanai SOLVER
export FUNDER=$(cast rpc eth_accounts -r "$RPC_URL" | jq -r '.[0]')
```

Sanity check:
```bash
cast code "$SETUP" -r "$RPC_URL" | head -c 2  # -> 0x
cast call "$SETUP" "TARGET()(address)" -r "$RPC_URL"  # = $TARGET
cast call "$SETUP" "isSolved()(bool)" -r "$RPC_URL"   # harus false sebelum solve
```

---

## ✅ Langkah Penyelesaian

### 1) Danai SOLVER (butuh > 1000 ETH)
```bash
cast send "$SOLVER"   --from "$FUNDER" --unlocked   -r "$RPC_URL"   --value 1200ether   --gas-limit 21000
```
Tujuan: **deposit** butuh **1000 ETH**, sisanya buat gas.

Cek saldo:
```bash
cast balance "$SOLVER" -r "$RPC_URL"
```

### 2) Deposit 1000 ETH
```bash
cast send "$TARGET"   "depositVoteCollateral(string,string)" "Sato" "shiNakamoto"   -r "$RPC_URL" --private-key "$PK_SOLVER"   --value 1000ether --gas-limit 300000
```
Tujuan: memenuhi syarat “boleh memilih”.

> Jika keluar `execution reverted: Already deposited`, artinya kamu sudah deposit. Lanjut ke vote.

### 3) Vote untuk CIM
```bash
# "CIM" -> bytes3 -> 0x43494d
cast send "$TARGET"   "vote(bytes3,string,string)" 0x43494d "Sato" "shiNakamoto"   -r "$RPC_URL" --private-key "$PK_SOLVER"   --gas-limit 300000
```
Tujuan: set pemenang menjadi **CIM**.

---

## 🔍 Verifikasi
```bash
cast call "$TARGET" "winner()(bytes3)" -r "$RPC_URL"   # -> 0x43494d
cast call "$SETUP"  "isSolved()(bool)" -r "$RPC_URL"   # -> true
```

---

## 🏁 Klaim Flag
Gunakan menu TCP (**bukan** JSON-RPC):
```bash
# urutan input: 3 -> SOLVER -> SETUP
( printf "3
"; sleep 0.2; printf "%s
" "$SOLVER"; sleep 0.2; printf "%s
" "$SETUP"; )   | nc -v 83.136.255.235 42915
# contoh output:
# HTB{h4sh_c0ll1s10n_t0_br1ng_b4ck_d3m0cr4cy}
```

---

## 🔎 Diagram Alur ASCII

```
+------------------+
|  TCP :42915      |
|  "1" -> INFO     |
+--------+---------+
         |
         v  (PK_SOLVER, SOLVER, TARGET, SETUP)
+--------+---------+
|  Env Vars        |
|  RPC=:36473      |
+--------+---------+
         |
         v
+--------+---------+         +-------------------------+
|  FUNDER (rich)   |  ETH -> |      SOLVER (your acc) |
+--------+---------+         +-----------+-------------+
                                         |
                                         v
                         +---------------+------------------+
                         |      TARGET (contract)          |
                         | 1) deposit 1000 ETH             |
                         | 2) vote bytes3("CIM")=0x43494d  |
                         +---------------+------------------+
                                         |
                                         v
                         +---------------+------------------+
                         |      SETUP (checker)            |
                         | isSolved() == true?             |
                         +---------------+------------------+
                                         |
                                         v
                         +---------------+------------------+
                         |   TCP :42915 -> "3" Get flag    |
                         |   send SOLVER, then SETUP       |
                         +----------------------------------+
```

---

## 🧯 Troubleshooting

- **`invalid HTTP version parsed`**  
  Kamu salah port; itu terjadi kalau `cast` diarahkan ke **42915**. Gunakan **36473** untuk `cast/forge`. Port **42915** hanya untuk **nc**.

- **`Insufficient funds for gas * price + value`**  
  Saldo pengirim kurang. Danai SOLVER dari akun FUNDER node.

- **`execution reverted: Already deposited`**  
  Deposit sudah pernah dilakukan. Langsung vote saja.

- **`invalid value '' for '[TO]': invalid string length`**  
  Variabel kosong (contoh `$TARGET` belum terisi). Echo semua env sebelum kirim tx.

- **`Conditions not satisfied!` saat klaim**  
  - Kamu salah **SOLVER** atau **SETUP** (pakai yang dari **menu 1**).
  - Kamu solve di chain/RPC lain.
  - Belum deposit 1000 ETH / belum vote CIM.

---

## 🧩 Kenapa Ini Bekerja?
- Kontrak mendesain **syarat deposit** sebagai “akses ke hak pilih”.  
- Pemenang diharapkan **CIM** (bukan demokrasi sungguhan).  
- `Setup.isSolved()` memverifikasi state di `Target` (pemenang == `0x43494d`).  
- Service TCP memetakan **instance kamu** (SETUP/TARGET/SOLVER) → kalau cocok & solved, flag keluar.

---

## ⚡ Skrip Otomasi

### `solve.sh`
Otomatis: **fund → deposit → vote → verifikasi**.  
Butuh: `RPC_URL`, `PK_SOLVER`, `TARGET`, `SETUP` (dan `jq`).

```bash
#!/usr/bin/env bash
set -euo pipefail

: "${RPC_URL:?set RPC_URL=http://83.136.255.235:36473}"
: "${PK_SOLVER:?set PK_SOLVER from menu 1}"
: "${TARGET:?set TARGET from menu 1}"
: "${SETUP:?set SETUP from menu 1}"

SOLVER=$(cast wallet address --private-key "$PK_SOLVER")

echo "[i] SOLVER        = $SOLVER"
echo "[i] TARGET        = $TARGET"
echo "[i] SETUP         = $SETUP"
echo "[i] RPC_URL       = $RPC_URL"

# pick a rich account on the node if FUNDER not provided
FUNDER="${FUNDER:-$(cast rpc eth_accounts -r "$RPC_URL" | jq -r '.[0]')}"
echo "[i] FUNDER        = $FUNDER"

# fund SOLVER if needed
BAL_WEI=$(cast balance "$SOLVER" -r "$RPC_URL")
NEED_WEI=$(cast --to-wei 1100ether)

if [ "$BAL_WEI" -lt "$NEED_WEI" ]; then
  echo "[+] Funding SOLVER with 1200 ETH from $FUNDER"
  cast send "$SOLVER" --from "$FUNDER" --unlocked -r "$RPC_URL" --value 1200ether --gas-limit 21000
else
  echo "[i] SOLVER already funded (balance $(cast --from-wei "$BAL_WEI") ETH)"
fi

echo "[+] Deposit 1000 ETH"
cast send "$TARGET" "depositVoteCollateral(string,string)" "Sato" "shiNakamoto"   -r "$RPC_URL" --private-key "$PK_SOLVER"   --value 1000ether --gas-limit 300000 || true

echo "[+] Vote CIM (0x43494d)"
cast send "$TARGET" "vote(bytes3,string,string)" 0x43494d "Sato" "shiNakamoto"   -r "$RPC_URL" --private-key "$PK_SOLVER"   --gas-limit 300000

echo "[?] winner   = $(cast call "$TARGET" "winner()(bytes3)" -r "$RPC_URL")"
echo "[?] isSolved = $(cast call "$SETUP"  "isSolved()(bool)" -r "$RPC_URL")"
```

### `auto-claim.sh`
Otomatis: **menu 3 → kirim SOLVER → kirim SETUP**.

```bash
#!/usr/bin/env bash
set -euo pipefail

HOST="${1:-83.136.255.235}"
PORT="${2:-42915}"
SOLVER="${3:?usage: $0 [host] [port] <solver_address> <setup_address>}"
SETUP="${4:?usage: $0 [host] [port] <solver_address> <setup_address>}"

echo "[i] Claiming on $HOST:$PORT"
{
  printf "3
"
  sleep 0.2
  printf "%s
" "$SOLVER"
  sleep 0.2
  printf "%s
" "$SETUP"
  sleep 0.2
} | nc -v -w 4 "$HOST" "$PORT" || {
  echo "[!] connection/claim failed"; exit 1;
}
```

---

## 📝 Catatan & Tips
- **Jangan** panggil `cast` ke port 42915 — itu **menu teks**, bukan JSON-RPC.  
- Selalu echo env sebelum kirim tx:
  ```bash
  echo "$RPC_URL"; echo "$SOLVER"; echo "$TARGET"; echo "$SETUP"
  ```
- Jika gas price tinggi dan gagal pendanaan, kurangi value/naikkan saldo FUNDER.

> Hasil akhir sukses terlihat sebagai:
> - `winner = 0x43494d`
> - `isSolved = true`
> - Menu TCP “3 – Get flag” mengeluarkan flag 🎉

---

Happy hunting & have fun! 🛠️🧩
