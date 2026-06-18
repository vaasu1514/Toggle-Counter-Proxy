# On-Chip Toggle-Counter Fingerprinting for Hardware Trojan Detection in FPGA AES Cores

A runtime Hardware Trojan detection framework for FPGA-based AES-128 cryptographic cores. Instead of external power probes or offline trace capture, the AES core is instrumented with small **digital toggle-counter sensors** that measure internal switching activity per encryption and use it as a **power side-channel proxy**. A Trojan-free baseline trains the detection thresholds; deviations at runtime flag malicious behaviour.

> **Status:** Manuscript under review at *IEEE Transactions on Information Forensics and Security (TIFS)*.
---

## Why this approach

Hardware Trojans can stay dormant through normal verification and fire only under a rare trigger, so functional testing alone often misses them. Classic side-channel detection works, but it usually needs an oscilloscope, current probe, or a platform like ChipWhisperer, plus trace alignment and offline processing — awkward for lightweight, deployed FPGA systems.

This project moves the measurement *inside* the chip. Because dynamic CMOS power scales with switching activity, counting bit toggles on key internal paths gives a cheap digital stand-in for a power trace — no external instrumentation required.

**What this needs:** an FPGA board, a UART cable, and a host script.
**What it does not need:** oscilloscope, logic analyzer, ChipWhisperer, or current/differential probes.

---

## How it works

### Toggle-counter sensor

Each monitored signal `x[t]` is compared against its previous value, the changed bits are counted, and the count is accumulated over one encryption:

```
Δx[t]   = x[t] XOR x[t-1]      # bits that flipped this cycle
a[t]    = popcount(Δx[t])      # number of toggles this cycle
A       = Σ a[t]               # accumulated activity for the transaction
```

The sensor is a short pipeline: **previous-value register → XOR → popcount → adder → accumulator register**.

### Monitored features

Four activity counters are captured per encryption:

| Feature  | What it watches                                  |
|----------|--------------------------------------------------|
| State    | AES internal state datapath                      |
| Key      | Round-key / key-schedule path                    |
| Control  | FSM / control signals (primary discriminator)    |
| Total    | Aggregate switching-activity proxy               |

The **control-path** counter is the most useful: a clean AES controller runs the same deterministic sequence every time, so any Trojan that perturbs control flow shows up immediately as a control-activity spike.

### System flow

```
Host PC  ──UART RX (0xAA + 16-byte plaintext)──►  FPGA: AES-128 core + activity sensors
Host PC  ◄─UART TX (0x55 + 16-byte ciphertext + 8-byte activity)──  FPGA
```

The host streams random plaintexts, checks the returned ciphertext against a software AES reference, and logs the four activity counters for analysis.

---

## Detection method

A Trojan-free run characterises each feature `f` with its mean and standard deviation, and a 3σ threshold is set:

```
T_f = mean_f + 3 · std_f
```

A transaction is flagged when any feature exceeds its threshold. The control path is fully deterministic in the clean design (`mean = 21`, `std = 0`), so its threshold collapses to a hard line at **21**, and the dedicated leakage trigger is detected with the event rule **Ctrl ≥ 24**.

---

## Trojan benchmarks

Two Trojan families are used for evaluation:

- **Performance Degradation Trojan** — a sequential time-bomb (small FSM watching for a rare plaintext-byte sequence). On trigger it fires a burst payload that stresses the datapath/control logic. Two variants are included: **Case 1** (stealthier, smaller footprint) and **Case 2** (stronger). Rare activation can cause intermittent ciphertext mismatches from reduced timing margin.
- **Information Leakage Trojan** — an AES-T1000-style confidentiality attack. On a specific trigger plaintext it leaks the fixed AES key through the output path while leaving normal encryptions untouched.
  - Trigger plaintext: `00112233445566778899AABBCCDDEEFF`
  - Leaked key: `000102030405060708090A0B0C0D0E0F`

---

## Results

Measured on a Xilinx Artix-7 (Basys 3).

**Control-path separation** — clean baseline sits flat at 21; Trojan runs cross the threshold:

| Run                                | Peak control activity |
|------------------------------------|-----------------------|
| Clean baseline                     | 21                    |
| Performance Degradation — Case 1   | 39                    |
| Performance Degradation — Case 2   | 57                    |
| Information Leakage                | 24                    |

**Leakage detection (event rule Ctrl ≥ 24)** — over 1,000 transactions:

| Metric    | Value |
|-----------|-------|
| True Positives  | 4   |
| False Positives | 0   |
| False Negatives | 0   |
| Precision / Recall / F1 | 100% |

The Performance Degradation Trojan also produced an observed ciphertext mismatch rate of about **0.1%**, confirming a real functional impact that ordinary testing tends to miss.

**Implementation overhead** (AES only → AES + sensors):

| Metric | AES only | AES + sensors | Change   |
|--------|----------|---------------|----------|
| LUT    | 977      | 1192          | +22.01%  |
| FF     | 771      | 947           | +22.83%  |
| Power  | 0.069 W  | 0.071 W       | +2.90%   |
| Timing | —        | —             | closure maintained (positive WNS) |

The sensors add modest logic but only ~2.9% power, and timing closure holds — practical for always-on runtime monitoring.

---

## Repository layout

> Adjust to match your actual file names.

```
.
├── rtl/                 # AES-128 core, UART, toggle-counter sensors, top module
├── trojans/             # Performance Degradation (C1/C2) and Information Leakage variants
├── sim/                 # Testbenches and simulation scripts
├── host/                # Python automation: plaintext injection, ciphertext check, logging
├── constraints/         # Basys 3 .xdc pin/clock constraints
├── results/             # Captured datasets, plots, overhead reports
└── docs/                # Paper, figures, notes
```

---

## Getting started

**Requirements**
- Xilinx Vivado (tested with the Basys 3 / Artix-7 flow)
- A Basys 3 board (or compatible Artix-7 target)
- Python 3 with `pyserial`, `numpy`, `pandas`, `matplotlib`

**Build and program**
1. Open the project in Vivado, run synthesis and implementation, and generate the bitstream.
2. Program the board over USB-JTAG.

**Collect activity data**
```bash
cd host
python collect.py --port COM3 --baud 115200 --n 10000 --out results/baseline.csv
```
The script sends random plaintexts, verifies each returned ciphertext against a software AES reference, and logs the four activity counters.

**Run detection**
```bash
python detect.py --baseline results/baseline.csv --test results/trojan.csv
```
This learns the 3σ thresholds from the clean baseline and reports threshold crossings and the Ctrl ≥ 24 leakage events.

---

## Authors

Sarath Srinivasan, **Vaasudev Enamandram**, Mukund Varshney, and Abhishek Kumar
Department of Electrical Engineering, Indian Institute of Technology Ropar.

## Citation

If you use this work, please cite:

```bibtex
@unpublished{srinivasan_toggle_counter_2026,
  author = {Srinivasan, Sarath and Enamandram, Vaasudev and Varshney, Mukund and Kumar, Abhishek},
  title  = {On-Chip Toggle-Counter Fingerprinting as a Runtime Power Side-Channel
            Proxy for Hardware Trojan Detection in FPGA-Based AES Cryptographic Cores},
  note   = {Manuscript under review, IEEE Transactions on Information Forensics and Security},
  year   = {2026}
}
```

## Acknowledgements

The authors thank the Department of Electrical Engineering, IIT Ropar, for the lab environment that supported this work. AI-assisted tools were used only for language and formatting support during manuscript preparation; the research idea, methodology, implementation, and analysis are the authors' own.
