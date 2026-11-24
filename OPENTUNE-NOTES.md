# OpenTune — reuse notes for OpenGPEC

*Written 2026-11-23 from a read of the `opentune-main` source (v1.0.0-beta.1). This is my working map of what OpenTune gives us for free and what we have to build ourselves, so we don't have to re-read the whole repo. Upstream: github.com/ebjerk01/opentune, GPL-3.0-or-later — compatible with OpenGPEC's GPL-3.*

---

## 0. The one-paragraph version

OpenTune is a **Rust workspace** (~55 crates) with a **Tauri desktop app** (Rust backend + web UI) and an Android app. It already has everything OpenGPEC's non-security phases need as *infrastructure*: a calibration map data model, a TunerPro XDF importer, a checksum engine with a clean plug-in trait, a flash-container decoder, a map-diff finder, and — critically — a **serial bootstrap-loader read** crate for a different CPU (`opentune-c167bsl`) that is a direct architectural template for the MPC5566 BAM read. What it does **not** have is anything for GPEC2 / MPC5566 / Continental-Chrysler. So our job is to *add a target*, not build a toolkit. The license matches, so our work can be upstreamed.

---

## 1. Stack / how to build it

- Rust, edition 2021, `rust-toolchain.toml` pins the toolchain, MSRV 1.75.
- Workspace manifest at repo root lists all crates under `crates/`.
- Desktop app: `desktop/` (Tauri — `src-tauri` Rust + `ui` web). CLI: `crates/opentune-cli` (`opentune …`).
- Standalone tools in `tools/` build with plain `rustc`, no deps.
- Our vendored copy lives at `OpenMopar/opentune-main/opentune-main/` (git-ignored; it's a reference, not part of our repo).

---

## 2. What we reuse vs. what we build

| Need | OpenTune gives us | We add |
|---|---|---|
| Calibration table model (address, type, endian, axes, scaling, min/max) | `opentune-core::ecudef` — `MapDef`, `AxisDef`, `Scaling`, `DataType`, `Endian` | nothing — just author data |
| ECU definition file | JSON in `ecudefs/`, schema + `TEMPLATE.json` | a `continental-gpec2-*.json` def |
| Import existing defs | `opentune-xdf` (TunerPro XDF), `opentune-a2l` (A2L), `opentune-ecuflash` (EcuFlash/RomRaider), `opentune-ols` (WinOLS) | nothing |
| Checksum verify/correct | `opentune-checksum` — `ChecksumAlgo` trait, `get(id)` registry, catalog, CRC helpers | **the GPEC2 checksum algorithm** (a new module) |
| Reverse-engineer an unknown checksum | `tools/checksum-search` (brute-forces config from ≥2 dumps) | run it on GPEC2 dumps |
| Find maps by diffing dumps | `opentune-mapfind` (`diff`, `relocate`) | run it; record addresses |
| Decode OEM flash containers | `opentune-container` (FRF/ODX/ODX-F/SGO/VBF, AES/compression) | only if we meet Chrysler containers |
| Serial boot-mode read (the Phase 3 read) | `opentune-c167bsl` — *template*, wrong CPU | **`opentune-bam`** for MPC5566 BAM |
| Seed/key (Phase 4) | `opentune-core::protocol::seedkey`, `opentune-sa2`, `opentune-seedkey-loader` | the Chrysler 0x27 algorithm |
| Flashing / write path (Phase 4) | `opentune-flash`, `docs/FLASHING.md` | GPEC2 write sequence |

**The single genuinely new engineering deliverable in Phase 1 is the GPEC2 checksum module.** Everything else in Phase 1 is authoring data into OpenTune's existing shapes.

---

## 3. The data model — `crates/opentune-core/src/ecudef.rs`

This is the heart. An `EcuDef` (one JSON file) carries: `id/name/maker/family`, `provenance`, `protocol/diag_layer/bitrate/tx_id/rx_id`, `flash_size`, `regions[]`, `checksum` (an algo id string), `seedkey`, `full_read`, `maps[]`, and `identify` (part/sw number prefixes).

Why it fits GPEC2 cleanly:

- **`Endian::Big` is the default.** PowerPC is big-endian, so our maps are right by default — unlike the Bosch/TriCore little-endian world most defs come from. (Still verify against a real dump; §2 of the XDF README warns endianness is historically ambiguous in XDF files.)
- **`DataType`**: `u8/s8/u16/s16/u32/s32/f32`. Byte width + sign; endianness is a separate axis.
- **`MapDef`**: `address` (a **decimal** file offset into the full image — the template is explicit that all numbers are decimal, convert hex yourself), `rows` (Y), `cols` (X), `scaling {factor, offset}` (linear only: `physical = raw*factor + offset`), `unit`, optional `x_axis`/`y_axis`, optional `min`/`max` safety clamps.
- **`AxisDef`**: three flavors mirroring A2L — inline `values`, a `stored` axis (`address`+`data_type`), or a shared `axis_ref`.
- **`Scaling` is linear-only by design.** Non-linear OEM scaling is intentionally out of scope; capture it as pre-scaled + a note. Fine for our rev-limit / shift / AFR targets.
- **`FullRead::BenchBoot`** is already a variant — i.e. the model expects some ECUs are read via bench/boot mode, not OBD. That's us.
- **`checksum`/`seedkey`** are just id strings resolved by the respective crates, so wiring GPEC2 is: write the algo, register its id, name the id in our def.
- **`identify.part_numbers`** keys an ECU to a def by part-number prefix (min 6 chars, longest match wins). We key on `68092683` / hardware `05150656`.

## 3a. Provenance / legal model — matches ours exactly

`provenance` is **required** for contributions: `sources[]` (citations), `license` (SPDX, normally `GPL-3.0-or-later`), and `evidence` ∈ `implemented | synthetic-tested | real-image-tested | bench-tested | vehicle-tested`. An `evidence: example` def is treated as placeholder and **never matches a real unit**. The template screams: never ship proprietary firmware, A2L/DAMOS, seed/key secrets, real VINs or plates; "own read, never committed." This is the same ships-empty posture CLAUDE.md mandates — we inherit it for free.

---

## 4. The checksum framework — `crates/opentune-checksum/`

Trait-based, one module per ECU family:

```rust
pub trait ChecksumAlgo {
    fn id(&self) -> &str;
    fn verify(&self, image: &[u8]) -> Result<Vec<BlockStatus>>;        // stored vs computed, per block
    fn correct(&self, image: &mut [u8]) -> Result<Vec<Fix>>;           // recompute + write in place
    fn correct_with_reference(&self, image, reference) -> …;           // bootstrap from a known-good stock dump
    // verify_report / correct_bad_blocks have sane defaults
}
```

- `get(id)` maps an id string → boxed algo; `algorithms()` is a data catalog (id, name, family, needs_reference, notes). **To add GPEC2: write `gpec2.rs`, add a `"gpec2"` arm to `get()`, add an `AlgoInfo` to `algorithms()`.** A test enforces `get()` and `algorithms()` stay in sync.
- Shared helpers already present: `crc32` (IEEE reflected), `crc32_forward(poly 0x04C11DB7)`, `crc32_bosch` (init 0). Big-endian CRC-32 is first-class here.
- **Anti-brick stance is baked in and non-negotiable:** a block whose correct value can't be computed is *never guessed* — it's left as-is or the image refused. `correct_bad_blocks` rewrites only mismatched blocks, leaving valid ones byte-for-byte. Matches CLAUDE.md's "recalc + verify before every write" rule. Honor it in our module.
- **`Recognition::Recognised` vs `Consistent`**: arithmetic agreement alone is never reported as "supported" — a family must be structurally identified. Good discipline; our GPEC2 module should only claim Recognised once validated on a real stock dump.
- **Closest existing references for us:** `simos.rs` and `pcr21.rs` are **Continental/Siemens** block-CRC schemes (self-describing `{init, crc, count, [start,end]...}` directory, Bosch forward CRC-32 over concatenated areas, header outside its own coverage so rewriting the CRC doesn't change covered data). GPEC2 is also Continental — if Chrysler reused a Continental-style secured-block layout, these are the closest-shaped prior art. `edc16.rs` also shows a `FADECAFE/CAFEAFFE` directory of big-endian sum blocks. Read these two first when we start the GPEC2 algo.

---

## 5. Reverse-engineering the GPEC2 checksum — `tools/checksum-search/`

A single-file, dependency-free Rust tool (`rustc --edition=2021 -O search.rs`). Give it ≥2 equal-length same-layout images and it brute-forces the config:

- `diff --edited <range> stock.bin modified.bin` → prints contiguous changed-byte runs (candidate checksum storage slots).
- `search --slot <off> --region <a>..<b> a.bin b.bin c.bin` → tries every config that reproduces *all* stored values; each successive image narrows candidates; empty output = no match.
- Search space: word sums (8/16/32-bit, both byte orders, sum / one's / two's complement / XOR), Fletcher-16, Adler-32, CRC-16 (0x8005, 0x1021) and CRC-32 (0x04C11DB7, 0x1EDC6F41) with init/xorout/reflection permutations, and a VAG EDC15-style accumulator.

**This is our Phase-1/Phase-5 checksum workflow.** The hard dependency: at least two GPEC2 images that differ in known bytes (ideally edit one byte in a cal region of a stock dump, or two stock dumps of different SW). Caution it flags: short sums collide — use independently-changed inputs and a held-out image to rule out coincidence.

---

## 6. The definition-authoring workflow

1. Copy `ecudefs/TEMPLATE.json` → `ecudefs/continental-gpec2-68092683.json`, set `id` to the filename (minus `.json`), delete every `_`-prefixed hint field.
2. Fill `identify.part_numbers` with `68092683` (and/or hw `05150656`).
3. Start with `checksum: null`, `maps: []` (a def with no maps is valid — identify + checksum still work; add maps in follow-ups). Every map's `notes` must carry a `CONFIDENCE: HIGH|MEDIUM|LOW` label or `EXAMPLE ONLY`.
4. `full_read: bench_boot`.
5. Validate: `opentune ecudefs validate ecudefs/<file>.json` (add `--image <stock-dump>` to bounds-check map addresses against a real read).
6. Importing community maps: `opentune import-xdf <file> --into ecudefs/<file>.json`. XDF importer notes to remember: `mmedtypeflags` bit 0x02 = little-endian (else big — our default), 0x04 = column-major (imported transposed), 0x10000 = float; non-linear math is pre-scaled + warned, never approximated; **verify endianness against a known-byte-order bin** before trusting a bulk import.

Relevant docs: `docs/ECU-DEFS.md`, `docs/MAP-DEFS.md`, `docs/CONTRIBUTING-ECUDEFS.md`, `docs/ecudef.schema.json`.

---

## 7. The Phase 3 read — `opentune-c167bsl` is our template

`docs/C167-BSL.md` + crate `opentune-c167bsl` do for an Infineon C167 (EDC15) exactly what we need for MPC5566: a **mask-ROM serial bootstrap read** that works when the normal programming session is locked. The architecture to copy:

- CLI shape `opentune bsl presets | plan | read`; `plan` prints the exact step sequence, `read` executes it.
- A **protocol engine + a simulator** (built/tested against a simulated BSL before any real hardware).
- **Refuses to run without `--i-understand`**; read-only, never writes/erases. (Mirror this for safety.)
- **Ships no proprietary loader/kernel** — the user supplies them (`--loader`/`--kernel`), recognized by SHA-256. Same ships-empty model we want for any GPEC2 loader stub.
- Per-board **presets** (region, CPU address, length, SFR setup), with sources/licenses cited in `docs/PRIOR-ART.md`.

Our equivalent would be an `opentune-bam` crate: MPC5566 BAM entry over eSCI/CAN, the `0xFEEDFACECAFEBEEF`-vs-custom password question (WHAT-EXISTS.md §5), upload a flash-reader stub to SRAM, dump ~3 MB. Build the simulator + planner first, hardware last — same as they did.

Other hardware/read docs worth reading: `docs/CHEAP-INTERFACES.md`, `docs/hardware.md`, `docs/FIRST-CAR-TEST.md`, `docs/FLASHING.md` + `docs/FLASHING-NOTES.md`.

---

## 8. Phase 4 seed/key — where to look later

`opentune-core::protocol::seedkey`, `opentune-core::protocol::uds` (service 0x27), `opentune-sa2` (ASAM SA2 seed/key bytecode), `opentune-seedkey-loader`, `docs/SEEDKEY.md`, `docs/SEEDKEY-LOADER.md`. The Chrysler 0x27 algorithm itself is still the closed research wall (WHAT-EXISTS.md) — these crates are the plumbing to *run* an algorithm once we have one, not the algorithm.

---

## 9. Highest-value docs to read next (in `opentune-main/opentune-main/docs/`)

- `MAP-DEFS.md` — the map format rationale, the full XDF import outcome table.
- `BOSCH-CHECKSUM.md` + `checksums-edc16.md` — worked checksum RE, the methodology to copy for GPEC2.
- `C167-BSL.md` — the bootstrap-read blueprint (see §7).
- `FIRST-CAR-TEST.md` — end-to-end "test on your own car" walkthrough.
- `CONTRIBUTING-ECUDEFS.md` + `ECU-DEFS.md` — how to get our def accepted upstream.
- `PRIOR-ART.md`, `IDEAS.md`, `AFTERMARKET-ECUS.md`, `PLUGINS.md` — context and the upstream wishlist.

---

## 10. Net for the Phase 1 plan

Phase 1 reduces to, in order:
1. **Author a minimal GPEC2 `EcuDef`** (identify keys, `bench_boot`, empty maps, `checksum: null`) — publishable immediately, zero dependency on a dump.
2. **Get ≥1 stock GPEC2 dump** — the gate for everything below (sourcing one is the critical path).
3. **Run `checksum-search`** on two dumps / a one-byte-edited pair → identify the GPEC2 cal checksum → implement it as `opentune-checksum/src/gpec2.rs` (model on `simos.rs`/`pcr21.rs`). **This is the one new deliverable and the best upstream contribution.**
4. **Run `opentune-mapfind`** to locate the three target maps (neutral rev limit, shift tables, cruise AFR) and write them into the def with confidence labels.
5. View/edit via the existing OpenTune desktop app or TunerPro (XDF round-trips) — no viewer to build.

Reuse the framework; the only real code we write is the checksum module and, later, the `opentune-bam` read crate.
