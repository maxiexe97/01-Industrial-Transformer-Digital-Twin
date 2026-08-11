# Industrial Transformer Digital Twin

## Overview

Development of a simplified, physics-based digital twin for a three-phase distribution transformer.

The system combines real environmental measurements from a physical sensor node with engineering-based models to estimate the electrical and thermal behavior of a transformer that is not directly instrumented.

The digital twin was implemented using Node-RED as the processing and integration environment, with InfluxDB for time-series storage and Node-RED Dashboard 2.0 for real-time visualization and historical analysis.

The project was designed as an extension of the physical sensing system developed in the related **Low-Power LoRa Sensor Node** project.

<p align="center">
  <img src="docs/images/dash01.png" alt="Industrial Transformer Digital Twin dashboard" width="850">
</p>

<p align="center">
  <i>Real-time monitoring dashboard for the simulated transformer.</i>
</p>

---

## System Architecture

```text
                    Physical Sensor Node
                            │
                            │ LoRa
                            ▼
                     Dragino Gateway
                            │
                            │ MQTT
                            ▼
                        Node-RED
                            │
              ┌─────────────┴─────────────┐
              │                           │
              ▼                           ▼
       Digital Twin Models            InfluxDB
              │                           │
              │                           ▼
              │                    Historical Data
              │
              ▼
          Dashboard
```
The physical sensor node provides environmental measurements such as ambient temperature and timestamp.

Node-RED processes these measurements together with the transformer parameters and a time-based load profile. The resulting electrical and thermal variables are stored in InfluxDB and presented through the dashboard.

## Digital Twin Model

The digital twin is organized into three main stages:
```text
                    Time of Day
                         │
                         ▼
                   Load Profile
                         │
                         ▼
                  Demand Model
                         │
                         ▼
                 Electrical Model
                    │    │    │
                    │    │    └── Active Power
                    │    └─────── Apparent Power
                    └──────────── Current / Voltage
                         │
                         ▼
                    Thermal Model
                         │
                         ▼
                Transformer Temperature
```
The model separates the transformer parameters from the calculation logic, allowing the same model structure to be adapted to different transformer ratings.

## 1. Load Demand Model

Transformer loading is estimated as a function of the time of day using a normalized daily load profile.

The profile is represented by discrete points and linearly interpolated between them, allowing the model to estimate the transformer load at any point during the day.
Example profile:

| Time  | Load |
| ----- | ---- |
| 00:00 | 35%  |
| 02:00 | 25%  |
| 04:00 | 20%  |
| 06:00 | 28%  |
| 08:00 | 50%  |
| 10:00 | 62%  |
| 12:00 | 70%  |
| 14:00 | 65%  |
| 16:00 | 72%  |
| 18:00 | 88%  |
| 20:00 | 100% |
| 22:00 | 75%  |
| 24:00 | 35%  |

The model receives the current time and returns the normalized transformer loading:
```text
load_pu
```

## 2. Transformer Parameters

The model stores the transformer characteristics separately from the processing logic.

The prototype configuration uses:
```text
Rated power:         315 kVA
Secondary voltage:   400 V
Power factor:        0.92
Voltage regulation:  4 %
Maximum temperature rise: 45 °C
Thermal time constant:    180 min
```
Separating these parameters from the model allows the same calculation logic to be reused with different transformer configurations.

## 3. Electrical Model

The electrical model converts the estimated transformer loading into operating variables.

The nominal current is calculated from the rated apparent power and secondary voltage:

```text
In = Sn / (√3 × Vn)
```

For a 315 kVA, 400 V transformer:

```text
In ≈ 454.7 A
```

The estimated operating variables are then calculated from the normalized load:

```text
I = load_pu × In

S = load_pu × Sn

P = S × PF
```

A simplified voltage regulation model is also used to represent the voltage drop as the transformer loading increases:

```text
V = Vn × (1 − Reg × load_pu)
```

The electrical model therefore provides:

- Estimated current
- Secondary voltage
- Apparent power
- Active power
- Transformer loading

---

## 4. Thermal Model

The thermal model estimates transformer temperature from ambient temperature and transformer loading.

The temperature rise is related to the normalized current:

```text
ΔT = ΔTmax × (I / In)²
```

The resulting target temperature is:

```text
Ttarget = Tamb + ΔT
```

Unlike the electrical variables, transformer temperature is not treated as an instantaneous value.

A first-order discrete model is used to represent the thermal inertia of the transformer:

```text
Tn = Tn-1 + K × (Ttarget − Tn-1)
```

The coefficient `K` is derived from the transformer thermal time constant and the sampling interval:

```text
K = 1 − e^(-dt / τ)
```

For the implemented model:

```text
dt = 15 min
τ  = 180 min
```

This produces a gradual temperature response as the estimated load changes, rather than an instantaneous temperature jump.

The model represents a simplified equivalent transformer temperature suitable for the digital twin rather than a detailed winding-level thermal simulation.

---

## Data Processing

After the electrical and thermal calculations, the processed data is normalized into a common structure containing:

```text
Timestamp
Ambient temperature
Transformer temperature
Load level
Current
Voltage
Apparent power
Active power
```

The resulting data is used both for real-time visualization and historical storage.

---

## Time-Series Storage

Transformer data is stored in InfluxDB using the measurement:

```text
dt_transformador
```

The stored variables include:

- Ambient temperature
- Transformer temperature
- Load level
- Current
- Voltage
- Apparent power
- Active power

Historical data can be queried and aggregated over time to generate hourly trends for the main transformer variables.

<p align="center">
  <img src="docs/images/dash02.png" alt="Historical transformer data" width="850">
</p>

<p align="center">
  <i>Historical transformer data retrieved from InfluxDB.</i>
</p>

---

## Dashboard

A custom Node-RED Dashboard 2.0 interface was developed to provide a real-time view of the transformer state.

The interface displays:

- Transformer temperature
- Ambient temperature
- Current
- Secondary voltage
- Active power
- Apparent power
- Load percentage
- Transformer nominal parameters

Historical charts provide additional visualization of transformer behavior over time.

---

## Node-RED Implementation

The complete processing pipeline was implemented using Node-RED Function nodes and dashboard components.

The main processing stages are:

```text
Input Data
    │
    ▼
Load Profile
    │
    ▼
Demand Model
    │
    ▼
Electrical Model
    │
    ▼
Thermal Model
    │
    ▼
Data Processing
    │
    ├──────────────► Dashboard
    │
    └──────────────► InfluxDB
```

<p align="center">
  <img src="docs/images/flow01.png" alt="Node-RED digital twin flow" width="1000">
</p>

<p align="center">
  <i>Main Node-RED flow implementing the digital twin.</i>
</p>

---

## Technologies

**Data acquisition:** LoRa · MQTT

**Processing & integration:** Node-RED

**Time-series database:** InfluxDB

**Visualization:** Node-RED Dashboard 2.0

**Modeling:** Load profile interpolation · Electrical modeling · First-order thermal model

**Hardware integration:** STM32 · ESP32 · Dragino Gateway

---

## Project Structure

```text
industrial-transformer-digital-twin/
│
├── README.md
│
├── flows/
│   └── flows.json
│
└── docs/
    └── images/
        ├── dashboard-main.png
        ├── historical-dashboard.png
        └── digital-twin-flow.png
```

---

## Relation to the Physical Sensor Project

The environmental data used by the digital twin originates from the physical LoRa sensor node developed in the related project:

**Low-Power LoRa Sensor Node**

The two projects form a complete acquisition-to-modeling pipeline:

```text
Physical Sensor
      │
      ▼
     LoRa
      │
      ▼
   Gateway
      │
      ▼
     MQTT
      │
      ▼
   Node-RED
      │
      ▼
 Digital Twin
      │
      ├────────► InfluxDB
      │
      └────────► Dashboard
```

The physical sensor provides real measurements, while the digital twin uses these measurements as inputs to estimate variables that are not directly measured on the transformer.

---

## Key Engineering Concepts

- Embedded sensor data integration
- MQTT-based telemetry
- Time-series data processing
- Load profile modeling
- Electrical parameter estimation
- Transformer voltage regulation modeling
- Thermal inertia modeling
- State-based simulation
- Real-time monitoring
- Historical data analysis
- Node-RED flow-based system integration
