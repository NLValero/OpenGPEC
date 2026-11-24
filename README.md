# OpenGPEC

**An open-source firmware-read and calibration-tuning toolkit for Continental GPEC2 engine control units** (Chrysler / Dodge / Jeep / Ram — Pentastar 3.6L and related).

> ⚠️ **Research / off-road use only.** Not emissions-legal for on-road use. This project ships **no factory keys and no copyrighted calibration data** — you supply your own.

---

## Why this exists

Reading and tuning a Continental GPEC2 ECU today means paying into a closed ecosystem — HP Tuners, DiabloSport, Autotuner, FoxFlash, or a mail-in unlock service. There is **no free, open route** to pull a stock image off a locked GPEC2, edit the calibration, and write it back.

OpenGPEC aims to be that route: read a stock image off a locked GPEC2, edit calibration tables, and write it back safely, with no proprietary paid tool in the loop.

## The target

| Item | Value |
|---|---|
| ECU family | Continental **GPEC2** |
| MCU | NXP/Freescale **MPC5566** — e200z6 **PowerPC** core (Qorivva / MPC55xx family), ~3 MB flash |
| Example module | 2012 Dodge Journey 3.6L V6 — PCM P/N `68092683AE`, hardware `05150656AA` |

GPEC2A is the later, more-common sibling (MPC5674F — also PowerPC).

## Status — Phase 0 complete

This repo currently holds the **Phase 0 "what exists" map** — a survey of the community state, the confirmed hardware, why PowerPC is a tractable target, and the concrete first experiment for the read.

👉 **Start here: [WHAT-EXISTS.md](WHAT-EXISTS.md).**

Key findings:
- **GPEC2 is still uncracked in the open.** Every current read/unlock route is paid.
- The chip is **PowerPC (MPC5566), not TriCore** — far better documented and tool-supported.
- An active pcmhacking.net crew is already cracking the **sibling** chip (MPC5554 / NGC4) for free. Same MPC55xx family — adjacent allies.
- [OpenTune](https://github.com/ebjerk01/opentune)'s container/definition framework is reusable, but its checksum engine covers **Bosch/VAG/Mercedes only** — the **GPEC2 checksum is work we must add.**

## Roadmap

| Phase | Goal |
|---|---|
| **0 — Map the territory** ✅ | Survey community work, confirm hardware, scope the read. |
| **1 — Non-security tooling** | GPEC2 definition file + calibration map viewer. Reuse OpenTune plumbing, add GPEC2 checksum. Zero legal risk, immediately useful. |
| **2 — Document the maps** | Locate the tuning targets (neutral rev limit, shift tables, cruise fueling) in real stock dumps. |
| **3 — The read** | Bench / BAM boot-mode extraction off a **junkyard** module. First hardware wall. |
| **4 — Security layer** | Seed-key (UDS 0x27) + write signature. The research frontier. |

Phases 1–2 deliver community value *before* any cracking — which is what attracts the collaborators who make Phases 3–4 feasible.

## Safety conventions

- Always read and save the stock image first; keep multiple backups.
- Practice every read/write on a **spare junkyard module**, never the running car's ECU.
- Keep a steady 12 V supply on the module during any flash (steady-voltage mode, not a swinging charge cycle).
- Recalculate and verify checksums before every write.
- Treat the bootloader / boot region as untouchable until Phase 4 understands it.

## Legal & licensing

OpenGPEC is licensed under the **GNU General Public License v3.0** — see [LICENSE](LICENSE).

The tool ships empty: **no factory keys, no copyrighted calibration data.** The caller supplies their own. Intended for off-road / research use; not emissions-legal for on-road use.

## Contributing

This is early. The most useful things right now:
- A **GPEC2 connector pinout** (power/ground, eSCI/CAN BAM pins, boot-config strap).
- A **junkyard GPEC2** to run the BAM read experiment on (see WHAT-EXISTS.md §5).
- Anyone from the MPC55xx / NGC4 reverse-engineering world who wants to compare notes.
