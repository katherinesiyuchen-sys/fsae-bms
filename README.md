# fsae-bms

*Doing this for the love of the game 🏎️*

Centralized battery management board for one **6S segment** of Energus 1s4p Li-ion modules.


## Cell voltage

Six cells in series stack up. Only cell 1 has a terminal at 0 V:

```
cell 6   18.0 V ── 21.6 V
cell 1      0 V ──  3.6 V
```

An ADC measures relative to its own ground and reads 0–3.3 V. Cell 6's top sits at 21.6 V, so we cannot wire it to a pin and measuring both ends and subtracting means finding a 3.6 V answer from two ~20 V readings, with all the error from both landing on the difference.

**A flying capacitor removes the pedestal mechanically.** Two ADG1408 muxes, addressed together with inputs offset by one tap, connect a 1 µF capacitor across the selected cell. The mux then disconnects both ends — the capacitor holds the cell's voltage with no memory of its former potential — and a second mux state presents it to the ADC as an ordinary ground-referenced source.

All six cells travel one signal path, so there is **one** gain error and **one** offset, common to all of them. In the cell-to-cell comparison balancing depends on, a common error largely cancels.

Rejected: a divider stack (error from both taps lands in full on a 3 V result), and six high-common-mode difference amplifiers (6× cost, six independent gain errors to trim).

### Channel map

Each mux has 8 channels; six cells use six, and the spare two do real work.

| Address | U1 — cap top | U2 — cap bottom | Phase |
|---|---|---|---|
| 0–5 | `CELL2`–`CELL7` | `CELL1`–`CELL6` | charge from cell 1–6 |
| 6 | `SELFTEST` | `GNDA` | known voltage through the chain |
| 7 | ADC divider | `GNDA` | read cap into ADC |

Address 7 *is* the transfer switch, so no second high-voltage switch is needed, and break-before-make — the property the mux was chosen for — protects the transfer too.
Address 6 verifies mux, capacitor, divider and reference every scan, before a cell reading is trusted. `SELFTEST` derives from `+3.3VA`, **not** `VREF`, so a reference failure cannot hide itself.

## Temperature 

The Energus module's built-in sensor is a **temperature-variable voltage shunt reference**, behaving like a zener whose voltage falls with temperature.

| Property | Consequence |
|---|---|
| Needs a **680 Ω pull-up**, not a divider | ~2.2 mA bias per channel |
| 2.44 V @ −40 °C → 1.30 V @ +120 °C, non-linear | datasheet lookup table, no β equation |
| **Galvanically isolated** from cell terminals | reads ground-referenced — no mux, no level shift |
| 4 sensors per module, analogue-OR'd | reports the module hot spot — 100 % coverage against a 30 % requirement |
| Shunt reference, not a ratio | reading largely immune to rail variation |

Six channels diode-OR into `TEMP_HOT`. Because sensor voltage *falls* with temperature, the hottest module is the lowest voltage, so a common-anode Schottky array extracts the minimum and one comparator watches all six.

**The diode drop is an error and it sets the threshold.** `TEMP_HOT` = lowest sensor + V_f, about 0.2 V at 154 µA, varying ±50 mV. At −11 mV/°C that's ±4.5 °C. So the hardware trip is aimed at **57 °C** to absorb the spread, and firmware, which reads all six channels individually, owns the precise per-module limit. Layered by design: the comparator is the backstop that works with firmware dead.

```
threshold  1.534 V (57 C) + 0.2 V Vf  =  1.734 V
divider    R18 662R / R19 1.50k from VREF
check      1.5 / 2.162 x 2.5  =  1.734 V
```

`TSENSE_RTN` returns to `GNDA`, not `GND`: you're measuring a voltage across the sensor relative to its own return, and the ADC measures relative to `VSSA`. The ~13 mA of sensor current in the analog ground is DC, so it's a fixed offset rather than noise.

---

## Protection 

A hung microcontroller that hasn't noticed an overvoltage is *indistinguishable* from a healthy one that sees nothing wrong. Both are silent. So the trip cannot depend on code
executing and the rules agree: the shutdown circuit must be latched open by non-programmable logic, resettable only by a person at the vehicle. **SW1, never a GPIO.**

| Source | Threshold | At | Gated |
|---|---|---|---|
| Overvoltage | 2.096 V | 4.192 V cell | address 7 only |
| Undervoltage | 1.246 V | 2.493 V cell | address 7 only |
| Overtemperature | 1.734 V | 57 °C | continuous |
| Watchdog timeout | ~200 ms | kicked every 20 ms | continuous |

`VCELL` is only a real cell voltage during the read slot — every other slot the divider
node is mid-transfer. So the voltage comparators are gated by an address decode, and one
gate does the whole job:

```
NOR(/ADDR7, VFAULT_N)  =  ADDR7 . fault
```

Both signals are active-low, and NOR is the AND of two active-low inputs.

All four thresholds derive from `VREF` — the same reference the ADC's scale is corrected against — so hardware trip points and firmware measurements can never disagree about what 4.2 V means.

**The fail-safe is structural.** The relay's normally-open contact sits in the shutdown circuit, held closed only while the latch says healthy *and* 12 V is present. A fault, a dead latch, a lost rail, an undriven gate. Every one gives coil de-energized, contact open, circuit broken. Q1's gate pulldown guarantees the last of those.

---

## Balancing

Passive dissipative, one optocoupler and one BSS138 per cell. The optocoupler solves a level-shift problem: cell 6's FET source sits ~21 V above ground, and a MOSFET switches on V_GS, gate voltage relative to *its own source*. A ground-referenced 3.3 V drive reads as −21 V to that gate.

```
LED     (3.3 - 1.2) / 470R  =  4.5 mA
opto    CTR ~50%            ->  ~2.2 mA available
gate    4.2 V / 101k        =  42 uA needed      (50x margin)
        -> phototransistor saturates, Vgs ~ 3.96 V

bleed   4.2 V / 100R  =  42 mA
        0.176 W per cell  ·  1.06 W all six  ·  1206 rated 0.25 W
```

100 Ω rather than 50 Ω halves current and dissipation — consistent with the argument below that heat is the binding constraint, not balance time. Shedding ~200 mAh at 42 mA takes about five hours, which is normal: balancing happens on the charger between sessions, not on track.

### Active balancing was evaluated and rejected

Without a dedicated controller the options are a switched-capacitor shuttle (a signal mux can't carry useful balance current, so it needs a second HV switch matrix whose timing must never overlap the first), adjacent-cell inductive buck-boost (five half-bridges with isolated gate drive, moving charge one hop at a time), or multi-winding flyback (custom magnetics).

Against that: active balancing recovers a fraction of the *imbalance* order 1–2 % of capacity at ~80 % transfer efficiency. And the problem passive balancing actually causes in a sealed accumulator is **heat**, better solved by limiting how many cells bleed at once. 

What would change the answer: a longer event, a thermally-limited pack, or a production vehicle where round-trip efficiency compounds over thousands of cycles.

---

### Buses

```
CELL[1..7]    cell taps -> frontend, balance
BAL_EN[1..6]  mcu -> balance
TSENSE[1..6]  J2 -> temp
TEMP[1..6]    temp -> mcu
FE_CTL        {MUX_A0 MUX_A1 MUX_A2 MUX_EN}
PROT          {WDT_KICK FAULT_N}
CAN           {CAN_TX CAN_RX}
SWD           {SWDIO SWCLK SWO NRST}
```

Bus aliases live in `feb-bms.kicad_pro`, so open the **project**, not individual sheets, or
they won't resolve.

## Key parts

| Ref | Part | Role | Datasheet |
|---|---|---|---|
| U1, U2 | ADG1408YRUZ | 8:1 analog mux, break-before-make | [ADI](https://www.analog.com/media/en/technical-documentation/data-sheets/ADG1408_1409.pdf) |
| U4 | LM339 | quad comparator, open collector | [ST](https://www.st.com/resource/en/datasheet/lm139.pdf) |
| U3/U5/U6/U7/U8/U11 | 74LVC1G10 / 1G02 / 1G32 | gating and latch | [TI](https://www.ti.com/lit/ds/symlink/sn74lvc1g02.pdf) |
| U9 | TPS3823-33 | watchdog supervisor | [TI](https://www.ti.com/lit/ds/symlink/tps3823.pdf) |
| U10 | ADR4525 | 2.500 V reference | [ADI](https://www.analog.com/media/en/technical-documentation/data-sheets/ADR4520_4525_4530_4533_4540_4550.pdf) |
| U18 | STM32F446RETx | LQFP64 | [ST](https://www.st.com/resource/en/datasheet/stm32f446re.pdf) |
| U19 | TJA1051T/3 | CAN transceiver, VIO = 3.3 V | [NXP](http://www.nxp.com/docs/en/data-sheet/TJA1051.pdf) |
| U20 | AP63203WU | 12 V → 5 V buck | [Diodes](https://www.diodes.com/assets/Datasheets/AP63200-AP63201-AP63203-AP63205.pdf) |
| U21 | MIC5219-3.3 | 5 V → 3.3 V LDO, low noise | [Microchip](http://ww1.microchip.com/downloads/en/DeviceDoc/MIC5219-500mA-Peak-Output-LDO-Regulator-DS20006021A.pdf) |
| U22 | TPS61040 | 12 V → 28 V boost | [TI](http://www.ti.com/lit/ds/symlink/tps61040.pdf) |
| Q1, Q2, Q3–Q8 | BSS138 | relay / lamp / balance | [onsemi](https://www.onsemi.com/pub/Collateral/BSS138-D.PDF) |
| Q9 | AO3401A | reverse-polarity blocker | [AOS](http://www.aosmd.com/pdfs/datasheet/AO3401A.pdf) |
| U12–U17 | LTV-817S | balance gate drive, isolated | [Liteon](http://www.us.liteon.com/downloads/LTV-817-827-847.PDF) |
| D3–D5 | BAT54A | diode-OR, common anode | [Diodes](http://www.diodes.com/_files/datasheets/ds11005.pdf) |
| D10 | SMAJ16A | load-dump clamp, unidirectional | [Littelfuse](https://www.littelfuse.com/media?resourcetype=datasheets&itemid=75e32973-b177-4ee3-a0ff-cedaf1abdb93&filename=smaj-datasheet) |
| K1 | SPDT relay | NO contact in the SDC | — |

## Compliance

Requirements below are from the **Formula Student 2026 rules v1.1**. The clause numbers
still need re-deriving from the FSAE rulebook — same requirements, different numbering.

| Requirement | Where it lands |
|---|---|
| Measure all cell voltages, TS current, ≥30 % of cell temps | 6 cells scanned; module sensors cover 100 %. **TS current is not on this board** — open question whether a segment board carries it |
| Max cell temp 60 °C or datasheet, whichever lower | 45 °C is binding while charging |
| Open SDC on persistent fault: >500 ms voltage, >1 s temp | 145 ms scan |
| Red latching cockpit "AMS" lamp | driven from the latch, not firmware |
| Loss of a measurement connection must open SDC | self-test slot + implausible-reading detection |
| Faults individually injectable for inspection | break-in header — **to add** |
| SDC latched by non-programmable logic, manual reset | SR latch + SW1 |
| De-energized state opens SDC | NO relay contact throughout |


## Opening this

KiCad 10.0.3. Open **`feb-bms.kicad_pro`**.
