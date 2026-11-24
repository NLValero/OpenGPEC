# OpenGPEC

A free, open attempt at reading and tuning the Continental GPEC2 — the engine computer in a lot of Chrysler, Dodge, Jeep and Ram vehicles with the 3.6 Pentastar.

Right now, if you want to pull the stock file off one of these, change a table, and write it back, you pay. HP Tuners, DiabloSport, Autotuner, FoxFlash, or you mail the module off to someone who unlocks it. There's no open way to do it. That's the whole reason this project exists.

Everything here is for research and off-road use. It is not emissions-legal on the road, and I'm not pretending otherwise. The repo ships with no factory keys and no copyrighted calibration data — you bring your own.

## The ECU

The target is the Continental GPEC2. Inside it is an NXP MPC5566, which is a PowerPC chip (the MPC55xx / Qorivva family), with about 3 MB of flash. That detail matters more than it sounds like it should — see below.

My test car is a 2012 Dodge Journey, 3.6 V6. The PCM on it is part number `68092683AE`, hardware `05150656AA`. The later and more common cousin of this ECU is the GPEC2A, which uses the MPC5674F — also PowerPC.

## Where things stand

So far this is really just the homework. I spent Phase 0 figuring out what already exists so I don't rebuild things, and writing down what I found. That lives in [WHAT-EXISTS.md](WHAT-EXISTS.md), and it's worth reading before anything else.

The short version:

- Nobody has cracked GPEC2 out in the open. Every route I could find still runs back to a paid tool or a paid service.
- The chip is PowerPC, not TriCore like I originally thought. That's good news — PowerPC is much better documented, and the tooling is far more mature.
- There's a crew on pcmhacking.net actively reverse-engineering the *sibling* chip (the MPC5554, used in the older NGC4) and giving it away for free. Same chip family as ours. Those are the people worth talking to.
- [OpenTune](https://github.com/ebjerk01/opentune) already handles a lot of the boring plumbing — reading flash containers, importing definition files — and it's GPL-3 like this. But its checksum code only covers Bosch, VAG and Mercedes ECUs. There's nothing for GPEC2, so that part is on us.

## The plan

Rough order:

0. **Map the territory** — done. Figure out what's already been done. (WHAT-EXISTS.md.)
1. **The safe tooling** — a GPEC2 definition file and a map viewer, built on OpenTune's framework, with the GPEC2 checksum added. No security involved, nothing legally touchy, and useful to people right away.
2. **Find the maps** — with real stock files, locate the tables I actually care about: the neutral rev limit, the shift points, and the cruise fueling.
3. **The read** — pulling a full image off a locked module on the bench. This is the first real wall, and it only ever happens on a junkyard ECU, never the car I drive.
4. **Security** — the seed-key handshake and the write signature. This is the hard research, and the longest part.

The reason the boring stuff comes first is simple: 1 and 2 are things people can use before anything gets cracked, and that's how you get other people interested enough to help with 3 and 4.

## Don't brick your ECU

A few hard rules, learned the usual way:

- Read and save the stock file first, and keep more than one copy.
- Do all your experimenting on a spare junkyard module. Never the one in the car you need to drive.
- Keep steady 12 V on the module the whole time you're writing to it — a bench supply holding voltage, not a charger swinging up and down. Bricks come from the write dropping out, not from the size of the edit.
- Recalculate and check the checksum before every write.
- Leave the bootloader alone until Phase 4 actually understands it. If you corrupt that, a backup won't save you.

## License

GPL-3. See [LICENSE](LICENSE). Keys and calibration data are yours to supply — none of that ships here.

## Want to help?

It's early, so the useful things are specific:

- A GPEC2 connector pinout — power, ground, the CAN/serial pins the boot mode listens on, and the pin that forces serial boot.
- A junkyard GPEC2 to actually run the read experiment on (that's §5 of WHAT-EXISTS.md).
- Anyone already working on MPC55xx / NGC4 who wants to compare notes.
