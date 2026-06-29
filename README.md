# Rincoin Core

![Version](https://img.shields.io/badge/version-1.1.0-blue.svg)
![License](https://img.shields.io/badge/license-MIT-green.svg)
[![RIPs](https://img.shields.io/badge/RIPs-rincoin--rips-blueviolet.svg)](https://github.com/Aevust/rincoin-rips)

Rincoin is a decentralized digital currency derived from the Litecoin codebase (itself a long-established Bitcoin derivative) that introduces a new Proof-of-Work hashing algorithm called **RinHash**. RinHash is a hybrid PoW algorithm combining BLAKE3, Argon2d, and SHA3-256, designed to provide security while enabling broad, accessible participation during the network's formative phase. This README provides an overview of Rincoin's specifications, the RinHash algorithm, and network parameters.

> **Current release: v1.1.0** — the *Customized Halving* release. Versions follow `v[GENERATION].[MAJOR].[MINOR]` per RIP-0001 (here: GENERATION=1, MAJOR=1, MINOR=0). The MAJOR field is incremented from the previous `v1.0.6` release because v1.1.0 carries protocol-level (hard fork) changes; per RIP-0001, a hard fork is not shipped as a maintenance (MINOR) release. See the **Release: v1.1.0** section below.

---

## 🛡️ Core Architecture & Network Sovereignty

Rincoin Core has been engineered and reviewed with the goal of mathematical and network independence from its upstream codebases. Recent milestones include:

- **P2P Network Sovereignty (confirmed in both Sim and Core):** Legacy network identifiers have been replaced with Rincoin-native values across two distinct layers. (1) **Message-start magic bytes** `0x52 0x49 0x4E 0x43` ("RINC"), defined in `src/chainparams.cpp`. (2) **Internal IPv6 prefix** `INTERNAL_IN_IPV6_PREFIX = FD 2D DD 82 F5 C8` — `0xFD + sha256("rincoin")[0:5]`, replacing the upstream `FD 6B 88 C0 87 24` (`sha256("bitcoin")[0:5]`), confirmed in `src/netaddress.h` with inline comment. This prefix is used when serializing non-IP peers (Tor/I2P/CJDNS) in ADDR messages. `CNetAddr::SetInternal()` in `netaddress.cpp` is a separate mechanism that hashes individual DNS seed names per-peer for internal tracking — already Rincoin-native by using Rincoin's own seeds. Note: minor residual upstream naming artifacts remain in `netaddress.h` (`#ifndef BITCOIN_NETADDRESS_H`, `bitcoin-config.h` include, copyright header) but have no functional impact on network sovereignty.
- **Customized Consensus & Emission (Scenario II):** Rincoin implements a multi-phase emission schedule. It begins with 210,000-block intervals (~145 days) and dilates to multi-million-block epochs after height 840,000 to slow subsidy decay. It culminates in a perpetual terminal reward (0.6 RIN), which is intended to support a long-term security budget. The dilation at block 840,000 is a hard fork, implemented in **v1.1.0** (see below). Consensus rules, including the custom base58 address prefix (prefix `60`), are validated by the test suite.
- **Continuous Integration:** The active validation and utility test suites report a 100% PASS state. Legacy upstream benchmarks that depend on obsolete upstream block data have been decoupled as not applicable to Rincoin, keeping the CI pipeline stable for ongoing development.

---

## 📊 Key Specifications

| Feature | Specification |
| :--- | :--- |
| **Coin Name / Ticker** | Rincoin (**RIN**) |
| **Consensus Mechanism**| Proof-of-Work (PoW) – **RinHash** algorithm |
| **Block Target Time** | 1 minute (60 seconds per block) |
| **Initial Block Reward**| 50 RIN |
| **Emission Schedule** | Custom multi-phase (Initial: 210k blocks, Dilated: up to 2.1M blocks, Terminal: 0.6 RIN) |
| **Difficulty Retarget**| 1,980-block intervals until block 30,000; DGW (per-block) thereafter |
| **BIP9 Confirmation Window (mainnet)** | 7,920 blocks (1,980×4); threshold 5,940 (75%) — see v1.1.0 |
| **Proof-of-Work Hash** | 256-bit output |
| **Address Format** | Base58 addresses start with **R** (version byte `60`) |
| **Network Ports** | P2P: `9555`, RPC: `9556` |
| **Network Magic** | `0x52` `0x49` `0x4E` `0x43` ("RINC") |

> **Note:** All network parameters (difficulty retarget, magic bytes, address prefixes) are defined in `src/chainparams.cpp`, which is the single source of truth. Values documented here are for reference only.

### 📉 Emission Schedule (Customized Halving: Scenario II)

Rincoin implements a piecewise emission schedule intended to support long-term network sustainability and slow subsidy decay.

| Phase | Block Height | Reward (RIN) | Duration (Blocks) |
| :--- | :--- | :--- | :--- |
| **Phase 0** | 0 - 209,999 | 50 | 210,000 |
| **Phase 1** | 210,000 - 419,999 | 25 | 210,000 |
| **Phase 2** | 420,000 - 629,999 | 12.5 | 210,000 |
| **Phase 3** | 630,000 - 839,999 | 6.25 | 210,000 |
| **Phase 4** | 840,000 - 2,099,999 | 4 | 1,260,000 |
| **Phase 5** | 2,100,000 - 4,199,999 | 2 | 2,100,000 |
| **Phase 6** | 4,200,000 - 6,299,999 | 1 | 2,100,000 |
| **Terminal**| 6,300,000+ | 0.6 | Perpetual |

The Phase 4 transition at block 840,000 is a hard fork, implemented in **v1.1.0**. See the **Release: v1.1.0** section, the technical roadmap, and RIP-0002 for the activation plan.

**Privacy layer (MWEB):** MWEB (MimbleWimble Extension Blocks) is implemented in the codebase. Per project policy, mainnet activation is intentionally deferred until the SQLite descriptor-wallet migration completes. As of **v1.1.0**, mainnet MWEB is sealed to `NEVER_ACTIVE` in `src/chainparams.cpp` (replacing the Litecoin-inherited height-based value `nStartHeight = 2,217,600`), matching the state validated in Rincoin-Sim. For validation purposes, the testnet begins MWEB signaling at block 840 and reaches a deterministic `ACTIVE` state at block 1260 (guaranteed activation; see RIP-0004).

---

## ⚙️ Proof-of-Work Algorithm: RinHash

RinHash is a custom proof-of-work algorithm using:

1. **BLAKE3**: Fast initial hashing
2. **Argon2d**: Memory-hard step that broadens hardware accessibility and supports fair early distribution
3. **SHA3-256**: Final standard cryptographic hash

A valid block satisfies:
`SHA3-256( Argon2d( BLAKE3(block_header) )) < Target`

This design provides fast verification, a memory-hard core that broadens early mining participation, and seamless compatibility with existing 256-bit PoW frameworks.

---

## 🌐 Network and Usage

- **Magic bytes:** see `src/chainparams.cpp` (documented in the table above for reference)
- **Ports:** `9555` (P2P), `9556` (RPC)
- **Mining:** Accessible to general-purpose hardware (CPU/GPU)
- **Wallet:** Full-node wallet with RIN units

---

## 🧭 Network Evolution & Mining Policy

Rincoin's Proof-of-Work parameters are treated as a phase-dependent engineering matter, not a permanent ideological commitment. The current RinHash configuration prioritizes broad, accessible participation and fair distribution while the network is establishing its hashrate base and economic weight.

As the network matures, the security model is expected to rest increasingly on the emission schedule, the accumulated economic weight of the chain, and a sustained long-term security budget. Rincoin makes **no permanent guarantee about supported or excluded mining hardware classes**. Any future change to consensus-relevant PoW parameters is an engineering decision subject to the project's formal improvement-proposal process, simulation-based verification, and maintainer approval, with advance notice to node operators and mining participants.

---

## 🚀 Release: v1.1.0

v1.1.0 is the *Customized Halving* release. It supersedes v1.0.6 and carries the protocol-level changes validated at a 1/1000 scale in Rincoin-Sim. Every consensus change is specified in a formal Rincoin Improvement Proposal (RIP). The version jumps from `1.0.6` to `1.1.0` (the MAJOR field) because this release introduces a hard fork; per RIP-0001, a hard fork is not shipped as a maintenance (MINOR) release.

### A. Customized Halving Hard Fork (Block 840,000), RIP-0002 / RIP-0010

At block 840,000, the block subsidy transitions from 6.25 RIN to a fixed 4.00 RIN (Phase 4 of the emission schedule), slowing subsidy decay to support a long-term security budget. `GetBlockSubsidy()` in `src/validation.cpp` expresses the phase boundaries as multiples of `nSubsidyHalvingInterval` (4×/10×/20×/30×), so mainnet (interval 210,000) and testnet/regtest (interval 210) share one rule with proportional timing. The activation height is `consensus.nRinHashForkHeight = 840000` in `src/chainparams.cpp`.

### B. RIN3 Transaction-Version Enforcement (Block 840,000), RIP-0009

From block 840,000, standard transactions must carry the replay-protection version marker `RIN_FORK_TX_VERSION = 0x52494E33` (ASCII "RIN3"). Enforcement is dual-layer: at mempool entry (`PreChecks`) and at block connection (`ContextualCheckBlock`). Coinbase, HogEx, and MWEB-only transactions are exempt (system-generated, not replay-attack targets). The error code on violation is `bad-tx-rinhash-version`. The same activation height (840,000) is shared with the CH hard fork above.

> The two changes above (CH dilation and RIN3) are the **only** consensus rules that activate at block 840,000. Minimum peer-version enforcement, previously scoped to this fork, has been separated; see the Roadmap section.

### C. MWEB & Taproot Mainnet Sealing, RIP-0004 / RIP-0011

Mainnet MWEB and Taproot are sealed to `NEVER_ACTIVE`, replacing the Litecoin-inherited height-based activation values (MWEB `nStartHeight = 2,217,600`; Taproot `nStartHeight = 2,161,152`). MWEB is deferred pending the BDB → SQLite descriptor-wallet migration; Taproot is deferred because its inherited activation heights are incompatible with Rincoin's corrected BIP9 window. Sealing before mainnet reaches those inherited heights prevents MWEB from activating under the BDB-locked wallet path.

### D. BIP9 Window Correction (mainnet)

| Item | Previous value | Corrected value | Impact |
| :--- | :--- | :--- | :--- |
| `nMinerConfirmationWindow` | `8064` (upstream) | `7920` (Rincoin: 1,980×4) | BIP9 soft fork activation window |
| `nRuleChangeActivationThreshold` | `6048` | `5940` (75% of 7,920) | BIP9 signaling threshold |

The MWEB/Taproot sealing and the window change land in the **same commit**, so no live BIP9 deployment ever observes the new window retroactively (atomicity guarantee).

### E. Upstream Parser Fix, Litecoin #1095

The `-vbparams` parser in `chainparams.cpp` is fixed to zero-initialize the `nStartHeight`/`nTimeoutHeight` locals (previously undefined behavior on the legacy 3-argument form). This is the Rincoin-side application of a root-cause fix contributed upstream as [litecoin-project/litecoin#1095](https://github.com/litecoin-project/litecoin/issues/1095).

### Hard fork coordination

The block-840,000 hard fork requires advance coordination: at minimum 30 days' notice to seed operators and mining pools, and block-height monitoring from block 800,000. Activation timing and migration details are published in the corresponding RIPs and the technical roadmap. All consensus changes are verified at a 1/1000 scale in Rincoin-Sim before mainnet deployment.

---

## ⏭️ Roadmap (post-v1.1.0)

The following items are scoped but are **not** part of v1.1.0. Each remains subject to the formal RIP process, simulation-based verification, and maintainer approval.

| Item | Notes |
| :--- | :--- |
| **Minimum peer version 70018** (peer gate) | Originally scoped to coincide with the block-840,000 hard fork, now separated into its own peer gate with **independent activation timing** (to be determined). It is **not** tied to block 840,000. |
| **Guix reproducible builds** | Deterministic build environment for third-party verification (includes Argon2d packaging and cross-compilation support). Scoped as a separate effort. |
| **`netaddress.h` naming artifacts** | Residual upstream identifiers (`BITCOIN_NETADDRESS_H`, `bitcoin-config.h` include, copyright header) remain. No functional impact on network sovereignty; cleanup is cosmetic. |
| **SQLite descriptor-wallet migration** | Precondition for any future MWEB mainnet re-activation. Tracked against upstream Litecoin's SQLite + MWEB integration. |

---

## 🛠️ Building Rincoin

For detailed build instructions for Linux and Windows, see the build notes in `doc/` (below).

**Quick start for building from source:**
- [Linux/Unix Build Notes](doc/build-unix.md)
- [Windows Build Notes](doc/build-windows.md)

---

## 💻 Developer Notes

- See `src/chainparams.cpp` for network configuration (ports, magic bytes, difficulty algorithm, address prefixes) and per-network consensus parameters (`nRinHashForkHeight`, BIP9 window, MWEB/Taproot deployment state). This file is authoritative.
- See `src/netaddress.h` for `INTERNAL_IN_IPV6_PREFIX` (`FD 2D DD 82 F5 C8`, confirmed `sha256("rincoin")`-derived). Minor naming artifacts (`BITCOIN_NETADDRESS_H`, `bitcoin-config.h`) remain but are functionally inert.
- See `src/primitives/block.cpp` for the `GetPoWHash()` RinHash implementation.
- See `src/validation.cpp` for `GetBlockSubsidy()` (CH emission logic) and the dual-layer RIN3 enforcement.
- See `src/primitives/transaction.h` for `RIN_FORK_TX_VERSION` (`0x52494E33`).
- The current release contents are tracked in the **Release: v1.1.0** section; deferred work is in the **Roadmap** section above.

---

## 💬 Community

Join the official Rincoin community to stay updated, get support, and discuss development:

[![Discord Banner 2](https://discord.com/api/guilds/1354664874176680017/widget.png?style=banner2)](https://discord.gg/H4Du5YuqFa)
