# Power Supply Design

This document describes the power section of the PSST board. The same PCB
serves both roles: the external unit (CR2032 coin cell, occasionally
connected to USB for programming) and the internal unit (USB-powered, no
battery).

## Goals

- Run the MCU from either a CR2032 coin cell or USB VBUS, automatically.
- When USB is plugged in, no **steady-state** current path into the cell
  (it's not rechargeable). Brief, benign current *out of* the cell during
  the USB plug-in transient is acceptable (see [Battery isolation]
  (#battery-isolation-single-p-channel-mosfet-switched-by-vbus) for the
  full caveat). True bidirectional isolation under all transients would
  require back-to-back FETs or an ideal-diode controller; the project
  doesn't need that.
- When USB is removed, the cell takes over with no manual intervention.
- One circuit, two populations: external unit populates the cell holder;
  internal unit populates the USB connector. Everything else is shared.

## Topology

High-level block diagram. USB-C is the top source, the coin cell is the
bottom source; both converge on the common VCC rail on the right.

```
   ┌──────────┐
   │  USB-C   │── VBUS ──┬──► D1 ─► LDO MCP1700 ─VOUT──┐
   │ C2765186 │          │          no-EN, no discharge │
   │          │          │                             │
   │          │          ├──► Module VBUS pin          │
   │          │          │    (powers MCU USB regulator│
   │          │          │     + presence via          │
   │          │          │     USBREGSTATUS)           │
   │          │          │                             │
   │          │          └──► PFET gate (direct tie)   │
   │          │                       │                │
   │          │── D+/D− ──► USBLC6 ESD ──► MCU         │
   │          │                       │                │
   │          │── CC1 ── R5 (5.1k) ── GND              │
   │          │── CC2 ── R6 (5.1k) ── GND              │
   │          │                       │                │
   │          │── GND                 │                ├──► VCC rail
   └──────────┘                       ▼                │      │
                              ┌───────────────┐        │      ├── C2  10 µF
                              │  P-MOSFET     │        │      ├── C3  100 nF
   ┌──────────┐               │  AO3401A      │        │      ├── C4  22 µF
   │  CR2032  │── BAT+ ──────►│  D         S  │── VCC ─┘      │
   │  holder  │               │  Gate ← VBUS, │               ▼
   │          │── BAT− ── GND │  R1↓ to GND   │           MCU + radio
   └──────────┘               └───────────────┘           VCC pin
```

The interesting bit is the battery-disconnect, drawn schematic-style:

```
                       VBUS
                        │
                        ●──────────────┐
                        │              │
                       R1 (100k)       │ gate
                        │              │
                       GND             ▼
                                 ┌───────────┐
                                 │  AO3401A  │
        BAT+ ───────────── D ────┤ P-MOSFET  ├──── S ─────► VCC
                                 └───────────┘
```

Source on VCC, drain on BAT+ — this orientation puts the body diode
anode on the cell side, so when VCC > BAT+ (USB present) the diode is
reverse biased and there's no DC path into the cell.

| VBUS  | Gate    | Source (VCC) | Vgs    | FET state                  |
|-------|---------|--------------|--------|----------------------------|
| 5 V   | ≈ 5 V   | 3.3 V        | +1.7 V | OFF — cell isolated (DC)   |
| 0 V   | 0 V     | ≈ 3 V        | −3 V   | ON — cell powers VCC       |

Key nets:

- **VBUS** — 5 V from USB. Feeds the LDO, the PFET gate, the ESD array,
  and the module's dedicated `VBUS` pin (which powers the MCU's internal
  USB regulator and is read via `USBREGSTATUS` for presence detection).
- **VCC** — 3.0–3.3 V common rail to the MCU. Sourced by the LDO when USB
  is present, or by the cell through the PFET when USB is absent.
- **PFET gate** — tied directly to VBUS, with R1 (100 kΩ) as pull-down to
  GND so the gate is at 0 V when USB is unplugged.
- **CC1, CC2** — each pulled to GND through 5.1 kΩ (R5, R6) so a USB-C
  host identifies the board as a sink and asserts VBUS.

## Key decisions

### Battery isolation: single P-channel MOSFET, switched by VBUS

P-FET **drain** on `BAT+`, **source** on the common `VCC` rail. Gate
tied directly to VBUS, with R1 (100 kΩ) pulling the gate to GND when
VBUS is absent.

- USB present (VBUS = 5 V) → gate = 5 V, source = VCC = 3.3 V →
  Vgs = +1.7 V → FET hard off. No DC path from VCC into the cell.
- USB absent (VBUS = 0 V) → gate pulled to 0 V through R1. The body
  diode initially pulls the source up to BAT+ − Vf, then with Vgs ≈ −3 V
  the FET turns on hard (Rds(on) typically <100 mΩ) and VCC = BAT+.

A symmetric divider (e.g. R1 = R2 = 100 kΩ) would leave the gate at the
midpoint (~2.5 V) when USB is plugged in — only Vgs = −0.5 V, which is
right at AO3401A's threshold (−0.4 to −1.1 V) and would leak. The direct
tie sidesteps that entirely.

**Body-diode orientation matters.** For a P-channel MOSFET the body
diode is anode = drain, cathode = source. With D on BAT+ and S on VCC,
when USB is in and VCC (3.3 V) > BAT+ (3 V), the diode is *reverse*
biased — no steady-state DC path from the LDO into the cell. The
opposite orientation (S on BAT+, D on VCC) would forward-bias the body
diode under exactly the steady-state USB-plugged condition and
trickle-charge the cell. Get this wrong and the whole circuit is
pointless.

**What this does *not* protect against.** During the USB plug-in
transient, the LDO ramps and the rail capacitance (C2 + C4 ≈ 32 µF)
charges from 0 toward VCC. While VCC < BAT+ the body diode is forward
biased and the cell sources a current pulse into the rail. Energy
delivered to the cap is ~½ × 32 µF × (3 V)² ≈ 144 µJ (plus diode-drop
losses) — benign for a CR2032 but not literally zero. The same applies
to any unusual condition that pushes VCC below BAT+. If a future
revision ever needs literal bidirectional isolation under all transient
and fault conditions, that requires back-to-back PFETs or an
ideal-diode controller (LTC4412 / LTC4413). For this project, a single
PFET with correct orientation is the right tradeoff.

Why not the alternatives:

- **Schottky-OR both sources.** A 0.3 V drop on a 3 V cell is brutal
  near end-of-life and shortens usable battery time substantially.
- **Power-mux IC (LTC4412, TPS211x).** Cleaner schematic but more parts
  and cost; popular cheap ones (TPS2113A) have ~2.8 V min input which is
  marginal for a near-EOL CR2032. A PFET + one resistor does the job.
- **Manual switch / remove the cell.** Ruled out as not foolproof.

### LDO selection: no EN pin, no active output discharge

When the unit runs from the cell with USB unplugged, VCC sits at ~3 V
(driven by the cell through the PFET) while raw VBUS is 0 V. The LDO's
output pin is wired directly to VCC, so the LDO experiences VOUT > VIN
with input collapsed. Two LDO features that are normally desirable
become **disqualifying** in this topology:

1. **Active output discharge.** Many modern LDOs (e.g. TPS7A02) include
   an internal FET that actively pulls VOUT to GND when the part is
   disabled. In our topology, "disabled" coincides with "VCC is being
   held at 3 V by the cell" — the discharge FET would sit there
   shorting the cell to GND through the LDO until something gave. Hard
   no.
2. **EN pin tied to anything other than VIN.** Tying EN to raw VBUS
   while VIN is post-D1 puts EN above VIN by the Schottky drop, which
   violates the EN absolute-max spec on most EN-having LDOs. Tying EN
   to VIN works but is one more pitfall to get right.

The cleanest answer is to pick a simple LDO with no EN pin and no
active discharge. Then:

- When VBUS = 5 V, D1 (B5819WS) forward-biases, LDO_VIN ≈ 4.6 V, LDO
  regulates VCC = 3.3 V normally (MCP1700 dropout is ~180 mV at full
  load — plenty of headroom).
- When VBUS = 0, D1 blocks, C1 (on LDO_VIN) discharges through the
  LDO's quiescent current path, and the LDO is effectively off. Its
  pass element's body diode does forward-bias from VCC into LDO_VIN
  (since VCC > LDO_VIN once C1 drops), but the only thing on the other
  side of that path is the LDO's Iq to ground — a continuous few µA
  drain on the cell. For a 1.6 µA-class LDO that's ~14 mAh/year, well
  inside the [current budget](#current-budget-tbd) noise floor.

D1 sits only in the LDO branch, not in the main VBUS rail. The module's
USB VBUS pin and the PFET gate still see the full 5 V — they don't have
a reverse path to worry about, and the PFET turn-off margin benefits
from not taking D1's ~0.4 V drop.

If a future revision really needs zero LDO leakage in battery mode, add
a second Schottky **D2** between LDO_OUT and VCC (anode at LDO_OUT).
That isolates VCC from the LDO entirely when USB is unplugged, at the
cost of ~0.3 V more headroom and an LDO that needs to put out ~3.6 V to
land VCC at 3.3 V (or accept VCC ≈ 3.0 V from a 3.3 V LDO — fine for
nRF52840). Not needed for the baseline design.

### USB peripheral wiring and VBUS detect

VBUS goes to the module's dedicated `VBUS` input pin (per the Raytac
MDBT50Q-1MV2 / nRF52840 datasheet — it powers the internal USB
regulator and is read via `USBREGSTATUS` for presence detection). D+
and D− go through USBLC6 ESD to the module's USB peripheral pins. No
separate GPIO divider is needed; using one in addition would just waste
a pin.

(If the design ever switches to "USB is power-only, programming via SWD
only" — no MCU USB peripheral — then the VBUS pin can be left floating
and a divided GPIO would be the only way to sense USB presence. Not the
plan here.)

### USB-C CC pulldowns

USB-C sinks must identify themselves with 5.1 kΩ pull-downs on both CC1
and CC2 to GND (R5, R6). Without these, a C-to-C cable will not deliver
VBUS — the host has no way to know the board's role. The resistors
draw negligible current and matter only when a cable is attached.

### Decoupling and TX-pulse support

nRF52840 TX bursts draw 10–15 mA. CR2032 ESR (10–40 Ω fresh, worse as
the cell ages) sags noticeably under that load. A 22 µF ceramic (C4)
placed right at the module's VCC pin keeps the rail up through TX
bursts. Standard 100 nF (C3) sits next to the module supply pin in
addition. C2 (10 µF) is the LDO output cap; C1 (1 µF) is the LDO input
cap and must be placed on the `LDO_VIN` side of D1, close to the LDO's
VIN pin — putting it on the raw VBUS rail would leave the LDO input
without local decoupling.

If field testing shows the cell still sags too much, add a 47–100 µF
bulk cap (C5) directly across `BAT+/BAT−`.

### USB ESD/TVS

A USBLC6-2 (or equivalent) clamps D+/D− and VBUS during hot-plug events.
Cheap insurance and standard practice.

## Current budget (TBD)

CR2032 viability for the external unit is **not** purely a power-section
decision — it's a system-level question that depends on firmware
behavior. The bulk cap (C4) handles brief TX/RX packet edges; it does
**not** make the 60-second receive window in [docs/design.md](design.md)
cheap.

Dominant loads on the external unit:

- **Sleep** (System OFF or System ON-Idle, radio off): low µA. Set by
  MCU leakage + LDO leakage (LDO is disabled when VBUS = 0) + PFET
  subthreshold + R1 leakage (~0, since gate node has no DC path on
  battery).
- **RX window** (per requirements: ~60 s after a button press, waiting
  for the reply): nRF52840 RX ≈ 5–6 mA continuous → ~0.1 mAh per cycle.
  This is the dominant per-cycle cost.
- **TX bursts**: brief, ms-scale, 7–15 mA peak. Negligible energy
  compared to the RX window.
- **LED indication**: depends entirely on color, drive current, and
  duty cycle. Easily comparable to or larger than the RX window if not
  constrained in firmware.

CR2032 nominal capacity is ~220 mAh at low discharge currents. Both
delivered capacity and pulse capability degrade significantly with age
(ESR climbs from ~10 Ω fresh to 40 Ω+ near end-of-life) and at higher
continuous currents than the datasheet's test conditions.

**Feasibility must be confirmed by measuring actual sleep + active
currents on prototype hardware**, then tuned in firmware (shorter
listen windows, lower LED brightness, fewer retries) until the
calls-per-day target lines up with the desired cell lifetime. If the
budget doesn't close, the options are: (a) firmware-side reductions,
(b) a larger cell (CR2450, CR2477), (c) a different chemistry, or (d)
accepting more frequent battery swaps.

## Components

Manufacturer part numbers are listed below. LCSC C-numbers change over
time, so search by MPN. Multiple options are given per role for sourcing
flexibility.

### Q1 — P-channel MOSFET (battery isolation)

**Selected:** AO3401A (Alpha & Omega) — LCSC [C15127](https://www.lcsc.com/product-detail/C15127.html), SOT-23.
Vds = −30 V, Vgs(th) = −0.4 to −1.1 V, Rds(on) ≈ 60 mΩ at Vgs = −2.5 V.
Comfortably meets all requirements (logic-level, low Rds(on) at the
−3 V Vgs we get on battery, body diode correctly oriented when
S=VCC / D=BAT+).

Alternatives considered:

| Manufacturer  | Part             | Notes                                    |
|---------------|------------------|------------------------------------------|
| Diodes Inc    | DMP2305U-7       | Common alternative                       |
| Vishay        | SI2301CDS-T1-GE3 | Slightly higher Rds(on), still fine      |

### U1 — 3.3 V LDO

**Selected:** MCP1700T-3302E/TT (Microchip) — LCSC [C39051](https://www.lcsc.com/product-detail/C39051.html), SOT-23-3.
3.3 V fixed output, 250 mA, 1.6 µA Iq, 178 mV dropout at 250 mA, no EN
pin, no active output discharge. Meets every hard requirement.

Hard requirements (see [LDO selection](#ldo-selection-no-en-pin-no-active-output-discharge)
for *why*):

- **No active output discharge.** Rules out TPS7A02, some TLV70x and
  LP5907 variants, and most "advanced" modern LDOs that list active
  discharge as a feature.
- **Tolerates VOUT > VIN** with low backfeed (just the Iq path).
- **No EN pin preferred** — simpler topology, no EN-vs-VIN absolute-max
  pitfall. If using an EN-having LDO, tie EN to `LDO_VIN` (post-D1),
  not to raw VBUS.
- ≥ 100 mA Iout, dropout ≤ 400 mV.

Alternatives considered:

| Manufacturer | Part             | Iout  | Iq      | EN | Notes                          |
|--------------|------------------|-------|---------|----|--------------------------------|
| Torex        | XC6206P332MR     | 200mA | 1 µA    | no | Very low Iq; SOT-23            |
| Holtek       | HT7333-A         | 250mA | 4 µA    | no | Cheapest; SOT-89               |

### USB ESD/TVS array

| Manufacturer | Part         | Notes                                       |
|--------------|--------------|---------------------------------------------|
| STMicro      | USBLC6-2SC6  | Industry standard, SOT-23-6                 |
| Onsemi       | SRV05-4-T7   | Functional equivalent, also widely stocked  |

### D1 — VIN-side Schottky (required)

**Selected:** B5819WS (Diodes Inc) — LCSC [C64886](https://www.lcsc.com/product-detail/C64886.html), SOD-123.
1 A forward, Vrwm = 40 V, Vf ≈ 0.4 V at our LDO input currents
(tens of mA), reverse leakage at our reverse voltage (~4 V vs the
30+ V the datasheet quotes) is in the low-µA range. Anode toward
VBUS, cathode toward LDO_VIN.

Requirements: in series with the LDO's VIN, anode toward VBUS.
≥ 200 mA forward, Vrwm ≥ 20 V, low Vf preferred (smaller drop = more
LDO headroom).

Alternatives considered:

| Manufacturer | Part         | Package | Vf @ 100 mA | Notes                    |
|--------------|--------------|---------|-------------|--------------------------|
| Nexperia     | PMEG2010EJ   | SOD-323 | ~0.3 V      | Smaller, slightly lower Vf|
| Generic      | SS14         | SMA     | ~0.4 V      | Oversized but stocked     |

### USB-C connector

| Manufacturer    | Part             | LCSC #    | Notes                  |
|-----------------|------------------|-----------|------------------------|
| (per design.md) | (already chosen) | C2765186  | Existing decision      |

### Coin-cell holder (CR2032)

| Manufacturer | Part         | Notes                                          |
|--------------|--------------|------------------------------------------------|
| MPD          | BAT-HLD-001  | SMT, very popular and well stocked             |
| Keystone     | 1042         | SMT, low profile                               |
| Keystone     | 3034         | Through-hole tabs; sturdier mechanically       |

### Passives

| Ref  | Value   | Notes                                                |
|------|---------|------------------------------------------------------|
| R1   | 100 kΩ  | PFET gate pull-down to GND, 0402 5 %                 |
| R5   | 5.1 kΩ  | USB-C CC1 pull-down to GND, 0402 1 %                 |
| R6   | 5.1 kΩ  | USB-C CC2 pull-down to GND, 0402 1 %                 |
| C1   | 1 µF    | LDO input cap, X7R 10 V 0402, **on LDO_VIN side of D1**, close to LDO VIN pin |
| C2   | 10 µF   | LDO output cap, X5R 10 V 0603                        |
| C3   | 100 nF  | Module VCC decoupling, X7R 0402, close to VCC pin    |
| C4   | 22 µF   | Bulk for TX bursts, X5R 10 V 0805, at module VCC pin |
| C5   | 47 µF   | Optional, across BAT+/BAT− if cell sag is observed   |

Any of the usual suspects work for these passives (Yageo, Samsung,
Murata, Walsin); LCSC's generic basic-part offerings are fine.

## Population per role

- **Internal unit (USB-powered):** populate USB-C connector, R5/R6 CC
  pulldowns, USBLC6, D1, LDO, C1/C2/C3/C4. Cell holder not populated.
  PFET + R1 can be populated (harmlessly inert — with BAT+ floating,
  the body diode has no anode connection) or DNP'd; populating saves
  having a separate BOM variant.
- **External unit, cell only, SWD programming:** populate cell holder,
  PFET, R1, C3, C4. USB-C connector, R5/R6, USBLC6, D1, LDO, C1, C2
  can all be DNP'd.
- **External unit, cell + USB DFU programming:** populate everything.
  The LDO **and** D1 are *not* optional in this configuration — when
  USB is plugged in, the PFET gate is pulled to VBUS and the cell path
  is disconnected, so VCC has no source unless the LDO is present and
  working. USBLC6 is required for ESD safety once D+/D− are wired to
  the MCU.
