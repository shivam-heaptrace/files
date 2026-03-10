# EWX Power / UKKA Platform Rebuild — Comprehensive Training Course

**Prepared by:** DevSavant Product Engineering
**Date:** 2026-03-10
**Source Documents:** Dashboard Requirements.docx, Dashboard Deliverables.docx, Requirements - ETL.docx, Data Definition.xlsx, Sample Data - Meta Data.xlsx, Sample Data - Data Series.xlsx, Multi Inverter Solark 12k Scenario.xlsx, Multi Inverter Solark 15k Scenario.xlsx, Dashboard Data Flow.vsdx, Rails API Codebase Audit, Baseline Technical Inventory
**Cross-references:** [Rails API Codebase Audit](rails-api-codebase-audit.md) | [Baseline Technical Inventory](baseline-technical-inventory.md) | [Platform Rebuild README](../README.md)

---

## Table of Contents

1. [Module 1: Project Context & Business Background](#module-1-project-context--business-background)
2. [Module 2: The Energy Domain — Understanding the Physical System](#module-2-the-energy-domain--understanding-the-physical-system)
3. [Module 3: External Data Sources & the ETL Pipeline](#module-3-external-data-sources--the-etl-pipeline)
4. [Module 4: The Dashboard UI — Every Screen Explained](#module-4-the-dashboard-ui--every-screen-explained)
5. [Module 5: The Data Model Deep Dive](#module-5-the-data-model-deep-dive)
6. [Module 6: Architecture & Non-Functional Requirements](#module-6-architecture--non-functional-requirements)
7. [Module 7: Open Questions, Risks & Knowledge Gaps](#module-7-open-questions-risks--knowledge-gaps)
8. [Assessment Quiz](#assessment-quiz)

---

## Module 1: Project Context & Business Background

### Who Is EWX Field Services?

EWX Field Services, LLC (branded as "EWX Power" at ewxpower.net) is an operator and field service provider that deploys, monitors, and maintains microgrid energy systems. Their locations are typically remote sites — oil well pads in rural Pennsylvania, ranches in South Texas — where there is no connection to the main utility grid. Each site combines solar panels, battery banks, diesel/gas generators, and Sol-Ark hybrid inverters into a self-contained power system.

EWX's business model centers on monitoring these distributed energy assets across multiple client companies (tenants). They need a software platform that gives operators real-time visibility into every location's energy balance: how much solar is being produced, whether the generator is running, what the battery state of charge is, and whether any equipment is faulting.

### Who Is Martin Janda?

Martin Janda is the technical lead / CTO at EWX Field Services. He is the sole author of every requirements document in the client-docs directory:

- Dashboard Requirements.docx (6.9 MB, 100+ pages of detailed screen specifications)
- Dashboard Deliverables.docx (project scope and deliverables)
- Requirements - ETL.docx (external data sources and ETL pipeline)
- Data Definition.xlsx (30+ sheets defining every data source)
- Sample Data - Meta Data.xlsx and Sample Data - Data Series.xlsx (real Tophole configuration and telemetry)
- Multi Inverter Solark 12k Scenario.xlsx and 15k Scenario.xlsx (Moran Ranch and Tophole deployment scenarios)

Martin is both the domain expert and the system architect. His documents reveal a deep understanding of both the energy domain and software engineering. He designed the entire data model, the ETL pipeline, the event-driven architecture, and the concurrency model. He also wrote the 19 outstanding questions that honestly acknowledge where design decisions are incomplete.

### What Happened with the Previous Development Team?

The current codebase was built by a prior development team. Based on the documentation and codebase audit, it's clear that their work was unsatisfactory:

- The codebase audit gave the overall system a grade of **D** — "Not production-ready"
- Authentication is disabled in production (the password check is commented out)
- Multi-tenant isolation is broken (the member? check doesn't scope to the requested tenant)
- Test coverage is approximately 3-5% (17 test files for 350+ application files)
- The ETL pipeline fails silently and has no retry logic
- Security vulnerabilities include hardcoded credentials, plaintext API secrets, and no CORS configuration

The root cause analysis in the audit identifies **technical competence gaps** compounded by **absent process discipline** — the team prioritized surface-level feature delivery over correctness, security, and testing.

### The Existing Tech Stack

| Component | Technology | Version |
|-----------|-----------|---------|
| Backend Framework | Ruby on Rails (API-only) | 8.0.4 |
| Language | Ruby | 3.2.2 |
| Database | MySQL | 8.x |
| Frontend Framework | Next.js | 15.4.2-canary.34 |
| UI Library | React | 19.1.1 |
| Background Jobs | Solid Queue | 1.2.4 |
| Deployment | AWS ECS Fargate via CDK | -- |
| Container Registry | AWS ECR | -- |
| File Storage | AWS S3 (Active Storage) | -- |
| IoT Platform | AWS IoT Greengrass | -- |
| External Data Source | Sol-Ark API (mysolark.com) | -- |
| Authentication (Legacy) | JWT via bcrypt + custom TokenManager | -- |
| Authentication (New) | Better Auth (JS library) | -- |
| Error Tracking | Sentry | -- |
| Serialization | Blueprinter | -- |

### What EWX Expects from DevSavant

The engagement is structured in two phases:

**Phase 1: Evaluation & Scoping**
- Audit the existing codebase (Rails backend and Next.js frontend)
- Assess the keep-vs-rebuild decision
- Produce a detailed project evaluation and estimate
- Deliver a Phase 1 Discovery report

**Phase 2: Rebuild**
The Dashboard Deliverables document specifies:

1. **GUI Model** — A working front-end demonstrating screen navigation, card configuration, responsive behavior, and all screen flows
2. **Technical Design / Proof of Concept** — Demonstrating:
   - Multi-tenant security working correctly
   - End-to-end ETL flow (Sol-Ark API → Staging → Component.Statistics → Location.Statistics → Company.Statistics)
   - Real-time subscriptions (WebSocket/SSE)
3. **Revised SDLC** — CI/CD pipeline, automated testing framework, deployment/rollback procedures

### Engagement History

Based on document revision dates and file metadata:

| Date | Milestone |
|------|-----------|
| November 6, 2024 | Martin publishes initial Dashboard Requirements |
| Late 2024 - Early 2025 | Previous development team builds initial codebase |
| October 2024 - February 2026 | 190 database migrations span this period |
| March 9, 2026 | Client documents provided to DevSavant |
| March 10, 2026 | DevSavant produces codebase audit and baseline inventory |

### Key Takeaway for Phase 1 Evaluation

This is not a simple "take over existing code" engagement. The existing codebase has 44 findings across every layer — 10 Critical, 13 High. The strategic recommendation from the audit is to **lean toward rebuild with domain model preservation**. The remediation cost (270-370h) is 35-45% of a full rebuild (640-800h) but carries higher risk because it preserves the architectural decisions that produced the problems.

The domain model (79 models, 63 database tables, entity relationships) is the most valuable asset and should be preserved regardless of the framework decision.

---

## Module 2: The Energy Domain — Understanding the Physical System

### What Is a Microgrid?

A microgrid is a self-contained energy system that can operate independently of the main electrical grid. In EWX's world, each **Location** is essentially a microgrid. Think of a remote oil well site in rural Pennsylvania or a ranch in South Texas — places where you can't just plug into the utility grid. These locations need to generate, store, and manage their own electricity.

Every EWX microgrid has the same fundamental challenge: **balance energy supply and demand in real time**. Solar panels produce power when the sun shines. Batteries store excess energy for later. Generators kick in when solar + battery can't meet the load. The Sol-Ark inverters sit at the center, routing power between all these sources and the loads that consume it.

The platform Martin designed exists to **monitor this balancing act** across every location a tenant (customer company) operates.

### The Component Types — What Each One Does Physically

#### 1. Solar Arrays (CategoryID: 1 — Solar)

A solar array in EWX is a **string of photovoltaic panels** wired in series. At the Tophole Pilot Unit, there are three solar arrays:

- **pv1** — 5x BlueSun BSM460M-72HBD panels (460W each, rated at 42.4 VDC max)
- **pv2** — 5x BlueSun BSM460M-72HBD panels (same)
- **pv3** — 5x BlueSun BSM460M-72HBD panels (same)

Each array's total theoretical output is 5 × 460W = **2,300W (2.3 kW)**. Across all three arrays, the Tophole location has **6,900W (6.9 kW)** of nameplate solar capacity.

The telemetry fields for solar come from the Sol-Ark inverter's PV inputs. Each inverter has up to 4 PV input channels, and the key measurements per channel are:

| Field | Measurement | Example |
|-------|-------------|---------|
| `Ppv1(W)/186` | PV Power (Watts) | 293W |
| `Vpv1(V)/109` | PV Voltage (VDC) | 171.8V |
| `Ipv1(A)/110` | PV Current (Amps) | 1.7A |

These panels connect to the inverter via **HVDC (High Voltage DC)** connections — specifically the PV1, PV2, and PV3 input ports on the Sol-Ark.

#### 2. Generators (CategoryID: 0 — Generation)

Generators are fuel-burning engines that produce AC electricity. At Tophole, there is one generator:

- **gen1** — Generac SP-250, rated at **150 kVA**, running at 1,800 RPM, diesel-fueled (FuelType: 0), with a 358-gallon fuel tank, configured for Three Phase 277/480 output (OutputType: 2)

The generator at Tophole has its `TelemetrySourceID` set to **-2**, meaning its telemetry comes from an **EWX Monitor** (not from the Sol-Ark API). However, in the Moran Ranch sample MetaData, the generator data series are configured to pull from the Sol-Ark inverters (`solark1` and `solark2`), specifically from the AC input fields like `AcL1Power(W)/167` and `AcL2Power(W)/168`. This is because when the generator feeds power *through* the Sol-Ark inverter, the inverter measures that power on its AC input port.

The EWX IoT data source provides much richer generator telemetry via the DSE controller:

| Field | What It Measures | Sample Value |
|-------|-----------------|--------------|
| `oilPressure` | Engine oil pressure (kPa) | 448 |
| `coolantTemperature` | Engine coolant temp (°C) | 90 |
| `fuelLevel` | Fuel tank level (%) | 50 |
| `engineSpeed` | RPM | 1802 |
| `voltage1Neutral` | L1 output voltage (VAC) | 272.6 |
| `current1` | L1 output current (A) | 76.0 |
| `watts1` | L1 power output (W) | 16,560 |
| `generatorRunTime` | Lifetime runtime (seconds) | 2,765,178 |
| `numberOfStarts` | Lifetime start count | 770 |
| `secondsToMaintenance` | Until next service | 988,781 |

#### 3. Battery Banks (CategoryID: 3 — Storage)

Battery banks store energy in chemical form. At Tophole:

- **bank1** — 10x Fortress Energy eVault Max 18.5 batteries in parallel
  - Each unit: 18.43 kWh stored energy, 46-56 VDC range, LFP chemistry
  - Total storage: 10 × 18.43 = **184.3 kWh**
  - Sustained power: **86,400W** (86.4 kW)
  - Max discharge current: 230A per unit
  - Expected discharge current: 180A per unit

The battery bank connects to the Sol-Ark inverter via the **DC (48 VDC)** connection port. The connection is **bidirectional** — power flows *into* the bank when charging and *out* when discharging. Key telemetry fields:

| Field | Measurement | Sample Value |
|-------|-------------|--------------|
| `BatteryPower(W)/190` | Charge/discharge power | 760W (charging) or -3,341W (discharging) |
| `BatteryVolt(V)/183` | Bank voltage | 52.31 VDC |
| `BatteryTemp(℃)/182` | Bank temperature | 33.7°C (converted to °F via `CtoFTemp`) |
| `BatteryCurrent(A)/191` | Charge/discharge current | 14.66A (charging) or -61.66A (discharging) |

Note the sign convention: **positive = charging, negative = discharging**. In the Tophole sample data, at 10:04 AM the battery was charging at 760W, but by 10:09 AM it was discharging at -3,341W — the solar production had shifted and the battery was supplying the load.

#### 4. Power Conditioning — Sol-Ark Inverters

This is the **heart** of every microgrid. The Sol-Ark inverter is a hybrid inverter/charger that:

- Converts DC from solar panels to AC for loads (DC/AC)
- Converts AC from the generator to DC for battery charging (AC/DC)
- Manages DC battery charging and discharging (DC/DC)
- Routes power between all sources and the load

At Tophole, there are **two** Sol-Ark 15k-208 inverters:

- **solark1** (SN: 2201064087) — master inverter, ExternalSourceID: 1, PlantID: 118529
- **solark2** (SN: 2201074043) — slave inverter, ExternalSourceID: 1, PlantID: 118529

The Sol-Ark 15k specs from the sample data:

| Attribute | Value |
|-----------|-------|
| Rated Power | 15 kVA |
| Burst Power | 24 kVA (for 10 seconds) |
| Output Type | Single phase 120/240 AND Three phase 120/208 |
| Conversion Efficiency | 96.5% (CEC) |
| Max Parallel Units | 12 |
| Supported Battery Types | Lead Acid, LFP |
| Solar Max Power | 19.5 kW |
| Solar Voltage Range | 150-500 VDC |
| Solar Starting Voltage | 125 VDC |
| MPPT Controllers | 3 units |
| Storage Voltage Range | 43-63 VDC |
| Storage Max Current | 275A |

Each Sol-Ark has **7 connection ports**:

| ConnectionID | PowerType | Flow Direction | Multiple Connections? |
|-------------|-----------|----------------|----------------------|
| DC | 48 VDC | Both (bidirectional) | Yes |
| Load | Three phase 120/208 | Out | Yes |
| AC | Three phase 120/208 | In | Yes |
| Generator | Three phase 120/208 | In | Yes |
| PV 1 | HVDC | In | Yes |
| PV 2 | HVDC | In | Yes |
| PV 3 | HVDC | In | Yes |

#### 5. Loads (CategoryID: 2 — Load)

The load represents everything that *consumes* power at the location. At Tophole:

- **AC1** — Rated at 80 kW, Three Phase 277/480 (PowerType: 2), TelemetrySourceID: -1 (from connection)

Load telemetry comes from the Sol-Ark's load output measurements:

| Field | Measurement | Sample Value |
|-------|-------------|--------------|
| `LoadL1Power(W)/176` | L1 load power | 586W |
| `LoadL2Power(W)/177` | L2 load power | 958W |
| `LoadL1(V)/157` | L1 load voltage | 120.0 VAC |
| `LoadL2(V)/158` | L2 load voltage | 120.0 VAC |
| `InvFac(W)/193` | Output frequency | 60.0 Hz |

#### 6. Sensors (CategoryID: 6 — Sensors)

Sensors measure environmental conditions. At Tophole:

- **temp1** and **temp2** — SAH XY-MD02 temperature sensors
  - Type: Temperature (SensorType: 0)
  - Range: -40°F to 140°F
  - Communication: Modbus (channel 1-247, baud 9600-19200)
  - TelemetrySourceID: -2 (from EWX Monitor)

These sensors connect to the EWX Monitor via a Modbus network, not to the Sol-Ark.

#### 7. EWX Monitors

EWX Monitors are custom IoT devices (running on AWS IoT Greengrass) deployed at locations to gather telemetry from equipment that doesn't have its own cloud API. At the Moran Ranch 12k scenario, the monitor **MRMonitor** (IoT Device ID: `0-87740-1`) is connected to a DSE74XX generator controller via Modbus, reading 55+ fields covering everything from oil pressure to DEF tank levels.

### Key Terms and the Data Hierarchy

#### The Definitional Chain

| Term | Definition | Example |
|------|-----------|---------|
| **Tenant** | A legal entity/company that has licensed the software. Identified by a numeric **CompanyID**. A tenant can ONLY see its own data. | CompanyID: 0 |
| **Location** | A physical (or virtual) site with interconnected components. Identified by a numeric **LocationID**, unique within a Company. | LocationID: 1, "Tophole - Pilot Unit" |
| **Component** | A piece of equipment at a location. Identified by a string **ComponentID**, unique across the location. | `pv1`, `gen1`, `bank1`, `solark1`, `AC1`, `temp1` |
| **Data Series** | A single time-series of measurements for one metric on one component. Identified by a numeric **SeriesID**, globally unique across ALL tenants. | SeriesID: 1 = PV Power for pv1 via solark1 |
| **Day** | Defined by the location's time zone. Tophole is America/Chicago. A "day" for Tophole starts and ends at midnight Central Time. | Summaries and "today" values use this boundary |

#### The Hierarchy: Company → Location → Component → Data Series

```
Company (CompanyID: 0)
├── Location: Tophole - Pilot Unit (LocationID: 1, Spraggs PA, America/Chicago)
│   ├── Component: pv1 (Solar Array, 5x BlueSun 460W)
│   │   ├── Series 1: PV Power (Ppv1(W)/186 from solark1)
│   │   ├── Series 2: PV Voltage (Vpv1(V)/109 from solark1)
│   │   └── Series 3: PV Current (Ipv1(A)/110 from solark1)
│   ├── Component: pv2 (Solar Array, 5x BlueSun 460W)
│   │   ├── Series 4: PV Power (Ppv2(W)/187 from solark1)
│   │   ├── Series 5: PV Voltage (Vpv2(V)/111 from solark1)
│   │   └── Series 6: PV Current (Ipv2(A)/112 from solark1)
│   ├── Component: pv3 (Solar Array, 5x BlueSun 460W)
│   │   ├── Series 7: PV Power (Ppv3(W)/188 from solark1)
│   │   ├── Series 8: PV Voltage (Vpv3(V)/113 from solark1)
│   │   └── Series 9: PV Current (Ipv3(A)/114 from solark1)
│   ├── Component: gen1 (Generac SP-250, 150 kVA)
│   │   ├── Series 10: Gen L1 Power (AcL1Power(W)/167 from solark1)
│   │   ├── Series 11: Gen L2 Power (AcL2Power(W)/168 from solark1)
│   │   ├── Series 13-17, 22: Voltage, Current, Frequency from solark1
│   │   ├── Series 23-29: Same measurements from solark2
│   │   └── (14 total series for gen1 across both inverters)
│   ├── Component: bank1 (10x Fortress eVault, 184.3 kWh)
│   │   ├── Series 30-32: Power, Voltage, Temp from solark1
│   │   ├── Series 33-35: Same from solark2
│   │   └── (6 total series for bank1)
│   ├── Component: AC1 (Load, 80 kW rated)
│   │   ├── Series 36-40: Power, Voltage, Frequency from solark1
│   │   ├── Series 41-45: Same from solark2
│   │   └── (10 total series for AC1)
│   ├── Component: solark1 (Sol-Ark 15k, SN: 2201064087)
│   ├── Component: solark2 (Sol-Ark 15k, SN: 2201074043)
│   ├── Component: temp1 (SAH temperature sensor)
│   └── Component: temp2 (SAH temperature sensor)
```

That's **45 data series** at the component level for Tophole alone.

### Walking Through the Tophole Pilot Unit

At **2024-09-26 10:04:36 AM** (Central Time), the master inverter (solark1) reports:

| What | Value | Meaning |
|------|-------|---------|
| Ppv1(W)/186 = 293 | Series 1 = 293W | PV String 1 producing 293 watts |
| Ppv2(W)/187 = 267 | Series 4 = 267W | PV String 2 producing 267 watts |
| Ppv3(W)/188 = 239 | Series 7 = 239W | PV String 3 producing 239 watts |
| **Total PV** | **799W** | Morning solar production |
| AcL1Power(W)/167 = 0 | Series 10 = 0W | Generator L1: not running |
| BatteryPower(W)/190 = 760 | Series 30 = 760W | Battery **charging** at 760W |
| LoadL1Power(W)/176 = 586 | Series 36 = 586W | Load consuming 586W on L1 |
| LoadL2Power(W)/177 = 958 | Series 37 = 958W | Load consuming 958W on L2 |
| **Total Load** | **1,544W** | Total load demand |

At 10:08:24, solark2 reports: Generator through solark2 L1 = 1,526W, L2 = 1,380W (generator IS running through solark2's AC input); Battery through solark2: -1,348W (discharging); Load through solark2: L1 = 406W, L2 = 206W.

The full picture: **the generator is running, feeding power through the slave inverter (solark2), which is also discharging the battery. Meanwhile, the master inverter (solark1) is receiving solar power and charging the battery.**

### The Connection Map at Tophole

| Source | Conditioning | Port | Power Type | Bidirectional |
|--------|-------------|------|------------|---------------|
| pv1 → | solark1 | PV 1 | HVDC | No |
| pv2 → | solark1 | PV 2 | HVDC | No |
| pv3 → | solark1 | PV 3 | HVDC | No |
| gen1 → | solark1 | AC | 3-phase 120/208 | No |
| gen1 → | solark2 | AC | 3-phase 120/208 | No |
| AC1 ← | solark1 | Load Out | 3-phase 120/208 | No |
| AC1 ← | solark2 | Load Out | 3-phase 120/208 | No |
| bank1 ↔ | solark1 | DC | 48 VDC | **Yes** |
| bank1 ↔ | solark2 | DC | 48 VDC | **Yes** |

Key observations:
- **Solar only connects to solark1** (the master has PV inputs, the slave does not in this config)
- **Generator and Load connect to BOTH inverters** (redundancy and load balancing)
- **Battery connects bidirectionally to BOTH inverters** (either can charge or discharge)

### Comparing the Two Real Deployments

| Attribute | Moran Ranch (12k Scenario) | Tophole (15k Scenario) |
|-----------|---------------------------|----------------------|
| Inverters | 2x Sol-Ark 12k (8 kW rated) | 6x Sol-Ark 15k (15 kVA rated) |
| Topology | 1 master + 1 slave | 3 masters + 3 slaves |
| Generator | Gen25k with DSE74XX controller | Gen125k (much larger) |
| Generator Monitoring | **EWX Monitor** via Modbus | Connection-based (through Sol-Ark) |
| Battery | Fortress eFlex | Fortress eVault |
| Solar | 3 PV strings on master only | 3 PV strings on master 1 only |
| Power Type | 120/240 Split Phase | 120/208 3 Phase |
| Location | San Antonio, TX area | Spraggs, PA |

---

## Module 3: External Data Sources & the ETL Pipeline

### Part 1: The Sol-Ark API — Primary Data Source

The Sol-Ark API is accessed via `mysolark.com` and requires OAuth2-style authentication with `client_id`, `client_secret`, `username`, and `password`.

From the Sample Meta Data for Tophole:

| Field | Value |
|-------|-------|
| ExternalSourceID | 1 |
| CompanyID | 0 |
| Name | "Tophole Solark" |
| URL | `https://mysolark.com` |
| DataSourceTypeID | 0 (Sol-Ark) |

#### The Four Sol-Ark Data Feeds

##### Feed 1: Locations (Every 60 Minutes)

Returns the list of physical plant sites. Key fields: `id` (PlantID), `name`, `latitude`, `longitude`, `status`.

##### Feed 2: Location Details (Every 60 Minutes)

Returns detailed plant configuration: `plantId`, `installedCapacity`, `plantType`, `numberOfInverters`, `status`.

##### Feed 3: Inverters (Every 60 Minutes)

Returns inverter units and serial numbers: `serialNumber`, `plantId`, `model`, `status`, `masterSlave`.

For Tophole: SN `2201064087` (Master), SN `2201074043` (Slave).

##### Feed 4: Inverter Telemetry (Every 5 Minutes)

The high-frequency feed. Returns ~50+ electrical measurements per inverter per reading:

| Field/FieldID | Measurement | Unit | Sample Value |
|---------------|-------------|------|------|
| `Ppv1(W)/186` | PV1 Power | W | 293 |
| `Ppv2(W)/187` | PV2 Power | W | 267 |
| `Ppv3(W)/188` | PV3 Power | W | 239 |
| `BatteryPower(W)/190` | Battery Power | W | 760 |
| `BatteryVolt(V)/183` | Battery Voltage | V | 51.88 |
| `BatteryTemp(℃)/182` | Battery Temp | °C | 33.7 |
| `AcL1Power(W)/167` | AC L1 Power (Gen) | W | 0 |
| `LoadL1Power(W)/176` | Load L1 Power | W | 586 |
| `LoadL2Power(W)/177` | Load L2 Power | W | 958 |
| `InvFac(W)/193` | Output Frequency | Hz | 60.0 |

The field naming convention: `FieldName(Unit)/FieldID` — e.g., `Ppv1(W)/186` means "PV input 1 power in Watts, Sol-Ark field number 186."

### Part 2: The EWX IoT Data Source — Generator Telemetry

For generators with their own controllers (DSE74XX family), EWX deploys **EWX Monitors** — custom edge devices running **AWS IoT Greengrass** that connect via **Modbus** and publish telemetry via MQTT.

#### TelemetrySourceID Convention

| Value | Meaning | Example |
|-------|---------|---------|
| Positive integer | Specific ExternalSourceID | `solark1`: TelemetrySourceID = 1 |
| -1 | From connection (derived from inverter data) | `AC1` (load) |
| -2 | From EWX Monitor | `gen1`, `temp1`, `temp2` |

### Part 3: The Full ETL Flow — Step by Step

#### Stage 0: Scheduling

| Feed | Interval | Purpose |
|------|----------|---------|
| Inverter Telemetry | Every 5 minutes | High-frequency operational data |
| Locations, Details, Inverters | Every 60 minutes | Configuration sync |
| Aggregations | Hourly / Daily / Monthly | Roll-up calculations |

#### Stage 1: Staging (Raw Data Capture)

The system calls the Sol-Ark API and stores the **entire raw response** in `External.Source.Staging` before any transformation.

| Field | Value |
|-------|-------|
| StagingID | (auto-increment) |
| ExternalSourceID | 1 |
| LocationID | 1 |
| ExternalID | "2201064087" |
| Day | 2024-09-26 |
| TimeStamp | 2024-09-26 10:04:36 |
| Data | `{"Ppv1(W)/186": 293, "BatteryPower(W)/190": 760, ...}` |

#### Stage 2: Component.Statistics (Field-Level Transformation)

Each raw field is mapped to a specific `SeriesID` using `Component.Statistics.MetaData`.

Example: `Ppv1(W)/186` from solark1 → SeriesID 1:

| MetaData Field | Value |
|----------------|-------|
| ComponentID | `pv1` |
| SeriesID | 1 |
| MeasurementID | 3 (Power) |
| UnitID | 7 (W) |
| ExternalSourceID | 1 |
| ExternalID | "2201064087" |
| ExternalField | `Ppv1(W)/186` |
| ConversionID | (null) |
| Frequency | 5 |

The resulting Component.Statistics record: CompanyID=0, LocationID=1, ComponentID=pv1, SeriesID=1, TimeStamp=10:04:36, Value=293, Version=1.

For `BatteryTemp(℃)/182` (SeriesID 32), `ConversionID = "CtoFTemp"` converts 33.7°C → 92.66°F before storage.

#### Stage 3: Component Data Updated Event

Fires after Component.Statistics are written, carrying: CompanyID, LocationID, Day, TimeStamp.

#### Stage 4: Location.Statistics (Imputation)

The most complex stage. It answers: "Given data from multiple components at different times, how do I create a coherent location-level view?"

**The Problem:** solark1 reports at :04, :09, :14... while solark2 reports at :03, :08, :13... They're offset by about 1 minute.

**The Algorithm:**

1. **Target timestamps** = union of all component timestamps
2. **For each target timestamp and series:**
   - Exact match → use directly
   - Between two known values → **linear interpolation**
   - Before first / after last → leave null
3. **Apply AggregationType:**
   - Sum: add component values (power)
   - Average: mean of components (voltage)
   - Max: highest value (temperature)
   - Min: lowest value
   - Latest: most recent non-null

**Example: Total Load Power at 10:09:40:**

- SeriesID 36 (solark1 LoadL1Power): Direct value = **1,069W**
- SeriesID 41 (solark2 LoadL1Power): Interpolated between 406W (10:08:24) and 518W (10:13:25)
  - Fraction: 76s / 301s = 0.2525
  - Value = 406 + (518 - 406) × 0.2525 = **434.3W**
- Location total = 1,069 + 434.3 = **1,503.3W** (Sum)

#### Stage 5: Company.Statistics

After Location.Statistics, a **Location Data Updated Event** triggers Company.Statistics aggregation — the same imputation process across all company locations.

#### The Complete Event Chain

```
Raw API Response
  → External.Source.Staging
    → Component.Statistics
      → [Component Data Updated Event]
        → Location.Statistics
          → [Location Data Updated Event]
            → Company.Statistics
              → [Company Data Updated Event]
                → Dashboard refresh
```

### Part 4: The .Current and .Calcs Records

**.Current:** A single JSON document with the latest value for every series. The dashboard reads this instead of querying the full time-series.

```json
{
  "Data": {
    "1": 1205,   // SeriesID 1 latest value
    "2": 192.4,  // SeriesID 2
    "3": 6.3     // SeriesID 3
  }
}
```

**.Calcs:** Running statistics per series per day: Samples, Sum, Average, Variance, StdDev, Min, Max.

### Part 5: Concurrency and Caching

**Version fields:** Every mutable record has a Version counter for optimistic locking.

**FIFO per location:** Events for the same LocationID process in order. Different locations can process in parallel.

**The .Current record IS the cache:** One read returns all current values instead of querying thousands of time-series rows.

### Part 6: Data Volume at Scale

For Tophole (2 inverters): 576 API calls/day, ~11,520 Component.Statistics data points/day.
For Tophole 15k (6 inverters): 1,728 API calls/day, ~34,560 data points/day.

---

## Module 4: The Dashboard UI — Every Screen Explained

### Screen Architecture Overview

```
DASHBOARD
├── Dashboard - Summary     (Company-level overview)
└── Dashboard - Geographic  (Map + ticker view)

DETAILS
├── Location Details        (Single location deep dive)
├── Solar Details           (Component-level solar)
├── Generation Details      (Component-level generation)
├── Load Details            (Component-level load)
└── Storage Details         (Component-level storage)

CONFIGURATION
├── External Sources        (API credential & feed mgmt)
├── Locations               (THE critical screen)
├── Notifications           (Alert groups & recipients)
└── Components Setup        (6 sub-screens for each type)

SECURITY
└── Users                   (User management)
```

### Universal Screen Requirements

1. **Card configuration:** Every card has saved show/hide and ordering per user, rendered in a 2D grid layout
2. **Real-time subscriptions:** Each screen subscribes to update events matching its context
3. **Data sourcing:** Cards read from `.Current` records (pre-computed JSON)
4. **Time zone handling:** Location screens use location timezone; summary screens use browser timezone
5. **Unit preferences:** All values respect user's preferred units
6. **Information bubbles:** Every field has a tooltip description
7. **Busy indicators:** Loading state for refreshing regions
8. **Deep linking:** Every screen+parameter combination reachable via URL

### Dashboard - Summary

Company-level overview with 10 cards:

#### Solar Production Card

| Field | Unit | Description |
|-------|------|-------------|
| CurrentProduction | kW | Total solar output across all locations |
| PredictedProduction | kW | Forecasted solar for current hour |
| DeployedCapacity | kW | Nameplate solar capacity |
| TotalProductionToday | kWh | Solar energy produced today |

Action: Double-click → Solar Details

#### Fuel Supply Card

| Field | Unit | Description |
|-------|------|-------------|
| CurrentFuelFlow | SCFM | Total fuel flow rate |
| CurrentFuelLevel | % | Total fuel as % of capacity |
| TotalUsageToday | SCF | Total fuel consumed today |

#### Generation Card

| Field | Unit | Description |
|-------|------|-------------|
| CurrentProduction | kW | Total generator output |
| DeployedCapacity | kW | Nameplate generation capacity |
| TotalProductionToday | kWh | Total energy generated today |

Action: Double-click → Generation Details

#### Battery Card

| Field | Unit | Description |
|-------|------|-------------|
| TotalEnergyOutput | kW | Power flowing out (discharging) |
| TotalEnergyInput | kW | Power flowing in (charging) |
| TotalStoredEnergy | kWh | Current stored energy |
| TotalDeployedCapacity | kWh | Nameplate total storage |

Action: Click → Storage Details

#### Load Card

| Field | Unit | Description |
|-------|------|-------------|
| TotalEnergyConsumption | kW | Total power consumption |
| TotalEnergyToday | kWh | Total energy consumed today |

Action: Double-click → Load Details

#### Data Feeds Card

Shows health of each external data source. Row color rules:
- **Red:** `LastKnownError >= LastKnownAccess`
- **Orange:** `LastKnownError < LastKnownAccess` AND error within 30 minutes
- **Green:** All clear

Action: Click → External Sources configuration

#### Locations Card

One row per location with: LocationName, City, State, TotalEnergyToday, TotalEnergyConsumption, TotalStoredEnergy, BatteryRuntime, CurrentSolarProduction, CurrentGeneration, CurrentErrors, CurrentWarnings.

Row highlighting: Red = errors > 0, Orange = warnings > 0, Green = normal.

Action: Double-click → Location Details

#### Map Card

Google Maps with colored pins per location. Pin colors match row highlighting. Mouse-over shows location summary tooltip.

#### Last Known Issues Card

Unresolved alerts: AlertID, CategoryID, LocationID, AlertType.

#### Trends Chart

Time-series chart with series selector (from Company.Statistics.MetaData), date range picker, and multi-series overlay.

### Details - Location Details

Single-location deep dive with 18 data sources feeding it.

Cards: Solar Production, Fuel Supply, Generation (with ForecastedNextRun), Battery, Load, **Weather** (location-only), **5 Alert Status Indicators** (Solar/Fuel/Generator/Battery/Load), **Equipment Schematic** (network diagram), **Alert Log**, **Location Information**, **Statistics** (from .Calcs), **Trends Chart**.

Alert indicators use green/orange/red based on InFlightAlerts filtered by CategoryID and AlertType severity.

Equipment Schematic shows the connection diagram with custom icons per component type, labeled paths with power type and max power.

### Detail Screens (Component-Level)

- **Solar Details:** Data grid per solar array with production vs. prediction, documentation, trends
- **Generation Details:** Data grid per generator with details card, maintenance card, documents, alerts/faults
- **Load Details:** Data grid per load with power consumption chart, documentation
- **Storage Details:** Data grid per battery bank with battery state card, documentation

### Configuration - External Sources

Manages API connections. Cards: External Link Status (green/orange/red), Authentication (credentials with Validate And Save), Static Data (cached API config), Incoming Data Grid (raw staging audit trail).

### Configuration - Locations (THE Most Important Screen)

Martin's own words: "This is probably the most important screen in terms of business logic."

#### General Configuration Card

Name, Active flag, GPS coordinates, City/State, Deployment date.

#### Assigned Components Section

Sub-panels for each component type with data grids and setup dialogs:

- **Solar Arrays:** ArrayID, Panel Model, # of Panels, TelemetrySourceID
- **Generation:** GenerationID, Generator Model, FuelType, OutputType, TelemetrySourceID
- **Storage:** StorageID, Storage Model, UnitsDeployed, TotalStorage, SustainedPower
- **Power Conditioning:** ConditioningID, Component Model, TelemetrySourceID
- **Load:** LoadID, RatedPower, PowerType, TelemetrySourceID
- **Sensors:** SensorID, Sensor Type, Area, TelemetrySourceID

#### Component Connections and Assignments

**Power Connections:** SourceComponentID ↔ ConditioningComponentID on a specific port, with power type and bidirectional flag.

**Connection Setup cascading validations:**
1. Select conditioning component → refresh port list
2. Select port → filter compatible connected components
3. Select component → auto-calculate max power and flow direction

#### The Save/Apply Changes Process

**For New Setup:**
1. For each power connection, look up external fields from Configuration.Conditioning.Connection.External.Fields
2. Create `Component.Statistics.MetaData` records (SeriesID + field mapping)
3. Create `Location.Statistics.MetaData` records (grouped by MeasurementID, with AggregationType)
4. Create `Company.Statistics.MetaData` records (if not already existing)
5. Save all atomically

**For Existing Setup:**
1. Generate prospective new MetaData records
2. Compare against existing records (match by ComponentID + TelemetrySourceID + Field + CategoryID + MeasurementID)
3. Keep matches, create truly new records, mark orphans for deletion
4. Display diff to user for confirmation
5. On confirmation: save new, send orphans to Remove Orphaned MetaData And Statistics cloud process

### Configuration - Notifications

Alert groups, recipients, call priority, effective schedules. (TBD in original docs.)

### Configuration - Components Setup (6 Sub-Screens)

EWX-managed only. Generation, Power Conditioning, Solar, Energy Storage, EWX Monitors, Sensors — each with specifications, deployed locations, and documentation.

### Security - Users

User management with RBAC permissions (6 boolean flags — currently unenforced).

---

## Module 5: The Data Model Deep Dive

### The Complete Data Source Map

```
1. STATISTICS (3 levels × 4 types = 12 data sources)
   Component/Location/Company × Statistics/Current/Calcs/MetaData

2. CONFIGURATION (18+ data sources)
   Location.General, Component specs (6 types), Connection, Monitor, Equipment Specs (5 types)

3. EXTERNAL (3 data sources)
   External.Source, External.Source.MetaData, External.Source.Staging

4. STATIC (lookup tables)
   Static.Measurement, Static.Unit, Static.Category, Static.PowerType

5. ALERTS
   Alerts, Alert.Groups, Alert.Notifications

6. DOCUMENTATION
   Documentation (file uploads)
```

~40+ distinct data sources total.

### Component.Statistics.MetaData — The Master Mapping Table

| Field | Type | Description |
|-------|------|-------------|
| CompanyID | Integer | Tenant |
| LocationID | Integer | Location |
| ComponentID | String | Target component |
| SeriesID | Integer | **Globally unique** data series ID |
| MeasurementID | Integer | FK → Static.Measurement |
| UnitID | Integer | FK → Static.Unit |
| CategoryID | Integer | Component category |
| ExternalSourceID | Integer | Which API to read from |
| ExternalID | String | Inverter serial number |
| ExternalField | String | JSON field to extract |
| ConversionID | String (nullable) | Conversion function |
| Frequency | Integer | Expected interval (minutes) |

#### Tophole MetaData Breakdown (45 records)

| Category | Component(s) | Series Count | ExternalID(s) |
|----------|-------------|-------------|---------------|
| Solar | pv1, pv2, pv3 | 9 | solark1 only |
| Generation | gen1 | 14 | solark1 + solark2 |
| Storage | bank1 | 6 | solark1 + solark2 |
| Load | AC1 | 10 | solark1 + solark2 |
| Conditioning | solark1, solark2 | 0 (spec only) | N/A |
| Sensors | temp1, temp2 | 0 (not configured) | N/A |
| **Total** | | **45** | |

### Component.Statistics — Time-Series Table

One row per value per timestamp:
- CompanyID, LocationID, ComponentID, SeriesID, Day, TimeStamp, Value, Version

Sample for SeriesID 1 (pv1 power): 293W → 485W → 561W → 611W → 679W (solar production climbing through the morning).

### Location.Statistics.MetaData — Aggregation Rules

21 records for Tophole. Key AggregationType assignments:

| Measurement | AggregationType | Physical Rationale |
|-------------|-----------------|-------------------|
| Power | **Sum** | Power is additive (300W + 300W = 600W) |
| Voltage | **Average** | Parallel arrays don't add voltage |
| Current | **Sum** | Parallel arrays add current |
| Temperature | **Max** | Safety: worst-case thermal reading |
| Frequency | **Average** | Frequency should be uniform |

### .Current Record Structure

```json
{
  "CompanyID": 0,
  "LocationID": 1,
  "ComponentID": "pv1",
  "Day": "2024-09-26",
  "TimeStamp": 1727362117000,
  "Version": 10,
  "Data": {
    "1": 1205,
    "2": 192.4,
    "3": 6.3
  }
}
```

Keys = SeriesIDs, values = latest readings. One document read replaces querying the full time-series.

### .Calcs Record Structure

| Field | Description |
|-------|-------------|
| Samples | Count of data points today |
| Sum | Running sum |
| Average | Sum / Samples |
| Variance | Running variance (Welford's algorithm) |
| StdDev | √Variance |
| Min | Day's minimum |
| Max | Day's maximum |

### Static Lookup Tables

**Static.Measurement:** 0=Voltage, 1=Current, 2=Temperature, 3=Power, 4=Frequency, 5=Energy, 6=Fuel Level, 7=Pressure

**Static.Unit:** 1=V, 2=A, 3=°C, 5=°F, 7=W, 8=kW, 9=Hz, 10=kWh

### Complete Data Lineage (PV1 Power Example)

```
Physical: BlueSun panels → Sol-Ark PV1 input
Configuration: Solar.Spec → Component.Solar (pv1) → Connection (pv1→solark1 PV1) → External.Fields
MetaData: Component.Statistics.MetaData → SeriesID 1
          Location.Statistics.MetaData → Sum aggregation
          Company.Statistics.MetaData → Sum aggregation
External: Source → MetaData → Staging (raw JSON)
Statistics: Component → Location (imputed) → Company (aggregated)
           Each level: .Current (cache) + .Calcs (running stats)
Dashboard: Summary Solar Card → Location Solar Card → Solar Details Grid → Trends Chart
```

One measurement touches **15+ data sources**.

---

## Module 6: Architecture & Non-Functional Requirements

### Multi-Tenant Security Architecture

Martin's requirement: *"A tenant will only ever have access to see its own data and never any other tenant's data."*

Three layers required:

1. **Authentication:** Valid, non-expired token identifying a user
2. **Tenant Authorization:** Every query filtered by user's CompanyID (server-side, not from request params)
3. **RBAC:** 6 granular permissions per role

**Current Implementation Failures:**

| Finding | Severity | Impact |
|---------|----------|--------|
| S-1: Password verification disabled | CRITICAL | Any email grants access |
| S-6: `member?` not scoped to tenant | CRITICAL | Cross-tenant data access |
| S-7: RBAC permissions never enforced | HIGH | All users equal access |
| S-8: `params.permit!` in 16+ places | HIGH | Mass assignment attacks |
| D-3: Plaintext credential storage | HIGH | Credential exposure |

### Performance Requirements

**The 10-Second Rule:** All queries must return in under 10 seconds (MAXIMUM, not target).

**The .Current Cache Strategy:** Instead of scanning thousands of time-series rows, the dashboard reads a single pre-computed JSON document per entity. Updated on every ETL cycle.

| Cache Layer | What | Invalidation |
|-------------|------|-------------|
| .Current records | Latest values | Every ETL cycle (5 min) |
| .Calcs records | Running statistics | Every ETL cycle |
| Static data | Measurements, Units, Specs | Configuration change + TTL |
| Client-side | Static data, preferences | TTL-based |

### Required Third-Party Integrations

1. **Google Maps API** — Location pins, geocoding, custom markers
2. **Weather API** — Current temperature, conditions per location
3. **Solar Production Forecast API** — Predicted output per location
4. **Charting Library** — Time-series visualization (recharts already in frontend)
5. **Network Diagram Library** — Equipment schematic (@xyflow/react already in frontend)
6. **Dashboard Library** — Configurable card grid with drag-and-drop

### Cloud Processes (9 Scheduled Jobs)

| Process | Description |
|---------|-------------|
| Collect External Data Source Statistics | Core ETL — Sol-Ark API calls |
| Calculate Component/Location/Company Statistics | Three-level cascade |
| Calculate Next Generation Run | Generator start forecasting |
| Download Weather Data | Per-location weather |
| Download Solar Forecasts | Per-location solar prediction |
| Remove Orphaned MetaData | Cleanup on config changes |
| Validate External Source Credentials | API credential testing |

### Automated Testing Requirements

Martin requires: *"All new functionality must have automated testing via Localstack or Nock or equivalent."*

Current state: **~3-5% effective coverage** (17 test files for 350+ app files). Zero model specs, zero ETL test coverage.

Required: Unit tests (models), service tests (business logic), request specs (API contract), job specs (ETL), integration tests (end-to-end pipeline), external service mocks (Nock/LocalStack), multi-tenant isolation tests.

### SDLC Requirements

AWS CDK infrastructure, Docker containerization, ECS Fargate orchestration, CI/CD pipeline, reversible migrations, feature flags, local Docker Compose development, seed data, structured logging.

### Data Concurrency Model

1. **Version Fields (Optimistic Locking):** Read version N → compute → write WHERE version = N → if 0 rows affected, retry
2. **FIFO Per Location:** Same-location events process sequentially; different locations process in parallel
3. **Globally Unique SeriesID Generation:** Must be concurrent-safe across all tenants

### Additional Non-Functional Requirements

| Requirement | Specification |
|-------------|---------------|
| Responsiveness | 1920x1080, 414x896, 768x1024 (portrait + landscape) |
| i18n | No hardcoded text; full translation support |
| Deep Linking | Every screen + parameters linkable via URL |
| Real-Time | WebSocket/SSE subscriptions per screen context |
| Card Configuration | User-controlled show/hide and ordering |
| Navigation | Browser forward/back with state restoration |
| Date/Time | Epoch milliseconds; location timezone for detail; browser timezone for summary |

---

## Module 7: Open Questions, Risks & Knowledge Gaps

### Martin's 19 Outstanding Questions

| # | Question | Status | Impact |
|---|----------|--------|--------|
| 1 | Outlier handling in statistics | **OPEN** | MEDIUM |
| 2 | Location status color rules (error vs. warning definitions) | **PARTIALLY ANSWERED** | HIGH |
| 3 | Planned outage windows for components | **OPEN** | LOW |
| 4 | Historical statistics tracking | **OPEN** | MEDIUM |
| 5 | Maximum age for stale data in summaries | **OPEN** | HIGH |
| 6 | Solar forecast API granularity | **OPEN** | MEDIUM |
| 7 | Network diagram custom icons support | **ANSWERED** (@xyflow/react) | LOW |
| 8 | Secure credential storage | **ANSWERED** (KMS/encrypted attrs) | LOW |
| 9 | Combining liquid/gas fuel flow units | **OPEN** | MEDIUM |
| 10 | Sorting without denormalized static data | **ANSWERED** (client-side cache) | LOW |
| 11 | Time-series normalization strategy | **ANSWERED** (Combining Data Series spec) | NONE |
| 12 | Missing data handling in charts | **OPEN** | LOW |
| 13 | Multi-language alerts and measurement labels | **PARTIALLY ANSWERED** | MEDIUM |
| 14 | Automatic alert expiration | **OPEN** | MEDIUM |
| 15 | Standard deviation at all three levels | **OPEN** | LOW |
| 16 | Company.Statistics update contention at scale | **OPEN** | HIGH |
| 17 | Synthetic/configurable statistics | **OPEN** | LOW (Phase 1) |
| 18 | User-defined component types | **ANSWERED** (not practical) | NONE |
| 19 | Partitioning / History tables | **OPEN** | MEDIUM |

**Summary:** 4 answered, 2 partially answered, 13 remain open. Highest-impact open questions: #2, #5, #16.

### Biggest Technical Risks

| ID | Risk | Severity | Likelihood |
|----|------|----------|------------|
| R1 | ETL pipeline silently loses data | CRITICAL | HIGH (proven) |
| R2 | Cross-tenant data leakage | CRITICAL | HIGH (proven) |
| R3 | Company.Statistics contention at scale | HIGH | MEDIUM |
| R4 | Save/Apply Changes atomicity failure | HIGH | MEDIUM |
| R5 | Frontend on unstable framework versions | HIGH | HIGH |
| R6 | Dual auth architecture creates security gaps | HIGH | HIGH |
| R7 | Missing integrations expand scope | MEDIUM | CERTAIN |
| R8 | Data volume exceeds MySQL capacity | MEDIUM | LOW (short-term) |
| R9 | 13 open design questions delay implementation | MEDIUM | HIGH |

### Information Missing for Phase 1 Evaluation

**Scale:** How many tenants, locations, inverters? Expected growth?

**Migration:** Existing production data? Historical telemetry volume? Active SeriesIDs to preserve?

**Sol-Ark API:** Rate limits? Webhook support? SLA history?

**EWX IoT:** Greengrass deployed and working? MQTT topics/formats? Edge device count?

**Business Rules:** Who are the actual end users? What actions do they take on alerts? Regulatory/compliance requirements?

**Infrastructure:** Why EU region for US locations? Who owns the AWS account? Working staging environment? Monthly AWS cost?

---

## Assessment Quiz

### Question 1 (Module 1 — Project Context)

The codebase audit gave the existing Rails application an overall grade of **D**. Name three of the ten **Critical** findings (the ones that block production deployment) and explain why each one is dangerous.

### Question 2 (Module 2 — Physical System)

At the Tophole Pilot Unit, solar arrays `pv1`, `pv2`, and `pv3` all read telemetry from inverter `solark1` (SN 2201064087), but **none** read from `solark2`. Why? What physical fact about the Tophole wiring explains this?

### Question 3 (Module 2 — Data Hierarchy)

A `SeriesID` must be unique at what scope — within a location, within a company, or across the entire platform? Why does Martin insist on this, and what specific concurrency risk does it create during the Save/Apply Changes process?

### Question 4 (Module 3 — ETL Pipeline)

Walk through what happens to the raw Sol-Ark telemetry field `BatteryTemp(℃)/182 = 33.7` as it flows from staging to `Component.Statistics`. What is the final stored value, and what MetaData field drives the transformation?

### Question 5 (Module 3 — Imputation)

At Tophole, `solark1` reports at 10:09:40 and `solark2` reports at 10:08:24 and 10:13:25. Explain how the system calculates `Location.Statistics` for total load power at timestamp 10:09:40. Which values are direct matches and which must be interpolated?

### Question 6 (Module 3 — Event Architecture)

What are the three events in the ETL cascade, in order? For each, state what **triggers** it and what **consumes** it. Then explain: why must events for the same LocationID be processed in FIFO order?

### Question 7 (Module 4 — Dashboard UI)

The Data Feeds Card on the Dashboard Summary uses a three-color status system. State the exact rules for when a data feed row appears **Red**, **Orange**, and **Green**, using the specific field names from the specification.

### Question 8 (Module 4 — Configuration)

Martin calls the Configuration - Locations screen "probably the most important screen." Explain why. Specifically: what does the Save/Apply Changes process create, at how many levels, and what would happen to the ETL pipeline if this process produced incorrect MetaData records?

### Question 9 (Module 5 — Data Model)

The `Location.Statistics.MetaData` table has an `AggregationType` field. For the Tophole deployment, explain why solar **Power** uses `Sum`, solar **Voltage** uses `Average`, and battery **Temperature** uses `Max`. Ground each answer in the physics of what's being measured.

### Question 10 (Module 5 — .Current Record)

Describe the structure of a `Component.Statistics.Current` record. What are its keys, and why does the dashboard read this single document instead of querying `Component.Statistics` directly? How does this relate to the 10-second query requirement?

### Question 11 (Module 6 — Security)

The codebase audit found that the `member?` method in the existing Rails code does NOT scope to the requested tenant. Describe the specific exploit scenario: a user belongs to Company A, sends a request for Company B's data — step by step, why does the current code allow it?

### Question 12 (Module 6 — Concurrency)

Explain the optimistic locking protocol using the `Version` field. Include: what happens on read, what happens on write, and what happens when two processes try to update the same record simultaneously. Why is this preferred over pessimistic locking for this system?

### Question 13 (Module 6 — Architecture)

The system requires both real-time subscriptions (WebSockets) and deep linking. For each, give a concrete example of how they'd work on the Location Details screen for Tophole (LocationID=1). What event triggers a WebSocket update, and what would a deep link URL look like?

### Question 14 (Module 7 — Open Questions)

Martin's Outstanding Question #5 asks about the maximum age of stale data. Explain the specific operational scenario he's worried about, why it's dangerous, and propose a solution with specific thresholds.

### Question 15 (Module 7 — Risk Assessment)

You're presenting to the client (Martin) and he asks: "What are the top 3 things that could go wrong with the rebuild, and how will you prevent them?" Answer as if you're speaking to Martin directly, referencing specific findings from the documents.

---

*DevSavant Product Engineering — Phase 1 Discovery*
*EWX Power Platform Rebuild Training Course*
