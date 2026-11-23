# OpenGPEC — Phase 0: "What Exists" Map

*Last updated: 2026-11-23. This is the Phase 0 deliverable: a snapshot of the community state, the confirmed hardware target, and the concrete first move for Phase 3.*

---

## 0. TL;DR

- **The test car's ECU is confirmed: Continental GPEC2** (NOT NGC4, NOT GPEC2A).
- **The chip is NXP MPC5566 — PowerPC, not Infineon TriCore.** CLAUDE.md's original "TriCore/AURIX" hardware line was wrong and has been corrected.
- **GPEC2 is still uncracked in the open.** No free read/unlock exists; every route is still paid. This is the gap OpenGPEC fills — unchanged.
- **The good news:** PowerPC MPC55xx is far more documented and tool-supported than TriCore, and an active pcmhacking crew is cracking the *sibling* chip (MPC5554 / NGC4) for free right now. The ladder is good even though the wall is real.
- **The one question that sets Phase 3 difficulty:** does this GPEC2 accept the **public BAM boot password** or a custom one? (Test plan in §5.)

---

## 1. Confirmed hardware target

| Item | Value |
|---|---|
| Vehicle | 2012 Dodge Journey, 3.6L Pentastar V6 |
| ECU family | **Continental GPEC2** |
| PCM service P/N | **68092683AE** (read off the module label) |
| Hardware code | **05150656AA** |
| MCU | **NXP/Freescale MPC5566** — e200z6 **PowerPC** core, Qorivva/MPC55xx family |
| Flash | ~3 MB (GPEC2 reads ~3 MB; GPEC2A/GPEC3 ~4 MB) |
| Visual tell | GPEC2 = grey + brown connector locking tabs |

**How it was identified:** the PCM label (not the VIN — the VIN alone doesn't encode the ECU). The module first photographed was the **TIPM** (`68143314AC`, the fuse/power box) — a common mix-up; the real PCM is the finned aluminum module with the engine harness.

**Watch out:** the "4 connectors = GPEC2" heuristic is unreliable — it fooled a pcmhacking member (and an AI) into calling an NGC4 a GPEC2. Trust the part number or the CPU marking, not connector count.

CPU-marking cross-check (if the lid ever comes off): `MPC5554` = NGC4 · `MPC5566` = GPEC2 · `MPC5674F` = GPEC2A.

---

## 2. Community state — what's already done (don't rebuild)

### OpenTune — reuse the framework, but it does NOT cover us
[github.com/ebjerk01/opentune](https://github.com/ebjerk01/opentune) — GPL-3, ships no keys. Good for: flash-container decode (**FRF/ODX/ODX-F/SGO/VBF**), definition import (**TunerPro XDF, ASAM A2L, EcuFlash/RomRaider XML, WinOLS .ols**), plus an A2L→XDF converter and a checksum *framework*.
- ⚠️ **Its ChecksumAlgo covers Bosch/VAG/Mercedes only** (EDC15/16/17, Simos PCR2.1, ME7, DQ-series TCUs). Its "TriCore" = **Bosch EDC17**, not ours. **No GPEC2, nothing Chrysler.**
- **Takeaway:** reuse the container/definition plumbing; the **GPEC2 checksum algorithm is work OpenGPEC must add** (candidate upstream contribution).

### Stock dumps on GitHub — nothing for GPEC2 yet
- [Moddler/chrysler-stock-calibrations](https://github.com/Moddler/chrysler-stock-calibrations) — real Chrysler dumps, but **1980s/90s LM/SMEC/SBEC only.** Wrong era.
- [kaivyy/ApexECUVault](https://github.com/kaivyy/ApexECUVault) — broad multi-brand vault (BMW/VAG/GM/Ford/Toyota, some TriCore/AURIX) but **no Chrysler/Mopar section at all.**
- **No public GPEC2/Pentastar stock image exists.** Sourcing one (yours, or a junkyard module) is a hard dependency for Phase 2 map work.

### CalibraX
[github.com/davgeek/calibrax-ecu](https://github.com/davgeek/calibrax-ecu) — 1996–2005 JTEC (68HC16) tuner. **UX/table-editor reference only.** Not our chip.

### The GPEC2 security crack — still entirely closed / commercial
- Every route is paid: bench/boot unlock services, mail-in unlocks (~$150 online / ~$250 mail-in), and OBD-patch tools (**FoxFlash, Autotuner, Alientech KESS3/K-TAG, Magic Flex, BitBox, KT200**).
- **bFlash published a GPEC2/GPEC2A boot protocol in early 2024** (~5 min read) — but it's a paid tool, not open.
- The 0x27 **seed-key algorithm is nowhere public.**

### pcmhacking.net — the active Chrysler RE is NGC4, not GPEC2
(Engineering & Reverse Engineering section. Note: the forum is behind Cloudflare and blocks automated fetching — read it from a real browser.)
- **Title search "GPEC2" = 0 hits; "Pentastar" = 0 relevant.** GPEC2 has essentially no RE presence here.
- The only GPEC work is **GPEC3** (user *The_Lishp* — locked modules, EFD files, a "security unlock script"; SKIM still unbeaten except via commercial ECU Villain / ECU Hero).
- **The live effort is [Chrysler NGC4 (MPC5554)](https://pcmhacking.net/forums/viewtopic.php?t=9277)** (started May 2026, 85+ posts) + a [MPC5554 Disassembly Project](https://pcmhacking.net/forums/viewtopic.php?t=6696).
  - Opening line: *"Apparently there are exactly zero free software options for anything Chrysler. Hoping to get the ball rolling."* — confirms OpenGPEC's premise.
  - **usbbdm** built a working **u-Link NT JTAG read/write** for NGC4: read 2 MB flash <4 s, EEPROM r/w <1 s, flash program ~22 s, 3D-printed JTAG jig.
  - Stock NGC4 bins are shared openly in-thread.
  - **Tazzi** (serious RE) names the real gap — and it *is* our Phase 1–2: *"All FCA ECMs have inbuilt bootloaders that handle reflashing. The part no-one has done work with is breaking down the OS/Cal sections, segment checksums, algos."*
- **Why this matters to us:** NGC4 = MPC5554 = **same MPC55xx PowerPC family** as our MPC5566. Their toolchain, JTAG jigs, and Ghidra setup are directly adjacent. These people (AngelMarc, usbbdm, Tazzi) are the collaborators Phases 1–2 were meant to attract — and they already exist.

---

## 3. Why PowerPC (MPC5566) is a better target than a TriCore would've been

**Not because you can just plug in JTAG and dump it** — a stock module is locked. But the ecosystem around the chip is night-and-day vs. AURIX TriCore:

- **The read path is public and well-trodden: serial boot via the Boot Assist Module (BAM).** This is exactly what the paid tools do (*TOOLS BOOT MODE → MOTOROLA FREESCALE BAM → CONTINENTAL GPEC2*).
- **Deep public documentation:** NXP reference manuals, [AN3787 censorship/recovery note](https://www.nxp.com/docs/en/application-note/AN3787.pdf), full [Lauterbach Qorivva MPC5xxx debugger docs](https://www2.lauterbach.com/pdf/debugger_mpc5500.pdf), and mature **Ghidra PowerPC/VLE** support.
- **Public attack research:** the ["BAM BAM!!" EMFI paper](https://eprint.iacr.org/2020/937.pdf) targets exactly the MPC5566 BAM password check.
- **Community on the sibling chip** (MPC5554/NGC4) with working tooling.

**Correction to the old mental model:** neither GPEC2 nor GPEC2A is TriCore. GPEC2A is MPC5674F, *also* PowerPC. The real difficulty gap between the two is flash size / mask / mitigations, not PPC-vs-TriCore. We're on the more approachable one (GPEC2, 3 MB).

---

## 4. The lock, precisely (MPC5566 "censorship")

- When a MPC5566 is **censored**, internal flash is disabled to external access, and **JTAG/Nexus can read the chip ID but no memory**.
- **MPC55xx-specific:** you **cannot** unlock JTAG with a password on this family (Lauterbach: *"The unlock is supported by all processors but MPC55XX"*). So the JTAG-password route that works on newer chips is dead here.
- The way in is **BAM serial boot**, gated by a **64-bit password**:
  - **Public constant** `0xFEEDFACECAFEBEEF`, **or**
  - a **custom password** stored in the shadow flash row.
- If custom: recovery options are AN3787's shadow-erase routine (⚠️ risky — can wipe config), fault injection/EMFI (advanced; the "BAM BAM" paper found the public-password config is actually the *most* glitch-resistant), or recovering the password (the research wall).

---

## 5. Phase 3 — concrete first step: the BAM read experiment

**Goal:** determine whether this GPEC2 accepts the public BAM password, and if so, dump the flash. **This single test decides how hard Phase 3/4 are.**

> ⚠️ **Junkyard module only.** Never the daily-driver PCM. Keep a steady 12 V bench supply (steady-voltage mode, not a swinging charge cycle). Make backups of everything you read, immediately, multiple copies.

**Hardware you'll need:**
- A spare/junkyard GPEC2 (same `05150656`/`68092683` family ideal) on the bench.
- Bench power: 12 V steady supply + the module's power/ground pins (from the connector pinout).
- A MPC55xx-capable interface. Cheapest community route = the same class of tool the NGC4 crew uses. Options, in rough order of cost/openness:
  1. **BAM via serial (eSCI) or CAN** — BAM boots from a UART/CAN interface; a cheap USB-serial or CAN adapter + a reader stub may suffice if the public password works.
  2. **JTAG/Nexus pod** (e.g. a u-Link-class or PEmicro/Lauterbach-compatible probe) — only useful if the unit turns out *uncensored*; ask the NGC4 thread what worked for them.

**Procedure (outline — fill in pinout from the GPEC2 connector map before touching it):**
1. **Identify the boot/comm pins** on the GPEC2 connector (power, ground, the eSCI/CAN pins BAM listens on, and the boot-config pin that forces serial boot). Pull this from a pinout before anything else.
2. **Power the module** on the bench (12 V steady).
3. **Force serial boot (BAM)** via the boot-config strap.
4. **Send the BAM entry sequence with the public password** `0xFEEDFACECAFEBEEF`. (Note: byte order matters on this family — if it's rejected, try the reversed byte order before concluding it's custom.)
5. **Branch:**
   - ✅ **Accepted** → upload a small flash-reader stub to SRAM, dump all ~3 MB, save multiple copies. **Phase 3 is essentially open.** Proceed to map work (Phase 2) on the real dump.
   - ❌ **Rejected (both byte orders)** → the module uses a **custom shadow password.** That's the research wall. Document it, bring it to the NGC4/pcmhacking crew, and scope the recovery options (shadow-erase vs. glitching vs. password recovery) as the Phase 4 frontier.

**First non-hardware task that unblocks this:** get the **GPEC2 connector pinout** (power/ground + eSCI/CAN + boot-config pin). Without it, step 1 is guesswork. Check the NGC4 thread, Continental/Chrysler service docs, and vendor connection manuals.

---

## 6. Open questions / next actions

- [ ] **Public vs. custom BAM password?** (§5 — the decisive test.)
- [ ] **GPEC2 connector pinout** — power, ground, BAM comm pins, boot-config strap.
- [ ] Source a **junkyard GPEC2** to practice on.
- [ ] Introduce OpenGPEC to the **pcmhacking NGC4 crew** (AngelMarc, usbbdm, Tazzi) — shared MPC55xx family, potential collaborators.
- [ ] Phase 1 can start now in parallel: stand up the OpenTune container/def framework and begin a GPEC2 definition skeleton.

---

*Sources are linked inline. Forum content was read via a real browser (pcmhacking blocks automated fetches). Vendor claims (BitBox/FoxFlash/N2 Speed/etc.) are promotional and unverified; NXP and Lauterbach material is authoritative on the silicon.*
