# DVFS-re-modeling-in-VisualSim-Hackathon-2026

# Heterogeneous Multi-Core DVFS Architecture (Big.LITTLE) in VisualSim Architect

[![VisualSim Hackathon](https://img.shields.io/badge/VisualSim_Hackathon-Challenge_2-blue.svg)](https://www.mirabilisdesign.com/)
[![Architecture](https://img.shields.io/badge/Architecture-2_Big_+_2_Little_Hybrid-brightgreen.svg)]()
[![Model](https://img.shields.io/badge/Model-Multi--Core_DVFS-orange.svg)]()

> **Submission for VisualSim Global Electronics Hackathon — Challenge 2: Re-Engineer and Improve an Existing VisualSim Model**  
> Selected Model: **Multi-Core DVFS Design**

---

##  Executive Summary

Modern mobile, automotive, and edge processors cannot rely on homogeneous multi-core configurations where all cores run at identical frequencies, voltages, and power profiles. 

In this project, we re-engineered the default **Multi-Core DVFS Design** VisualSim model from an idealized **4-core homogeneous architecture** into an **asymmetric 2+2 Big.LITTLE heterogeneous architecture**:
- **2 High-Performance Big Cores (P-Cores):** $2.5\times$ clock speed, $2500\text{ MHz}$, $1.2\text{ V}$, $3.5\text{ mW}$ active power.
- **2 Energy-Efficient Little Cores (E-Cores):** $1.0\times$ baseline clock speed, $1000\text{ MHz}$, $0.8\text{ V}$, $1.0\text{ mW}$ active power.
- **Asymmetric Power Management:** Refactored `PowerTable2` with dedicated power states (`BigActPwr`, `LittleActPwr`, `BigStbyPwr`, `LittleStbyPwr`).
- **Dynamic Speed Dispatching:** Upgraded the task dispatcher script inside `Threads_and_Cores` to assign per-core processing rates dynamically.

---

##  Key Improvements & Results

| Metric | Original Baseline Model | Re-Engineered Hybrid (Big.LITTLE) | Engineering Impact |
|---|---|---|---|
| **Core Architecture** | 4 Identical Cores (Symmetric) | 2 Big (P-Core) + 2 Little (E-Core) | Matches modern ARM/Intel mobile SoCs |
| **Clock Frequencies** | Uniform ($1000\text{ MHz}$) | $2500\text{ MHz}$ (Big) / $1000\text{ MHz}$ (Little) | High-performance compute on critical paths |
| **Supply Voltage** | Single rail ($1.0\text{ V}$) | Dual domain ($1.2\text{ V}$ Big / $0.8\text{ V}$ Little) | Significant leakage & dynamic energy savings |
| **Peak Task Latency** | $\mathbf{\sim 16.0 \times 10^{-6}\text{ s}}$ ($16\,\mu\text{s}$) | $\mathbf{\sim 6.5 \times 10^{-6}\text{ s}}$ ($6.5\,\mu\text{s}$) |  **$\sim 2.5\times$ Latency Reduction** |
| **Power Signature** | Uniform steps ($2.5\,\text{mW}$ steps) | Differentiated steps ($3.5\,\text{mW}, 4.5\,\text{mW}, 7.0\,\text{mW}, 8.0\,\text{mW}$) | Granular power visibility per core type |
| **Thermal Profile** | Uniform heating bursts | Localized thermal headroom ($40.0^\circ\text{C} - 40.11^\circ\text{C}$) | Prevents premature thermal runaway |

---

##  Architecture & Model Changes

### 1. Top-Level Parameter Hierarchy
Added top-level architectural parameters to `DigitalSimulator` / `Threads_and_Cores`:
* `Big_Core_Speed = Core_Power * 2.5`
* `Little_Core_Speed = Core_Power * 1.0`
* `BigActPwr = Core_Power * 1.4 / 100.0` ($3.5\text{ mW}$ active)
* `LittleActPwr = Core_Power * 0.4 / 100.0` ($1.0\text{ mW}$ active)
* `BigStbyPwr = 0.02`
* `LittleStbyPwr = 0.005`

### 2. Dispatcher Re-Engineering (`Threads_and_Cores.Script`)
Replaced the symmetric array allocation with index-specific processor speeds:
```text
// Original:
Proc_Speed = newArray(Num_Cores, Core_Speed)

// Re-Engineered:
Proc_Speed = newArray(Num_Cores, Core_Speed)
Proc_Speed(0) = Big_Core_Speed      /* Big Core 0 (P-Core) */
Proc_Speed(1) = Big_Core_Speed      /* Big Core 1 (P-Core) */
Proc_Speed(2) = Little_Core_Speed   /* Little Core 2 (E-Core) */
Proc_Speed(3) = Little_Core_Speed   /* Little Core 3 (E-Core) */
```

### 3. Hardware Resource Differentiation (`SystemResource`)
* `SystemResource` (Core 0) & `SystemResource2` (Core 1): `Clock_Rate_Mhz = Big_Core_Speed`
* `SystemResource3` (Core 2) & `SystemResource4` (Core 3): `Clock_Rate_Mhz = Little_Core_Speed`

### 4. Heterogeneous Power Manager (`PowerTable2`)
Updated `Manager_Setup` table to represent independent voltage rails and power states:
```text
Architecture_Block   Standby         Active         Wait  Idle  Existing  OffState  OnState  t_OnOff   Mhz     Volts ;
Scheduler_0          BigStbyPwr      BigActPwr      0.0   0.0   Standby   Standby   Active   100.0e-9  2500.0  1.2   ;
Scheduler_1          BigStbyPwr      BigActPwr      0.0   0.0   Standby   Standby   Active   100.0e-9  2500.0  1.2   ;
Scheduler_2          LittleStbyPwr   LittleActPwr   0.0   0.0   Standby   Standby   Active   100.0e-9  1000.0  0.8   ;
Scheduler_3          LittleStbyPwr   LittleActPwr   0.0   0.0   Standby   Standby   Active   100.0e-9  1000.0  0.8   ;
```

---

## Simulation Results & Analysis

### 1. Latency (`Threads_and_Cores.Latency`)
* **Baseline:** Tasks took between $0.4\times 10^{-5}\text{ s}$ and $1.6\times 10^{-5}\text{ s}$ ($4 - 16\,\mu\text{s}$).
* **Hybrid Model:** Task latency dropped to $1.5\times 10^{-6}\text{ s} - 6.5\times 10^{-6}\text{ s}$ ($1.5 - 6.5\,\mu\text{s}$). High-priority thread execution benefited directly from the $2.5\times$ compute frequency on the Big cores.

### 2. Instantaneous & Average Power (`Instantaneous_Average_Power`)
* **Baseline:** Each active core rigidly pulled $2.5\text{ mW}$ regardless of workload, stepping to $5.0\text{ mW}$ and $7.5\text{ mW}$.
* **Hybrid Model:** Distinct power tiers appear in the waveform:
  - Single Big Core Active: $3.5\text{ mW}$
  - Big Core + Little Core Active: $4.5\text{ mW}$
  - 2 Big Cores Active: $7.0\text{ mW}$
  - 2 Big Cores + 1 Little Core Active: $8.0\text{ mW}$

### 3. Thermal Modeling (`Temperature`)
* The thermal subsystem (`MyChipTherm`) receives the aggregated instantaneous power dissipated across the die at $40.0^\circ\text{C}$ ambient temperature. The hybrid architecture keeps temperature spikes controlled while completing bursts significantly faster.

---

## How to Open and Run in VisualSim Architect

1. Launch **VisualSim Architect**.
2. Go to **File $\rightarrow$ Open** and select the model file:
   ```
   Multi_Core_DVFS_Hybrid_BigLittle.xml
   ```
3. To view or adjust core ratios and clock rates:
   - Double-click the green **`Threads_and_Cores`** block to inspect `Big_Core_Speed` and `Little_Core_Speed`.
   - Double-click the red **`PowerTable2`** block to inspect power values in `Manager_Setup`.
4. Click the green **Run** button on the top toolbar.
5. The 4 analytical plot windows will automatically open:
   - `Waveform_Plot`: Shows task execution across cores.
   - `Latency`: Displays task processing delays over simulation time.
   - `Instantaneous_Average_Power`: Displays power consumption profile.
   - `Temperature`: Plots thermal response via `MyChipTherm`.

---



---

##  Overall:

- **Improved Modeling Fidelity:** Replaced an unrealistic homogeneous 4-core queue with a production-grade big.LITTLE SoC topology.
- **New Engineering Insight:** Quantified latency speedup vs. power draw across asymmetric processor cores.
- **Modeling Efficiency & Reuse:** Maintained model stability and parameterization without increasing block count.
- **Design-Space Exploration:** Enabled rapid exploration of Big/Little clock multipliers and power settings via top-level parameters.

---
*Developed for the VisualSim Global Electronics Hackathon.*
