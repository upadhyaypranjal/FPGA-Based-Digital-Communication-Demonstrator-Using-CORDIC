# FPGA-Based CORDIC Digital Communication Demonstrator

---

**This project performs REAL hardware computation.** Every SIN, COS, and TAN value used in this repository is computed inside the FPGA fabric using the CORDIC algorithm, expressed purely as shift-and-add operations. Python never calls `math.sin()`, `math.cos()`, or `math.tan()` to produce a result — those functions are used only, and exclusively, as a reference for validating what the FPGA computed. The Python GUI is a visualization and control layer, not a computation engine.

---

## Overview

Digital communication systems — from Wi-Fi to satellite links to 5G — are built on a deceptively simple mathematical foundation: rotating a signal's phase to encode information, using nothing more than **sine and cosine**. Every phase-shift-keyed modulation scheme maps a bit or a group of bits onto a point on a circle, and that point is defined entirely by `(cos θ, sin θ)`.

Most educational projects teach this concept using `numpy.sin()` and `numpy.cos()` in a Jupyter notebook. This project deliberately does **not** do that. Instead, it asks and answers a harder, more interesting question:

> *What does it actually take to compute sine, cosine, and tangent using only addition, subtraction, and bit-shifts — on a real, resource-constrained FPGA — and then use those hardware-computed values to build a real modulator?*

The answer is the **CORDIC algorithm** (COordinate Rotation DIgital Computer), the same class of algorithm used inside real-world DSP chips, software-defined radios, and DDS (Direct Digital Synthesis) hardware, precisely because it avoids multipliers entirely. On a small FPGA like the SLG47910 — which has no dedicated multiplier blocks and a tight CLB budget — CORDIC isn't an academic curiosity, it's the *only* practical way to get trigonometric functions in hardware.

### Why FPGA implementation matters

A microcontroller computing `sin()` calls a library function that ultimately reduces to a polynomial or table approximation running on a general-purpose ALU, sequentially, at clock speeds bottlenecked by fetch-decode-execute overhead. An FPGA implementation, by contrast, is a purpose-built circuit: every clock cycle, the same small set of adders and shifters is doing exactly the useful work it was wired to do, with deterministic, cycle-accurate latency.

### Why digital communications need SIN and COS

Every phase-shift-keying (PSK) scheme transmits information by placing a symbol at a specific phase angle `θ` on the unit circle. To actually generate or demodulate that waveform, a radio needs the **I/Q components** `(cos θ, sin θ)` of that angle. This project makes that dependency explicit and tangible: the same CORDIC core that answers "what is sin(37°)?" is the core that generates the BPSK and QPSK symbols later in the pipeline.

### Why this project is educational

This repository is designed to be read, not just run. It intentionally exposes:
- The exact fixed-point format used at every stage
- The exact number of CORDIC iterations and why that number was chosen
- The exact resource trade-offs made to fit inside a **140-CLB** FPGA

If you are learning CORDIC, fixed-point arithmetic, SPI protocol design, or digital modulation, this project is meant to be a working reference you can step through end-to-end.

---

## Hardware Used

| Component | Role | Details |
|---|---|---|
| **Vicharak Shrike Lite** | Main hardware platform | Integrated development board containing both the **RP2040** and **Renesas ForgeFPGA SLG47910**. |
| **USB-Serial Link** | Host communication | USB CDC serial connection between the Python GUI and the Vicharak Shrike Lite. |

---

## Architecture

The following diagram illustrates the complete system architecture and data flow of the FPGA-Based CORDIC Digital Communication Demonstrator.

<p align="center">
  <img src="images/system_architecture.png" alt="FPGA-Based CORDIC Digital Communication Demonstrator Architecture" width="9000">
</p>

The Python Host GUI communicates with the Vicharak Shrike Lite board through a USB serial connection. The on-board RP2040 acts as a communication bridge, forwarding commands to the Renesas ForgeFPGA SLG47910 over the internal SPI interface. The FPGA performs quadrant reduction, CORDIC rotation, SIN/COS computation, and Linear Divider-based TAN computation entirely in hardware. The computed results are then transferred back through the RP2040 to the GUI, where they are visualized as modulation waveforms, I/Q constellation points, and processing results for both BPSK and QPSK communication modes.
---

## The CORDIC Algorithm

CORDIC computes rotations of a 2D vector using only **shifts and additions**, entirely avoiding multipliers — which is precisely why it was chosen for the SLG47910, a device with no dedicated multiplier hardware.

### Circular Rotation Mode

Starting from a vector `(x₀, y₀) = (K, 0)` and a target angle `z₀`, each iteration `i` applies:

```
if zᵢ ≥ 0:
    xᵢ₊₁ = xᵢ − (yᵢ >> i)
    yᵢ₊₁ = yᵢ + (xᵢ >> i)
    zᵢ₊₁ = zᵢ − atan(2⁻ⁱ)
else:
    xᵢ₊₁ = xᵢ + (yᵢ >> i)
    yᵢ₊₁ = yᵢ − (xᵢ >> i)
    zᵢ₊₁ = zᵢ + atan(2⁻ⁱ)
```

After `n` iterations, `xₙ → K · cos(z₀)` and `yₙ → K · sin(z₀)`, where `K` is the accumulated CORDIC gain.

### Why Quadrant Reduction Comes First

CORDIC rotation only converges reliably over roughly `[-90°, +90°]`. Real messages need to be modulated across the full 0°–360° range, so the angle-reduction logic reduces *any* input angle down into `[0°, 90°]` first, recording which quadrant it came from as two sign bits. This is what allows the system to correctly compute values for angles like `315°`, `-720°`, or angles far outside a single revolution — the reduction happens before CORDIC ever sees the angle.

### The atan Lookup Table

Each iteration needs the constant `atan(2⁻ⁱ)` for that specific iteration index. Rather than computing this at runtime (which would require a divider and an arctangent — the very things CORDIC exists to avoid), these constants are precomputed offline and stored in a tiny combinational lookup table indexed by the iteration counter.

### CORDIC Gain

Each rotation step doesn't preserve vector length — it scales it by `√(1 + 2⁻²ⁱ)`. The product of these scale factors over all iterations converges to a constant, `K ≈ 0.6072529`. Rather than dividing by this gain after the fact (another division!), the vector is simply initialized at `x₀ = K` instead of `x₀ = 1`, so the gain is pre-compensated for free.

### Fixed-Point Arithmetic and the Q1.14 Format

All values that cross the SPI boundary use **Q1.14 fixed-point**: a 16-bit signed word with 1 integer bit and 14 fractional bits, giving a representable range of roughly `[-2.0, +1.99994]` with a resolution of `1/16384 ≈ 6.1×10⁻⁵`. This single, consistent format is what lets the GUI, the RP2040 firmware, and every Verilog module agree on how to interpret a raw 16-bit integer without any additional metadata.

> **Resource-driven precision trade-off:** to fit the CORDIC and its TAN divider on the 140-CLB SLG47910, the internal rotation math runs in a narrower format than the external Q1.14 interface — angles are truncated on the way in and rescaled on the way out via free bit-slices, so the SPI protocol and GUI never see the difference, only a modest reduction in decimal precision.

### Why CORDIC Instead of Multipliers

A direct Taylor-series or polynomial approximation of sine/cosine needs several multiply-accumulate operations per evaluation. The SLG47910 has zero dedicated multiplier blocks — implementing multiplication in raw LUT fabric is extremely expensive in CLBs. CORDIC sidesteps this entirely: every operation in the algorithm is a shift (free — just wire routing) or an add/subtract (cheap, one adder per operand). This is the same reason CORDIC is used in real DDS chips and software-defined radio front ends.

TAN is computed the same way, in hardware, using a second linear-CORDIC divider stage rather than a conventional divider — never in Python, and never via `math.tan()`.

---

## BPSK (Binary Phase Shift Keying)

### Theory

BPSK is the simplest phase-shift-keying scheme: each bit is mapped to one of exactly two phase states, 180° apart on the unit circle.

### Equation

```
s(t) = A · cos(2π f_c t + θ)
where θ = 0°   for bit = 1
      θ = 180° for bit = 0
```

### Constellation

| Bit | Phase (θ) | I | Q |
|:---:|:---:|:---:|:---:|
| `1` | 0° | `+1` | `0` |
| `0` | 180° | `−1` | `0` |

### Advantages

- Maximum noise immunity per symbol among common PSK schemes (largest possible phase separation)
- Extremely simple demodulation — a single sign decision
- Lowest hardware complexity, making it ideal as a first demonstration on constrained hardware

### Waveform Behavior

Because the phase flips a full 180° between a `1` and a `0`, the BPSK waveform shows a sharp, visible phase discontinuity at every bit transition — this discontinuity is exactly what the CORDIC core is asked to compute a new `(cos θ, sin θ)` pair for, symbol by symbol.

---

## QPSK (Quadrature Phase Shift Keying)

### Theory

QPSK doubles the spectral efficiency of BPSK by encoding **2 bits per symbol**, mapping each 2-bit group to one of four phase states, 90° apart.

### Gray Coding

Adjacent constellation points differ by only **one bit**, minimizing the bit-error impact of a symbol being mistaken for its nearest neighbor due to noise:

| Bit Pair | Phase (θ) | I | Q |
|:---:|:---:|:---:|:---:|
| `00` | 45° | `+0.707` | `+0.707` |
| `01` | 135° | `−0.707` | `+0.707` |
| `11` | 225° | `−0.707` | `−0.707` |
| `10` | 315° | `+0.707` | `−0.707` |

### Phase Mapping Equation

```
s(t) = A · cos(2π f_c t + θ), θ ∈ {45°, 135°, 225°, 315°}
```

### Advantages

- Twice the bit rate of BPSK for the same symbol rate and bandwidth
- Still relatively simple to demodulate compared to higher-order QAM schemes
- Directly exercises all four quadrants of the CORDIC's angle range, making it the scheme that most thoroughly tests the angle-reduction logic

---

## SPI Communication Protocol

A minimal, byte-oriented command set drives the entire system. Every command is a single byte sent by the RP2040 (as SPI controller); replies are shifted back on the same transaction.

| Command | Direction | Payload | Description |
|:---:|:---:|---|---|
| `0xA1` | RP2040 to FPGA | 4 bytes, little-endian | Sends a new angle (Q1.14 fixed-point, radians × 16384) and triggers a new computation |
| `0xA2` | FPGA to RP2040 | 1 bit (in LSB of reply byte) | SIN/COS ready flag — polled until set before reading results |
| `0xA3` | FPGA to RP2040 | 1 byte | SIN result, low byte |
| `0xA4` | FPGA to RP2040 | 1 byte | SIN result, high byte |
| `0xA5` | FPGA to RP2040 | 1 bit (in LSB of reply byte) | TAN ready flag — polled after `0xA2`, since the linear divider runs after the rotation CORDIC finishes |
| `0xA6` | FPGA to RP2040 | 1 byte | COS result, low byte |
| `0xA7` | FPGA to RP2040 | 1 byte | COS result, high byte |
| `0xA8` | FPGA to RP2040 | 1 byte | TAN result, low byte |
| `0xA9` | FPGA to RP2040 | 1 byte | TAN result, high byte |

<details>
<summary><strong>Click to expand: example transaction sequence</strong></summary>

<br/>

```text
1. RP2040 sends: 0xA1, angle_byte0, angle_byte1, angle_byte2, angle_byte3
2. RP2040 polls: 0xA2  → repeat until LSB == 1
3. RP2040 reads: 0xA3 (sin_low), 0xA4 (sin_high)
4. RP2040 reads: 0xA6 (cos_low), 0xA7 (cos_high)
5. RP2040 polls: 0xA5  → repeat until LSB == 1
6. RP2040 reads: 0xA8 (tan_low), 0xA9 (tan_high)
```

All 16-bit results are two's-complement Q1.14 fixed-point; the GUI/firmware converts them back to floating-point by dividing by `16384.0`.

</details>

---

## Installation

### 1. Clone the Repository

```bash
git clone https://github.com/<your-username>/fpga-cordic-digital-comm-demonstrator.git
cd fpga-cordic-digital-comm-demonstrator
```

### 2. Install Python Requirements

```bash
pip install -r requirements.txt
```

This installs PyQt, Matplotlib, NumPy, and PySerial for the GUI and validation utilities.

### 3. Flash MicroPython Firmware to the RP2040

Open the firmware folder in **Thonny**, connect the Shrike Lite over USB, and flash the MicroPython firmware to the RP2040's filesystem.

### 4. Program the FPGA Bitstream

Using the ForgeFPGA toolchain (GateForge / Logic Studio), synthesize and generate the bitstream from the Verilog sources, then flash it to the SLG47910 via the RP2040's `shrike.flash()` utility.

### 5. Run the GUI

```bash
python gui/main.py
```

---

## Running the Project

1. **Power on** the Vicharak Shrike Lite and connect it to your computer via USB.
2. **Launch the GUI** and select the correct serial port.
3. **Choose a modulation scheme** — BPSK or QPSK.
4. **Type a message** to transmit.
5. Watch the message get encoded to ASCII, binary, phase angles, and Q1.14 fixed-point values.
6. Observe the FPGA converge on SIN, COS, and TAN for each symbol.
7. View the resulting constellation diagram and waveform as they populate in real time.
8. Optionally, send arbitrary angles directly and compare FPGA output against the software reference value.

---

## Results

The system reliably reproduces textbook SIN, COS, and TAN values for arbitrary input angles, including angles well outside a single revolution (e.g. `10009393°`, `-720°`), confirming that angle reduction, sign restoration, and both CORDIC engines behave correctly across the full range. Because the internal CORDIC datapath was deliberately narrowed to fit the SLG47910's CLB budget, results carry a modest, consistent numerical error relative to double-precision software trig — small enough to be functionally invisible in the resulting BPSK/QPSK constellations, but visible if you compare raw values directly.

BPSK constellations show two cleanly separated points at 0° and 180°; QPSK constellations show four Gray-coded points at 45°, 135°, 225°, and 315° — both populated entirely from FPGA-returned I/Q pairs, with no software-side trigonometric substitution at any point in the transmit chain.

---

## Future Enhancements

- [ ] **8PSK** — extend the phase mapper to 3 bits/symbol
- [ ] **16QAM** — combine phase and amplitude modulation
- [ ] **DDS (Direct Digital Synthesis)** — continuous carrier generation directly from the CORDIC core
- [ ] **OFDM** — multi-carrier transmission built on top of the existing symbol mapper
- [ ] **UART interface** — an alternative to SPI for host communication
- [ ] **Hardware acceleration** — pipelined, multi-symbol-per-cycle CORDIC for higher throughput
- [ ] **Real DAC output** — drive an actual analog waveform out of the board instead of only visualizing it in software

---

## Author

<div align="center">

**Pranjal Upadhyay**

[![Portfolio](https://img.shields.io/badge/Portfolio-000000?style=for-the-badge&logo=About.me&logoColor=white)](https://your-portfolio-link.example.com)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white)](https://linkedin.com/in/your-linkedin)
[![GitHub](https://img.shields.io/badge/GitHub-100000?style=for-the-badge&logo=github&logoColor=white)](https://github.com/your-username)
[![Email](https://img.shields.io/badge/Email-D14836?style=for-the-badge&logo=gmail&logoColor=white)](mailto:your.email@example.com)

</div>

> Update the portfolio, LinkedIn, GitHub, and email links above before publishing.
