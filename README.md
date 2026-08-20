# ARM EL3 Memory Fault Detector & Predictive Failure Analysis (PFA) Tool

## Overview
This tool provides real-time monitoring of ARM EL3 hardware errors, mapping ECC memory faults to physical DIMM topologies for rapid diagnostics. It intercepts DRAM ECC events and decodes complex syndrome data into actionable Predictive Failure Analysis (PFA) alerts.

The project is heavily inspired by large-scale DRAM failure studies in data centers, including:
- *Predicting Memory Failures in Data Centers (USENIX ATC '19)*
- *DRAM errors in the wild: a large-scale field study*
- *DRAM failure prediction in AIOps: Empirical evaluation challenge*

By analyzing error rates, persistence, and affected DQ lanes, the tool categorizes DRAM errors and predicts impending Uncorrectable Errors (UCE) or hardware faults before they cause system downtime.

## Key Features
- **Real-Time EL3 Log Parsing**: Continuously monitors and parses DRAM ECC events from EL3 hardware logs.
- **Physical Topology Mapping**: Maps logical addresses and errors to physical topology, including Socket, Channel, SubChannel, DIMM, Rank, CID, Bank Group, Bank, Row, and Column.
- **DQ Signal Analysis**: Identifies Active DQ Errors and specific affected DQ lanes using `phy_lanes_bitmask`.
- **Predictive Failure Analysis (PFA)**:
  - **Single Device/Chip Fault**: Detects multi-bit CE storms often preceding a complete chip failure.
  - **Link/RCD Interface Signal Noise**: Flags signal integrity issues (PreReplay errors).
  - **Hard Stuck-at Faults**: Uses persistence ratio (`persist_cnt` / `correctable_cnt`) to detect permanent cell damage and recommends page offlining.
  - **CE Storms & UCE Risk**: Alerts on high CE counts indicating an imminent Uncorrectable Error (UCE) risk.
  - **Patrol Scrubbing/Sparing**: Differentiates standard memory scrubbing corrections from active runtime errors.

## Error Categorizations
Based on the empirical findings from AIOps and large-scale data center studies, the tool implements the following failure signatures:
1. `CRITICAL UCE`: Uncorrectable Error detected.
2. `[PFA: Single Device/Chip Fault (Critical)]`: ≥ 4 Active DQ lanes affected simultaneously.
3. `[PFA: Link/RCD Interface Signal Noise]`: Transient interface noise detected (`pre_replay`).
4. `[PFA: Hard Stuck-at Fault (Action: Page Offline)]`: High correctable error count with ≥ 80% persistence ratio.
5. `[PFA: CE Storm / Imminent UCE Risk]`: Correctable error burst (≥ 1,000 errors).

## Usage
Run the tool directly on an ARM-based server. It automatically fetches DIMM topology (`--dimm-info`) and begins monitoring EL3 logs.

```bash
go run main.go
```

By default, the tool invokes the underlying `arm_tool` to fetch and tail system logs (`--log -r --type el3`).
