# RTL-to-GDSII Physical Design — Memory Block (90nm)

A complete RTL-to-GDSII implementation of a **synchronous memory design** using **Cadence Innovus 21.15** on a 90nm process node. This project covers the full physical design flow — from synthesized netlist import through floorplanning, power planning, placement, clock tree synthesis (CTS), routing, and final GDSII generation — with clean timing closure and zero DRC/PG/connectivity violations.

> **Tool:** Cadence Innovus Implementation System 21.15-s110_1  
> **Process:** 90nm | **Design:** `memory` | **Corner:** PVT_0P9V_125C (worst-case)  
> **OS:** Linux x86_64

---

## Flow Overview

```
RTL Netlist (memory.v)
        │
        ▼
   Synthesis (Cadence Genus)
        │  → Gate-level netlist
        ▼
   Floorplanning
        │  → Core area definition, I/O pin placement
        ▼
   Power Planning
        │  → VDD/VSS rings and stripes (power_plan_done.enc.dat)
        ▼
   Placement (GigaPlace)
        │  → Standard cell placement, timing-driven optimization
        ▼
   Clock Tree Synthesis (CTS)
        │  → Clock buffer insertion, skew minimization
        ▼
   Post-CTS Optimization
        │  → Hold fixing, timing closure
        ▼
   Routing (NanoRoute)
        │  → Global + detailed routing, DRV cleanup
        ▼
   Signoff Verification
        │  → verify_drc, verify_PG_short, verifyConnectivity
        ▼
   GDSII Export (final.gds)
```

---

## Key Results

| Metric | Value |
|---|---|
| **WNS (default/all)** | 0.927 ns ✅ |
| **TNS** | 0.000 ns ✅ |
| **Violating paths** | 0 ✅ |
| **Clock skew** | 0.015 ns ✅ |
| **Max clock latency** | 0.179 ns |
| **Hold violations (post-CTS)** | 0 ✅ |
| **DRC violations** | 0 ✅ |
| **PG short violations** | 0 ✅ |
| **Connectivity violations** | 0 ✅ |
| **Routing overflow** | 0.00% H, 0.00% V ✅ |
| **Cell density** | ~53–61% |
| **Max transition** | 0.280 ns |
| **Metal layers used** | Metal1–Metal6 |

---

## Stage-by-Stage Walkthrough

### 1. Design Setup & Import

- Loaded synthesized gate-level netlist, LEF/DEF libraries, and timing constraints (SDC)
- Operating condition: `PVT_0P9V_125C` (worst-case slow-slow corner, 0.9V, 125°C)
- Verified design properties using `report_design`

### 2. Floorplanning

- Defined core area and aspect ratio for the memory block
- Placed I/O pins along the die boundary (visible as yellow arrows in layout)
- Set utilization targets to maintain routing headroom

### 3. Power Planning

- Created VDD and VSS power rings around the core
- Added internal power stripes across the standard cell rows
- Verified power plan: `power_plan_done.enc.dat` generated with 0 warnings, 0 errors

### 4. Placement

- Used Cadence **GigaPlace** engine for timing-driven global placement
- Ran `place_opt_design` with multiple optimization passes (DrvOpt iterations)
- Pre-CTS timing after placement:

  | Mode | WNS (ns) | TNS (ns) | Violating Paths |
  |---|---|---|---|
  | all | 0.804 | 0.000 | 0 |
  | reg2reg | 2.259 | 0.000 | 0 |
  | default | 0.804 | 0.000 | 0 |

- Density: 53.25% | Routing overflow: 0.00%
- Pre-CTS timing reports saved to `./reports/placement/`

### 5. Clock Tree Synthesis (CTS)

- Defined CTS spec for clock `mclk` using `cts.tcl`
- Inserted clock buffers (`CLKBUFX6LVT`, `CLKBUFX2LVT`, `CLKBUFX3LVT`) across the clock tree
- Clock tree debugger view showed a clean balanced 2-level buffer tree

**CTS Results:**

| Metric | Value |
|---|---|
| Clock skew | **0.015 ns** |
| Max latency | **0.179 ns** (at `mem_reg[11][2]/CK`) |
| Clock net | `mclk` |
| Analysis view | `func_ss` |

- Worst clock path: `clk_i → CTS_ccl_a_buf_00037 → CTS_ccl_a_buf_00006 → mem_reg[11][2]`

### 6. Post-CTS Optimization

- Ran `optDesign -postCTS` (setup) and `optDesign -postCTS -hold` (hold)
- Fixed hold violations — reduced from 3 violating paths to 0
- Post-CTS timing:

  | Mode | WNS Setup (ns) | Hold Violations |
  |---|---|---|
  | all | 1.407 | 0 |
  | reg2reg | 2.366 | 0 |

### 7. Routing

- Used Cadence **NanoRoute** for global and detailed routing
- Ran `routeDesign` followed by post-route optimization (`optDesign -postRoute`)
- All 2664 un-routed nets resolved
- Post-route verification:
  - `verify_drc -limit 0` → **0 violations**
  - `verify_PG_short` → **0 short violations**
  - `verifyConnectivity` → **0 violations, 0 warnings**

### 8. GDSII Generation

- Exported final layout as `final.gds` to `~/dakshaa/pnr/outputs/`
- Design boundary: (0.0, 0.0) to (83.6, 82.08) µm

---

## Repository Structure

```
rtl-to-gdsii-cadence-innovus/
├── README.md
├── scripts/
│   ├── floorplan_new.tcl       # Floorplan and I/O constraints
│   ├── placement.tcl           # Placement script
│   ├── cts.tcl                 # CTS spec and execution
│   └── route.tcl               # Routing script
├── reports/
│   ├── placement/
│   │   ├── preCTS.summary.gz   # Pre-CTS timing summary
│   │   ├── preCTS.fanout.gz    # Fanout report
│   │   └── preCTS_reg2reg.tarpt.gz  # Reg-to-reg timing paths
│   ├── latency.txt             # Clock latency report
│   └── skew.txt                # Clock skew report
└── screenshots/
    ├── 01_floorplan_powerplan.png
    ├── 02_placement_global.png
    ├── 03_placement_zoomed.png
    ├── 04_post_cts_layout.png
    ├── 05_clock_tree_debugger.png
    ├── 06_routed_layout.png
    ├── 07_routed_zoomed.png
    ├── 08_drc_pg_connectivity_clean.png
    └── 09_gdsii_output.png
```

---

## Screenshots

### Floorplan + Power Plan
After floorplanning and power ring/stripe creation. Blue horizontal lines are metal routing tracks; the orange ring is the power boundary.

![Floorplan](screenshots/01_floorplan_powerplan.png)

### Placement (Global View)
Standard cells placed across the core area after `place_opt_design`. Density ~53%.

![Placement Global](screenshots/02_placement_global.png)

### Placement (Zoomed)
Zoomed view showing individual standard cell instances (`mem_reg`, `rdata_o_reg`, combinational logic).

![Placement Zoomed](screenshots/03_placement_zoomed.png)

### Post-CTS Layout
Layout after clock tree synthesis showing clock buffer (`CTS_ccl_a_buf`) instances inserted by Innovus.

![Post-CTS Layout](screenshots/04_post_cts_layout.png)

### Clock Tree Debugger
Balanced 2-level clock tree — root buffer fans out to all leaf flip-flops. Skew: 0.015 ns.

![Clock Tree](screenshots/05_clock_tree_debugger.png)

### Routed Layout (Global)
Fully routed design with all six metal layers visible (Metal1–Metal6, Via1–Via5).

![Routed Global](screenshots/06_routed_layout.png)

### Routed Layout (Zoomed)
Detailed view of routing showing Metal1 (horizontal) and Metal2 (vertical) signal routing.

![Routed Zoomed](screenshots/07_routed_zoomed.png)

### DRC / PG / Connectivity Clean
`verify_drc`: 0 violations | `verify_PG_short`: 0 shorts | `verifyConnectivity`: 0 violations.

![Verification](screenshots/08_drc_pg_connectivity_clean.png)

### GDSII Output
Final `final.gds` file generated in `~/dakshaa/pnr/outputs/`. Binary GDSII stream viewed in text editor showing cell geometry data.

![GDSII](screenshots/09_gdsii_output.png)

---

## Tools & Technologies

| Category | Tool/Technology |
|---|---|
| Implementation | Cadence Innovus 21.15-s110_1 |
| Synthesis | Cadence Genus |
| Scripting | Tcl, Linux bash |
| Process node | 90nm |
| Libraries | Standard cell LEF/LIB (LVT variants: DFFQXLLVT, CLKBUFX6LVT, etc.) |
| Timing analysis | MMMC, func_ss (setup) / func_ff (hold) |
| Sign-off checks | verify_drc, verify_PG_short, verifyConnectivity |

---

## What I Learned

- Full RTL-to-GDSII flow execution in an industry-standard EDA tool (Cadence Innovus)
- Power planning methodology: ring + stripe topology for standard cell power delivery
- Reading and interpreting `timeDesign` timing reports (WNS, TNS, violating paths)
- Clock tree synthesis: buffer insertion, skew targets, latency analysis using `report_clock_timing`
- Hold fixing post-CTS using `optDesign -postCTS -hold`
- Physical verification: DRC, PG connectivity, net connectivity checks
- Tcl scripting for automating PD flow stages
- Linux command-line workflow for EDA tool operation

---

## About

Built as part of a hands-on Physical Design training at **NMIT Bengaluru** (Electronics — VLSI Design and Technology).  
Part of an ongoing portfolio targeting **ASIC Physical Design** roles in the semiconductor industry.

---

*Topics: `vlsi` `physical-design` `cadence-innovus` `eda` `rtl-to-gdsii` `semiconductor` `asic` `floorplanning` `cts` `routing` `gdsii`*
