# RFID Hub v2 rev A — pre-order safety review

**Verdict: do not order this revision as-is.**

The input protection, the USB ideal diode, and the “outputs off until software says so” behavior are real and are wired the way the schematic claims. Two holes are large enough to smoke a board in an escape room: a passive PoE injector (or any 24/48 V source) on a reader jack back-feeds the 3.3 V rail through the port eFuse, and there is no over-voltage clamp on the rail that feeds the ESP32. A closed PLA box makes the second problem worse, because the only over-temperature shutdown is firmware and it is set above the temperature where PLA softens.

Reviewed from the KiCad 9 board (`work/kicad/rfid_hub/rfid_hub.kicad_pcb`), the fab BOM, and the ESPHome YAML. Part limits were taken from the manufacturer datasheets (TI LM74502, LM66100, TPS2596, TPS54202; AOS AON6262E; Same Sky PJ-063AH/BH; Littelfuse 451; Ckmtw R-RJ45R08P-A004), not from `work/review/calculations.md`. Numbers below are recomputed in `evidence/measurements-and-calcs.txt`.

The board is 110 × 100 mm, 1.6 mm, 4 layers, 1 oz outer / 0.5 oz inner, 0.15 mm minimum copper clearance. All 394 footprints are on the top. Mounting holes are unplated.

---

## BLOCKER

### B1. A PoE injector or a 24/48 V pair on an RJ45 jack back-feeds the reader rail

Each reader jack puts 3.3 V straight onto pin 7 (`J101` pad 7 = `P1_3V3`, same on ports 2–8) and ground on pin 5. The only thing between pin 7 and the `3V3_PORT` buck is a TPS259630 (`U101` and the other port eFuses). That part’s OUT pin is specified to `min(21 V, VIN + 0.3 V)`. It does not block reverse current: if the cable drives pin 7 above the 3.3 V rail, the MOSFET body diode conducts into `3V3_PORT` whether EN is high or low.

What that means in the room:

- A normal Ethernet switch (no PoE) mostly stresses magnetics and a few 0402 resistors. Pin 7 and pin 8 are one pair, so the transformer winding sits across the port 3.3 V and the unused pin. The eFuse current-limits at about 0.24 A and, if firmware is alive, turns the port off. The hub should survive. The switch port may not.
- A compliant 802.3af/at switch looks for a 25 kΩ signature before it applies 48 V. This jack does not present that signature, so a compliant PSE should not turn on. Do not treat that as protection.
- A passive injector, which is the usual “PoE” brick in the same building as cameras and props, applies 24 V or 48 V immediately between pairs 4–5 and 7–8. Pair 4–5 is MISO and GND, so that side of the injector is clamped to hub ground through pin 5. Pair 7–8 is then at ±24 V or ±48 V relative to GND, directly on pin 7. Either polarity is outside the eFuse pin (OUT minimum is −0.3 V, maximum is VIN+0.3 V). That is already above the eFuse absolute maximum (21 V) and above the 10 V rating of the port 4.7 µF capacitors (`C104` and the other `Cxx4`, Samsung 10 V). The body diode dumps it onto `3V3_PORT`. The 22 µF bulk caps on that rail are 25 V. A 48 V source is past those too. The port eFuse and the reader buck are then in a cascade that can smoke the board and take every reader with it. There is no series fuse on pin 7.

Silkscreen on F.SilkS says `RFID READERS ONLY` / `NOT ETHERNET`. That is a label, not a barrier. Staff plug cables into jacks that look like Ethernet.

Fix before fab: on each port, block reverse voltage into `3V3_PORT` (ideal diode or a back-to-back arrangement ahead of the eFuse output) and add a series PTC or similar on pin 7 so a 24/48 V source cannot sit on the rail. A TVS on pin 7 alone will burn if the injector does not current-limit.

Evidence: `design.py` port jack pins; board pads of `J101` and `U101`; TPS2596 SLVSET8A absolute maximum, VOUT = min(21 V, VIN+0.3 V); port caps in the JLC BOM (`C23733` is 10 V, `C12891` is 25 V).

### B2. The ESP32 3.3 V rail has no over-voltage clamp, and USB may be plugged in while 12 V is on

`3V3_PORT` (the reader buck, `U6`) has an OVLO divider into every port eFuse. `3V3_BUCK` / `+3V3` (the logic buck, `U4`, then `U5`) does not.

`U4` is a TPS54202. Feedback is `R20` 47 kΩ from `3V3_BUCK` to `FB2`, `R21` 10 kΩ to GND. If `R20` opens or `R21` shorts, the error amp runs to full duty and the output rises toward the 12 V input. The parts on that rail are not rated for it:

- `U5` LM66100 absolute maximum is 6 V on VIN and on VOUT (SLVSEZ8A). The chip is there specifically so USB and 12 V can be connected together. Above 6 V it is outside its rating and can fail short, which connects the runaway buck to the ESP32 3.3 V pins.
- The ESP32 module, both TCA9555s, the three ADS1115s and the SHT40 sit on `+3V3`. A 12 V excursion on that pin smokes the module. The 7 A input fuse will not open for the current a buck-limited failure draws.

USB at the same time, which is the case Ronnie called out:

- The hub does **not** drive the FNK0090 5 V pins. `FNK_5V` only feeds `R49` 1 MΩ / `R50` 150 kΩ into GPIO36, about 4 µA at 5 V. Confirmed on `J3`: the 5 V socket pins are net `FNK_5V`, not a hub supply.
- `U5` is wired as TI’s reverse-blocking circuit: pin 1 VIN = `3V3_BUCK`, pin 6 VOUT = `+3V3`, pin 3 CE = `+3V3`. While the buck is in regulation (~3.40 V) and the module’s own regulator is still powered from USB 5 V, the module regulator’s input is above its output, so it does not push current back out the USB cable. That normal case is fine.
- The laptop is at risk in the fault above, not in normal operation. If `+3V3` is forced above the FNK0090’s 5 V rail, current can come back through the module regulator. Whether a series diode on the FNK0090 then blocks the USB connector is **not in this design package** (Freenove does not publish that schematic here). Do not sign the laptop off until someone checks a real FNK0090: diode between VBUS and the 5 V pin, yes or no.

Fix: the same kind of OVLO already used on the readers, or a zener-plus-fuse / crowbar, on `3V3_BUCK` before `U5` and on `+3V3`. The reader rail shows the designer already considered this failure. The logic rail was left open.

Evidence: `U4`/`U5` pad nets on the board; LM66100 pin table (1 VIN, 3 CE, 6 VOUT) and 6 V absolute maximum; `R20`/`R21` in `power.kicad_sch` / `design.py`; `J3` 5 V pins net `FNK_5V`.

---

## SHOULD FIX

### S1. Maglock kickback exceeds the eFuse pin rating, and the diode is not at the pin

`D30` and `D32` (SS34) are oriented correctly: cathode (SMA pad 1) on `OUT1_12V` / `OUT2_12V`, anode on GND. That is the right way around for a high-side switch and a load returned to GND. Polarity is not the bug.

TPS2596 OUT absolute minimum is −0.3 V. An SS34 at half an amp to a couple of amps drops about 0.45–0.55 V. The pin is already past its absolute maximum for the whole time the maglock current is decaying. On top of that, the diode is not next to the pin: `U9` pin 5 to `D30` pin 1 is 15.5 mm, `U8` pin 5 to `D32` pin 1 is 14.1 mm. TI’s layout section says the Schottky has to be physically at the OUT pin because the trace between them is the inductance that the pin sees during commutation. There is a local 1 µF (`C34` / `C39`), which helps the first instant, and then the diode has to hold the current.

A damaged output FET that fails short leaves that jack stuck on, ignoring GPIO32/GPIO33. For a door maglock that is the wrong way to fail: the door stays locked with nobody able to turn it off from Home Assistant. Put a low-forward-drop Schottky on the eFuse OUT pin and GND, and still fit a diode at the lock.

Evidence: board pad nets of `D30`/`D32`; TPS2596 absolute maximum VOUT min −0.3 V and section 10.1; measured pad-to-pad distances.

### S2. A shorted reader cable does not trip the eFuse. Firmware is the only disconnect.

TPS259630 current-limits. It does not pull FLT for an overload or a short (SLVSET8A Table 3). FLT goes low for thermal shutdown, OVLO, or a shorted ILM pin. Thermal shutdown on the latch-off variant (`x0`, which is the part in the BOM) sticks off until EN is toggled. That **does** protect a dead short on the 12 V jacks: dissipation is on the order of 12 V × 1.9 A until the die hits 157 °C, then it latches. Firmware is only a second layer there.

A short on RJ45 pin 7 is different. Limit is about 0.24 A (RILM 3.9 kΩ, `ILIM = 903/R + 0.0112`). Power in the eFuse is about 3.4 V × 0.27 A ≈ 0.9 W. At the datasheet 52.7 °C/W (exposed pad soldered) that is about +47 °C, so the die sits near 90–110 °C in a warm box and never reaches 157 °C. Each PowerPAD on this board has two 0.3 mm vias (`U101`, `U201`, `U801` checked), so the real thermal resistance is worse than 52.7 °C/W, but not so much worse that a 0.9 W short is guaranteed to latch. The port just cooks.

The firmware over-current path needs the ADS1115 and I2C, waits through a 1.2 s blanking time plus two samples, and can be defeated by an ESP32 that is hung but not reset. `ports_safety_off` and the 180 mA threshold are real, and they are not a hardware limit. Same for the HA numbers: output trip can be set to 1.8 A, input trip to 6.0 A, temperature trips to 90 °C and 105 °C, and those values are restored. If I2C dies, `on_value` never runs and nothing turns off.

Fix: a port limit low enough that a short is obviously inside the part’s thermal shutdown, or an eFuse configuration that latches on over-current. Do not describe the HA automations as the protection.

Evidence: TPS2596 electrical table (ILIM, TSD 157 °C, RθJA 52.7 °C/W with the pad soldered, 119.8 °C/W without) and fault-response table; `R101` = 3.9 kΩ; two GND vias under `U101` pad 9; `rfid-hub.yaml` number entities and the 500 ms / 1 s sensors.

### S3. The plastic box will get hotter than the shutdown thinks, and the bucks cannot be turned off

Owner’s update: no metal standoffs. PLA or PETG ledges and clips hold the board.

Continuous heat on the board, both 12 V outputs working and the readers on, is a few watts: the two bucks (TI’s own loss estimate is in the 0.4–0.6 W range each at these loads), the ESP32 (Wi-Fi bursts), about 0.2 W in the input FETs, and I²R in the 12 V eFuses (about 0.5 W each at 1.9 A if Rds is hot). The 12 V loads themselves heat the prop, not the PCB, except when an output is in current limit.

PLA softens around 60 °C. PETG is useful to about 80 °C and then creeps. The firmware trip that turns the 12 V outputs off defaults to 80 °C and can be raised to 90 °C. The port trip defaults to 95 °C. Both are board-NTC readings (`TH1` between `U4` and `U6`, `TH2` between `U8` and `U9`), updated every 5 s. A PLA clip on the copper next to a buck can be soft before that trip. The bucks’ EN pins are a resistor divider from `12V_BUS` (`R22`/`R23`, `R72`/`R73`), not a GPIO, so an over-temperature trip cannot shut the converters off. They keep making heat after the outputs and the readers are off.

The thermal path off the packages is thin for the datasheet numbers those loss figures assume:

- `U8`/`U9` exposed pads: two 0.3 mm vias each. TI’s 52.7 °C/W figure is for the pad tied into a plane with a real via array.
- `U4`/`U6` are SOT-23-6 with no exposed pad. Heat leaves through GND pin 1. There is one GND via at that pin. JEDEC RθJA for this package is 119 °C/W; 0.6 W is a 70 °C rise on that number, which puts the board under the part near 100 °C if the air in the box is already warm.

Enclosure rules that should be written down before this board is closed up:

- Vent over `U4`, `U6`, `U8` and `U9`. Do not seal it.
- PETG rather than PLA if the box touches the board near the bucks. If it has to be PLA, set the temperature trip near 55 °C and accept that it is still only software.
- No carbon-filled or other conductive filament. The bottom is a GND plane under soldermask, and the barrel-jack and RJ45 pins are bare metal on the bottom.
- No parts on the bottom (confirmed). The jack and RJ45 tails still stick through. The ledges have to clear those tails, and the floor has to be thick enough that a 12 V tail cannot touch a metal shelf the box sits on.
- Support the board at the jack edge. `C2` (470 µF) is 8 mm from `J1`, `C4` (1206, 12 V) is 13 mm from `J1`. Plugging a barrel jack into an unsupported 1.6 mm board is how 1206 ceramics crack and short 12 V to GND. The fuse would then be the thing that saves it.

Mounting holes do **not** need a copper change for plastic bosses. `H1`–`H4` are 3.2 mm non-plated holes with no net, and there are keepouts named `hole_H1`…`hole_H4`. Nearest 12 V copper is `D33` to `H4`, 5.2 mm from the hole center to the pad. A plastic boss is fine. A metal screw is a bad idea later; don’t add one, and don’t plate the holes.

Evidence: NTC placement and `t_out_off` / `t_ports_off` in `rfid-hub.yaml`; buck EN nets `EN2`/`EN3` not on a GPIO; via counts under `U4`, `U6`, `U8`, `U9`; hole pads have an empty net; `C2`/`C4` distances to `J1`.

### S4. “12 V 5 A covers both outputs at full limit” is not an honest margin

TPS2596 recommended RILM is 453 Ω to 7.87 kΩ, and recommended continuous current is 2 A. The 12 V outputs use 470 Ω (`R31`, `R37`). Typical limit is `903/470 + 0.0112 = 1.93 A`. The characterized window at 453 Ω is 1.83–2.15 A. 470 Ω is only 4 % above the minimum (a 1 % resistor still stays above 453 Ω; a 5 % resistor would not). Scaling that window puts the high side near 2.1 A, which is above the 2 A continuous rating.

Two outputs at that high side, plus the hub (eight readers at the 1.23 A design number and the logic rail), is about **4.8 A at 12 V**. A 5 A wall supply is then at roughly 95 %, before cable drop and before the supply’s own optimism. The 7 A fuse does not protect that supply. Littelfuse’s own guide is to run a fuse at no more than 75 % for long life (5.25 A at 25 °C). 4.8 A is inside that fuse rule and outside a comfortable 5 A brick.

Practical maglocks are a few hundred milliamps, and a 5 A supply is plenty for those. The quote and the manual should not say both jacks can sit on the eFuse ceiling at once from a 5 A brick. Either spec a 12 V supply of at least 6 A, or set RILM near 600 Ω (about 1.5 A typical, worst case still under 2 A) so the sum stays near 4 A.

The fuse itself (Littelfuse 0451007, melting I²t 10.32 A²s, 125 V, 50 A interrupt) is a reasonable backstop for a hard fault. It is not the thing that saves a cheap 5 A adapter from a 4.8 A load.

Evidence: TPS2596 recommended conditions and the 453 Ω ILIM row; `R31`/`R37`; Littelfuse 451 series table for 0451007 and the 75 % application note.

### S5. Reader over-voltage trip is 40 mV under a 4.0 V absolute maximum

Port OVLO is one divider for all eight eFuses: `R74` 2.2 kΩ / `R75` 1 kΩ from `3V3_PORT`. TPS2596 OVLO rising threshold is 1.17–1.22 V. With 1 % resistors the trip is 3.69 V minimum, 3.84 V typical, **3.96 V maximum**. The MFRC522 supply absolute maximum used in this design is 4.0 V. The part can be at 3.9 V, above its operating maximum, before the eFuse is guaranteed to open. Move the divider down so the worst-case trip is under 3.6 V.

The divider impedance was lowered on purpose (about 0.7 kΩ). That part is fine. The setpoint is the problem.

Evidence: TPS2596 VOVLO(R) 1.17/1.20/1.22 V; `R74`/`R75`.

### S6. 24 V is blocked only by the series FETs. The TVS does not clamp a 24 V brick.

This path is wired correctly, and a 24 V brick with a healthy `U1`/`Q1`/`Q2` does not reach the bucks:

- `U1` pads match the LM74502 DDF pinout. EN (pin 1) and VS (pin 5) are both `VIN_F`. OV is the `R7` 120 kΩ / `R8` 10 kΩ divider. Cutoff, including the 1.165–1.333 V threshold and 1 % resistors, is **14.9 V to 17.7 V**. 24 V is above the top of that range, so GATE stays off. The FETs are off until the charge pump runs, and Q1’s body diode faces the wrong way to pass forward current, so a hot-plug of 24 V does not sneak through before the comparator wakes up.
- `Q1` drain and tab are `VIN_F` (input side), `Q2` drain and tab are `VIN_P` (load side), sources tied on `Q_SRC`. That is the back-to-back arrangement. AON6262E is 60 V, VGS ±20 V. LM74502 gate drive tops out at 13.9 V, so the gates are inside the FET rating. LM74502 VS is rated −65 V to +65 V, so a reversed 24 V brick is inside the controller rating too.
- `D2` SMBJ33CA is **after** `F1`, on `VIN_F`. Standoff 33 V, so a regulated 24 V brick does not make it conduct. That is intentional, and it means the TVS is not a second layer against 24 V. It is a surge clamp. If `Q2` fails short, 24 V reaches `12V_BUS`. The bus TVS `D3` (SMAJ17A, breakdown about 19–21 V) will then conduct and burn; the 7 A fuse is what eventually opens. The 12 V eFuses are only rated to 21 V on IN. A slow rise might be caught by their own OVLO (divider `R34` 150 kΩ / `R35` 10 kΩ trips at about 19.2 V, which is under 21 V). A step to 24 V is already an absolute-maximum violation for the 1.3 µs OVLO delay.

Good enough to believe the primary circuit. Not good enough to call 24 V a solved problem if a FET fails. A bus clamp that is actually sized to hold the rail under 21 V, with the fuse, would be the second layer. Do not move the input TVS down to 17 V: it would fight the cutoff and a slightly high 12 V brick.

`C2` is 470 µF **25 V** on `12V_BUS`. It is fine at the 17.7 V cutoff. It is not fine if 24 V gets through.

Evidence: `U1`, `Q1`, `Q2`, `D2`, `F1`, `D3`, `R7`, `R8` pad nets; LM74502 electrical table; AON6262E pinout and VGS; TPS2596 IN absolute maximum 21 V.

---

## NICE TO HAVE

- **LM74502 gate slew.** There is no CdVdt on GATE. TI’s equation for 470 µF and a few amps of inrush wants a few nanofarads there. The 60 µA gate drive still soft-starts: two FETs are about 90 nC, so the gate moves in about 1.5 ms. Even a pessimistic 8 A triangular pulse for 1 ms is 0.02 A²s, against 10.32 A²s melting I²t on the 7 A fuse. The fuse will not nuisance-open. A small gate capacitor would still be kinder to the FETs’ linear-mode SOA.
- **ESD on RJ45 pin 7.** Signal pins have SRV05-4 (`D101` pad 2 = GND, pad 5 = `P1_3V3`, the other pads on SS/SCK/MOSI/MISO; `D102` covers RST and pin 8). That matches the Semtech pinout on the standard SOT-23-6 footprint. Pin 7 relies on the 4.7 µF and the eFuse. A real TVS from pin 7 to GND would take ESD without using the eFuse as the clamp. Barrel jacks already have SMAJ/SMBJ devices. This is a 12 V SELV board, not a mains product; 0.15 mm soldermask clearance is enough for the voltages that are supposed to be there. Exposed jack gaps are the connector’s own spacing (PJ-063 withstands 500 Vac; the RJ45 is rated 125 Vac).
- **Confirm RJ45 pin 1 on a physical jack before fab.** The custom footprint matches the Ckmtw drawing on the things the drawing actually dimensions: 1.02 mm pitch, 1.78 mm row stagger, 12.70 mm between the 3.25 mm plastic holes. The drawing does not print a pin number. If pin 1 is on the wrong end, cable pin 7 is not 3.3 V and the MFRC522 gets 3.3 V on an IO pin. The eFuse limits the current, so this is a dead reader, not a fire. Ronnie was already going to check a breakout. Do that against this footprint, not against the schematic alone.
- **Output-jack back-feed.** `J2`/`J5` are PJ-063BH (2.5 mm pin) on the same land pattern as the 2.0 mm input. A 5.5×2.1 mm brick does not mate. A 5.5×2.5 mm brick does, and it back-feeds `12V_BUS` through the eFuse body diode (OUT absolute max is VIN+0.3 V). Bottom silkscreen already says not to plug a supply in. A series diode would close it. Low priority next to B1.
- **`R49` short.** The USB sense divider is 1 MΩ series into GPIO36, which is the right way to do it. If that resistor shorts, GPIO36 sees the FNK 5 V pin directly.
- **More vias under the eFuse pads and a real GND pour on buck pin 1.** This is the layout half of S3. Two vias is electrically connected; it is not the thermal design the 52.7 °C/W number assumes.

---

## VERIFIED OK

These were checked on the board pads and against the datasheet, not taken from the 67-pass spreadsheet.

**Reverse polarity.** Back-to-back AON6262E as above, LM74502 rated to −65 V, bidirectional SMBJ33CA that does not conduct at a reversed 12 V or 24 V (33 V standoff). A center-negative 12 V plug leaves the board off. It does not power the bucks backwards.

**Input fuse vs inrush and vs a hard fault.** 0451007 is after the jack and before the TVS, which is the right order: a TVS that clamps hard opens the fuse instead of burning forever. 4.5–4.8 A is under the 75 % long-life current at room temperature. Hot-plug inrush into `C2` is limited by the 60 µA charge pump, and the I²t is a small fraction of 10.32 A²s. Interrupt rating 50 A is more than a wall brick can deliver.

**USB + 12 V, normal case.** See B2. `U5` pinout on the board matches SLVSEZ8A, CE is tied to VOUT, and the hub does not source the FNK 5 V pin. With the buck in regulation this does not back-feed a laptop. The failure case is B2, not this case.

**12 V outputs fail off when the ESP32 resets, boots, or is flashed.** `OUT1_EN` is GPIO32, `OUT2_EN` is GPIO33, each with `R30`/`R36` 100 kΩ to GND. Those are not strapping pins. ESPHome `restore_mode: ALWAYS_OFF`. During reset the pins are high-Z and the pull-downs hold EN at 0 V, which is under the TPS2596 UVLO-falling minimum of 1.08 V. A crash that actually resets the chip (the ESP-IDF watchdog) turns the maglocks off and they stay off. That is the right way for a room with people in it: power loss unlocks. A hang that does not reset leaves the jacks where firmware left them; the eFuse still current-limits and a hard short still latches off (S2).

**Reader ports fail off through reset.** TCA9555 powers up as inputs with about a 100 kΩ pull-up. The board has 10 kΩ to GND on each EN (`R103` and the rest). Divider voltage is about 0.31 V, under 1.08 V. Firmware writes the output latch low before it turns the pins into outputs (`rfid_hub.cpp`), so there is no 0xFF glitch onto EN. Ports stay off until the hub decides to probe them.

**12 V jack short.** Current limit, then thermal shutdown, then latch. FLT is what firmware sees after the fact. The copper from `U9`/`U8` pin 5 to `J2`/`J5` pin 1 is 2.0 mm (1 oz), with a 0.61 mm neck only 0.24 mm long at the SOIC pin. IPC-2221 gives a 2.0 mm external 1 oz trace about 4.0 A at 10 °C and 5.4 A at 20 °C. IPC-2152 with the inner plane under the trace is looser than that, so 2 A is inside both. The skinny 0.09 mm tracks on `OUT2_12V` are a parallel autorouter snake, not the path to the jack (the jack is on the 2.0 mm segments at y = 187.46). They are ugly and worth deleting; they are not the fuse.

**Input and 12 V plane copper.** Jack to fuse is 2.0 mm (`VIN_JACK`, about 9 mm). The shunt’s power path is 2.0 mm into a cluster of six 0.3 mm vias on the 12 V side of `R2`, then the In2 `12V_BUS` pour (2570 mm²). A 0.5 oz internal pour that wide is on the order of 8 A at 10 °C by the IPC-2221 internal formula, and the three In2 pours do not overlap. The 0.25 mm tracks near `R2` are the INA226 Kelvin sense, not the power path. Port 3.3 V routing is narrow (0.09–0.12 mm in places, including 6.6 mm of 0.09 mm on `P3_3V3`) but at the 0.24 A limit that is about 2 mW. IPC-2221 gives 0.09 mm about 0.4 A at 10 °C even as a long trace. Fine at the eFuse limit. Not fine if a port eFuse ever fails short and the buck dumps an amp into it; that is a second fault.

**Connectors.** PJ-063AH and PJ-063BH are 8 A, 24 Vdc (Same Sky, September 2025 datasheet). 4.8 A in and 2 A out are inside that. The 24 V misuse case is inside the jack’s voltage rating; the electronics are what have to block it (S6). RJ45 contacts are 1.5 A, operating range 0 to 70 °C. 0.27 A is inside the current rating. The 70 °C connector rating is another reason not to let the box air sit at the 80 °C firmware trip.

**Reader-port behavior that is contained.** Pin 7 shorted to pin 5: that port current-limits, other ports stay up, the ESP32 is on the other buck. Pin 7 shorted to a data pin: the data pin is already a 3.3 V logic net, and the series 100 Ω + 100 Ω resistors (`R110`–`R117` and the copies) hold buffer current to about 16 mA. A crossover cable does not swap pins 7 and 5. A custom cable that swaps them reverse-powers the reader; the hub is current-limited and the reader chip is not. That is a damaged reader, which is why B1’s series protection is the real fix.

**Buzzer.** `D42` cathode is on `3V3_BUCK`, anode on the low-side drain. The coil is powered from `3V3_BUCK`, upstream of `U5`, so USB-only bench power does not run the buzzer. Gate has `R61` 100 kΩ to GND.

**Clearance for this voltage.** 0.15 mm with soldermask is the right ballpark for SELV under 30 V. Nothing here is mains. Creepage is not the reason to hold the order.

---

## What this means for the order

Do not send rev A to JLCPCB. The two changes that matter before copper is cut:

1. Stop a foreign voltage on RJ45 pin 7 from reaching `3V3_PORT` (B1).
2. Clamp `3V3_BUCK` and `+3V3` so a failed feedback resistor cannot put 12 V into the ESP32 and, if USB is plugged in, into the path toward the laptop (B2).

With those, the 12 V input circuit, the ideal diode, and the default-off outputs are in good enough shape to build. Still do S1 (maglock diode at the pin) on the same spin if maglocks are a real load. S3 is mostly the box: vent it, don’t use a sealed PLA shell, and don’t expect the 80 °C Home Assistant trip to protect the plastic. S4 is a one-line change to two resistors or a change to the supply Ronnie actually buys.

The designer’s calculation log scores 67 passes and 0 fails. Several of those passes are true (cutoff math, fuse I²t, LM66100 connection, GPIO32/33 pull-downs). The ones that are not true are the ones that treat a label, a firmware threshold, or a 40 mV margin as protection.
