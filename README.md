# USB-C to Gigabit Ethernet Adapter

A small open-hardware dongle that adds a wired Gigabit Ethernet port to any host with a USB-C port. Built around the Microchip **LAN7800** (USB 3.x ↔ 10/100/1000 Ethernet bridge) and a **TI HD3SS3220** USB-C SuperSpeed mux for cable-orientation handling.

No drivers required on most modern hosts — the LAN7800 enumerates as a standard USB Ethernet class device (inbox drivers on Windows/macOS/Linux/Android/ChromeOS).

> **Status:** work in progress / learning project. Schematic is functional on paper but not yet fabricated or tested. See [Known Issues](#known-issues--open-items) before ordering boards.

---

## Table of Contents

- [Features](#features)
- [Hardware Overview](#hardware-overview)
- [Block Diagram](#block-diagram)
- [Bill of Materials (key parts)](#bill-of-materials-key-parts)
- [Repository Structure](#repository-structure)
- [Tools Used](#tools-used)
- [Building the Board](#building-the-board)
- [Known Issues / Open Items](#known-issues--open-items)
- [Bring-Up & Test Plan](#bring-up--test-plan)
- [References](#references)
- [License](#license)

---

## Features

- Full Gigabit Ethernet (10/100/1000BASE-T) over USB 3.x SuperSpeed (5 Gbps link)
- USB-C **receptacle** input — works with any standard USB-C cable
- Reversible-cable support via an active SuperSpeed mux (HD3SS3220) — no "flip it and try again"
- Driverless enumeration on modern OSes (standard USB Ethernet / CDC-ECM-class device)
- No external Ethernet PHY needed — LAN7800 integrates the Gigabit PHY and MAC
- Optional EEPROM for a unique factory-assigned MAC address (or run off internal OTP defaults for a first prototype)

## Hardware Overview

| | |
|---|---|
| Bridge IC | Microchip LAN7800-I/Y9X (48-pin QFN, exposed pad) |
| USB-C mux/CC controller | TI HD3SS3220RNHR (30-pin WQFN) |
| USB-C connector | Receptacle, SuperSpeed-capable (24-signal) |
| Ethernet connector | Gigabit RJ45 MagJack (integrated magnetics), sourced as an LCSC extended part |
| Power rails | 5V (VBUS) → 3.3V (LDO) + 2.5V (LDO, PHY analog). LAN7800 core rail (~1.2V) generated internally via integrated switcher + external inductor |
| Reference clock | Single 25 MHz crystal (18 pF load) |
| Board | 4-layer, controlled impedance (90Ω USB3 diff pairs, 100Ω Ethernet diff pairs) |

## Block Diagram

```
USB-C Receptacle → HD3SS3220 (SS mux + CC logic) → LAN7800 → RJ45 Gigabit MagJack
                          │                            │
                     Power tree ──────────────────┴── 25 MHz XTAL, EEPROM, LEDs
```

The HD3SS3220 detects cable orientation via CC1/CC2 and routes whichever physical SuperSpeed pair is live (of the two exposed by the receptacle) through to the LAN7800's single SuperSpeed TX/RX pair. USB 2.0 D+/D- run straight from the connector to the LAN7800 with no mux, since those lines don't change with cable orientation.

## Bill of Materials (key parts)

| Ref | Part | LCSC # | Notes |
|---|---|---|---|
| U1 | LAN7800-I/Y9X | C633488 | Main bridge IC |
| U2 | HD3SS3220RNHR | C165155 | USB-C SS mux + CC logic |
| U3 | 3.3V LDO (e.g. AP2112K-3.3) | C51118 | Main digital rail |
| U4 | 2.5V LDO (e.g. AP2127K-2.5) | — | Dedicated analog rail for Gigabit PHY, keep isolated from digital 3.3V |
| Y1 | 25.000 MHz crystal, 18pF load | C13738 (verify) | Single reference clock for both USB and Ethernet PLLs |
| U5 | I²C EEPROM (e.g. 24AA025E48) | C88445 | Optional — unique MAC address; can be skipped for prototype using internal OTP |
| J1 | USB-C receptacle, SuperSpeed | C165948 (verify) | Must expose both SS pairs, CC1/CC2, D+/D- |
| J2 | Gigabit RJ45 MagJack | Extended part | Not in JLCPCB basic library — footprint drawn manually from vendor mechanical drawing |

Full BOM with passives, decoupling, and strapping resistors is in [`docs/usbc_ethernet_adapter.pdf`](docs/usbc_ethernet_adapter.pdf).

## Repository Structure

```
.
├── docs/
│   ├── usbc_ethernet_adapter.tex     # Full design document (LaTeX source)
│   └── usbc_ethernet_adapter.pdf     # Compiled design document
├── hardware/
│   ├── kicad/                        # KiCad project (schematic, PCB, footprints)
│   └── footprints/                   # Custom footprints (e.g. Gigabit MagJack)
├── README.md
└── LICENSE
```

*(Adjust this section to match your actual folder layout once the KiCad project is committed.)*

## Tools Used

- **EDA:** KiCad 8.x
- **Fabrication / assembly:** JLCPCB (PCB + SMT assembly)
- **Sourcing:** LCSC (basic + extended parts)

## Building the Board

1. Clone this repo and open `hardware/kicad/*.kicad_pro` in KiCad.
2. Verify every IC pin number against the current datasheet revision before trusting the schematic symbols:
   - LAN7800: Microchip DS00001992 (current rev. H)
   - HD3SS3220: TI SLLSE21 (current rev. E)
3. Draw/import the Gigabit MagJack footprint (see [Known Issues](#known-issues--open-items)) before running DRC.
4. Set stack-up to 4-layer and enable controlled impedance in JLCPCB's order options; target 90Ω differential for USB3 pairs and 100Ω differential for the Ethernet MDI pairs.
5. Run DRC, export Gerbers + BOM + CPL (pick-and-place), and submit to JLCPCB.

## Known Issues / Open Items

- [ ] **VDD5 on U2 (HD3SS3220) is currently wired to 3.3V — must be corrected to 5V/VBUS.** VCC33 (pin 8) stays on 3.3V; VDD5 (pin 30) needs its own connection to the VBUS rail.
- [ ] Confirm **ENn_CC (pin 29)** is tied to GND — required for the CC logic block to be active at all.
- [ ] Confirm **ADDR (pin 22)** is left floating/NC for GPIO mode (tying to GND instead selects I²C mode at a fixed address — not wrong, just a different intended config).
- [ ] Gigabit MagJack footprint not yet verified against a physical part / mechanical drawing.
- [ ] LAN7800 configuration strap table (USB/PHY mode selects, EEPROM size select) not yet finalized — copy exactly from datasheet Section 4 before final routing.
- [ ] No board has been fabricated or bench-tested yet.

## Bring-Up & Test Plan

1. **Power-only check** (no cable): confirm no shorts VBUS→GND before ever connecting a host.
2. **Power sequencing**: verify 3.3V and 2.5V rails are stable under a bench supply/host.
3. **Enumeration**: connect to a host, check `lsusb` / Device Manager for a recognized USB Ethernet adapter.
4. **Link-up**: connect an Ethernet cable to a known-good switch, confirm link LED and OS-reported link state.
5. **Throughput**: run `iperf3` to confirm real Gigabit throughput rather than a silent fallback to 100 Mbps.

## References

- Microchip LAN7800 datasheet — DS00001992
- TI HD3SS3220 datasheet — SLLSE21
- [JLCPCB impedance calculator / capabilities](https://jlcpcb.com/)
- [LCSC parts catalogue](https://www.lcsc.com/)

## License

Choose and add a license (e.g. CERN-OHL-S for open hardware, or MIT/CC-BY-SA if you prefer a simpler permissive option) — not yet decided for this project.

---

*This is a personal learning project. Pin numbers and component values should always be re-verified against the current manufacturer datasheets before fabrication.*# USB-3.1Gigabit-Ethernet-to-USB-C-Adapter
