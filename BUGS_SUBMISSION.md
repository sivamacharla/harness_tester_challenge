# comma.ai Harness Tester Challenge — Bug List

Prepared for submission via https://forms.gle/US88Hg7UR6bBuW3BA

---

## Firmware (`firmware/firmware.ino`, `firmware/CY8C9560.*`)

1. **Inverted start-button logic.** `loop()`: `if (digitalRead(PIN_BTN_TEST) == LOW) return;`. BTN_TEST is pulled HIGH via R4 (10k to +3.3V) and pulled LOW when pressed (per schematic). As written, the tester runs continuously while the button is untouched and *stops* when you press it — exactly backwards from "press start button to begin test."

2. **Pass/fail logic is OR instead of AND.** `passed` starts `false` and is set `true` if *any single pin* out of the 40 matches its expected connectivity:
   ```cpp
   if (values == EXPECTED_CONNECTIONS[i]) {
     passed = true;
   }
   ```
   One correctly-wired pin marks the *entire* 40-pin harness "PASSED" even if the other 39 are wired wrong — this defeats the entire purpose of the tester.

3. **32-bit shift overflow / UB.** `uint64_t output_mask = 1 << i;` (also in the debug-print loop `1 << j`). `1` is a 32-bit `int` literal; shifting it by `i >= 31` is either wrong or undefined behavior in C++, so pins ~31–39 of the 40-pin harness are never driven/read correctly. Needs `1ULL << i`.

4. **Firmware's linear bit-index assumption doesn't match the expander's real pin layout — breaks roughly half the harness test.** Verified by mapping every `GPortN_BitM` pin function on U4 to its actual `CBL_x` net via the schematic netlist:
   - Bits 0–19 → `CBL_0`–`CBL_19` (contiguous, matches what the firmware assumes)
   - **Bits 20–23 don't exist as GPIO at all** — Port 2's bits 4–7 are the chip's fixed-function `SCL`/`SDA` pins (confirmed: U4 pin 24 = `SCL` → net `CY_SCL`, pin 28 = `SDA` → net `CY_SDA`), not general-purpose I/O.
   - Bits 24–43 → `CBL_20`–`CBL_39` (shifted 4 bits higher than where the firmware's `1 << i` loop expects them).

   The firmware's test loop (`firmware.ino`) assumes harness pin `i` lives at bit `i` of the expander's 64-bit I/O register — true for `CBL_0`–`CBL_19`, but completely wrong for `CBL_20`–`CBL_39`, which actually live 4 bits higher. This is a separate, independent bug from #3 above (the int-overflow bug) — even after fixing `1 << i` to `1ULL << i`, the back half of the connector would still be tested against the wrong physical pins.

5. **Missing `pinMode(OUTPUT)` for GPS control lines.** In `setup()`, `digitalWrite(PIN_UBX_SAFEBOOT, LOW)` and `digitalWrite(PIN_UBX_RST_N, HIGH)` are called without ever setting those pins to `OUTPUT`. They default to `INPUT`, so these calls just toggle an internal pull resistor instead of actually driving the u-blox module's SAFEBOOT/RESET lines — GPS boot state is left undefined.

6. **Unbounded NMEA buffer write (overflow).** `nmea_buf[64]` in `loop()`: `nmea_buf[nmea_idx++] = UBX_SERIAL.read();` has no bounds check. A sentence without `\n`/`\r` in the first 64 bytes overflows the buffer, and `process_nmea`'s `buf[len] = 0;` can write one byte past the end of the array.

7. **`CY8C9560::begin()` never releases the expander from reset.** Verified against the real schematic netlist (see below). The reset sequence in `firmware/CY8C9560.cpp`:
   ```cpp
   digitalWrite(CY_RST, HIGH);
   delay(10);
   digitalWrite(CY_RST, LOW);
   delay(100);
   WIRE.begin(); ...
   return read_id() == 0x06;
   ```
   `CY_RST` drives the chip's active-low `RESET_N` pin (netlist confirms: Teensy pin 22 → net `CY_RST_N` → U4 pin 62 `RESET_N`). The sequence ends on `LOW` (reset asserted) and never drives it back `HIGH` before `WIRE.begin()`/`read_id()` run — every I2C transaction happens while the expander is held in reset. `begin()` can never succeed, and no harness-pin testing can ever work.

8. **No I2C error checking anywhere in the CY8C9560 driver.** `read_register()`, `read_registers()`, `write_register()`, and `write_registers()` in `CY8C9560.cpp` never check the return value of `WIRE.endTransmission()` or `WIRE.requestFrom()`. If the I2C bus fails for any reason (e.g. bug #16's dead SDA pull-up, a bad reset per bug #7, a disconnected chip), every read silently returns garbage/`0xFF` and every write is silently dropped — the firmware has no way to detect or report an I2C fault, it just proceeds as if everything succeeded.

9. **No debounce or single-shot logic — the harness test re-runs and re-logs every single `loop()` iteration while triggered.** There's no state tracking of "test already ran, wait for button release." As soon as the trigger condition in `loop()` is satisfied, `log_result()` (which appends a line to `results.txt` on the SD card) fires again on the very next iteration, with no delay anywhere in that code path. `loop()` runs continuously and extremely fast, so holding the button (or leaving it in the untriggered state, per bug #1) floods the SD card with thousands of duplicate log entries per second instead of logging once per test. This is independent of bug #1 — it exists regardless of which direction the button logic points.

## Schematic — verified via `kicad-cli sch export netlist`

10. **No current-limiting resistor on the RGB status LED (D3).** `LED_R`/`LED_G`/`LED_B` wire straight from Teensy GPIOs to the LED with no series resistor anywhere in the schematic/BOM — this will overdrive the LED and/or the GPIO pins.

11. **No pull-up / defined idle state on the GPS module's `~RESET`/`~SAFEBOOT` lines.** Combined with bug #5, whenever the Teensy pin isn't actively driven (e.g. during MCU boot/reset), the GPS module's reset/boot state floats — can hold the module in reset or an undefined boot mode.

12. **Reverse-polarity-protection MOSFET (Q1) has Source and Drain swapped.** Netlist confirms Q1's Source pins (S1–S3) tie to the *protected* `+12V` rail (feeding the L7805, decoupling caps, everything downstream), while Q1's Drain ties to the *raw, unprotected* input straight from J1 (net `Net-(D1-A1)`). The gate is pulled to `GND` via R1 — standard practice for a high-side P-channel "ideal diode," which is only correct when Source faces the raw input and Drain faces the protected rail. As wired here (reversed), the MOSFET's body diode blocks forward current entirely — the board would never power up even with a correctly-polarized 12V input. (D1, the TVS diode, is correctly placed straight across the raw input to GND for transient clamping — that part checks out fine.)

13. **NEO-M8N's `VDD_USB` pin (pin 7) is tied directly to `GND` instead of +3.3V/VCC.** Netlist confirms pin 7 (`pinfunction VDD_USB_7`) sits on the exact same net named `GND` as the module's real ground pins (10, 12, 13, 24). Per u-blox's hardware integration manual, `VDD_USB` must be tied to VCC even when USB isn't used (it isn't — `USB_DM`/`USB_DP` are correctly left unconnected here since UART is used instead). Grounding a VDD-labeled supply pin outright is a genuine wiring error — at best the internal USB regulator is left in an undefined state, at worst it's an effective short that stresses the module.

14. **Silkscreen advertises "Ethernet" and "USB Host" connector locations that were never actually designed in.** The top silkscreen has labeled cutout areas for "Ethernet" and "USB Host" (visible in the rendered board), but the schematic includes no RJ45 jack/magnetics footprint and no USB-A host connector — confirmed via netlist: none of the associated Teensy 4.1 pins (Ethernet: `D+`/`D-`/`LED`/`T+`/`T-`/`R+`/`R-` on pins 60–67; USB Host: the second `D+`/`D-` pair) are wired to anything. These are silkscreen-only labels for features that don't exist in the design.

15. **R3 (I2C SDA pull-up) is wired to GND instead of +3.3V.** Net `CY_SDA` contains only R3 pin 2, U2 pin 17 (Teensy SDA), and U4 pin 28 (SDA) — but R3 pin 1 sits on `GND`, not `+3.3V` (compare R2, the SCL pull-up, whose pin 1 correctly sits on `+3.3V`). R3 is a pull-*down*, not a pull-up: SDA has no real pull-up and is instead fighting toward ground, breaking I2C communication with the CY8C9560 expander — the entire harness test depends on this bus.

## PCB layout

16. **Wrong footprint/package for U4 (CY8C9560A-24AXIT).** The schematic symbol and PCB footprint use 100-pin numbering (pads run to pin 100, e.g. `GPort0_Bit0_PWM7` on pin 95, `GPort0_Bit2_PWM3` on pin 99). The real CY8C9560A only ships in a 68-pin QFN/TQFP package — the chip physically cannot be soldered onto this footprint.

17. **D3 (status LED) common pin is completely unconnected on the PCB.** KiCad DRC's connectivity check reports a missing connection: Pad 1 of D3 (`+3.3V`, the common-anode pin) has no net connection on the copper layer — there's a stray 0.68mm track stub nearby that never reaches the pad, even though the schematic net is correct. Combined with bug #10, the LED circuit is broken twice over.

18. **199 trace-width violations** — board's own design rules require ≥0.2mm trace width; many tracks are routed at 0.127mm. Concentrated on the actual harness signal nets (`CBL_20`, `CBL_38`, `CBL_39`, `CBL_22`, `CBL_24`, `CBL_4`, etc. — 22 distinct nets total), plus `LED_R`/`LED_G`/`LED_B` and the GPS `VCC_RF` net. These are below what most fabs can reliably manufacture — real risk of opens on the very lines the tester exists to check.

19. **510 clearance violations** — declared minimum clearance is 0.2mm; actual spacing goes down to 0.089mm. Root cause: the GND/+3.3V copper-pour zones are configured with a 0.15mm pad clearance setting, which is *narrower than the board's own stated 0.2mm minimum clearance rule* — an internal inconsistency that trips DRC wherever a pour runs near a `CBL_x` signal, `GND`, `+3.3V`, `CY_INT`/`CY_RST_N`, or the I2C bus (`CY_SCL`/`CY_SDA`). Risk of solder bridges/shorts during fabrication.

20. **Courtyard overlap + silkscreen-over-copper on the GPS front-end.** Footprints for C6 and U5 (MAX2679 LNA) physically overlap (courtyard violation). Related: U5's silkscreen artwork is printed directly over an exposed copper pad of C6 — a legend/solder-mask defect.

21. **Board-edge clearance violations (4 instances).** J1 (12V input connector)'s mounting hole sits 0.25mm from the board edge (rule requires 0.5mm); Teensy pins 1 (GND) and 48 (+5V) sit 0.48mm from the edge cut — both risk being nicked or exposed during panel depaneling/edge routing.

22. **Silkscreen text below manufacturable minimums.** U2's footprint reference text is 0.7mm tall (min 0.8mm required); the "Hardware Hiring Challenge" board title text has insufficiently thick strokes for reliable printing.

23. **D1 (TVS diode) has no polarity marker on its silkscreen.** Unlike D2 and Q1 nearby, D1's silkscreen outline has no cathode band/line — an assembly risk (easy to place backwards), though lower-confidence than the netlist/DRC-verified items above.

24. **83 silkscreen-clipped-by-board-edge violations, concentrated on J3.** DRC's silkscreen-edge-clearance check found 68 instances on J3 (the 40-pin cable header) alone, plus 3 on AE1 and 12 on U2 — meaning J3's own pin-reference silkscreen is placed so close to the board edge that most of it is cut off by the board outline. This is the connector whose pin identity matters most for a harness tester, and its labels are largely unreadable/unprinted as laid out.

25. **32 isolated (floating) copper islands in the GND plane.** DRC flags 32 separate disconnected fragments of the main ground pour across F.Cu/B.Cu/In2.Cu — i.e. the "solid" ground plane is actually broken into disconnected islands rather than one continuous return path. Isolated copper like this doesn't function as a ground plane where it's floating and can act as an unintended antenna/EMI contributor.

---

## Retracted (do not submit)

~~`CY_RST`/`CY_IRQ_N` pins swapped~~ — this was based on manually reading pin numbers out of the PDF-exported schematic text, which came out jumbled/unordered. After exporting the actual netlist and checking real net membership, Teensy pin 22 → net `CY_RST_N` → U4 pin 62 (`RESET_N`), and Teensy pin 23 → net `CY_INT` → U4 pin 45 (`INT`) — i.e. `CY_RST=22` / `CY_IRQ_N=23` in the firmware **is correct**. The real bug in this area is #7 above (reset never released).

---

## Appendix: granular DRC data (if you want to split #18/#19 into more line items)

### Track-width violations (199 total) by net — all below the 0.2mm board minimum, down to 0.127mm:
```
20x CBL_20        13x Net-(U3-VCC_RF)   11x CBL_24         8x CBL_13       6x LED_B
16x CBL_38        13x LED_G             10x CBL_4          8x CBL_35       6x CBL_9
15x CBL_39        13x CBL_22             8x CBL_36         8x CBL_5       6x CBL_7
12x LED_R                                                                  5x +3.3V
                                                                            3x CBL_10
                                                                            2x Net-(AE1-A)
                                                                            2x CBL_0
```

### Clearance violations (510 total, 95 unique net-pairs), all below the 0.2mm board minimum, down to 0.089mm:
```
19x +3.3V/GND             8x +3.3V/UBX-RXD       5x CBL_22/GND          3x CBL_28/GND
16x GND/UBX-TIMEPULSE     8x CBL_5/GND           5x GND/Net-(U3-RF_IN)  3x GND/LED_B
15x CY_INT/GND            8x +3.3V/CBL_39        4x CBL_11/GND          3x GND/LED_G
14x GND/Net-(U3-VCC_RF)   8x CBL_20/GND          4x CBL_26/GND          3x GND/LED_R
13x CY_INT/CY_RST_N       8x CBL_36/GND          4x +3.3V/CBL_26        3x GND/Net-(D1-A1)
13x CY_RST_N/GND          7x CBL_18/GND          4x CBL_23/GND          2x CBL_9/GND
11x GND/UBX-SAFEBOOT      7x CY_SDA/GND          4x +3.3V/CBL_25        2x CBL_16/CY_RST_N
11x +3.3V/CBL_38          7x CY_SCL/GND          4x CBL_8/GND           2x GND/Net-(L1-Pad2)
11x +3.3V/LED_R           7x +3.3V/CBL_23        4x CBL_21/GND          2x +3.3V/CBL_6
 9x +3.3V/CY_SCL          7x +3.3V/CBL_27        4x CBL_39/GND          2x +3.3V/CBL_8
 9x CBL_2/GND             7x CBL_38/GND          4x CBL_4/GND           2x GND/Net-(D2-A)
 9x +3.3V/CBL_21          7x +3.3V/CBL_22        4x +3.3V/LED_B         2x +3.3V/Net-(U3-VCC_RF)
 9x +3.3V/CBL_17          6x +3.3V/BTN_TEST      3x +3.3V/CBL_18        1x GND/Net-(AE1-A)
 9x +3.3V/LED_G           6x +3.3V/CY_SDA        3x GND/UBX-TXD         1x CBL_10/GND
 8x GND/UBX-RST_N         6x CBL_16/GND          3x GND/UBX-RXD         1x +3.3V/CY_INT
                          6x +3.3V/CBL_19        3x CBL_12/GND          1x +3.3V/UBX-SAFEBOOT
                          6x CBL_25/GND          3x CBL_33/GND          1x +3.3V/CY_RST_N
                          6x CBL_37/GND          3x CBL_14/GND          1x +3.3V/UBX-RST_N
                          6x +3.3V/CBL_37        3x CBL_15/GND          1x +3.3V/UBX-TIMEPULSE
                          6x CBL_13/GND          3x CBL_34/GND          1x +3.3V/CBL_16
                          5x CBL_29/GND          3x CBL_7/GND           1x CBL_0/GND
                          5x +3.3V/UBX-TXD                              1x GND/Net-(C5-Pad2)
                          5x BTN_TEST/GND
                          5x CBL_6/GND
                          5x CBL_1/GND
                          5x CBL_30/GND
                          5x CBL_3/GND
                          5x CBL_19/GND
                          5x CBL_27/GND
                          5x CBL_31/GND
                          5x CBL_17/GND
                          5x +3.3V/CBL_7
                          5x CBL_32/GND
                          5x CBL_35/GND
                          5x CBL_24/GND
                          5x +3.3V/CBL_24
                          5x CBL_22/GND
```

### Silkscreen-edge-clearance violations (83 total) by footprint:
```
68x J3   12x U2   3x AE1
```

Full breakdown (all 22 track-width net-groups + all 95 clearance net-pairs) saved at `kicad_files/full_appendix.txt`. Full raw DRC JSON at `kicad_files/drc_report.json`. Full raw ERC JSON at `kicad_files/erc_report.json`. Full exported netlist at `kicad_files/netlist.net`.

## Ruled out (checked, confirmed fine — don't submit)

- RF matching network (L1=12nH, DC-block cap=1nF, decoupling=100nF) matches Maxim's own MAX2679 EVKIT reference values exactly — correct as designed.
- GPS `VCC_RF` net (U3 pin 9) only ties to a decoupling cap and the LNA's Vcc — this is correct; it's the module's own internally-generated RF supply output, not something needing external power.
- All footprints are on the top side only — no mirrored/backwards-mounted parts.
- J3's `CBL_x` numbering is sequential and correct (pin *N* → `CBL_(N-1)`), no shuffle.
- I2C pull-up value/placement (R2 for SCL) and I2C bus routing to both U2 and U4 is otherwise correct.
