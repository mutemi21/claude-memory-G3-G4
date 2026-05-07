# CHK Power Board — Protocol Reference (corrected/extended)

This is the working reference for the **CHK 2 kW induction power board** UART
protocol. It rewrites the supplier's `UART_Protocol_CHK_V2 - EngVer 2.docx`
to match what we have actually observed on the wire and in firmware testing.
Where our findings disagree with the doc, both are stated and the source of
the deviation is called out.

The same CHK board is used on both **G3** and **G4**. This file is mirrored
in both projects' `memory/` directories — keep them in sync.

---

## 1. Physical & line parameters

| Item            | Value                                  |
|-----------------|----------------------------------------|
| Bus             | UART                                   |
| Baud rate       | 4800                                   |
| Data bits       | 8                                      |
| Stop bits       | 1                                      |
| Parity          | none                                   |
| Endianness      | bytes are sent in the order listed     |
| Pull-up         | 5.1 kΩ on **both** NTCs (per §4.2)     |

---

## 2. Frame structure (all directions)

```
+------+------+--------+----------+-----------+----------+------+
| 0xAA | 0x55 |  LEN   | DataType | DataPayld | Checksum | 0xFF |
+------+------+--------+----------+-----------+----------+------+
                |                                  ^
                |  LEN = DataType(1) +              |
                |        Data(n)    +               |  Checksum =
                |        Checksum(1)                |   sum of bytes
                                                    |   from 0xAA up
                                                    |   to and including
                                                    |   the last data byte,
                                                    |   modulo 256.
```

- `0xAA 0x55` is the start-of-frame.
- `LEN` is **Data Length** — the count of bytes from `DataType` through the
  checksum, **not** including the SOF or EOF.
- `0xFF` is end-of-frame.
- Total frame length on the wire = `LEN + 4`.

### Frame catalogue

| Direction | DataType | LEN  | What it is               | Total bytes |
|-----------|----------|------|--------------------------|-------------|
| MCU → CHK | `0x00`   | 0x0D | Initialization (write)   | 17          |
| MCU → CHK | `0x10`   | 0x09 | Control / heat command   | 13          |
| CHK → MCU | `0x10`   | 0x16 | Status / telemetry       | 26          |

(`0x20` Barcode and `0x30` Calibration data types exist per the doc but are
"for internal use" — we don't use them.)

---

## 3. Initialization frame (MCU → CHK)

Sent once at boot to configure load detection and power limits. Per supplier
§1, byte map:

| Idx | Byte                                   | Supplier default | Our value | Notes |
|-----|----------------------------------------|------------------|-----------|-------|
| 0   | `0xAA`                                 | -                | `0xAA`    | SOF |
| 1   | `0x55`                                 | -                | `0x55`    | SOF |
| 2   | Data Length                            | `0x0D`           | `0x0D`    |       |
| 3   | Data Type                              | `0x00`           | `0x00`    | init |
| 4   | Returned Data Type                     | `0x10`           | `0x10`    | tells board to reply with status |
| 5   | Returned Data Length                   | `0x20`           | **`0x14`** | see §13 — supplier value truncates board replies |
| 6   | Detection Load Parameter               | -                | `0x30`    | interval=3 s, load strength=0 |
| 7   | Maximum Limit PPG                      | `0xF8`           | `0xF8`    |       |
| 8   | Rated Voltage Ripple Threshold         | `0x15` (525 W)   | `0x15`    | unit = 25 W |
| 9   | Maximum Allowable Back-Pressure Count  | `0x3C` (60)      | `0x3C`    |       |
| 10  | Ripple Discrimination Time             | `0x18` (24)      | `0x18`    | "reserved in program, not called" per doc |
| 11  | Min Operating Power Code (N=Pmin/25)   | -                | `0x00`    | 0 W   |
| 12  | Max Operating Power Code (N=Pmax/25)   | -                | `0x50`    | 2000 W |
| 13  | Rated Voltage AD Value                 | -                | `0x00`    | TODO: hardware-cal dependent |
| 14  | IO_Config                              | -                | `0x00`    | all digital input |
| 15  | Pot Movement Parameter                 | `0x2C`           | `0x2C`    |       |
| 16  | Checksum                               | -                | computed  |       |
| 17  | `0xFF`                                 | -                | `0xFF`    | EOF |

**Init-frame settle time:** the firmware delays its first control frame by
~100 ms after queueing the init frame (`INIT_SETTLE_TIME_MS`) to let the
init bytes finish transmission and let the board process them — without
this gate, the first control frame races the init frame and the board
silently rejects the entire setup.

---

## 4. Control frame (MCU → CHK)

Sent every 110 ms by `TimerIsr` (TIM7). Byte map per supplier §3 (we use
the 13-byte form, no Music byte at index 11):

| Idx | Byte                            | Notes                                   |
|-----|---------------------------------|-----------------------------------------|
| 0   | `0xAA`                          | SOF                                     |
| 1   | `0x55`                          | SOF                                     |
| 2   | Data Length = `0x09`            |                                         |
| 3   | Data Type = `0x10`              | control                                 |
| 4   | Return Data Type = `0x10`       | board should reply with status          |
| 5   | Return Data Length = `0x14`     | see §13                                  |
| 6   | Peripheral Control              | bit 2 = fan, bits 1:0 = buzzer (00 off / 01 100 ms / 02 200 ms / 03 800 ms) |
| 7   | Power Output Switch             | see §4.1                                |
| 8   | PowerSetM (heating power)       | M = P/25, so 2000 W = `0x50`            |
| 9   | Fan-speed control               | unused → `0xFF`                         |
| 10  | Music / Reserved                | depends on tail mode (see below)        |
| 11  | Checksum                        | computed                                |
| 12  | `0xFF`                          | EOF                                     |

### 4.1 Power Output Switch (byte 7)

| Value     | Name      | Effect                                                                |
|-----------|-----------|-----------------------------------------------------------------------|
| `0x10`+   | HEAT      | normal heating; high nibble != 0 ⇒ heat                                |
| `0x00`    | STOP      | stop power, do not initialise operational data                         |
| `0x01`    | PAUSE     | stop power, **keep** operational data — used for thermal cuts          |
| `0x02`    | LOAD_DET  | periodic load detect at the init-frame interval                        |
| `0x03`    | NO_DET    | skip load detect on next power cycle                                   |

The firmware uses `0x10` for HEAT, `0x01` for PAUSE during a thermal cut or
when the fan needs to stay on, and `0x00` for STOP at full power off.

### 4.2 Tail byte (idx 10) values used

The firmware sets idx 10 to `0xE0` while `CurrentPowerState` is ON and
`0x3C` otherwise. The supplier doc labels this byte "Music / Reserved
features"; the values come from observing the supplier's PC tool.

---

## 5. Status frame (CHK → MCU)

Sent in response to every control frame (so also at ~110 ms). Per
supplier §4 the frame is 26 bytes (LEN = `0x16`, payload = 20 data bytes
+ checksum + EOF).

| Idx | Byte                                          | Notes / firmware use |
|-----|-----------------------------------------------|----------------------|
| 0   | `0xAA`                                        | SOF |
| 1   | `0x55`                                        | SOF |
| 2   | LEN = `0x16`                                  |  |
| 3   | DataType = `0x10`                             | status |
| 4   | **IHStatus** (see §6)                         | flags + error nibble |
| 5   | Voltage AD                                    | actual V = 127·g/128 + 3 |
| 6   | Current AD                                    | "do not use; use P/V instead" — supplier |
| 7   | IGBT temperature AD (high 8 bits)             | NTC, see §7 |
| 8   | Cooktop surface temperature AD (high 8 bits)  | NTC, see §7 |
| 9   | AN3: operating mode                           | 0=continuous / 1=chopping / 2=wave-drop |
| 10  | Reverse-pressure count                        |  |
| 11  | **Actual power AD high 8 bits**               | actual W = byte * 25 |
| 12  | Target power AD low 8 bits                    |  |
| 13  | Target power AD high 8 bits                   |  |
| 14  | PPG signal low 8 bits                         |  |
| 15  | PPG signal high 8 bits                        |  |
| 16  | **PowerStatus** (see §9)                      | flags |
| 17  | Load-detect pulses (low 4) + pan-detect (high 4) |  |
| 18  | Version (high 4 = protocol, low 4 = mainboard) |  |
| 19  | Oscillation cycle count (low 8)               |  |
| 20  | Oscillation cycle count (high 8)              |  |
| 21  | **ADDxL** (see §10)                           | low nibbles for IGBT and Top |
| 22  | IO_status                                     | AN3 low 4 + IO levels |
| 23  | CurPpgNum                                     |  |
| 24  | Checksum                                      |  |
| 25  | `0xFF`                                        | EOF |

---

## 6. IHStatus byte (status idx 4)

| Bits | Name                  | Values                                |
|------|-----------------------|---------------------------------------|
| 7    | IH init flag          | 0 = uninit, 1 = init                  |
| 6    | Power stability flag  | 0 = unstable, 1 = stable              |
| 5    | (Reserved per doc)    | observed always 1 in normal operation |
| 4    | Pot presence flag     | **0 = pot present, 1 = pot absent**   |
| 3:0  | Exception code (see §6.1) |                                   |

> **Observation:** bit 5 is documented as reserved/unused but is set in
> every status frame we have ever captured. It is harmless to ignore.

### 6.1 Exception code table

The supplier doc only defines codes `0x0` through `0x7`. We have observed
the board emit several "Reserved" codes during real cooking, especially at
high glass temperatures. Both sets are listed below with provenance.

| Code | Supplier definition                     | Observed in firmware testing | Mapped in `DetectError` to |
|------|-----------------------------------------|-------------------------------|-----------------------------|
| `0x0` | Working correctly                      |                               | `COOKER_ERROR_NO_ERROR` |
| `0x1` | Reserved                               | seen at both low and high glass temps; semantics unclear | not mapped |
| `0x2` | Circuit fault (no sync, load pulses 0) | seen during PAUSE windows at high glass temp | `COOKER_ERROR_WIRING_ERROR` |
| `0x3` | Reserved                               | observed in IGBT-NTC-open lab tests; **treated as IGBT NTC open-circuit** | `COOKER_ERROR_IGBT_SENSOR_OPEN_CIRCUIT` |
| `0x4` | IGBT sensor short-circuit              | also seen as transient single-frame blips at high glass temp | `COOKER_ERROR_IGBT_SENSOR_SHORT_CIRCUIT` |
| `0x5` | Reserved                               | observed in surface-NTC-open lab tests; **treated as surface NTC open** | `COOKER_ERROR_SURFACE_TEMP_SENSOR_OPEN_CIRCUIT` |
| `0x6` | Induction coil sensor short-circuit    |                               | `COOKER_ERROR_SURFACE_TEMP_SENSOR_SHORT_CIRCUIT` |
| `0x7` | Induction coil sensor failure          |                               | `COOKER_ERROR_SURFACE_TEMP_SENSOR_FAILURE` |
| `0x8` | (not defined)                          | observed exclusively at glass temp ≥ ~150 °C; strong candidate for an **undocumented "glass over-temperature" indicator** | not yet mapped — see §13.1 |

> **Glass over-temperature is not advertised by the board** in the supplier
> doc. The closest documented code is `0x7` (coil sensor failure), but it is
> labelled as a *failure*, not over-temperature. The firmware therefore
> **synthesises** an over-temp error from the glass NTC °C reading (see §11
> and §13.2).

---

## 7. NTC AD value → Celsius (status idx 7 = IGBT, idx 8 = Surface)

Both NTCs use a **5.1 kΩ** pull-up divider per supplier §4.2.

```
y = 5.1 · 255 / (5.1 + x)
```

- `y` = AD value the board reports (8-bit, 0–255).
- `x` = NTC resistance in kΩ.
- HOT NTC ⇒ low resistance ⇒ HIGH AD value.
- COLD NTC ⇒ high resistance ⇒ LOW AD value.

### 7.1 Reference points (from supplier doc)

| AD value | Celsius |
|---------:|--------:|
| `0xC0` (192) | 150 °C |
| `0xDA` (218) | 180 °C |
| `0xE6` (230) | 200 °C |

### 7.2 Lookup table

The firmware uses a **256-entry ADC-indexed table** (`CHKTempLookUp.h`).
Origin: ported from the G4 `Temp-Control` branch, validated against the
supplier reference points (matches exactly at 0xC0 and 0xE6, ±1 °C at
0xDA) and against a bench-recorded Highway 2000 W run (matches reported
glass temp within ±1 °C up to AD ≈ 180, then diverges slightly at the very
hot end where the table saturates).

There is also an older resistance-indexed Highway table
(`HighwayTempLookUp.h`, 326 entries indexed by integer kΩ) that we do
**not** use. The integer-kΩ truncation produces a 25 °C plateau between
AD 184 and 213 — fine for Highway's looser regulation but unsuitable for
CHK where the cut threshold sits in that exact range.

### 7.3 IGBT vs surface divider

Supplier doc §4.2 says **both** sensors use a 5.1 kΩ pull-up. Highway uses
10 kΩ on the IGBT side and 5.1 kΩ on the pot side, but Highway is a
different board. CHK firmware uses the same 5.1 kΩ table for both NTCs.
If a future bench test shows the CHK IGBT ADC behaves like a 10 kΩ
divider, we'd split into per-sensor tables; until then, one table is
correct per the doc.

---

## 8. Voltage / current / actual power conversions

| Quantity     | Source byte | Formula                        | Notes |
|--------------|-------------|--------------------------------|-------|
| Voltage      | idx 5       | `V = 127·byte/128 + 3`         | supplier §4.4 |
| Current      | idx 6       | `I = ActualPower / Voltage`    | supplier explicitly says **don't use the raw byte** — there's an offset that makes it inaccurate |
| Actual power | idx 11      | `P_W = byte * 25`              | supplier §4.8 |

---

## 9. PowerStatus byte (status idx 16)

| Bit | Meaning                                                       |
|-----|---------------------------------------------------------------|
| 0   | Low-voltage current limiting flag                             |
| 1   | Maximum PPG limiting flag                                     |
| 2   | Reverse-voltage power limiting flag                           |
| 3   | Power stability flag                                          |
| 4   | Voltage surge flag (cleared when pan-detect timer reaches `F`) |
| 5   | Current surge flag (cleared when pan-detect timer reaches `F`) |
| 6   | Reverse-voltage early PPG reduction                           |
| 7   | Not used                                                      |

> No bit in this byte signals over-temperature. Surge flags (bits 4, 5) are
> the closest fault-style flags but they relate to mains transients, not
> sensor temperatures.

We do not currently parse this byte in firmware. There is a deferred work
item to expose it as a richer fault decoder.

---

## 10. ADDxL byte (status idx 21)

Carries the low 4 bits of the IGBT and surface NTC ADC readings, so the
combined values are 12-bit:

| Bits 7:4         | Bits 3:0                      |
|------------------|-------------------------------|
| Low 4 of IGBT AD | Low 4 of cooktop surface AD   |

Reconstruction in firmware:

```c
IgbtAdc12    = ((uint16_t)Data[3] << 4) | (ADDxL >> 4);
SurfaceAdc12 = ((uint16_t)Data[4] << 4) | (ADDxL & 0x0F);
```

In practice the firmware then immediately drops the low 4 bits again
(`>> 4`) before indexing the lookup table, because the supplier's
reference formula and table are 8-bit. Keeping the 12-bit reconstruction
in case we ever do interpolation.

---

## 11. Checksum

```
Checksum = (sum of all bytes from 0xAA through the last data byte) mod 256
```

Byte coverage: SOF1 + SOF2 + LEN + DataType + every data byte. Does **not**
include the EOF (`0xFF`).

`IsDataValid` in the firmware passes `DataIndex - 2` to `CalculateChecksum`,
which makes the function sum exactly the bytes that the protocol demands
and compares against the byte at `Data[Size]` (= the checksum slot, two
positions before the end of the buffer that includes the EOF).

---

## 12. Direction-of-data summary

```
                          ┌─────────────┐
       init frame (1×)    │             │
   ─────────────────────► │   CHK 2 kW  │
       control frame      │  power      │
   ─────────────────────► │  board      │
   ◄───────────────────── │             │
       status frame       └─────────────┘
```

- The MCU initiates every exchange. The board only replies with status
  frames in response to control / init frames.
- Cadence is set by the MCU's TIM7 ISR at **110 ms**. The supplier doc
  does not name a cadence; this value comes from packet capture of the
  supplier's PC tool.

---

## 13. Observed deviations & quirks (vs supplier doc)

### 13.1 The board emits "Reserved" exception codes during real cooking

| Code | When | Tentative meaning |
|------|------|-------------------|
| `0x1` | both at low and high temps | unclear |
| `0x3` | when IGBT NTC is unplugged in lab tests | **IGBT NTC open-circuit** |
| `0x5` | when surface NTC is unplugged in lab tests | **surface NTC open-circuit** |
| `0x8` | exclusively at glass °C ≥ ~150 °C | **strong candidate for "glass over-temperature"** — needs more bench data to confirm |

`0x3` and `0x5` are wired into `DetectError` because the lab-test
correlation is solid. `0x8` is **not yet wired in** — the data so far is
consistent (4 events in one run, all at high glass temp) but we want
more confirmation before relying on it.

### 13.2 Glass over-temperature must be detected by the firmware

There is **no documented and no consistently-observable wire signal** for
glass over-temperature. The firmware therefore enforces it itself:

- Read glass NTC AD → convert to °C via `CHKNTCTempFromAdc[adc]`.
- Compare to `GLASS_OVERTEMP_OFF_C = 155` (cut) and `GLASS_OVERTEMP_ON_C =
  150` (resume) with bang-bang hysteresis.
- When latched, the next control frame transmits `switch=PAUSE` (0x01)
  and `PowerSetM=0`. Fan stays running through the peripheral byte so
  the board can shed heat.
- An IGBT side mirrors the same logic with thresholds 80/65 °C.
- Sensor-fault codes (open/short/failure) force the cut latch ON
  unconditionally — fail-safe.

Result: the firmware effectively manufactures `E3` from temperature data
that is reliable, instead of relying on a board-side flag that does not
exist.

### 13.3 `RETURNED_DATA_LENGTH 0x14` instead of `0x20`

The supplier doc shows `0x20` (= 32 data bytes) in the example init and
control frames at byte index 5. With that value the board's reply would
be 36 bytes total — but in our captures, the board only ever sends 26-byte
status frames regardless of what we ask for. With `0x14` (= 20 data
bytes) the resulting frame size matches what the board actually emits
(LEN `0x16`, total 26 bytes including header / checksum / EOF) and the
replies parse cleanly.

The supplier's PC tool also uses `0x14`, so we believe this is a
documentation oversight rather than a firmware bug on the board.

### 13.4 Buzzer enum labels are inverted vs wire behaviour

`BUZZER_100MS = 0x01` actually produces the **longer** beep on the wire,
and `BUZZER_800MS = 0x03` produces the **shorter** click. The firmware
keeps the inversion deliberately and uses `BUZZER_800MS` for the keypad
click. Documented inline in `CHKDriver_SingleBeep`.

### 13.5 ADC ⇒ raw counts are not °C

Earlier firmware treated the raw NTC ADC byte (0–255) as if it were
already °C. With the lookup table now in place this is no longer true,
but it's worth noting because:

- Highway's PID controller compares against °C values converted via its
  own lookup. Anyone porting heating logic between products needs to know
  whether each side is in raw AD or in °C.
- Older logs predating the lookup-table port show the surface "temp"
  hovering at 12–155 (raw ADC) — those numbers are not °C.

---

## 14. Firmware policy (informative — not protocol)

These are choices the firmware makes that aren't dictated by the supplier
doc; they're recorded here so changes are deliberate.

| Policy                         | Value                          | Rationale |
|--------------------------------|--------------------------------|-----------|
| Init settle time               | 100 ms                         | covers init-frame TX (~38 ms at 4800 baud) plus board processing |
| Control TX cadence             | 110 ms (TIM7)                  | matches the supplier PC tool's cadence; faster cadences observed to cause silent board state |
| Glass thermal-cut OFF          | 155 °C                         | derived from Highway lab data |
| Glass thermal-cut ON           | 150 °C                         | 5 °C hysteresis, taken from Highway logs |
| IGBT thermal-cut OFF           | 80 °C                          | conservative; user-tuned down from 100 °C |
| IGBT thermal-cut ON            | 65 °C                          | 15 °C hysteresis, taken from Highway logs |
| Sensor fault → force cut       | open / short / failure all latch the corresponding cut, regardless of temperature | fail-safe — a faulted NTC may read normal-looking values |
| TIM7 is the only writer to UART TX | enforced | mid-frame interleaving caused buffer overflow + silent state in earlier builds |

---

## 15. Maintenance

When this file changes, update **both** copies (G3 and G4 memory
directories) so future Claude sessions on either project see the same
content. The `~/.claude/CLAUDE.md` instructions remind both projects to
read this file before any CHK protocol or driver work.

Last reviewed: 2026-05-07.
