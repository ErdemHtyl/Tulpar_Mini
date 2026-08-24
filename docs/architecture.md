# MiniTULPAR — System Architecture and Design Rationale

**Board:** MiniTULPAR FCB · **Schematic revision:** Rev A (2026-08-10) + updated CubeMX pin configuration
**MCU:** STM32F405RGT6 (LQFP64, 168 MHz, 1 MB Flash, 192 KB RAM)
**Document status:** Derived from the Rev A schematic and the current CubeMX pinout; firmware decisions are proposals.
**Location:** `docs/architecture.md`

> This document does not record *what* was built — it records **why it was built
> this way**. Every decision lists the alternative that was rejected and the
> reason it was rejected. When the board is revised, this document gets revised
> too. If the schematic and this document ever disagree, **the schematic is
> right and this document is wrong** — fix the document.

---

## 0. Executive summary — the architecture in three sentences

A single ICM-42670-P IMU sits alone on **SPI1** and is sampled on a data-ready
interrupt; this is the only high-rate path on the board and it gets a bus to
itself. Slow sensors — barometer and magnetometer — share **I2C2**, because
their sampling rates cannot stress the bus and sharing buys back pins. All four
motors are driven from **TIM8** over DShot through a single DMA stream (TIM8_UP
with DMAR burst), and everything else flows through UART/DMA pairs that never
touch the CPU.

---

## 1. Block diagram

```mermaid
flowchart TB
    subgraph PWR["Power Tree"]
        XT[["Rail / XT60<br/>VCC"]]
        USB5[["USB-C 5 V"]]
        OR{{"D2 / D3 Schottky OR<br/>+ JP1 jumper"}}
        LDO["U1 LD39200PU33R<br/>3.3 V / 2 A LDO"]
        XT --> OR
        USB5 --> OR
        OR --> LDO
        LDO --> V33[("+3.3 V")]
        LDO -. "FB1 600Ω@100MHz" .-> VDDA[("VDDA")]
    end

    subgraph MCU["STM32F405RGT6 @ 168 MHz — HSE 8 MHz (X1)"]
        CORE["Cortex-M4F<br/>FPU + DSP"]
    end

    subgraph FAST["High rate — SPI1 @ 21 MHz"]
        IMU["U2 ICM-42670-P<br/>Accel + Gyro<br/>CS = PA4<br/>INT1 → PC4"]
    end

    subgraph SLOW["Slow sensor bus — I2C2 @ 400 kHz"]
        BARO["U4 BMP581<br/>0x46 · INT → PB1"]
        MAG["U6 IIS2MDC<br/>0x1E · DRDY → PB2"]
    end

    subgraph LOG["Logging — SPI2 @ 21 MHz"]
        FLASH["U7 W25Q128JVS<br/>16 MB · CS = PB12"]
    end

    subgraph ACT["Actuators"]
        ESC["4× ESC · DShot600<br/>TIM8_CH1..CH4<br/>PC6 PC7 PC8 PC9"]
        SERVO["Aux PWM<br/>TIM1_CH1/CH3 · PA8 PA10<br/>Gimbal · PB4 PB5 (TIM3)"]
    end

    subgraph COMM["Serial links"]
        RC["J10 CRSF/RC<br/>UART4 · PA0/PA1"]
        GPS["J12 GPS<br/>USART2 · PA2/PA3"]
        TLM["J11 Telemetry<br/>UART5 · PC12/PD2"]
        AUX["J5 Spare UART<br/>USART3 · PC10/PC11"]
        EXTI2C["J6 External I2C1<br/>PB8/PB9"]
    end

    subgraph MISC["Other"]
        ADCIN["J7 ADC → PC2<br/>J8 Battery → PC3"]
        SWD["J13 SWD<br/>PA13/PA14"]
        USBP["USB-C · PA11/PA12<br/>USBLC6-2SC6 ESD"]
        LEDS["LD1..LD3<br/>PC13/PC14/PC15"]
    end

    V33 --> MCU
    IMU <-->|SPI1| CORE
    BARO <-->|I2C2| CORE
    MAG <-->|I2C2| CORE
    FLASH <-->|SPI2| CORE
    CORE --> ESC
    CORE --> SERVO
    RC <--> CORE
    GPS <--> CORE
    TLM <--> CORE
    AUX <--> CORE
    EXTI2C <--> CORE
    ADCIN --> CORE
    SWD <--> CORE
    USBP <--> CORE
    CORE --> LEDS
```

**How data actually flows:** the IMU's INT1 line raises an EXTI interrupt → a
SPI1 DMA burst read starts → the DMA completion interrupt runs the filter, PID
and mixer chain → the result is encoded into a DShot frame and handed to the
TIM8 DMA. The whole chain completes **inside one interrupt context**, never
waiting on any other peripheral. Everything else — baro, mag, GPS, telemetry,
logging — runs outside that chain at lower priority.

### 1.1 Complete pin assignment

STM32F405RGT6, LQFP64. This table is the intended single source of truth — the
firmware's `board.h` should be derived from it, not maintained in parallel.

| Pin | Port | Signal | Peripheral / alternate function |
|---|---|---|---|
| 20 | PA4 | SPI1_CS1 | GPIO output — IMU chip select |
| 21 | PA5 | SPI1_SCK | SPI1 |
| 22 | PA6 | SPI1_MISO | SPI1 |
| 23 | PA7 | SPI1_MOSI | SPI1 |
| 24 | PC4 | INT_IMU1 | EXTI4 — IMU data ready |
| 33 | PB12 | SPI2_CS | GPIO output — flash chip select |
| 34 | PB13 | SPI2_SCK | SPI2 |
| 35 | PB14 | SPI2_MISO | SPI2 |
| 36 | PB15 | SPI2_MOSI | SPI2 |
| 29 | PB10 | I2C2_SCL | I2C2 — internal sensor bus |
| 30 | PB11 | I2C2_SDA | I2C2 |
| 27 | PB1 | INT_BAR | EXTI1 — BMP581 |
| 28 | PB2 | INT_MAG | EXTI2 — IIS2MDC DRDY |
| 61 | PB8 | I2C1_SCL | I2C1 — external connector |
| 62 | PB9 | I2C1_SDA | I2C1 |
| 37 | PC6 | DSHOT_1 | **TIM8_CH1** |
| 38 | PC7 | DSHOT_2 | **TIM8_CH2** |
| 39 | PC8 | DSHOT_3 | **TIM8_CH3** |
| 40 | PC9 | DSHOT_4 | **TIM8_CH4** |
| 41 | PA8 | Aux PWM 1 | TIM1_CH1 |
| 43 | PA10 | Aux PWM 2 | TIM1_CH3 |
| 57 | PB5 | CAM_GIMBAL1 | TIM3_CH2 |
| 56 | PB4 | CAM_GIMBAL2 | TIM3_CH1 |
| 14 | PA0 | UART4_TX | UART4 — CRSF / RC |
| 15 | PA1 | UART4_RX | UART4 |
| 16 | PA2 | USART2_TX | USART2 — GPS |
| 17 | PA3 | USART2_RX | USART2 |
| 51 | PC10 | USART3_TX | USART3 — spare |
| 52 | PC11 | USART3_RX | USART3 |
| 53 | PC12 | UART5_TX | UART5 — telemetry |
| 54 | PD2 | UART5_RX | UART5 |
| 10 | PC2 | ADC1 | ADC1_IN12 — external sense |
| 11 | PC3 | Battery_Voltage | ADC1_IN13 |
| 44 | PA11 | USB_OTG_FS_DM | USB |
| 45 | PA12 | USB_OTG_FS_DP | USB |
| 42 | PA9 | USB_OTG_FS_VBUS | USB VBUS detect |
| 46 | PA13 | SWDIO | SYS_JTMS-SWDIO |
| 49 | PA14 | SWCLK | SYS_JTCK-SWCLK |
| 2 | PC13 | LD1 | GPIO — see §6.7 ⚠ |
| 3 | PC14 | LD2 | GPIO — see §6.7 ⚠ |
| 4 | PC15 | LD3 | GPIO — see §6.7 ⚠ |
| 5 | PH0 | RCC_OSC_IN | HSE 8 MHz (X1) |
| 6 | PH1 | RCC_OSC_OUT | HSE |
| 7 | NRST | Reset | S1 button + 100 nF |
| 60 | BOOT0 | Boot select | S2 SPDT switch |
| 1 | VBAT | Backup supply | |
| 13 / 12 | VDDA / VSSA | Analog supply | FB1 + 0.1 µF + 1 µF |
| 31 / 47 | VCAP1 / VCAP2 | Core regulator | 2.2 µF each |
| 19, 32, 48, 64 | VDD | Digital supply | 100 nF per pin |
| 18, 63 | VSS | Ground | |

**Unused and available:** PA15 (50), PB0 (26), PB3 (55), PB6 (58), PB7 (59),
PC0 (8), PC1 (9), PC5 (25). Notable options among these: PB6/PB7 are USART1,
PB0/PB1 are TIM3_CH3/CH4, and PC0/PC1 are unrestricted GPIOs — see §6.7 for why
that last pair matters. Leave test points on all of them in the layout; it is
cheap insurance.

---

## 2. Bus allocation and rationale

### 2.1 Summary

| Sensor / device | Bus | Clock | Sample rate | Why this bus |
|---|---|---|---|---|
| ICM-42670-P (accel/gyro) | **SPI1**, CS = PA4 | 21 MHz | up to 1.6 kHz | The only high-rate path; I²C bandwidth does not fit (§2.2) |
| BMP581 (baro) | **I2C2** | 400 kHz | ~50–100 Hz | Slow; saves pins; not worth occupying an SPI |
| IIS2MDC (mag) | **I2C2** | 400 kHz | 100 Hz | Same; I²C is also the part's native interface |
| W25Q128JVS (log) | **SPI2** | 21 MHz | continuous block writes | Must not share a bus with the IMU (§2.4) |
| CRSF / RC | UART4 | 420 kbaud | 250–500 Hz packets | Framed protocol; DMA + IDLE line costs zero CPU |
| GPS | USART2 | 115200 | 5–10 Hz | Same |
| Telemetry | UART5 | 57600–115200 | ~10 Hz | Same |
| Spare | USART3 | — | — | ESC telemetry / VTX / future use |
| External I²C | I2C1 | 400 kHz | ≤100 Hz | Leaves the board on a cable — kept **separate** from the internal bus (§2.5) |
| Camera gimbal | TIM3_CH1/CH2 (PB4/PB5) | 50–333 Hz | — | Servo-rate PWM; deliberately *not* on TIM8 (§3.1) |

### 2.2 DECISION: the IMU goes on **SPI1**, not on I²C

**Decision:** the ICM-42670-P sits alone on SPI1 with a dedicated chip select
(PA4).

**Rationale — the numbers.** One sample read is 1 register address byte + 14
data bytes (TEMP 2 + GYRO 6 + ACCEL 6) = 15 bytes.

*On SPI1 (APB2 = 84 MHz, prescaler /4 → 21 MHz, below the ICM-42670-P's 24 MHz
limit):*

```
15 bytes × 8 bits = 120 bits
120 bits ÷ 21 MHz  ≈  5.7 µs
```

*The same read over I²C (Fast-mode, 400 kHz):*

```
START + ADDR_W(9) + REG(9) + REPEATED START + ADDR_R(9) + 14×9 bits + STOP
= 9 + 9 + 9 + 126  ≈  155 bit-times  (plus start/stop overhead)
155 ÷ 400 kHz  ≈  388 µs
```

**The same work takes roughly 68× longer on I²C.** What that costs:

| Loop rate | Period | SPI load | I²C load |
|---|---|---|---|
| 1 kHz | 1000 µs | 0.6 % | **39 %** |
| 1.6 kHz (the part's ceiling) | 625 µs | 0.9 % | **62 %** |
| 4 kHz | 250 µs | 2.3 % | **impossible** |
| 8 kHz | 125 µs | 4.6 % | **impossible** |

Because 388 µs > 125 µs, 8 kHz sampling physically does not fit on I²C — a
single read consumes three times the whole period. At 1.6 kHz it technically
"fits", but handing 62 % of a bus to one sensor means nothing else can ever
share that bus.

And there is no escape upward. The ICM-42670-P's own I²C interface is rated to
1 MHz, but **the STM32F405's I²C peripheral cannot exceed 400 kHz** — there is no
Fast-mode Plus on this MCU, so the sensor's headroom is unreachable. Even if it
were reachable, 155 bit-times at 1 MHz is 155 µs, still longer than a 125 µs
period: 8 kHz would not fit on I²C at *any* clock this pairing can produce.

(The SPI side has margin in hand by comparison: the part is rated to 24 MHz and
the chosen 21 MHz sits just under it, arrived at by the APB2 ÷ 4 prescaler
rather than by pushing the sensor.)

**The reasons that have nothing to do with bandwidth — these matter just as
much:**

1. **Determinism.** An SPI transfer takes a fixed, predictable time. On I²C a
   slave may apply **clock stretching**, so the transfer takes as long as the
   slave feels like. Jitter in the control loop period shows up directly as
   noise on the derivative term.
2. **Bus lockup.** I²C is open-drain. If a slave resets at the wrong moment it
   can hold SDA low and **wedge the entire bus**. Recovery means manually
   clocking out nine SCL pulses — not something to be doing mid-flight. SPI has
   no equivalent failure mode; deasserting CS resynchronises the device.
3. **Noise immunity.** The ESCs switch tens of amps at 20–40 kHz. I²C's slow,
   pull-up-driven rising edges are vulnerable in that environment; SPI's
   push-pull drivers are far more robust.
4. **STM32F4 I²C errata.** The F4 family's I²C peripheral has documented errata,
   including a BUSY flag that can latch under specific conditions. The
   flight-critical path should not be built on that peripheral.
5. **DMA friendliness.** SPI + DMA moves 15 bytes at **zero** CPU cost. I²C +
   DMA works on the F4, but the START/ADDR/RESTART phases still need interrupt
   servicing — it is not free.

**Rejected alternatives:**

- *IMU on I²C* — see the table above. This would be reasonable for a 100 Hz tilt
  sensor. It is not reasonable for a flight controller.
- *Sharing one SPI between the IMU and the flash* — see §2.4.
- *Polling instead of using the INT line* — the delay between the sampling
  instant and the read instant becomes variable, which is a timestamp error in
  the fusion filter. This is exactly why INT1 is routed to PC4, and it
  **must be used**.

### 2.3 DECISION: barometer and magnetometer share **I2C2**

**Decision:** BMP581 (0x46) and IIS2MDC (0x1E) share I2C2 on PB10/PB11.

**Rationale.** The physics of these sensors is already slow.

- Barometer: altitude estimation gains nothing above 50–100 Hz — measurement
  noise and the sensor's own conversion time dominate.
- Magnetometer: the IIS2MDC tops out at 100 Hz ODR, so reading faster buys
  nothing.

Load calculation, with both sensors reading 6 bytes at 100 Hz:

```
6-byte read ≈ 9 + 9 + 9 + 54 = 81 bit-times ≈ 203 µs @ 400 kHz
2 sensors × 100 Hz × 203 µs = 40.6 ms/s  →  bus load ≈ 4 %
```

Four percent load, ninety-six percent idle. Moving these to SPI would spend two
more chip-select pins and gain nothing. **I²C's weaknesses — indeterminate
timing, lockup risk — are acceptable here**, because neither sensor is on the
critical path of the stabilisation loop. If the baro or mag stalls for a few
periods, or dies outright, the quad keeps flying with degraded altitude and
heading. If the gyro stalls, it falls. That asymmetry is the whole basis of the
split:

> **The critical path lives on SPI. Everything non-critical lives on I²C.**

**The obligation this puts on firmware:** the I2C2 driver must be written as a
**non-blocking state machine**, every transfer must have a timeout, and a
timeout must trigger the bus recovery routine (nine SCL pulses followed by a
STOP). A failure of these two sensors must never stall the main loop.

### 2.4 DECISION: flash on its own bus (SPI2), not shared with the IMU

**Decision:** the W25Q128JVS gets SPI2 (PB12–PB15) to itself.

**Rationale.** Blackbox logging writes **long and unpredictable** blocks by
nature:

| Operation | Duration (W25Q128JV typ / max) |
|---|---|
| 256-byte page program | 0.7 ms / 3 ms |
| 4 KB sector erase | 45 ms / 400 ms |

A sector erase can take **up to 400 ms**. If the flash shared a bus with the
IMU, a gyro read would sit behind that queue — or the firmware would need
elaborate transfer-splitting and priority logic. Two separate SPIs make the
problem disappear at design time: **no shared resource, no priority problem.**

Bandwidth check: at 1 kHz with ~60 bytes per record, logging produces 60 kB/s.
The flash sustains 256 B / 0.7 ms ≈ 365 kB/s — comfortable margin. The real
constraint is erase time:

```
Worst case: 400 ms erase × 60 kB/s = 24 KB of RAM buffering required
```

The F405's 192 KB (128 KB SRAM + 64 KB CCM) covers that easily, but **the buffer
size should not be left to chance**. The better approach is to erase ahead
during idle time — erase the next blocks before arming — so that only page
programs happen in flight and the required buffer drops to 3 ms × 60 kB/s ≈
200 bytes.

### 2.5 DECISION: two separate I²C buses — internal (I2C2) and external (I2C1)

**Decision:** on-board sensors on I2C2; the off-board J6 connector on its own
bus, I2C1 (PB8/PB9).

**Rationale.** The external bus leaves the board on a cable, which brings three
distinct risks:

1. **ESD and shorts.** A reversed connector or a chafed cable takes down
   everything on that bus. Isolating the internal baro and mag means one cable
   fault cannot kill altitude estimation.
2. **Lockup isolation.** If an external device — a GPS module's compass, say —
   holds SDA low and wedges its bus, the internal sensors are untouched.
3. **Capacitance and edge rates.** A cable adds roughly 50–100 pF per metre. On
   a shared bus that would degrade the internal sensors' edges too.

**A concrete finding that falls out of this:** the schematic pulls both buses up
with 4.7 kΩ (I2C1: R5/R6, I2C2: R3/R4). Fast-mode allows a maximum rise time of
300 ns:

```
t_r ≈ 0.8473 × R × C   →   C_max = 300 ns / (0.8473 × 4700 Ω) ≈ 75 pF
```

The internal bus — two devices, short traces — lands around 20–40 pF, so
**4.7 kΩ is fine there**. On the external bus, even 30 cm of cable adds 30–50 pF,
and together with the remote module the 75 pF budget is **exceeded**.
Recommendation: **drop the I2C1 pull-ups to 2.2 kΩ** (R5/R6). That allows about
160 pF instead of 75 pF, and the sink current stays at 3.3 V / 2.2 kΩ = 1.5 mA,
well under the 3 mA I²C limit.

---

## 3. Timer and DMA allocation

### 3.1 The TIM3 constraint — already resolved, do not undo it

The pins the board uses can each be reached by more than one timer, and two of
those options overlap:

| Net | Pin | Timer channels that can reach this pin |
|---|---|---|
| DSHOT_1 | PC6 | TIM3_CH1 · **TIM8_CH1** ← chosen |
| DSHOT_2 | PC7 | TIM3_CH2 · **TIM8_CH2** ← chosen |
| DSHOT_3 | PC8 | TIM3_CH3 · **TIM8_CH3** ← chosen |
| DSHOT_4 | PC9 | TIM3_CH4 · **TIM8_CH4** ← chosen |
| CAM_GIMBAL2 | PB4 | **TIM3_CH1** (only option) |
| CAM_GIMBAL1 | PB5 | **TIM3_CH2** (only option) |
| Aux PWM 1 | PA8 | **TIM1_CH1** (only option) |
| Aux PWM 2 | PA10 | **TIM1_CH3** (only option) |

**The latent conflict:** PB4 and PC6 are two alternate pins for the *same*
channel, TIM3_CH1 — and a timer channel can only be routed to **one pin at a
time**. Route DShot through TIM3 and the gimbal outputs go dead; route the
gimbal through TIM3 and DSHOT_1/DSHOT_2 go dead.

**The current CubeMX configuration assigns DShot to TIM8, which avoids this
entirely.** The gimbal keeps TIM3_CH1/CH2 and all four motor outputs work. This
section exists so that nobody "simplifies" the design later by moving DShot to
TIM3 — PC6–PC9 map to TIM3 just as naturally, the change looks harmless, and it
silently kills the gimbal outputs.

**The assignment to preserve:**

| Function | Timer | Channel | Pin | Timer clock |
|---|---|---|---|---|
| Motor 1 | **TIM8** | CH1 | PC6 | 168 MHz (APB2) |
| Motor 2 | **TIM8** | CH2 | PC7 | 168 MHz |
| Motor 3 | **TIM8** | CH3 | PC8 | 168 MHz |
| Motor 4 | **TIM8** | CH4 | PC9 | 168 MHz |
| Aux PWM 1 | TIM1 | CH1 | PA8 | 168 MHz (APB2) |
| Aux PWM 2 | TIM1 | CH3 | PA10 | 168 MHz |
| Camera gimbal 2 | TIM3 | CH1 | PB4 | 84 MHz (APB1) |
| Camera gimbal 1 | TIM3 | CH2 | PB5 | 84 MHz |

**Why TIM8 rather than TIM3:**

1. It resolves the conflict — all four aux PWM channels stay usable.
2. TIM8 lives on APB2, so its timer clock is **168 MHz** (TIM3 is on APB1 at
   84 MHz). That is twice the resolution for DShot bit timing (§3.2).
3. TIM8 is an advanced-control timer: it has dead-time generation, a break input
   and a repetition counter. The break input is a path to hardware motor cutoff
   for failsafe later; TIM3 has none of this.
4. All four motors on **one timer** means they are bit-synchronous, and one DMA
   stream (TIM8_UP + DMAR burst) drives all of them.

**Note:** the four servo-rate outputs are split across TIM1 (PA8, PA10) and TIM3
(PB4, PB5), so driving them at a common rate means programming both timers to
the same period (e.g. 50 Hz / 20 ms, or 333 Hz) separately — and the prescaler
values will differ, because TIM1 runs at 168 MHz and TIM3 at 84 MHz. For a
camera gimbal this is harmless; the two axes only need to agree with themselves.

### 3.2 DShot timing numbers (TIM8 @ 168 MHz)

DShot encodes a bit by pulse width: high for **37.5 %** of the bit period is a
"0", high for **75 %** is a "1". In PWM mode the CCR value changes per bit.

| Protocol | Bit rate | ARR | CCR for "0" | CCR for "1" | 16-bit frame |
|---|---|---|---|---|---|
| DShot150 | 150 kbit/s | 1119 | 420 | 840 | 107 µs |
| DShot300 | 300 kbit/s | 559 | 210 | 420 | 53.3 µs |
| **DShot600** | 600 kbit/s | **279** | **105** | **210** | **26.7 µs** |
| DShot1200 | 1200 kbit/s | 139 | 52 | 105 | 13.3 µs |

```
ARR = (168 MHz / bit_rate) − 1        →  168e6 / 600e3 − 1 = 279
CCR_0 = 0.375 × 280 = 105
CCR_1 = 0.750 × 280 = 210
```

**Recommendation: DShot600.** A 26.7 µs frame is 2.7 % of a 1 kHz period, 4.3 %
at 1.6 kHz. DShot1200 is fussy about cable length and ESC compatibility and buys
nothing here. DShot300 is a more forgiving first target; move to 600 once the
board is proven.

**Bidirectional DShot (RPM telemetry)**, if added later: the same pin is turned
into an input roughly 30 µs after the frame is sent, and the ESC's 21-bit GCR
response is read by input capture. That needs a separate DMA stream per channel
and changes the budget in §3.4 — handled there.

### 3.3 Why DShot rather than PWM or OneShot

| Protocol | Resolution | Frame | Calibration | CRC |
|---|---|---|---|---|
| PWM (1000–2000 µs) | ~1000 steps | 2 ms @ 500 Hz | Required | None |
| OneShot125 | ~1000 steps | 250 µs | Required | None |
| **DShot600** | **2048 steps** | **26.7 µs** | **Not needed** | **4-bit** |

DShot is digital: it does not depend on the ESC's interpretation of an analogue
pulse width, so **ESC calibration stops being a step that exists**. Every frame
carries a 4-bit CRC, so a command corrupted by noise gets discarded by the ESC
instead of silently applying the wrong throttle. On a noisy power board that
difference is a safety property, not a convenience.

### 3.4 DMA allocation

On the STM32F4, each peripheral request is hard-wired to **specific
stream + channel** combinations (RM0090 Tables 42/43). DMA allocation is
therefore not a free choice — it is a **constraint-satisfaction problem**. The
table below is a conflict-free solution under those constraints.

#### DMA2 (drives SPI1, TIM8, ADC1)

| Stream | Ch | Assignment | Why |
|---|---|---|---|
| **S0** | 3 | **SPI1_RX** — IMU read | The critical path. SPI1_RX exists only on S0C3 or S2C3 |
| **S1** | 7 | **TIM8_UP** — DShot burst output | The only option for TIM8_UP. DMAR writes CCR1–CCR4 in one go |
| **S2** | 7 | TIM8_CH1 (bidirectional DShot capture, M1) | Optional — §3.4.1 |
| **S3** | 7 | TIM8_CH2 (M2) | Optional |
| **S4** | 7 | TIM8_CH3 (M3) | Optional |
| **S5** | 3 | **SPI1_TX** — IMU command | SPI1_TX: S3C3 or S5C3. S3 was given to TIM8_CH2 |
| S6 | — | *free* | Reserve |
| **S7** | 7 | TIM8_CH4 (M4) | Optional |

#### DMA1 (drives SPI2, the UARTs, I²C)

| Stream | Ch | Assignment | Why |
|---|---|---|---|
| **S0** | 4 | **UART5_RX** — telemetry in | |
| **S1** | 4 | **USART3_RX** — spare UART | |
| **S2** | 4 | **UART4_RX** — CRSF / RC | **Critical.** 420 kbaud = 42 kB/s; interrupt-per-byte would mean 42 000 ISRs per second |
| **S3** | 0 | **SPI2_RX** — flash | |
| **S4** | 0 | **SPI2_TX** — flash / blackbox | Keeps long block writes off the CPU |
| **S5** | 4 | **USART2_RX** — GPS | Circular mode + IDLE line interrupt |
| **S6** | 4 | **USART2_TX** — GPS configuration (UBX) | |
| **S7** | 4 | **UART5_TX** — telemetry out | |

**What is left without DMA, and why:**

- **I2C1 and I2C2 → interrupt-driven.** I2C2_RX's options are S2C7 and S3C7, and
  both went to higher-value work (CRSF RX, flash RX). Acceptable: 6-byte
  transfers at 100 Hz produce ~1200 interrupts per second, about a thousandth of
  the CPU.
- **UART4_TX (CRSF telemetry back-channel) → interrupt.** Its S4C4 option
  collides with SPI2_TX. The back-channel is a few hundred bytes per second;
  interrupts are fine.
- **USART3_TX → interrupt.** Same reason (S3C4/S4C7 both collide).
- **ADC1 → interrupt or direct read.** ADC1's DMA options are DMA2 S0C0 and
  S4C0, both taken. Battery voltage and current are read at 100 Hz; a two-channel
  scan conversion serviced in an interrupt is negligible.

#### 3.4.1 The trade-off: bidirectional DShot or ADC DMA?

In the table above, DMA2 streams S2/S3/S4/S7 are reserved for bidirectional
DShot. **If bidirectional DShot is not used**, those four streams free up and
S0C0 or S4C0 opens for ADC1. The decision:

> **No bidirectional DShot on the first flight.** Run the ADC without DMA and
> leave the streams unused. RPM telemetry — for RPM-based notch filtering —
> comes later, in a separate firmware version, once basic flight is stable.

The reason: bidirectional DShot also requires support on the ESC side
(BLHeli_32 / AM32), GCR decoding is unpleasant to debug, and it muddies the
picture during bring-up. Get it flying first, then filter it.

### 3.5 Interrupt priority table (NVIC)

Allocating DMA and timers correctly is not enough; **which interrupt is allowed
to preempt which** is part of the architecture too. On Cortex-M4, lower number =
higher priority.

| Priority | Interrupt | Rationale |
|---|---|---|
| 0 | EXTI4 (PC4 — IMU INT1) | Timestamps the sampling instant. Any delay corrupts the timestamp |
| 1 | DMA2_Stream0 (SPI1_RX complete) | Triggers the control chain |
| 2 | DMA2_Stream1 (TIM8_UP) | Motor frame output |
| 3 | DMA1_Stream2 (UART4_RX — RC) | RC loss means failsafe; low tolerance for delay |
| 4 | DMA1_Stream5 (USART2_RX — GPS) | |
| 5 | I2C2_EV / I2C2_ER | Baro/mag; delay is harmless |
| 6 | SysTick | Timebase |
| 7 | DMA1_S3/S4 (flash), UART5, USB | None of these affect flight |

**The rule:** nothing runs at a higher priority than the interrupt that executes
the PID chain (priority 1). Blackbox writes, USB and telemetry must **never**
preempt the control loop.

---

## 4. Timing budget

The table below is an estimated budget for a 1 kHz control loop (1000 µs
period). These are targets — the real numbers should be measured with the DWT
cycle counter and written back into this table.

| Step | Duration | Occupies the CPU? |
|---|---|---|
| IMU SPI burst read | 5.7 µs | No (DMA) |
| Gyro/accel filtering (3× biquad + PT1) | ~10 µs | Yes |
| Attitude update (Mahony, 1 kHz) | ~20 µs | Yes |
| PID (3 axes, rate + angle) | ~12 µs | Yes |
| Mixer + motor limiting | ~4 µs | Yes |
| DShot frame encoding (4 channels × 16 bits) | ~6 µs | Yes |
| DShot600 frame transmission | 26.7 µs | No (DMA) |
| **Total CPU** | **~52 µs** | **5.2 % load** |

About 5 % core utilisation at 168 MHz leaves the remaining 95 % for the baro/mag
state machine, GPS parsing, telemetry, blackbox — and **margin for error**. That
margin is deliberate: the first firmware that actually works is always heavier
than the estimate.

---

## 5. Sampling architecture and the ODR ceiling

### 5.1 The ICM-42670-P's ODR ceiling

*Verified against the TDK ICM-42670-P datasheet (DS-000451), not from memory.*

**ODR — output data rate** — is the rate at which the sensor writes a *new*
number into its output registers. The IMU runs on its own internal clock: it
samples the MEMS structure, filters, decimates, and updates the registers at
this fixed rate. **The ICM-42670-P's ODR range is 12.5 Hz – 1600 Hz for the
gyroscope and 1.5625 Hz – 1600 Hz for the accelerometer.** 1.6 kHz is the
ceiling. This is the low-power member of the family — it is *not* the
ICM-42688-P, which reaches 32 kHz.

**Why the bus speed cannot rescue this.** At 1.6 kHz ODR the register contents
change once every 625 µs. Reading them faster returns the same value again:

```
t = 0   µs → read → 142.7 °/s   (new)
t = 125 µs → read → 142.7 °/s   ← identical
t = 250 µs → read → 142.7 °/s   ← identical
t = 375 µs → read → 142.7 °/s   ← identical
t = 500 µs → read → 142.7 °/s   ← identical
t = 625 µs → read → 148.1 °/s   (new)
```

An 8 kHz loop would execute 8000 times per second on 1600 distinct pieces of
information. The extra 6400 iterations add nothing — and they actively harm the
derivative term, which now sees a staircase: four periods of zero slope followed
by a step. That is the worst possible input to a D term.

**So the ceiling is set by the sensor, not by the bus.** The SPI decision in
§2.2 remains correct — I²C would consume 62 % of the bus even at 1.6 kHz — but
the second half of the argument ("8 kHz does not fit on I²C") never gets
exercised on this board, because 8 kHz is unreachable anyway.

**What a 1.6 kHz ODR would normally cost you — and why it does not here.** By
Nyquist, a 1.6 kHz sample rate can faithfully represent content only up to
800 Hz. Anything above that would normally fold back and appear at the wrong
frequency. For a 5-inch quad at 25 000 rpm the arithmetic looks alarming:

| Vibration component | True frequency | Would alias to, at 1.6 kHz |
|---|---|---|
| Fundamental (25000/60) | 417 Hz | 417 Hz — in band, correct |
| 2nd harmonic | 833 Hz | 767 Hz — wrong place |
| **3rd harmonic** | **1250 Hz** | **350 Hz** — indistinguishable from real motion |

That third row is the classic failure: a 1250 Hz mechanical vibration would
present itself to the gyro as a genuine 350 Hz rotation, the PID would chase it,
and the motors would heat up while the aircraft fought itself.

**The datasheet resolves this in the part's favour.** The ICM-42670-P's signal
path is: ADC → **anti-alias filter (AAF)** → programmable 1st-order low-pass →
ODR decimation. The AAF has fixed coefficients, is **not user-configurable and
cannot be bypassed**, and it sits ahead of decimation — which is exactly where
it needs to be. The aliasing scenario above is therefore handled in silicon, not
left to you.

**One condition attached, and it is the actionable part:** the datasheet
describes the AAF as active in **Low-Noise Mode**. Low-Power Mode is a different
signal path. So:

> **Run the gyro in Low-Noise Mode.** This is not a performance preference — it
> is what keeps the anti-alias filter in the path. Do not let a power-saving
> change quietly move the part into Low-Power Mode.

**The architectural consequences:**

1. **An 8 kHz control loop is not possible on this board** — sensor-limited, not
   bus-limited. The SPI decision stands regardless; it is just that the
   8 kHz half of the argument never gets exercised here.
2. **Target a 1 kHz PID loop**, gyro sampled at 1.6 kHz ODR. This is not a
   concession: ArduPilot and PX4 have flown 400 Hz–1 kHz loops for years.
   Betaflight's 8 kHz exists for racing feel and RPM-based filtering, neither of
   which is a first-board requirement.
3. **Set `GYRO_UI_FILT_BW` deliberately** — the programmable low-pass after the
   AAF is yours to choose, and its reset default is not a design decision.
4. **Still soft-mount the IMU.** The AAF prevents high-frequency energy from
   *aliasing*, but it does not prevent it from saturating the sensor or from
   consuming dynamic range. Keeping vibration out mechanically remains the
   cheapest filter on the aircraft.
5. **The FIFO is available if wanted:** 2.25 KB total, of which 1 KB is
   allocated to the FIFO by default (the rest is reserved for the APEX motion
   features, and can be reclaimed by disabling them). Not needed for a
   straightforward interrupt-per-sample design, but it is there if batching ever
   becomes useful.

**Recommendation:** target 1 kHz with this board — the architecture is sound and
the learning value is intact. If Rev B wants a higher loop rate, the
**ICM-42688-P** is the upgrade; it is in the same 14-pin LGA 2.5 × 3.0 mm class,
but the **pin map is not identical**, so the footprint and schematic must be
checked against the datasheet.

### 5.2 Interrupt-driven sampling, not polling

INT1 is routed to PC4 — use it:

```
IMU INT1 (PC4) → EXTI4 → start SPI1 DMA burst read
                          ↓
                 DMA2_S0 TC ISR → filter → PID → mixer → DShot DMA
```

This makes the delay between the sampling instant and the use of that data
**constant**. With periodic timer polling, the sensor's internal ODR clock and
the MCU clock drift against each other, so occasionally the same sample gets
read twice or one gets skipped — a silent and very hard-to-diagnose error source
in sensor fusion.

**INT2 (pin 9) is unconnected in the schematic.** It may be wanted later for a
FIFO watermark interrupt; for now one INT line is sufficient.

---

## 6. Findings from the schematic review

These came out of reading Rev A against the current pin configuration. None of
them makes the board non-functional.

### 6.1 Block diagram sheet vs schematic — ✅ resolved

The Rev A block diagram sheet disagreed with the schematic on the barometer part
number, the IMU count and the bus quantities. This has been corrected. Keep the
habit: whenever the schematic changes, the block diagram sheet changes in the
same commit, or the two drift apart again.

### 6.2 TIM3 channel conflict — ✅ avoided by design

The latent conflict between DShot and the gimbal outputs over TIM3_CH1/CH2 is
documented in §3.1. The current configuration puts DShot on **TIM8**, which
avoids it. Record this in `docs/pinout.md` as a constraint, not a preference —
the failure mode if someone moves DShot to TIM3 later is silent.

### 6.3 Magnetometer placement

The IIS2MDC is on-board. The magnetometer is the sensor most affected by the
magnetic field that motor currents produce, and a few millimetres from a 30 A
power trace it is not measuring the Earth's field — it is measuring throttle
position.

**Recommendation:** keep the IIS2MDC as a backup and use the GPS module's
compass over **J6 (external I2C1)** as the primary. In the PCB layout, put the
IIS2MDC as far from the power section as the board allows, ideally in the
opposite corner. Also plan a calibration step that measures compass deviation
against throttle.

### 6.4 3.3 V LDO thermal budget

The LD39200PU33R can supply 2 A, but everything dropped across an LDO becomes
heat:

```
P = (V_in − 3.3 V) × I_load
At 5 V in:  1.7 V × I
```

Load estimate: MCU ~60 mA + sensors ~5 mA + flash ~15 mA (while writing) +
whatever the connectors feed. **The important part:** J5, J6, J7, J10, J11 and
J12 all supply +3.3 V, and a GPS module alone can draw 100–150 mA.

| Total load | Dissipated in the LDO (5 V in) |
|---|---|
| 100 mA | 0.17 W — fine |
| 300 mA | 0.51 W — watch it |
| 500 mA | **0.85 W** — needs thermal vias and copper pour under the DFN |

**To do:** put the 3.3 V load budget into a table (`docs/power-tree.md`) and place
a thermal via array under the LDO. If the number of externally powered devices
is going to grow, consider a switching regulator instead — but then the IMU
needs an LC filter plus a low-noise LDO stage so switching noise stays out of it.

### 6.5 The battery voltage divider is off-board

The `Battery_Voltage` net runs from J8 straight to PC3; the board carries only
C29 (0.1 µF) and no divider resistors. The divider is expected externally.

**Things to watch:**
- The STM32F4 ADC wants a low source impedance for 12-bit accuracy (up to about
  50 kΩ with a long sampling time, but under 10 kΩ is the safe answer).
- PC3 = ADC1_IN13, PC2 = ADC1_IN12 — correctly noted on the schematic.
- **There is no overvoltage protection.** If the divider is miscalculated or the
  external cable fails, something above 3.3 V goes straight into an MCU pin.
  Recommendation for Rev B: a 1 kΩ series resistor into PC3 plus a Schottky
  clamp to 3.3 V.

### 6.6 SBUS is not supported unless you add hardware

The block diagram mentions an "S-BUS Connector", but the **STM32F405's UARTs
have no hardware signal inversion** (the F7/H7 do; the F4 does not), and SBUS is
an inverted signal.

- **Using CRSF / ELRS:** no problem, CRSF is not inverted and J10 is ready.
- **Using SBUS:** you need an external inverter on the board (one NPN and two
  resistors, or a 74LVC1G04). Rev A does not have one.

### 6.7 Smaller notes

- **⚠ LD1/LD2/LD3 on PC13/PC14/PC15 — check the LED polarity.** These three pins
  sit in the backup power domain and are fed through an internal power switch.
  The STM32F405 datasheet is explicit about them: their total current is limited
  to **3 mA for all three combined**, they must not be driven above 2 MHz, and
  **they must not be used as a current source — the datasheet names driving an
  LED as the example.** In the Rev A schematic the LEDs are wired
  pin → LED → 1 kΩ → GND, which is exactly the sourcing configuration the
  datasheet warns against, and at ~1.3 mA each, two lit at once already consumes
  the budget for all three.

  Two fixes, both cheap:
  1. **Flip the LEDs to sink** — anode to +3.3 V, cathode through the resistor to
     the pin, driven low to light. No extra parts, just a schematic change, and
     it turns a forbidden configuration into an allowed one. Firmware logic
     inverts.
  2. **Move LEDs to PC0/PC1** — both are unused normal I/Os in the current
     pinout, with no backup-domain restriction. Two of the three LEDs could
     move there for free.

  Recommendation: do both — move two LEDs to PC0/PC1 and flip whatever stays on
  PC13–PC15 to sink configuration. Also note that using PC14/PC15 means **no LSE
  crystal can be fitted** (the RTC runs from the internal LSI only), which is
  irrelevant for a flight controller.
- **Crystal load capacitors.** C13/C14 = 10 pF, R1 = 0 Ω. Load capacitance is
  chosen as `C = 2 × (CL − C_stray)`; assuming C_stray ≈ 3–5 pF, 10 pF
  corresponds to a crystal with CL ≈ 7–7.5 pF. Verify the CL of the crystal you
  actually chose. R1 = 0 Ω is a placeholder for now — measure the drive level and
  replace it with something in the 47–220 Ω range if needed.
- **VDDA filtering is correct:** FB1 (600 Ω @ 100 MHz) + C15 (0.1 µF) + C16
  (1 µF). On LQFP64, VREF+ is bonded internally to VDDA — there is no separate
  pin, which matches the schematic.
- **VCAP1/VCAP2 = 2.2 µF** ✓ the value ST requires.
- **The BOOT0 switch (S2)** is SPDT, so BOOT0 is never left floating — correct.
- **Flash /HOLD and /WP** pulled to 3.3 V through 10 kΩ ✓.
- **USB protection:** USBLC6-2SC6 + polyfuse + 5.1 kΩ CC resistors ✓ correct.
- **IIS2MDC C1 pin** decoupled with 220 nF to GND ✓ as the datasheet requires.
- **Unused pins:** PA15, PB0, PB3, PB6, PB7, PC0, PC1, PC5 — listed with their
  options in §1.1. PB0 and PC5 are worth calling out: together they are exactly
  what a second IMU would need (a chip select and an interrupt line), so if
  redundancy comes back on the table in Rev B, the pins are already free.

---

## 7. Proposed firmware task architecture

This does not need an RTOS; prioritised interrupts plus a super-loop are
sufficient and have more predictable timing.

| Rate | Task | Trigger |
|---|---|---|
| 1.6 kHz | IMU read + filtering | IMU INT1 → EXTI |
| 1 kHz | PID → mixer → DShot | IMU read counter (÷n) |
| 1 kHz | Attitude estimation (Mahony/Madgwick) | Same chain |
| 100 Hz | Baro + mag state machine | SysTick counter |
| 100 Hz | ADC (battery, current) | SysTick |
| 50 Hz | RC packet processing (CRSF) | UART4 DMA IDLE |
| 10 Hz | GPS parsing (UBX/NMEA) | USART2 DMA IDLE |
| 10 Hz | Telemetry transmission | SysTick |
| 1 kHz | Blackbox enqueue | From the PID loop |
| Idle | Blackbox flush to flash, pre-erase | Main loop |

**The safety layer — non-negotiable:**

- **IWDG (independent watchdog)**, ~50 ms timeout, fed **only** by the main
  control loop. If the PID chain stops, the board resets and the motors stop.
- **Gyro timeout:** if N consecutive samples fail to arrive (the INT line goes
  quiet) → immediate disarm.
- **RC timeout:** ~500 ms with no packet → failsafe (progressive throttle cut,
  or return-to-home if GPS is available).
- **Arming interlock:** motors are **always** disarmed at power-up; arming
  requires low throttle plus an explicit arm command plus a sensor health check.
- **DShot pins after reset:** GPIOs are high-impedance at MCU reset. Confirm the
  ESCs do not apply throttle in that state; add pull-down resistors on the motor
  lines in Rev B if they do.

---

## 8. Decision summary

| # | Decision | One-line rationale |
|---|---|---|
| D1 | IMU alone on SPI1 | One read takes 388 µs on I²C — 39 % of the bus at 1 kHz, impossible at 8 kHz |
| D2 | Baro + mag share I2C2 | Combined bus load 4 %; neither is on the critical path, so their failure does not end the flight |
| D3 | External I²C on its own bus (I2C1) | A cable means ESD, capacitance and lockup risk — isolate the internal sensors from it |
| D4 | Flash on its own SPI (SPI2) | A sector erase can take 400 ms and must never delay a gyro read |
| D5 | Motors on TIM8, not TIM3 | Resolves the PB4/PB5 pin conflict; the 168 MHz timer clock doubles DShot resolution |
| D6 | DShot600 | No calibration, CRC-protected, 26.7 µs frame — a digital guarantee in a noisy environment |
| D7 | No bidirectional DShot in v1 | ESC firmware dependency plus painful debugging; get it flying first |
| D8 | ADC without DMA | ADC1's streams are taken; two channels at 100 Hz in an interrupt is negligible |
| D9 | I²C driver as a non-blocking state machine | An I²C lockup must never stall the control loop |
| D10 | Interrupt-driven sampling, not polling | Constant latency = correct timestamps = correct fusion |

---

## 9. Next steps

1. **Fix the LED drive on PC13–PC15** (§6.7) — the one item here that is an
   actual datasheet violation rather than a preference. Flip to sink
   configuration and/or move two LEDs to the unused PC0/PC1.
2. **Record the 1 kHz target loop rate in `README.md`** (§5.1) — the ODR ceiling
   and the anti-alias behaviour are now confirmed from the datasheet; what
   remains is making sure the gyro is initialised in **Low-Noise Mode** with
   `GYRO_UI_FILT_BW` set deliberately, and writing that into the IMU driver as a
   commented requirement rather than a bare register value.
3. **Revisit the I2C1 pull-ups** (§2.5) — 4.7 kΩ → 2.2 kΩ for the cabled bus.
4. **Decide SBUS vs CRSF** (§6.6); if SBUS, an inverter has to go on the board.
5. **Add battery-sense protection** (§6.5) — 1 kΩ series into PC3 plus a clamp.
6. **Create `docs/power-tree.md`** — the 3.3 V load budget and the LDO thermal
   calculation (§6.4).
7. **Derive `board.h` from §1.1** — do not maintain the pin map in two places.
8. **Firmware bring-up order:** SWD + LED blink, then raw IMU reads over SPI1,
   then DShot output (props off!), then closed loop.

---

*Document revision 1.2 · 2026-08-24 · Sources: MiniTULPAR Rev A schematic
(2026-08-10) and the current CubeMX pin configuration; STM32F405 datasheet and
the RM0090 reference manual; TDK ICM-42670-P datasheet DS-000451 (ODR range,
AAF behaviour, 24 MHz SPI / 1 MHz I²C limits and FIFO size verified against it).*
*DMA stream/channel assignments follow RM0090 Tables 42/43; verify them against
the reference manual once the final pin and peripheral selections are frozen.*
