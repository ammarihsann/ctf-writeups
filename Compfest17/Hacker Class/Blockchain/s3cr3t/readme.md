# Write-up — CTF “S3CR3T” (COMPFEST 17)


# Ringkasan singkat tantangan

* **Event:** COMPFEST 17
* **Challenge:** S3CR3T
* **Goal:** Dapatkan flag dengan memecahkan / mengeksploitasi smart contract yang dibuat lewat `Setup.sol` → deploy `Secret.sol`.
* **Inti kelemahan:** kontrak menyimpan *hash* dari sebuah kalimat (bytes32 `SECRET`) dengan format plaintext yang diketahui (hanya 6 digit berubah) → preimage space cuma `10^6` sehingga feasible untuk brute-force.

---

# Analisis singkat kontrak

* `Setup.sol`

  * Deploy `Secret` dan ekspose alamatnya via `challenge()`.
  * Sering juga ada `isSolved()` public untuk mengecek hasil.
* `Secret.sol`

  * Menyimpan `bytes32 public SECRET`.
  * Memiliki `ReferedBy` (string) yang juga dapat dibaca on-chain.
  * Fungsi `guess(string sentence, string referrer)`:

    * Require pembayaran `msg.value >= 0.5 ether`.
    * Cek `keccak256(abi.encodePacked(sentence)) == SECRET`.
    * Cek `keccak256(abi.encodePacked(referrer)) == keccak256(abi.encodePacked(ReferedBy))`.
    * Jika lolos -> set `solved = true`.
  * Ada hint berupa format kalimat:

    ```
    "Can i join COMPFEST 17? Here is my secret number: [6-digit number]"
    ```
* **Kenapa mudah dipecahkan:** format sangat terbatas — hanya 1,000,000 kemungkinan (000000..999999). Brute-force hash space ini sangat cepat di mesin lokal.

---

# Eksploitasi — langkah ringkas (apa yang saya lakukan)

1. Dapatkan alamat `Setup` (diberikan di UI challenge).
2. `cast` / ethers call `Setup.challenge()` → alamat `Secret`.
3. Baca `ReferedBy()` dari `Secret` (dipakai sebagai argumen referrer).
4. Brute-force semua 6-digit (zero-padded), hitung `keccak256(sentence)` sampai cocok dengan `SECRET`.
5. Panggil `Secret.guess(sentence, referrer)` dengan `value = 0.5 ETH`.
6. Verifikasi `isChallengeSolved()` dan `setup.isSolved()`.

---

# Artefak (hasil)

* **Preimage (kalimat lengkap) ditemukan:**

  ```
  Can i join COMPFEST 17? Here is my secret number: 198514
  ```
* **Keccak256(preimage):**

  ```
  0x726061b2dddd96177e52d122926939b0608ac99930f2f980c08045e863f6ecf3
  ```

  → sama dengan `SECRET` di kontrak.
* **Alamat Secret (di instance yang Anda gunakan):**

  ```
  0x8d999a32A861695d949D8CFbC64b49A91f35B6f6
  ```
* **Transaksi exploit (Anda kirim):**

  ```
  0x89d0c7d883570ac1eb3936effdde6be1946af7696cd37087b6bd7f517e4c20ec
  ```

  (mined in block 2 — menandai solved)
* **Flag (ditampilkan di UI setelah solved):**

  ```
  COMPFEST17{finally_COMPFEST17_Y3333333AYYYY!_32e0debeb1}
  ```

  (verifikasi di UI challenge Anda).

---

# Reproduksi — skrip & perintah

> **Jangan membocorkan PRIVATE\_KEY**. Pakai .env / environment variables.

## 1) Ethers.js (versi kompatibel)

Skrip `exploit_compat.js` yang mendukung ethers v5/v6 (saya pakai ini di environment Anda):

```javascript
// exploit_compat.js
require('dotenv').config();
const E = require('ethers');
const RPC_URL = process.env.RPC_URL;
const PRIVATE_KEY = process.env.PRIVATE_KEY;
const SETUP_ADDR = process.env.SETUP_ADDR;

if (!RPC_URL || !PRIVATE_KEY || !SETUP_ADDR) {
  console.error("Missing RPC_URL/PRIVATE_KEY/SETUP_ADDR in .env");
  process.exit(1);
}

function makeEthersHelpers(E) {
  if (E.providers && E.providers.JsonRpcProvider) {
    return {
      provider: new E.providers.JsonRpcProvider(RPC_URL),
      Wallet: E.Wallet,
      Contract: E.Contract,
      parseEther: E.utils.parseEther,
      keccak256: (s) => E.utils.keccak256(E.utils.toUtf8Bytes(s))
    };
  }
  if (E.JsonRpcProvider) {
    return {
      provider: new E.JsonRpcProvider(RPC_URL),
      Wallet: E.Wallet,
      Contract: E.Contract,
      parseEther: E.parseEther ? E.parseEther : (s) => E.utils.parseEther(s),
      keccak256: (s) => (E.keccak256 ? E.keccak256(E.toUtf8Bytes(s)) : E.utils.keccak256(E.toUtf8Bytes(s)))
    };
  }
  throw new Error("Unknown ethers");
}

(async () => {
  const { provider, Wallet, Contract, parseEther, keccak256 } = makeEthersHelpers(E);

  const setupAbi = ["function challenge() view returns (address)", "function isSolved() view returns (bool)"];
  const secretAbi = [
    "function SECRET() view returns (bytes32)",
    "function ReferedBy() view returns (string)",
    "function guess(string calldata sentence, string calldata referrer) external payable",
    "function isChallengeSolved() view returns (bool)"
  ];

  const setup = new Contract(SETUP_ADDR, setupAbi, provider);
  const secretAddr = await setup.challenge();
  console.log("Secret address:", secretAddr);

  const secret = new Contract(secretAddr, secretAbi, provider);
  const referer = await secret.ReferedBy();
  const sentence = "Can i join COMPFEST 17? Here is my secret number: 198514";
  console.log("Using sentence:", sentence, "referer:", referer);

  // verify on-chain SECRET matches computed keccak256
  const onchainSecret = await secret.SECRET();
  const computed = keccak256(sentence);
  console.log("onchain SECRET:", onchainSecret);
  console.log("computed keccak:", computed);
  if (onchainSecret !== computed) {
    console.error("Preimage mismatch — aborting.");
    process.exit(1);
  }

  const wallet = new Wallet(PRIVATE_KEY, provider);
  const secretWithSigner = secret.connect(wallet);
  const tx = await secretWithSigner.guess(sentence, referer, { value: parseEther("0.5"), gasLimit: 200000 });
  console.log("Tx sent:", tx.hash);
  await (tx.wait ? tx.wait() : provider.waitForTransaction(tx.hash));
  console.log("Done.");
})();
```

**Cara pakai:**

```bash
npm init -y
npm install dotenv ethers
# buat .env berisi RPC_URL, PRIVATE_KEY, SETUP_ADDR
node exploit_compat.js
```

---

## 2) Foundry (Solidity) — `script/Exploit.s.sol` (broadcast)

File yang memanggil `guess` secara otomatis, dengan verifikasi `SECRET`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.18;
import "forge-std/Script.sol";

interface ISetup { function challenge() external view returns (address); }
interface ISecret {
    function ReferedBy() external view returns (string memory);
    function SECRET() external view returns (bytes32);
    function guess(string calldata sentence, string calldata referrer) external payable;
}

contract Exploit is Script {
    function run() external {
        address setupAddr = vm.envAddress("SETUP_ADDR");
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        address secretAddr = ISetup(setupAddr).challenge();
        ISecret secret = ISecret(secretAddr);
        string memory referer = secret.ReferedBy();
        string memory sentence = "Can i join COMPFEST 17? Here is my secret number: 198514";
        require(secret.SECRET() == keccak256(bytes(sentence)), "preimage mismatch");
        secret.guess{value: 0.5 ether}(sentence, referer);
        vm.stopBroadcast();
    }
}
```

**Run:**

```bash
export RPC_URL=...
export SETUP_ADDR=0x3BF655...
export PRIVATE_KEY=0x...
forge script script/Exploit.s.sol:Exploit --rpc-url $RPC_URL --broadcast
```

---

## 3) `cast` step-by-step (manual)

Gunakan `foundry`/`cast` untuk debugging jika ingin memeriksa setiap langkah:

```bash
RPC_URL="http://ctf.compfest.id:7401/eb8871c8-..."
SETUP_ADDR=0x3BF655...
SECRET_ADDR=$(cast call $SETUP_ADDR "challenge() returns (address)" --rpc-url $RPC_URL)
echo "Secret: $SECRET_ADDR"

REFERER=$(cast call $SECRET_ADDR "ReferedBy() returns (string)" --rpc-url $RPC_URL)
echo "Referer: $REFERER"

SENTENCE='Can i join COMPFEST 17? Here is my secret number: 198514'
CALDATA=$(cast calldata "guess(string,string)" "$SENTENCE" "$REFERER")
cast send $SECRET_ADDR "$CALDATA" --value 0.5ether --rpc-url $RPC_URL --private-key $YOUR_KEY
```

Setelah itu:

```bash
cast call $SECRET_ADDR "isChallengeSolved() returns (bool)" --rpc-url $RPC_URL
cast call $SETUP_ADDR  "isSolved() returns (bool)" --rpc-url $RPC_URL
cast receipt <tx-hash> --rpc-url $RPC_URL
```

---

# Mitigasi & lessons learned

1. **Jangan simpan hash dari string dengan low entropy** — jika format plaintext dapat diprediksi, brute force trivial. Gunakan nonce/salt yang only-known off-chain atau random oracle yang tidak terprediksi.
2. **Jangan publikasikan referensi atau pola plaintext di code / hint** jika Anda ingin menyimpan secret on-chain. Atau, gunakan HSM / offchain verification.
3. **Gunakan keamanan ekonomi**: kalau ada payment requirement, pertimbangkan gating lain (multi-factor).
4. **Jika perlu verifikasi on-chain**, gunakan signature scheme (pemilik menandatangani string) daripada memverifikasi preimage langsung.

---

# Kesimpulan

* Vulnerability: predictable preimage (format diketahui → 1e6 search space) + secret publik (on-chain constant/variable) → brute force practical.
* Saya berhasil menemukan preimage `...198514` → memanggil `guess(...)` → `isSolved()` true → flag ditampilkan di UI.
* Artefak utama: secret contract address, tx hash, preimage, keccak256, flag (lihat bagian Artefak).

---
