# 2019 F-150 IPC tach reflash — the Ford UDS SecurityAccess (0x27) keygen, L3 secrets, which Ford software by model year, and why Level-1 is the real wall

**TL;DR** — The legacy Ford SecurityAccess algorithm is a known 24-bit LFSR, and **Level-3 (config/As-Built) secrets are trivial to recover** by sniffing one unlock and solving. **Level-1 (firmware-flash) is the wall**: you can only sniff an L1 unlock *while a module is actually being flashed*, and the Ford software will only flash a module that is **behind on its calibration**. If everything's up to date, there is no way to make it write, so there's nothing to sniff. The L1 secret is **not** in the downloadable firmware files either — it lives in the module's resident primary bootloader in censored flash. Posting this so the next person doesn't spend the money/time to re-discover it.

---

## What I was actually trying to do
I put a set of aftermarket **camshafts** in my 5.0L Coyote — they make peak power around **8,100 RPM** and I run them toward an **~8,500 RPM** redline. The stock F-150 cluster tops the tach scale at **7,000**, so past that it's meaningless. The plan was to run an **8,000-RPM-scaled cluster** (the scaling exists on other Ford variants) and/or **reflash the IPC firmware** so the tach sweep is correct for the new RPM range. Purely cosmetic/informational — **not** touching odometer, VIN, or anything safety-related. Turns out the tach *scaling is in firmware*, not in As-Built config — which is why this became an L1 (flash) problem.

---

## 1) Which modules use which algorithm
On a 2019 F-150 (HS-CAN1, 500k, 11-bit), probing SecurityAccess (service `0x27`) sorts modules into two buckets:

- **Legacy 24-bit LFSR (SOLVABLE):** 3-byte seed, 3-byte key. This is the one we can beat. Confirmed on **IPC, RCM, OCS, PSCM**, and others reject with NRC `0x35` (= our algorithm, wrong key → crackable by capture+solve).
- **Different algorithm (NOT this one):** longer multi-byte key, rejects with NRC `0x13` (wrong key *length*). Seen on **ABS and IPMA (fwd camera)**. This is a different/stronger scheme — the LFSR solver does not apply.

**We only have the equation for the LFSR bucket.** The `0x13` modules are a separate problem we did not solve.

---

## 2) The LFSR keygen — the actual equation
It's a **24-bit Galois LFSR**. Two constants define it (same for every legacy Ford module):

- **Initial state (loaded before anything):** `0xC541A9`
- **Feedback taps:** bits **20, 15, 12, 5, 3** (= `0x109028`)

**How one tick works:**
1. `feedback = (next input bit) XOR (bit 0, the bit shifting out)`
2. Shift the whole 24-bit register right by one.
3. If `feedback == 1`, flip the 5 tap bits (20/15/12/5/3). If `0`, don't.
4. The `feedback` bit itself enters at bit 23.

**The full run (64 ticks):**
- Load `0xC541A9`.
- Clock in the **24 seed bits** (ticks 1–24), then the **40 secret bits** (ticks 25–64). Seed = 3 bytes, secret = 5 bytes, 24 + 40 = 64.
- After 64 ticks, take the final 24-bit register and **rearrange its 6 nibbles in a fixed order** → that's your 3-byte key. (The nibble shuffle is just one more baked-in constant; no new math.)

**Verified published test vector:**
```
seed = 1F 7C 69   secret = 00 00 FA 5F C0   ->   key = 9A 64 CE
```
(Matches two independent public implementations across thousands of random vectors.)

A real run — seed `73 88 A2` + a secret, 64 ticks, watch the bits shift and the gold taps fire, key `50 88 B5` drops out:

![LFSR keygen running](LFSR_RUN.gif)

---

## 3) The compact "plug-and-solve" form
Because every operation is XOR (it's linear over GF(2)), the whole thing collapses to:

```
key = (A · seed)  XOR  (B · secret)  XOR  C
```
- `C = 0x293F78` (what `0xC541A9` becomes after 64 zero-input ticks — a fixed constant)
- `A`, `B` are fixed bit-selector matrices (24×24 and 24×40). Each output key bit = XOR of a fixed set of seed bits + secret bits + a constant bit.

**Why this is the whole ballgame:** flip it around —
```
B · secret = key  XOR  (A · seed)  XOR  C
```
**One captured `(seed, key)` pair** gives 24 XOR equations in the 40 unknown secret bits → solve by Gaussian elimination. 40 unknowns, 24 equations → **2^16 = 65,536 secrets all satisfy it, and every one unlocks.** You can never recover the *unique* factory secret from bus captures, but you don't need to — any class member works for all seeds.

**Note for anyone with multiple modules:** each module's 65,536-member class is **disjoint** from the others (different `B·secret` signature). Knowing one module's secret tells you nothing about another's. Same algorithm, different locks.

---

## 4) L3 secrets recovered (proof the method works)
- **IPC (cluster), req `0x720`/resp `0x728`:** L3 secret = `00 00 4E 65 6F`  ← the cosmetic target, safe to share.
- **RCM / OCS / PSCM:** we recovered valid L3 class members for all three too (same method, one capture each), confirming it generalizes. The secrets for the modules are 0000E8FEB8, 00001DE305, 0000011DE4 respectively.

---

## 5) How to sniff an unlock (hardware + the gotchas that cost us time)
- **Rig:** an OBD2 **Y-splitter**, one adapter running the Ford tool / J2534, a second adapter **passively monitoring** (`ATMA`).
- **GOTCHA 1 — monitor protocol:** on an **OBDLink EX (STN chip)** you must use the **STN-native setup** (`STP53` / `STPBRxxxxxx` / `STPO`), **not** `ATSP6`. With `ATSP6` the EX captures **zero** frames in monitor mode. This cost us a lot of confusion.
- **GOTCHA 2 — you must be on the module's bus.** The OBDLink EX only taps **HS-CAN (pins 6/14)**. Modules on another bus are invisible. The IPC is on 6/14 (worked). **APIM/SYNC is on a different bus** (HS-CAN3), so the EX never sees it — and a J2534 tool *can* reach it, which fooled us into thinking the sniffer was broken.
- **GOTCHA 3 — APIM/SYNC updates over USB, not CAN.** SYNC firmware is a signed package you load from a **USB stick**; the unit flashes itself. **No CAN SecurityAccess happens**, so there is nothing to sniff for the APIM no matter what bus you tap. Don't chase it.

L3 (As-Built/config) unlocks are easy to catch this way with FORScan doing the config write. The seed/key is at the very start of the exchange, before any bulk traffic.

---

## 6) Which Ford software by model year (this was genuinely confusing — get it right)
All come from the Ford Diagnostic Subscription (~**$120 / 2-day**) at motorcraftservice.com.

- **IDS (Integrated Diagnostic Software):** ~**1996–2018** model years. **Does NOT cover 2019.**
- **FDRS (Ford Diagnostic & Repair System):** the current-generation native tool for **newer** vehicles (2018+/2021+ platforms).
- **FJDS (Ford J2534 Diagnostic Software):** **module reprogramming for 2018+** via a **J2534 pass-thru** device. **This is the one that reached the 2019 F-150 IPC Module Reprogramming for us.**

For a **2019 F-150**: use **FJDS** (with a J2534 interface) for reprogramming. IDS is the wrong tool (no 2019 coverage). We also hit a hardware wrinkle: older **J2534 clones (VXDIAG, ~2021 driver library)** could not run the newest IDS version due to a measurement-library mismatch — FJDS was happier.

---

## 7) The $120 subscription reality — the Level-1 wall
This is the part nobody tells you before you pay:

- **There is NO way to just open and edit firmware** in the Ford software. It is not an editor.
- It **reads your modules**, compares each module's current calibration to what's available, and **only offers to reprogram a module that is BEHIND**.
- **During that reprogram, the L1 SecurityAccess unlock happens on the bus — THAT is the only moment you can sniff an L1 seed/key.**
- If every module reports **"No later calibration available"** (everything up to date), the software gives you **no write option at all**. No flash → **no L1 unlock → nothing to capture.**

So to sniff **L1**, you need a module that is **genuinely behind on its calibration**. On a well-maintained truck, you may have none. We did not — every cluster we tried (including spares) was current.

---

## 8) Why Level-1 is especially hard (even with the firmware files)
- The IPC MCU is an **NXP SPC5645S** (PowerPC **VLE** core) — a dedicated instrument-cluster chip.
- L1 (programming) SecurityAccess is handled by the **resident primary bootloader** in the MCU's **protected/censored flash** — **Ford does not distribute that as a VBF.** The downloadable files are: the **app/EXE** (handles only L3), the **SBL** (a RAM flash-driver that runs *after* unlock — no secret), and the **graphics DATA** partition. We confirmed the L1 secret is in **none** of them.
- The chip's **censorship** (flash read-out lock, factory-set) blocks a straight JTAG/BDM dump, and clearing it typically **mass-erases**. Getting the bootloader out realistically means a **voltage-glitch attack** — a real but uncertain bench project, no published turnkey recipe for this exact part.

---

## Bottom line for the next person
- **L3 (config) secrets:** easy. Sniff one unlock, solve the LFSR, done.
- **L1 (firmware) secret:** you need **a module that's behind on calibration** to trigger a flash you can sniff, OR a **chip-level dump** of the censored bootloader. No behind-module + no chip attack = **you cannot get L1** from the subscription software, and it's **not in the firmware files**.
- **If your goal is a cosmetic IPC reflash (tach rescale, gear display, splash):** it's an **L1** job. Budget for either finding a cluster that's behind on calibration, or a bench glitch of the SPC5645S. The $120 subscription alone will not do it if your cluster is current.

Hope this saves someone the weekend and the money. Happy to answer questions.

---

*Disclaimer: this was all done on my own vehicle for legitimate repair/modification (tachometer rescale for aftermarket cams). No odometer, VIN, or safety-system tampering. Check your local laws before modifying any vehicle module.*
