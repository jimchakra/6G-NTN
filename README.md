# Ka-Band LEO & 6G RAN — RF/SoC Architecture Studies

Three self-contained interactive HTML tools covering Ka-band LEO satellite
link budget / DPD architecture, GNSS-free acquisition, and 5G/6G RAN DU
hardware acceleration. Each opens directly in any modern browser — no server,
no build step, no dependencies.

---

## Quick Access

| | Tool | Link |
|---|---|---|
| 📡 | **Ka-Band Link Budget · High-MCS & DPD/Shaper** | [ka_link_budget.html](https://jimchakra.github.io/6G-NTN/ka_link_budget.html) |
| 🛰️ | **Ka-Band GNSS-Free Acquisition · M1–M5 Framework** | [ka_pnt_acquisition.html](https://jimchakra.github.io/6G-NTN/ka_pnt_acquisition.html) |
| 🔬 | **5G gNB / 6G aNB RAN DU Architecture & HW Acceleration Study** | [6g_du_rd_ai.html](https://jimchakra.github.io/6G-NTN/6g_du_rd_ai.html) |

---

## Tools

### 1. [`ka_link_budget.html`](https://jimchakra.github.io/6G-NTN/ka_link_budget.html) — Ka-Band Link Budget · High-MCS & DPD/Shaper

**Topic:** Can a Ka-band LEO satellite phased array reach 256-QAM or 1024-QAM,
and what does DPD cost vs. save?

A full downlink link budget simulator for LEO/MEO Ka-band phased arrays,
focused on the MCS ceiling set by PA nonlinearity and the power economics
of Digital Pre-Distortion. Parameters flow from constellation presets
(Kuiper, Starlink) through the full RF chain to MCS selection and DC
power comparison.

**Six tabs:**

| Tab | Content |
|---|---|
| **EVM Budget** | Side-by-side EVM breakdown for 256-QAM and 1024-QAM. All impairment sources (PA residual, phase noise, IQ imbalance, quantisation, jitter, thermal, misc) shown as individual bars. Pass/fail per MCS limit. |
| **MCS Feasibility** | Full MCS table from QPSK to 4096-QAM. C/N margin and EVM gating shown separately — identifies whether the link or the PA is the binding constraint. SCS selector wired to BW slider: changing SCS updates throughput live. |
| **Link Budget** | Full parameter chain: EIRP → path loss → C/N → Shannon capacity → throughput. |
| **PA Power Saving** | DC power savings from DPD at the current constellation/array settings. ESN system power cost (182 mW) vs array-level PA savings. ROI computed live. |
| **Power at MCS** | Per-MCS DC power comparison and bar chart. Includes 2025–2030 battery-operated CT roadmap: ~18W table-mount today → ~8W portable (2028, ESN well-tuned + N3E PA) → ~5W handheld (2030). |
| **DPD Notes** | Appendix: ESN standard vs well-tuned vs GRU-NN — three modes compared, the three prerequisites for ESN well-tuned converging at N3E (2028), shaper/PAPR synergy with ESN, signal chain terminology. |

**Constellation presets (sidebar):**
- Kuiper Production — 590km, Ka 28 GHz, est. ~2048 elem (5 m² panel)
- Kuiper High-Shell — 630km (the shell that demonstrated 1.28 Gbps, Sep 2025)
- Starlink V2 Mini — 550km, Ka 20 GHz, est. ~1500 Ka elem (11.07 m² panel, confirmed)
- Starlink V3 — 570km, est. ~6000 elem, 1 Tbps/sat

**DPD modes (progressive complexity):**

| Mode | OBO recovery | PA EVM residual | Notes |
|---|---|---|---|
| No DPD | 0 dB | PA floor | Baseline |
| Shaper / PAPR | +1.0 dB | — | Clip+filter pre-PA. 2022 Kuiper approach. Zero feedback HW. |
| OTA single-model | +3.5 dB | 1.5% | Gateway observes feeder link only — not user link (separate antennas). |
| GMP (Memory poly) | +2.5 dB | 2.5% | 50–100 coeff/sub-array. Standard Ka DPD. |
| Sub-array ESN ★ | +4.5 dB | 1.8% | Generic random reservoir. No calibration needed. Works day 1. |
| ESN well-tuned | +4.5 dB | 1.2% | GRU-NN offline (NOC server) informs reservoir design from bench PA IQ. Burned at OTP — tape-out decision, not field upgrade. Prerequisite: precision cal terminal network. |
| GRU-NN DPD | +5.0 dB | 1.0% | Offline design tool only — never runs on satellite. Produces optimal ESN reservoir for next tape-out. |

**Key findings encoded in the tool:**
- All DPD models are SISO. Sub-array ESN reuses the DBF Rx ADC path — incremental satellite HW ~$30 BOM.
- ESN standard = blind calibration (generic random reservoir, no PA measurement). ESN well-tuned = bench characterisation (GRU-NN on server, IQ from precision cal terminal, burned at integration). GRU-NN = offline design tool, not runtime.
- Higher QAM requires more OBO for PA linearity → lower efficiency → more DC wasted. DPD recovers OBO → higher efficiency → DC savings. 1024-QAM without DPD: 13% PA efficiency. With ESN well-tuned: ~37%.
- Battery CT roadmap: 8W portable target (2028) requires ESN well-tuned + N3E PA (PAE 52%). DPD is the single largest lever — reduces TX array power 3.5×.

---

### 2. [`ka_pnt_acquisition.html`](https://jimchakra.github.io/6G-NTN/ka_pnt_acquisition.html) — Ka-Band GNSS-Free Acquisition · M1–M5 Framework

**Topic:** How does a Ka-band LEO terminal acquire a satellite and achieve
positioning without GPS, and what does each approach cost in waveform overhead?

**Eight tabs:**

| Tab | Content |
|---|---|
| **M1–M5 Proposals** | All four proposals with mechanism, reference, patent flags, and OH. M1b checked by default. Interactive checkboxes affect TTFF live. |
| **TTFF Timeline** | Cold (~400ms M1b path) / warm (~185ms) / hot (~10ms). ZC collapses PSS+SSS in one correlation — no SSS step. |
| **CW Collision** | Simulated Doppler spectrum. Per-shell CW allocation. Kuiper 3-shell cross-shell collision at zenith: geometrically inevitable, fix is per-shell subcarrier (0.042% OH). |
| **Sync Design** | Subcarrier allocation canvas: ZC sync comb + PBCH ephemeris + CW pilot + PDSCH. NR vs proprietary SSB comparison. |
| **Overhead Budget** | Per-signal OH bars. PTRS/DPD synergy. |
| **Ranging Accuracy** | CRB timing σ vs sync BW. TCXO clock floor ~30 cm — full-BW sync wasteful above 240 SC. |
| **Acq. Link Budget** | Full single-element acquisition link budget. Wide-beam EIRP penalty (−25 dB) vs ZC processing gain (+24 dB). SNR after MF = +7.0 dB. PSS margin 13 dB ✓, PBCH margin 17 dB ✓. Single unpointed element decodes PSS+PBCH in one 20ms SSB period. |
| **Rain & Cold-Start** | CW resilience in downpour (26 dB margin at 25mm/hr — survives any practical rain). Sub-array necessity (single element fails at 25mm/hr, 16-elem sub-array passes with 7.4 dB margin). Beam search bins: CW Doppler collapses sky to ~24 azimuth bins → ~480ms search. TTFF: ~400ms clear sky → ~880ms downpour (2.2× slower, still under 1 second). |

**The M1–M5 framework:**

| ID | Name | TTFF (cold) | OH | Notes |
|---|---|---|---|---|
| M1a | Terminal beam sweep (HOOC-EM) | 10–40 s | 0% | Terminal-side only. Phase 1. |
| M1b | Satellite wide-beam SSB | ~1.5 s | 0% incr. | 30° half-angle from nadir (~700km ground radius). Visible to any terminal >25° elevation. Patent risk: US20220216896A1. Phase 2. |
| M2 | CW pilot tone | ~100 ms detect | 0.004% | Per-shell subcarrier for Kuiper 3-shell. Rides inside M1b. Survives downpour (26 dB margin). |
| M5 | Ephemeris in PBCH | 0 (enables) | 0% | 190-bit nav payload (BPSK, rate-1/4 polar). Zero incremental OH. |

**Key simulation findings:**
- ZC replaces both PSS and SSS: one correlation gives satellite ID + timing + channel estimate simultaneously. SSS eliminated entirely.
- Acquisition link budget: noise reference is full channel BW (200 MHz), not sync BW. Raw SNR = −17.2 dB → +7.0 dB after 24.1 dB ZC processing gain.
- Rain resilience: CW is essentially impervious (narrow FFT bin, 100ms integration). Sub-array (16 elem, +12 dB) is necessary for downpour PBCH decode. Beam search is the bottleneck, not SNR.

---

### 3. [`6g_du_rd_ai.html`](https://jimchakra.github.io/6G-NTN/6g_du_rd_ai.html) — 5G gNB / 6G aNB RAN DU Architecture & HW Acceleration Study

**Topic:** What hardware acceleration does a 5G/6G RAN Distributed Unit
actually need, sized correctly through simulation — and how should the
RTL/FW/DV development be structured to de-risk tape-out?

**Navigation (Co-Design Flow + 9 simulation sections):**

| Section | Content |
|---|---|
| **Co-Design Flow** | Principal architect methodology: YAML micro-arch spec as single source of truth before any RTL or SystemC is written. YAML fans out to RTL, SystemC (LLS HW), ISS (LLS FW), FW headers, and UVM — all from the same spec. FW functional before tape-out. DV closed at tape-out. Bring-up as confirmation rather than discovery. |
| **Methodology** | How a principal engineer approaches SoC design: bottom-up from product envelope, explicit bias correction at each stage. |
| **① Product Envelope** | Company-A DU100 / QRU100 reference spec. 6G aNB target. |
| **② Static Latency** | Minimum latency through a single CC/UE pipeline with no contention. |
| **③ Static Peak** | Coprocessor sizing under worst-case load. |
| **④ Contention Simulation** | SimPy Monte Carlo — closes the static analysis bias. FR3 6G aNB. |
| **⑤ DVFS Analysis** | Accelerator → frequency → voltage → power mapping. |
| **⑥a VU Tradeoff** | Vector unit time-share vs dedicated at TSMC N3E. |
| **⑥b O-RAN Partition** | HRT/PRE functional split. |

**Co-Design methodology detail:**

The YAML micro-architecture spec captures state machines, register maps,
pipeline stages, interrupt behavior, DMA protocols, and timing constraints —
more precise than English spec, less detailed than RTL. One change propagates
to all downstream artifacts by construction, not by cross-team review.
SystemC + ISS (not RTL sim) is the right LLS platform: RTL simulation is
10,000–100,000× slower than silicon and cannot support real FW development.

**Key simulation findings:**

| # | Finding | Headline number |
|---|---|---|
| F01 | Two-layer HRT/PRE architecture is correct | Cache non-determinism from PRE bleeds into HRT if mixed |
| F02 | Scheduler is the tightest path, not PDCCH | 72% of DCI fields pre-computable; last-mile patch ~1.2 µs |
| F03 | FR1 needs very little HW acceleration | 4–5 of 9 knobs sufficient for FR2 |
| F04 | AI Scheduler is deployable now | GPU scheduler: 2,576 µs → **92 µs** (28×). No 3GPP change needed. |
| F05 | LDPC decode is the PUSCH bottleneck | HW: 432 µs vs SW: 6,914 µs (**16×**). |
| F06 | Bit processing: pervasive 4× gain | Custom ISA ~4×; coprocessors ~10× |
| F07 | 3GPP independently confirmed the DCI bottleneck | Rel-18 DCI Format 1_3/0_3 |
| App-A | 4 parallel LDPC engines optimal for multi-CC | 8CC P99 grows only **1.16×** vs 1CC |
| App-B | SRS mixed-µ is a real scheduling hazard | **7.9% miss rate** in simulation (3,000 runs) |

---

## Common Design Threads

**DPD and PNT share the same PTRS signal.**
PTRS is present for 256-QAM phase coherence. Sub-metre carrier-phase PNT
is a free byproduct — paid for by the throughput roadmap, not a separate budget.

**The 1 Gbps competitive target.**
Kuiper demonstrated 1.28 Gbps from 630km in September 2025. The link budget
tool quantifies the DPD path to 256-QAM / 1024-QAM as the software-defined
route to competitive parity without a satellite redesign.

**GPS-free acquisition and high-MCS are complementary.**
M1b+M2+M5 adds 0.042% waveform overhead. The two system priorities converge
on the same waveform decisions.

**3GPP Rel-20 will not solve these for Ka-band.**
Rel-20 GNSS-resilient NTN targets L-band/FR1/IoT. Kuiper's Ka-band proprietary
waveform is ahead of standardisation in both DPD and GPS-free acquisition.

---

## File Summary

| File | Tabs | Topic | Interactive? |
|---|---|---|---|
| [`ka_link_budget.html`](https://jimchakra.github.io/6G-NTN/ka_link_budget.html) | 6 | Ka LEO link budget, DPD/Shaper, MCS, PA power, battery CT roadmap | ✓ Live sliders, presets |
| [`ka_pnt_acquisition.html`](https://jimchakra.github.io/6G-NTN/ka_pnt_acquisition.html) | 8 | GNSS-free acquisition, M1–M5, CW simulation, rain resilience | ✓ Live simulation, canvas |
| [`6g_du_rd_ai.html`](https://jimchakra.github.io/6G-NTN/6g_du_rd_ai.html) | 9+ | Co-design methodology, 5G/6G DU HW acceleration, SimPy Monte Carlo | ✓ Interactive simulator |

All tools are fully self-contained HTML — open in any modern browser, no server required.

---

## References

- Morgan & Humphreys, "HOOC-EM," ION GNSS+ 2024 (Amazon Kuiper funded)
- Kassas et al., Starlink Ku-band opportunistic PNT, ION GNSS+ 2024
- Iannucci & Humphreys, fused LEO GNSS <1.6% OH, IEEE TAES 2022
- Lynk Global, US20220216896A1 (filed Jan 2021) — wide-beam navigation signal patent
- DPD-NeuralEngine, 22 nm CMOS GRU-RNN DPD at 250 MSps, ISCAS 2025
- ITU-R P.838-3, specific rain attenuation model at Ka-band
- Starlink V2 Mini antenna panel area 11.07 m², arxiv 2306.06657
- Kuiper terminal apertures (18/28/50×78 cm), DSEI Sep 2025
- 3GPP TS 38.211/212/213, TR 38.843 (AI/ML for NR air interface)
- 3GPP Rel-18 DCI Format 1_3/0_3 (multi-CC scheduling)
- O-RAN Alliance WG4 fronthaul interface specifications
