# DAO Governance Smart Contract

## Overview

`DAO` adalah smart contract governance berbasis DAO (Decentralized Autonomous Organization) yang dibangun menggunakan OpenZeppelin Governor Upgradeable. Kontrak ini memungkinkan pemegang token untuk membuat proposal, melakukan voting, dan mengeksekusi keputusan melalui sistem timelock.

Kontrak ini menggunakan arsitektur yang sama seperti DAO besar seperti Compound dan OpenZeppelin Governor.

---

# Arsitektur Kontrak

Kontrak `DAO` menggunakan beberapa modul dari OpenZeppelin:

| Modul                                  | Fungsi                                                   |
| -------------------------------------- | -------------------------------------------------------- |
| GovernorUpgradeable                    | Kontrak utama governance                                 |
| GovernorSettingsUpgradeable            | Mengatur voting delay, voting period, proposal threshold |
| GovernorCountingSimpleUpgradeable      | Sistem voting For / Against / Abstain                    |
| GovernorStorageUpgradeable             | Menyimpan data proposal on-chain                         |
| GovernorVotesUpgradeable               | Voting berdasarkan token (ERC20Votes)                    |
| GovernorVotesQuorumFractionUpgradeable | Mengatur quorum (persentase minimal voting)              |
| GovernorTimelockControlUpgradeable     | Integrasi timelock sebelum eksekusi                      |
| TimelockControllerUpgradeable          | Delay eksekusi proposal                                  |
| Initializable                          | Untuk upgradeable contract                               |

---

# Penjelasan Setiap Bagian Kode

## 1. Import Library

```solidity
import {GovernorUpgradeable} from "@openzeppelin/contracts-upgradeable/governance/GovernorUpgradeable.sol";
import {GovernorSettingsUpgradeable} from "@openzeppelin/contracts-upgradeable/governance/extensions/GovernorSettingsUpgradeable.sol";
import {GovernorCountingSimpleUpgradeable} from "@openzeppelin/contracts-upgradeable/governance/extensions/GovernorCountingSimpleUpgradeable.sol";
import {GovernorStorageUpgradeable} from "@openzeppelin/contracts-upgradeable/governance/extensions/GovernorStorageUpgradeable.sol";
import {GovernorVotesUpgradeable} from "@openzeppelin/contracts-upgradeable/governance/extensions/GovernorVotesUpgradeable.sol";
import {GovernorVotesQuorumFractionUpgradeable} from "@openzeppelin/contracts-upgradeable/governance/extensions/GovernorVotesQuorumFractionUpgradeable.sol";
import {GovernorTimelockControlUpgradeable} from "@openzeppelin/contracts-upgradeable/governance/extensions/GovernorTimelockControlUpgradeable.sol";
import {TimelockControllerUpgradeable} from "@openzeppelin/contracts-upgradeable/governance/TimelockControllerUpgradeable.sol";
import {IVotes} from "@openzeppelin/contracts/governance/utils/IVotes.sol";
import {Initializable} from "@openzeppelin/contracts/proxy/utils/Initializable.sol";
```

### Penjelasan:

* Library OpenZeppelin digunakan untuk membangun sistem DAO tanpa harus membuat dari nol.
* `IVotes` digunakan untuk integrasi dengan token ERC20Votes.
* `Initializable` digunakan karena kontrak bersifat upgradeable (proxy pattern).

---

## 2. Deklarasi Kontrak

```solidity
contract MyDAO is
    Initializable,
    GovernorUpgradeable,
    GovernorSettingsUpgradeable,
    GovernorCountingSimpleUpgradeable,
    GovernorStorageUpgradeable,
    GovernorVotesUpgradeable,
    GovernorVotesQuorumFractionUpgradeable,
    GovernorTimelockControlUpgradeable
```

### Penjelasan:

Kontrak ini menggunakan multiple inheritance dari beberapa modul OpenZeppelin untuk membangun sistem governance yang lengkap:

* Membuat proposal
* Voting
* Quorum
* Timelock
* Eksekusi proposal

---

## 3. Constructor

```solidity
constructor() {
    _disableInitializers();
}
```

### Penjelasan:

* Digunakan untuk kontrak upgradeable.
* Mencegah kontrak implementation di-initialize secara langsung.
* Ini adalah best practice untuk proxy contract.

---

## 4. Fungsi Initialize

```solidity
function initialize(
    IVotes _token,
    TimelockControllerUpgradeable _timelock
) public initializer {
```

### Penjelasan:

Fungsi ini menggantikan constructor pada kontrak upgradeable.
Fungsi ini mengatur semua parameter DAO saat pertama kali deploy.

---

### Validasi Address

```solidity
require(address(_token) != address(0), "Invalid token");
require(address(_timelock) != address(0), "Invalid timelock");
```

Digunakan untuk memastikan alamat token dan timelock valid.

---

### Inisialisasi Governor

```solidity
__Governor_init("MyDAO");
```

Memberikan nama DAO.

---

### Voting Settings

```solidity
__GovernorSettings_init(
    7200,     // Voting delay
    50400,    // Voting period
    100e18    // Proposal threshold
);
```

Penjelasan:

* `votingDelay`: Waktu tunggu sebelum voting dimulai (dalam block).
* `votingPeriod`: Lama voting berlangsung (dalam block).
* `proposalThreshold`: Jumlah minimum token untuk membuat proposal.

---

### Voting System

```solidity
__GovernorCountingSimple_init();
```

Menggunakan sistem voting sederhana:

* For
* Against
* Abstain

---

### Proposal Storage

```solidity
__GovernorStorage_init();
```

Menyimpan data proposal secara on-chain.

---

### Token Voting

```solidity
__GovernorVotes_init(_token);
```

Menghubungkan DAO dengan token ERC20Votes agar voting power berdasarkan jumlah token.

---

### Quorum

```solidity
__GovernorVotesQuorumFraction_init(4);
```

Artinya minimal 4% dari total voting power harus ikut voting agar proposal valid.

---

### Timelock

```solidity
__GovernorTimelockControl_init(_timelock);
```

Timelock digunakan untuk memberi delay sebelum proposal dieksekusi.
Ini mencegah perubahan langsung yang berbahaya.

---

# 5. Fungsi Override

Karena menggunakan multiple inheritance, beberapa fungsi harus dioverride.

## votingDelay

```solidity
function votingDelay()
    public
    view
    override(GovernorUpgradeable, GovernorSettingsUpgradeable)
    returns (uint256)
{
    return super.votingDelay();
}
```

Mengembalikan delay sebelum voting dimulai.

---

## votingPeriod

Mengembalikan durasi voting.

---

## quorum

Mengembalikan jumlah minimal voting agar proposal valid.

---

## proposalThreshold

Jumlah minimal token untuk membuat proposal.

---

# 6. Governor + Timelock Functions

Fungsi-fungsi ini mengatur alur proposal dari dibuat hingga dieksekusi.

| Function             | Fungsi                               |
| -------------------- | ------------------------------------ |
| state                | Status proposal                      |
| proposalNeedsQueuing | Apakah proposal perlu masuk timelock |
| _propose             | Membuat proposal                     |
| _queueOperations     | Memasukkan proposal ke timelock      |
| _executeOperations   | Mengeksekusi proposal                |
| _cancel              | Membatalkan proposal                 |
| _executor            | Siapa yang mengeksekusi proposal     |

---

# Alur Kerja DAO

## 1. User memiliki token

User harus memiliki token ERC20Votes.

## 2. Delegate Voting Power

User harus mendelegasikan voting power ke dirinya sendiri sebelum bisa voting.

## 3. Membuat Proposal

User dengan token di atas `proposalThreshold` dapat membuat proposal.

## 4. Voting Delay

Proposal menunggu sebelum voting dimulai.

## 5. Voting Period

Pemegang token melakukan voting.

## 6. Quorum Check

Jika jumlah voting tidak mencapai quorum, proposal gagal.

## 7. Queue

Jika proposal lolos, proposal masuk ke Timelock.

## 8. Timelock Delay

Menunggu sebelum bisa dieksekusi.

## 9. Execute

Proposal dijalankan.

---

# Teknologi yang Digunakan

| Teknologi     | Fungsi               |
| ------------- | -------------------- |
| Solidity      | Smart contract       |
| OpenZeppelin  | Library governance   |
| ERC20Votes    | Token voting         |
| Timelock      | Delay eksekusi       |
| Proxy Pattern | Upgradeable contract |

---

# Kesimpulan

`MyDAO` adalah smart contract governance DAO yang memungkinkan:

* Proposal system
* Token-based voting
* Quorum system
* Timelock execution
* Upgradeable contract

Kontrak ini mengikuti best practice OpenZeppelin dan dapat digunakan sebagai dasar sistem DAO governance di blockchain.
