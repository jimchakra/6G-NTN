# Ka-Band LEO & 6G RAN — RF/SoC Architecture Studies

Three self-contained interactive HTML tools covering Ka-band LEO satellite
link budget / DPD architecture, GNSS-free acquisition, and 5G/6G RAN DU
hardware acceleration. Each opens directly in any modern browser — no server,
no build step, no dependencies.

---

## Tools

### 1. `ka_link_budget.html` — Ka-Band Link Budget · High-MCS & DPD/Shaper

**Topic:** Can a Ka-band LEO satellite phased array reach 256-QAM or 1024-QAM,
and what does DPD cost vs. save?

A full downlink link budget simulator for LEO/MEO Ka-band phased arrays,
focused on the MCS ceiling set by PA nonlinearity and the power economics
of Digital Pre-Distortion. Parameters flow from constellation presets
(Kuiper, Starlink) through the full RF chain to MCS selection and DC
power comparison.

**Five tabs:**

| Tab | Content |
|---|---|
| **EVM Budget** | Side-by-side EVM breakdown for 256-QAM and 1024-QAM. All impairment sources (PA residual, phase noise, IQ imbalance, quantisation, jitter, thermal, misc) shown as individual bars. Pass/fail per MCS limit. |
| **MCS Feasibility** | Full MCS table from QPSK to 4096-QAM. C/N margin and EVM gating shown separately — identifies whether the link or the PA is the binding constraint. |
| **Link Budget** | Full parameter chain: EIRP → path loss → C/N → Shannon capacity → throughput. Corrected formulas: spectral efficiency already includes code rate; no double-counting. |
| **PA Power Saving** | DC power savings from DPD at the current constellation/array settings. ESN system power cost (182 mW) vs array-level PA savings. ROI computed live. |
| **Power at MCS** | Per-MCS DC power comparison table and bar chart. Shows exactly how much power is saved at 16-QAM, 64-QAM, 256-QAM, and 1024-QAM for the selected DPD mode. |

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
| OTA single-model | +3.5 dB | 1.5% | Gateway observes feeder link only — **not** user link (separate antennas). |
| GMP (Memory poly) | +2.5 dB | 2.5% | 50–100 coeff/sub-array. Standard Ka DPD. |
| Sub-array ESN ★ | +4.5 dB | 1.8% | K=16 ESNs. Fixed reservoir (ROM). Output weights only adapt. DBF Rx ADC reused — incremental HW ~\$30/satellite. |
| ESN well-tuned | +4.5 dB | 1.2% | N_res=500, optimal spectral radius. Enables 1024-QAM at LEO. |
| GRU-NN DPD | +5.0 dB | 1.0% | Best linearisation. 22 nm, 250 MSps demonstrated (ISCAS 2025). |

**Key findings encoded in the tool:**
- All DPD models are SISO (single-input, single-output). Topology choice (per-element / sub-array / OTA) is independent of model type.
- Sub-array ESN reuses the DBF Rx ADC path — the only new satellite hardware is K=16 passive directional couplers + RF switches.
- ESN inference runs at DAC rate (~500 MSps) in a hardwired ASIC datapath (sparse MAC array). ESN learning (coefficient update, 100 ms period) runs on the existing Prometheus-class control CPU — a Cortex-M solves the 100×100 linear system in ~55 ms.
- OTA DPD via gateway is valid for the feeder link only. It cannot correct the user-link PA chain because Kuiper uses separate antennas for the two links.
- PA efficiency model: η(OBO) = η_sat × 10^(−OBO/20), η_sat = 40% (Ka-band GaAs class AB).

---

### 2. `ka_pnt_acquisition.html` — Ka-Band GNSS-Free Acquisition · M1–M5 Framework

**Topic:** How does a Ka-band LEO terminal acquire a satellite and achieve
positioning without GPS, and what does each approach cost in waveform overhead?

An interactive reference covering five GPS-free acquisition proposals (M1a,
M1b, M2, M4/M5), CW pilot tone Doppler collision analysis specific to Kuiper's
3-shell Ka architecture, proprietary sync signal design, and the ranging
accuracy ceiling imposed by sync bandwidth vs. terminal clock quality.

**Six tabs:**

| Tab | Content |
|---|---|
| **M1–M5 Proposals** | All four proposals with mechanism, reference, patent flags, and OH. Interactive checkboxes affect TTFF estimates live. |
| **TTFF Timeline** | Step-by-step cold / warm / hot start sequences with per-step timestamps. Cold: ~400 ms (M1b+M2+M5). Warm: ~150 ms. Hot (handoff): ~10 ms. |
| **CW Collision** | Simulated Doppler spectrum for the selected constellation. Per-shell CW subcarrier allocation shown visually. Collision statistics including the geometrically inevitable 0.15% cross-shell zenith collision unique to Kuiper's 3-shell architecture. |
| **Sync Design** | Subcarrier allocation canvas for one OFDM symbol: ZC sync comb + PBCH ephemeris + CW pilot + PDSCH (rate-matched). NR SSB vs proprietary 2-symbol SSB comparison. |
| **Overhead Budget** | Per-signal OH bars with notes. PTRS/DPD synergy: sub-metre carrier-phase PNT is a free byproduct of the 256-QAM PTRS infrastructure. |
| **Ranging Accuracy** | CRB timing σ vs sync bandwidth. Identifies the TCXO clock floor (~30 cm) that makes full-BW sync wasteful for code-phase PNT above 240 SC. |

**The M1–M5 framework:**

| ID | Name | TTFF (cold) | OH | Published | Notes |
|---|---|---|---|---|---|
| M1a | Terminal beam sweep (HOOC-EM) | 10–40 s | 0% | ✓ ION GNSS+ 2024, Amazon Kuiper funded | Terminal-side only. Satellite unchanged. Phase 1. |
| M1b | Satellite wide-beam SSB | ~1.5 s | 0% incr. | — (proposed) | DBF weight change. Patent risk: US20220216896A1 (Lynk Global). Phase 2. |
| M2 | CW pilot tone | ~100 ms detect | 0.004% | ✓ Kassas group, Starlink Ku observed | Per-shell subcarrier for Kuiper 3-shell. Rides inside M1b. |
| M5 | Ephemeris in PBCH | 0 (enables) | 0% | ✓ Analog: Iridium STL | 190-bit nav payload. Zero incremental OH. Enables direct Ka array steering. |

**M1b is the container. M2 and M5 are its payload.** One wide-beam SSB burst
carries the CW subcarrier (M2) and ephemeris (M5) — one transmission, three
functions.

**Key simulation finding — Kuiper 3-shell CW collision:**  
The 0.15% cross-shell Doppler collision at zenith is geometrically inevitable
— not probabilistic. Two satellites from different shells pass zenith
simultaneously, giving identical zero radial velocity. The collision rate does
not improve with longer integration time.  
Fix: per-shell CW subcarrier allocation (3 subcarriers / ~6667 total = **0.042% OH**).
This finding is specific to Kuiper's 3-shell Ka architecture and is not in
published literature.

**Key sync design finding:**  
Matched filter SNR = Pr × T_sym / N0 — independent of how many subcarriers the
sync occupies. Wider sync BW does not improve detection SNR; it only improves
ranging resolution. Above 240 SC (18 MHz), the ranging accuracy is clock-limited
(TCXO ~30 cm floor), not noise-limited. Full-BW sync only pays off with OCXO or
PTRS carrier-phase ranging.

---

### 3. `6g_du_rd_ai.html` — 5G gNB / 6G aNB RAN DU Architecture & HW Acceleration Study

**Topic:** What hardware acceleration does a 5G/6G RAN Distributed Unit
actually need, sized correctly through simulation?

A principal-engineer-level, simulation-driven analysis of DU hardware
acceleration — working bottom-up from product envelope to static latency,
worst-case peak sizing, Monte Carlo resource contention simulation, DVFS
analysis, and architecture decisions. Grounded in Company-A DU100 platform
analysis (public datasheet data, verified July 2026).

**Navigation (11 sections):**

| Section | Content |
|---|---|
| **① Product Envelope** | Company-A DU100 / QRU100 reference spec. 6G aNB target. Public datasheet comparison. |
| **② Static Latency** | Minimum latency through a single CC/UE pipeline with no contention. |
| **③ Static Peak** | Coprocessor sizing under worst-case load. |
| **④ Contention Simulation** | SimPy Monte Carlo — closes the static analysis bias. FR3 6G aNB. |
| **⑤ DVFS Analysis** | Accelerator → frequency → voltage → power mapping. |
| **⑥a VU Tradeoff** | Vector unit time-share vs dedicated at TSMC N3E. |
| **⑥b O-RAN Partition** | HRT/PRE functional split in the O-RAN stack (HRTC / PRE). |
| **RU Budget** | RU/DU interface timing budget model. |
| **Appendix** | Full reasoning, methodology, and task timeline. |

**Key findings:**

| # | Finding | Headline number |
|---|---|---|
| F01 | Two-layer HRT/PRE architecture is correct | Cache non-determinism from PRE bleeds into HRT if mixed |
| F02 | Scheduler is the tightest path, not PDCCH | 72% of DCI fields pre-computable; last-mile patch ~1.2 µs |
| F03 | FR1 needs very little HW acceleration | 4–5 of 9 knobs sufficient for FR2 |
| F04 | AI Scheduler is deployable now | GPU scheduler: 2,576 µs → **92 µs** (28×). No 3GPP change needed. |
| F05 | LDPC decode is the PUSCH bottleneck | HW: 432 µs vs SW: 6,914 µs (**16×**). K1=1 URLLC fails even with HW. |
| F06 | Bit processing: pervasive 4× gain | Custom ISA instructions ~4×; coprocessors ~10× |
| F07 | 3GPP independently confirmed the DCI bottleneck | Rel-18 DCI Format 1_3/0_3 is the standardised response |
| App-A | 4 parallel LDPC engines is optimal for multi-CC | 8CC P99 grows only **1.16×** vs 1CC. Serial: **4.54×** |
| App-B | SRS mixed-µ is a real scheduling hazard | Company-A DU100: **7.9% miss rate** in simulation (3,000 runs) |

**Methodology note:** Absolute timings are order-of-magnitude estimates. Relative
comparisons (SW vs HW, O(CC) vs O(CC^0.4)) are the credible findings.

---

## Common Design Threads

These three tools were developed together and share a set of connected insights:

**DPD and PNT share the same PTRS signal.**  
PTRS is already present in the waveform for 256-QAM phase coherence. The
same signal infrastructure enables sub-metre carrier-phase PNT ranging as a
free byproduct — the PNT capability is paid for by the throughput roadmap,
not a separate budget.

**The 1 Gbps competitive target.**  
Starlink V3 (Starship-deployed, 1 Tbps/sat) targets 1 Gbps to a Performance
terminal in 2026. Kuiper demonstrated 1.28 Gbps from 630km production satellites
in September 2025. The link budget tool quantifies the DPD path to 256-QAM /
1024-QAM as the software-defined route to competitive parity without a V3-class
satellite redesign.

**GPS-free acquisition and high-MCS are complementary, not competing.**  
M1b+M2+M5 adds 0.042% waveform overhead. PTRS for 256-QAM phase coherence
is already budgeted. The two system priorities (PNT resilience and throughput)
converge on the same waveform decisions.

**3GPP Rel-20 will not solve these for Ka-band.**  
Rel-20 GNSS-resilient NTN (targeted March 2027) operates at L-band/FR1/IoT
level. Kuiper's Ka-band proprietary waveform is ahead of standardisation in
both DPD architecture and GPS-free acquisition. The 6G DU study similarly
identifies gaps between what 3GPP has standardised (Rel-18/19 DCI reform)
and what production hardware actually needs.

---

## File Summary

| File | Size | Topic | Interactive? |
|---|---|---|---|
| `ka_link_budget.html` | ~36 KB | Ka LEO link budget, DPD/Shaper, MCS, PA power | ✓ Live sliders, presets |
| `ka_pnt_acquisition.html` | ~43 KB | GNSS-free acquisition, M1–M5, CW simulation | ✓ Live simulation, canvas |
| `6g_du_rd_ai.html` | ~390 KB | 5G/6G DU HW acceleration, SimPy Monte Carlo | ✓ Interactive simulator |

All tools are fully self-contained HTML — open in any modern browser, no server required.

---

## References

- Morgan & Humphreys, "HOOC-EM," ION GNSS+ 2024 (Amazon Kuiper funded)
- Kassas et al., Starlink Ku-band opportunistic PNT, ION GNSS+ 2024
- Iannucci & Humphreys, fused LEO GNSS <1.6% OH, IEEE TAES 2022
- Lynk Global, US20220216896A1 (filed Jan 2021) — wide-beam navigation signal patent
- DPD-NeuralEngine, 22 nm CMOS GRU-RNN DPD at 250 MSps, ISCAS 2025
- Starlink V2 Mini antenna panel area 11.07 m², arxiv 2306.06657
- Kuiper terminal apertures (18/28/50×78 cm), DSEI Sep 2025
- 3GPP TS 38.211/212/213, TR 38.843 (AI/ML for NR air interface)
- 3GPP Rel-18 DCI Format 1_3/0_3 (multi-CC scheduling)
- O-RAN Alliance WG4 fronthaul interface specifications
