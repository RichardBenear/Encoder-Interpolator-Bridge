# RP2040 Encoder Interpolator / ODrive SPI Bridge (AMT232 → Virtual AEAT9922 18-bit)

Author: Richard Benear

This project turns an **RP2040** into a small “encoder bridge” that upsamples the resolution for ODrive-style SPI encoder inputs. This implementation uses modified ODrive3.6(AEAT9922) open source code which can be found elsewhere in my repositories. 

**Concept:** Enhancing the CUI AMT23 14-bit SPI encoder to 18 bits via software interpolation to reduce "LSB hunting" in a direct-drive Alt/Az mount.

**Engineering Analysis:**

- **The Problem:** 14-bit resolution provides $\approx 79$ arcseconds per step. Sidereal tracking at 15 arcsec/sec means the encoder only ticks once every 5+ seconds. This causes the PID loop to "hunt" or oscillate between two discrete steps.
    
- **The Solution:** Use an RP2040 to "upsample" the 14-bit signal. By treating the Earth's rotation as a deterministic constant, we can use a **Predictive Filter** to bridge the gaps between the physical 14-bit steps.
    
- **Result:** A "synthetic" 18-bit signal ($4.9$ arcseconds per step) allows the PID controller to see a smooth ramp instead of a staircase.

## Implementation
In an effort to stay somewhat "standard", the AEAT9922-18 bit encoder was chosen as the target for the virtual 18 bits that are generated in case someone wanted to use a "real" AEAT9922 in place of this bridge.

The interpolator reads a **CUI AMT232** encoder frame (16-bit, 14-bit position + 2 check bits) using a **PIO-based SPI master** on **core1**, then publishes a response on **SPI0 in slave mode** so an **ODrive 3.6 / ODESC** can read either:

- **BYPASS mode**: the raw 16-bit AMT232 frame (ODrive “CUI” style checkbits), or
- **VIRT18 mode**: a **virtual AEAT9922-compatible** 24-bit SPI frame with:
  - 18-bit position
  - an error flag bit (set on bad AMT checkbits)
  - an even-parity bit

An optional **SSD1306 OLED** shows live values and parity/checkbit statistics. A single **NeoPixel** indicates mode.

---

## Features

- **PIO encoder sampling (core1)**  
  Generates AMT232 SPI clock, samples MISO, and returns 16-bit frames at a fixed rate.

- **Virtual 18-bit interpolation / smoothing**  
  A lightweight **alpha-beta filter** (steady-state constant-velocity Kalman form) unwraps the 14-bit angle and generates an 18-bit output.

- **ODrive SPI Slave responder (core0 ISR)**  
  Uses a **CS rising-edge IRQ** to reset PL022 state, then preloads the next 2 or 3 bytes into the TX FIFO.

- **Lock-free cross-core publish**  
  A ping-pong `PublishedFrame` buffer + `dmb()` barriers ensures the ISR reads a consistent frame.

- **Optional OLED diagnostics**
  Displays mode, 14-bit & 18-bit values, parity OK/FAIL, angle in degrees, and error statistics.

---

## How it works (high level)

1. **core1** samples the AMT232 at a fixed interval (`SAMPLE_PERIOD_US`) using a PIO program.
2. The AMT232 frame is validated using the ODrive/CUI checkbits (2-bit check value over the 14-bit position).
3. If valid, the 14-bit position is fed into the alpha-beta filter and updated as a **virtual 18-bit angle**.
4. core1 **publishes** either:
   - **2 bytes**: raw AMT232 16-bit frame (bypass), or
   - **3 bytes**: AEAT9922-like 24-bit frame (virt18)
5. **core0 ISR** fires on **ODRIVE CS rising edge**, resets the SPI peripheral, snapshots the published frame, and loads it into the SPI TX FIFO ready for the next master transaction.

---

## Output frame formats

### BYPASS mode (ODrive reads “CUI frame”)
- 16-bit frame from AMT232:
  - bits 13..0: position
  - bits 15..14: checkbits (ODrive/CUI style)

### VIRT18 mode (ODrive reads “AEAT9922-style frame”)
24-bit MSB-first frame with:
- bits 17..0 : position (18-bit)
- bit 18     : error flag (set when AMT checkbits fail)
- bit 19     : even parity over bits 18..0
- bits 23..20: 0

---

## Hardware

### RP2040 pins (as defined in code)

#### Mode select
- `BYPASS_PIN = 10`  
  Pulled up internally. **LOW = BYPASS**, **HIGH = VIRT18**.

#### AMT232 (PIO SPI master)
- `ENC_CS_PIN   = 3`
- `ENC_SCK_PIN  = 0`
- `ENC_MISO_PIN = 2`

#### ODrive SPI (RP2040 SPI0 slave)
- `ODRIVE_CS_PIN   = 5`
- `ODRIVE_SCK_PIN  = 6`
- `ODRIVE_MISO_PIN = 7`
- `ODRIVE_MOSI_NC_PIN = 4` (ODrive MOSI not used; held pulldown to avoid floating)

#### OLED (optional, I2C on Wire1)
- `OLED_SDA_PIN = 14`
- `OLED_SCL_PIN = 15`
- SSD1306 address: `0x3C`

#### Status NeoPixel
- Data pin: `16` (one pixel)

---

## Build / Flash

This is written for an RP2040 Arduino environment that supports:
- `pico/stdlib.h`, `pico/multicore.h`
- `hardware/pio.h`, `hardware/spi.h`, `hardware/gpio.h`
- Adafruit SSD1306 / GFX
- Adafruit NeoPixel
- A generated PIO header: `amt232_16b_falling_sample.pio.h`

### Typical toolchains
- **Arduino-Pico** core (Earle Philhower), or
- PlatformIO with the RP2040 Arduino core

> Make sure the PIO program is assembled and the generated header is available in the include path.

---

## Configuration knobs

These are the main constants you might tune:

- `AMT232_MAX_SCLK_HZ`  
  Encoder SPI clock rate (PIO SM clock divider derived from this).

- `SAMPLE_PERIOD_US`  
  Sampling interval (default 100 µs → 10 kHz).

- `OLED_UPDATE_MS`  
  OLED refresh rate.

- Alpha-beta filter constants inside `AlphaBeta18`:
  - `G_NUM / G_SHIFT` (position correction)
  - `H_NUM / H_SHIFT` (velocity correction)

---

## Notes / Diagnostics

- The OLED shows:
  - Mode (BYPASS / VIRT18)
  - 14-bit and 18-bit values
  - last checkbits status
  - degrees (scaled for 14 or 18-bit depending on mode)
  - total bad frames and percentage
  - windowed bad frames and percentage (window resets every `PARITY_WINDOW_N` samples)

- NeoPixel:
  - **Blue** = BYPASS
  - **Green** = VIRT18

---

