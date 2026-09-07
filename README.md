# 🌐 AI for Adaptive Networks: Multi-Layer Fault Diagnosis Engine

[![Python Version](https://img.shields.io/badge/Python-3.9%2B-blue.svg)](https://www.python.org/)
[![Domain](https://img.shields.io/badge/Domain-DWDM%20%7C%20IP%20Routing%20%7C%20Log%20Analytics-green.svg)](#architecture)
[![Status](https://img.shields.io/badge/Phase-Data%20Prep%20%26%20EDA-orange.svg)](#roadmap)
[![License](https://img.shields.io/badge/License-MIT-purple.svg)](LICENSE)

---

## 📌 Executive Summary

**AI for Adaptive Networks** is an intelligent, automated fault-diagnosis and root-cause localization platform designed for long-haul transport networks combining **DWDM (Dense Wavelength Division Multiplexing)** optical transport and **IP routing layers**.

When an anomaly or physical failure occurs on a high-speed fiber path (e.g., fiber cut, optical degradation, line-card crash, or link flapping), alarms propagate almost instantaneously across multiple network layers. Dashboard alerts frequently surface at higher layers (e.g., IP Interface or LAG level) while the root failure originated deep within the optical transport span. 

This repository implements the multi-layer correlation methodology, identifier translation engine, and timeline-alignment pipeline required to process multi-layer logs automatically, answer **WHERE** a fault began and **WHY** it happened, and eliminate manual root-cause investigation.

---

## 🎯 What the System Must Do — The Two Core Tasks

Every network incident evaluated by the diagnostic system is mapped to two fundamental questions:

| Task | Core Question | Definition |
| :--- | :--- | :--- |
| **Task 1** | **WHERE did the fault originate?** | Pinpoints the precise physical layer, node, and span where the failure started — distinguishing the true source from downstream symptom alarms. |
| **Task 2** | **WHY did it happen?** | Identifies the root cause: fiber cut, optical SNR degradation, router chassis failure, power/environmental trip, or misconfiguration. |

```
                              ┌────────────────────────┐
                              │  Observed Alarm Burst  │
                              └───────────┬────────────┘
                                          │
                                          ▼
                      ┌───────────────────────────────────────┐
                      │  Timeline & Layer Topology Alignment  │
                      └───────────────────┬───────────────────┘
                                          │
                   ┌──────────────────────┴──────────────────────┐
                   ▼                                             ▼
        ┌─────────────────────┐                       ┌─────────────────────┐
        │       TASK 1        │                       │       TASK 2        │
        │ Fault Origin (WHERE)│                       │ Root Cause (WHY)    │
        └─────────────────────┘                       └─────────────────────┘
```

---

## 🏗️ Architecture — Network & Layer Stack

The platform models the network across three primary structural views:

```
[ Router Endpoint A ] <== DWDM Mux/Demux ==> [ Intermediate Optical Relays ] <== DWDM Mux/Demux ==> [ Router Endpoint B ]
  ├── Logical Interface                         (Amplifiers / ROADMs)                                 ├── Logical Interface
  ├── Link Aggregation (LAG)                                                                          ├── Link Aggregation (LAG)
  ├── Physical Port                                                                                   ├── Physical Port
  └── DWDM Transponder                                                                                └── DWDM Transponder
```

### 1. End-to-End Physical Path
Traffic originates from routed endpoints, descends through the router's internal layer stack, converts to optical signals via DWDM transponders, and traverses multiple intermediate optical amplifier / relay (ROADM) stations before re-ascending the stack at the remote endpoint.

### 2. DWDM Optical Node Decomposition
Each optical site maintains two primary signal paths:
- **Receive Path**: Pre-amplifier $\rightarrow$ Wavelength Selective Switch (WSS / ROADM) $\rightarrow$ Channel Demultiplexer $\rightarrow$ Router Drop.
- **Transmit Path**: Channel Multiplexer $\rightarrow$ Dispersion Conditioning Module $\rightarrow$ Boost Amplifier $\rightarrow$ Long-haul Fiber Span.
- **Optical Supervisory Channel (OSC)**: Out-of-band management channel providing real-time node state to the central EMS/NMS log collector.

### 3. Router Internal Layer Stack
Within each router endpoint, log data is produced independently across four distinct layers:

```
┌─────────────────────────────────────────────────────────┐  ▲ Fault Detection
│ 4. Interface Sheet  (Logical IP interfaces, OSPF/BGP)   │  │ Propagates UPWARDS
├─────────────────────────────────────────────────────────┤  │ (Symptom Surfacing)
│ 3. LAG Sheet        (Link Aggregation Group bundles)    │  │
├─────────────────────────────────────────────────────────┤  │
│ 2. Port Sheet       (Physical Ethernet / Optics interfaces)│
├─────────────────────────────────────────────────────────┤  │
│ 1. DWDM Sheet       (Optical transponders, lambda paths)│  │ Data Flows DOWNWARDS
└─────────────────────────────────────────────────────────┘  
```

---

## 🧠 Diagnostic Methodology

The engine synthesizes multi-layer alarm streams into structured diagnostic decisions using two core principles:

### 1. The Bottom-Up (Earliest-and-Lowest) Principle
* **Asymmetry Rule**: Data flows **DOWN** the stack ($\text{Interface} \rightarrow \text{LAG} \rightarrow \text{Port} \rightarrow \text{DWDM} \rightarrow \text{Fibre}$), but fault detection propagates **UPWARDS**.
* **Ordering Logic**: A lower-layer failure (e.g., optical loss of signal) causes dependent upper layers to fail seconds or milliseconds later. By sorting all alarms chronologically across all layers, the alarm that fires **earliest in time** and sits **lowest in the stack** is identified as the fault origin.

### 2. The Silent-Layer Principle
* **Isolation Rule**: Layers that remain completely silent during an incident provide explicit boundaries regarding fault location:
  * **DWDM Silent + Upper Layers Firing**: Fault is strictly contained within the router/chassis/interface layer (not the optical transport fiber).
  * **DWDM Firing Alone**: Optical degradation or fiber issue contained below the router (packets/upper layers unaffected).

---

## 📚 Catalogues & Correlation Rules

The platform compiles three structured reference knowledge bases:

### 1. Alarm-Code Catalogue
Catalogues every alarm code across device classes (Nokia 7750 IP Routers, Ciena 6500 DWDM equipment) with plain-language definitions and mapped fault scenarios (e.g., AIS, Loss of Signal, FEC Degrade, Transponder Fail).

### 2. Fault-Scenario Catalogue
Organizes network failures into four primary scenario classes:

```
┌───────────────────────────────────────────────────────────────────────────┐
│ Group A: Optical Origin (Fiber cuts, optical power drop, WSS failure)    │
│ Group B: IP / Router Origin (Chassis failure, port flap, MTU mismatch)   │
│ Group C: Compound / Environmental (Power trip, cooling failure)          │
│ Group D: Partial Propagation (Sub-threshold degradations, transient drops) │
└───────────────────────────────────────────────────────────────────────────┘
```

### 3. Correlation Rules Matrix
Maps observed firing signatures $(\text{Fired Layers}, \text{Timestamp Order}, \text{Silent Layers}) \rightarrow (\text{Fault Origin}, \text{Root Cause}, \text{Recommended Action})$.

---

## 🛠️ Automated Code Translation Engine

Raw EMS log sheets arrive using legacy device identifier codes ("Army Format"). The translation framework converts all site names, system names, CLFIs, components, and IPs into standardized working conventions ("Navyug Format") prior to timeline unification.

### Scripts Included in Repository (`scripts/`):

1. **`scripts/router_site_mapping.py`**:
   - **4-Part Site Name Regex Splice**: Extracts `<Command><Station><RouterCode><RouterType><Number>`. Handles variable-length router types dynamically.
   - **Object Name Lookup**: Performs whole-cell standard dictionary lookups.
   - **Odometer Site ID IP Mapping**: Maps router site IDs into deterministic, anonymized IP subnets (`192.168.X.Y`) per command band.

2. **`scripts/dwdm_mapping.py`**:
   - **System Name 4-Part Splice**: Splices fixed-width System Name strings (`WC|DEL|TXN|CNA|002`) into Command, Station, Legend, Hardware, and Digits.
   - **CLFI & Component Translation**: Direct dictionary lookups for optical circuit identifiers and card components.
   - **Circle IP Re-assignment**: Re-assigns circle-specific IP bands (`193.167.X.Y`) deterministically.

---

## 📂 Repository Structure

```
.
├── AI_Adaptive_Networks_Documentation.docx  # Complete technical documentation docx
├── scripts/
│   ├── router_site_mapping.py               # Router & Site Name code translation script
│   └── dwdm_mapping.py                      # DWDM system & component translation script
├── .gitignore                               # Git ignore rules for datasets & python artifacts
└── README.md                                # Project documentation & architecture overview
```

---

## 🚀 Quickstart & Usage Guide

### Prerequisites
- **Python 3.9+**
- Required Python libraries: `pandas`, `openpyxl`

```bash
pip install pandas openpyxl
```

### Running Router Log Translation

```bash
python scripts/router_site_mapping.py \
  "path/to/TRANSLATION_CODES.xlsx" \
  "path/to/raw_router_alarms.xlsx" \
  "path/to/translated_router_alarms.xlsx"
```

### Running DWDM Log Translation

```bash
python scripts/dwdm_mapping.py \
  "path/to/TRANSLATION_CODES.xlsx" \
  "path/to/raw_dwdm_alarms.xlsx" \
  "path/to/translated_dwdm_alarms.xlsx"
```

---

## 🗺️ Project Roadmap & Current Status

- [x] **Architecture Reconstruction**: Full end-to-end DWDM + IP topology documented.
- [x] **Diagnostic Method Definition**: Bottom-up and silent-layer principles formalized.
- [x] **Reference Catalogues**: Alarm codes, fault scenarios (Groups A-D), and correlation rules compiled.
- [x] **Translation Engine**: Python scripts written and verified for router & DWDM identifier translation.
- [/] **Data Unification & Timeline Alignment**: Applying UTC-ms resolution timestamps and shared topology keys across all log sheets.
- [ ] **Exploratory Data Analysis (EDA)**: Validating catalogued fault signatures against live production log data.
- [ ] **Automated AI Classifier**: Training automated Machine Learning / Rule-based inference models for real-time incident diagnosis.

---

## 📄 Document & Confidentiality Notice

The complete underlying technical architecture documentation is available in [`AI_Adaptive_Networks_Documentation.docx`](file:///Users/sanskarranwa/Desktop/Navyug/Ai%20For%20Adaptive%20Networks/AI_Adaptive_Networks_Documentation.docx). Sensitive IP addresses, customer site mappings, and live proprietary logs have been omitted or anonymized in accordance with network security standards.

---

## 📜 License

Distributed under the MIT License. See `LICENSE` for more information.
