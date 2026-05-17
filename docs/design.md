# Polite Spouse Summoning Terminal — Design

## Overview

A little wireless doorbell for politely bugging your spouse while they're
working from their home office. There are two units. The *external* one sits
on the wall outside the office door — you push its button when you want to
get their attention. The *internal* one lives on their desk and lights up to
let them know. They hit one of two buttons to answer: green for "come in",
red for "busy", and the external unit flashes the same color back at you.

Both units are built from the same PCB. Which role a board plays comes down
to hardware switches and which parts get populated.

## Functional Requirements

### External Unit

Mounted outside the office door. Coin-cell powered, so it sleeps almost all
the time.

- One populated button: the "summon" button.
- One RGB LED for status feedback.
- Default state: deep sleep, drawing as little current as practical.
- On button press:
  1. Wake up.
  2. Transmit a summon request wirelessly to the internal unit.
  3. Stay awake listening for a reply for a bounded window (~1 minute).
  4. On reply, flash the LED green ("come in") or red ("busy").
  5. Return to deep sleep.
- If no reply arrives within the window, indicate timeout on the LED and
  return to deep sleep.

### Internal Unit

Sits on the desk inside the office. USB-powered, always on.

- Two populated buttons: green ("I'm free, come in") and red ("I'm busy").
- An LED that flashes when a summon request arrives, until a button is pressed
  (or the request times out).
- Default state: idle, listening for summon requests.
- On summon received:
  1. Flash the LED to get attention.
  2. Wait for a green or red button press.
  3. Transmit the corresponding reply back to the external unit.
  4. Return to idle.
- Power consumption should be kept reasonable, but is a much lower priority
  than for the external unit.

### Shared

- A single PCB design serves both roles. The role is selected by hardware
  switches and which components are populated (e.g. one button vs. two,
  coin-cell holder vs. USB input).
- The two units must pair/identify each other so only the intended pair
  communicates.

## Technical Decisions

- **MCU / radio**: Raytac MDBT50Q-1MV2 (nRF52840 module) on both units. Same
  module on both sides keeps the PCB and firmware shared.
- **Wireless protocol**: TBD. nRF52840 supports BLE, 802.15.4, and proprietary
  2.4 GHz ESB. To be chosen based on latency, range, and external-unit power
  budget.
- **External unit power**: coin cell. Specific chemistry/size TBD.
- **Internal unit power**: USB-C. Connector: LCSC C2765186.
- **Role selection**: hardware switches on the PCB + selective population of
  buttons and the power input. Details TBD.

## Components

- Raytac MDBT50Q-1MV2
- USB-C Connector: [LCSC C2765186](https://www.lcsc.com/product-detail/C2765186.html)
