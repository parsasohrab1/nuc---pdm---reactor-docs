# Software Requirements Specification (SRS)

## For the PdM-Nuclear System (Predictive Maintenance for Nuclear Reactor)

**Version:** 1.0
**Date:** 2026-05-30
**Status:** Initial draft

---

## 1. Introduction

### 1.1 Document Purpose
The purpose of this document is to specify the software requirements for an anomaly detection and predictive maintenance system for nuclear reactors. This document serves as the basis for design, implementation and testing.

### 1.2 Product Scope
The product includes three main modules:
1. **Synthetic data generator** (a reactor physics simulator with anomaly injection capability)
2. **Anomaly detection engine** (LSTM Autoencoder + One-Class SVM + adaptive threshold)
3. **Operator dashboard** (real-time display, alerts, SHAP root-cause analysis)

### 1.3 Definitions, Abbreviations and Acronyms

| Term | Definition |
|-------|-------|
| PdM | Predictive Maintenance |
| LSTM | Long Short-Term Memory – a type of recurrent neural network |
| SHAP | SHapley Additive exPlanations – a model explainability method |
| MAE | Mean Absolute Error – mean absolute error metric |
| SCADA | Supervisory Control and Data Acquisition |

### 1.4 References
- IEEE 830-1998 standard for SRS
- ANSI/ANS-3.2-2022 standard for reactor control systems

### 1.5 Overview
This document defines the functional and non-functional requirements of the target system.

---

## 2. Overall Description

### 2.1 Product Vision
The system is installed as a software layer on top of the reactor's existing SCADA system and provides early failure detection (at least 20 minutes before a critical event).

### 2.2 User Characteristics

| Role | Description | Access |
|-----|------|---------|
| Reactor operator | View the dashboard, receive alerts | Read-only |
| Maintenance engineer | Set thresholds, view reports | Read + change settings |
| System administrator | Install, configure, manage users | Full |
| Data researcher | Retrain the model with new data | Access to the model API |

### 2.3 Operating Environment
- **Hardware:** Industrial server with GPU (at least 8 GB GPU RAM, 32 GB system RAM)
- **Operating system:** Linux (Ubuntu 22.04 or later) or Windows Server 2019+
- **Database:** InfluxDB (for time-series data)
- **Network infrastructure:** Local network with latency under 10 milliseconds

### 2.4 Design Constraints
- Processing latency of each data window must be under 500 milliseconds
- The system must work without an internet connection (offline operation)
- All data must be encrypted on the local system

### 2.5 Prerequisites and Dependencies
- Access to the sensor data stream at a rate ≥ 1 Hz via OPC-UA or Modbus TCP
- At least 500 GB of storage for one month of data

---

## 3. Specific Requirements

### 3.1 Functional Requirements

#### 3.1.1 Synthetic Data Generator

| Code | Requirement | Priority |
|----|----------|--------|
| FR-01 | The system must be able to generate synthetic data for 4 variables: temperature, pressure, neutron flux, vibration | Essential |
| FR-02 | Data must be generatable at a selectable frequency of 0.1 to 10 Hz | Essential |
| FR-03 | The system must be able to inject 4 types of standard anomalies (Table 1) | Essential |
| FR-04 | The generator output must be a CSV file with the columns timestamp, temp_c, pressure_bar, neutron_flux, vibration_ms2, label | Essential |
| FR-05 | Ability to generate data for a period of 1 day to 12 months | Essential |

**Table 1 – Injectable anomaly types:**

| Type | Name | Pattern | Default value |
|-----|-----|------|---------------|
| A1 | Sudden temperature rise | Step of +15 to +30 degrees | 200 events per month |
| A2 | Abnormal pressure fluctuation | Gaussian noise with mean +10 | 150 events per month |
| A3 | Neutron flux drop | Multiply by 0.3 | 100 events per month |
| A4 | High-frequency vibration | Increase of 0.8 to 1.5 units | 250 events per month |

#### 3.1.2 Data Acquisition and Preprocessing

| Code | Requirement | Priority |
|----|----------|--------|
| FR-06 | The system must support the OPC-UA protocol for real-time data acquisition | Essential |
| FR-07 | Missing data must be reconstructed with linear interpolation | Essential |
| FR-08 | Data normalization must be done with the Min-Max method (range 0 to 1) | Essential |
| FR-09 | Dynamic windowing: window length of 10 seconds for neutron flux, 3600 seconds for temperature | Essential |

#### 3.1.3 Anomaly Detection Engine

| Code | Requirement | Priority |
|----|----------|--------|
| FR-10 | The system must include an LSTM Autoencoder with a specific architecture (input: 64, encoder: 32-16, decoder: 32-64) | Essential |
| FR-11 | The model must be trained only on data labeled 0 (normal) | Essential |
| FR-12 | Reconstruction error must be calculated with the MSE metric | Essential |
| FR-13 | The detection threshold must be adaptive with the formula μ + kσ (k=3 by default) and updated every 24 hours | Essential |
| FR-14 | The system must be able to detect an anomaly in less than 1 second after receiving the window | Essential |
| FR-15 | The engine output must include: 1/0 anomaly, reconstruction error value, confidence score (0 to 100) | Essential |

#### 3.1.4 Explainability and Root-Cause Analysis

| Code | Requirement | Priority |
|----|----------|--------|
| FR-16 | The system must calculate the importance of each sensor for every alert using the SHAP library | Essential |
| FR-17 | For each anomaly, a probable cause must be provided based on a rules table (Table 2) | Essential |
| FR-18 | A natural-language cause report must be generated (e.g., "The abnormal temperature rise is probably caused by a cooling pump failure") | Desirable |

**Table 2 – Root-cause rules:**

| Sensor pattern | Probable cause |
|--------------|--------------|
| Temperature ↑↑, pressure ↓ | Cooling failure |
| Neutron ↓↓, temperature ↓ | Control rod insertion |
| Vibration ↑↑ | Mechanical imbalance |
| Pressure ↑↑, temperature ↑ | Circuit blockage |

#### 3.1.5 User Interface and Alerts

| Code | Requirement | Priority |
|----|----------|--------|
| FR-19 | A web-based dashboard able to display 4 online charts (updated every 1 second) | Essential |
| FR-20 | Alerts must be displayed audibly (beep) and visually (red popup) | Essential |
| FR-21 | The history of the last 100 alerts with details must be viewable on the dashboard | Essential |
| FR-22 | Ability to export a PDF report of the alerts in a time interval | Desirable |

#### 3.1.6 Data Storage and Management

| Code | Requirement | Priority |
|----|----------|--------|
| FR-23 | Raw sensor data must be stored in the InfluxDB database for at least 3 months | Essential |
| FR-24 | The trained model must be saved and loaded as a .h5 or .onnx file | Essential |
| FR-25 | Automatic backup of all data and the model every 24 hours | Essential |

### 3.2 Non-Functional Requirements

#### 3.2.1 Performance

| Code | Requirement | Value |
|----|----------|-------|
| NFR-01 | Response time from data reception to alert display | ≤ 1 second |
| NFR-02 | True detection rate (TPR) | ≥ 95% |
| NFR-03 | False alarm rate (FPR) | ≤ 2% |
| NFR-04 | Concurrent processing capacity | At least 1000 sensors |

#### 3.2.2 Reliability and Availability

| Code | Requirement |
|----|----------|
| NFR-05 | System availability (Uptime) ≥ 99.9% |
| NFR-06 | In case of power loss, the system must be able to recover automatically within 2 minutes |
| NFR-07 | Mean time between failures (MTBF) ≥ 720 hours |

#### 3.2.3 Security

| Code | Requirement |
|----|----------|
| NFR-08 | Two-factor authentication for administrator access |
| NFR-09 | Data encryption at rest with AES-256 |
| NFR-10 | Logging all accesses and configuration changes |
| NFR-11 | Separation of the operational network from the enterprise network |

#### 3.2.4 Maintainability

| Code | Requirement |
|----|----------|
| NFR-12 | Source code in Python 3.10+ using the TensorFlow, FastAPI, InfluxDB-client libraries |
| NFR-13 | API documentation as Swagger/OpenAPI |
| NFR-14 | Ability to update the model without stopping the system |

#### 3.2.5 Usability

| Code | Requirement |
|----|----------|
| NFR-15 | Operator training time to work with the dashboard ≤ 2 hours |
| NFR-16 | The dashboard must be switchable between Persian and English |

---

## 4. Data Models

### 4.1 Sensor Input Data Structure

| Field | Type | Range | Description |
|------|-----|------|-------|
| timestamp | datetime64[ns] | - | Time with second precision |
| temp_c | float32 | 250 - 350 | Reactor core temperature (Celsius) |
| pressure_bar | float32 | 140 - 160 | Coolant pressure (bar) |
| neutron_flux | int32 | 800 - 1200 | Neutron flux (in relative units) |
| vibration_ms2 | float32 | 0.1 - 2.0 | Vibration acceleration (meters per second squared) |

### 4.2 Detection Output Structure

| Field | Type | Description |
|------|-----|-------|
| anomaly_label | int8 | 0 = normal, 1 = anomaly |
| reconstruction_error | float32 | MSE reconstruction error |
| confidence_score | float32 | 0 to 100 |
| probable_cause | string | Cause explanation in natural language |
| affected_sensors | list[string] | List of high-error sensors |

---

## 5. Interface Requirements

### 5.1 User Interface (UI)
- Framework: React.js
- Template: Responsive dashboard
- Components: Live-updating line charts, alerts table, settings panel

### 5.2 Application Programming Interface (API)
- Style: RESTful
- Data format: JSON
- Main endpoints:

| Method | Path | Description |
|------|------|-------|
| POST | /api/v1/predict | Receive a data window, return the detection result |
| GET | /api/v1/alerts | Get the alert history |
| POST | /api/v1/model/retrain | Request model retraining |

### 5.3 Hardware Interface
- OPC-UA protocol (freeopcua library)
- Modbus TCP protocol (pymodbus library)

---

## 6. Testing and Validation Requirements

| Code | Test requirement |
|----|----------------|
| TEST-01 | Unit Test for each preprocessing function |
| TEST-02 | Integration Test between the data generator and the detection engine |
| TEST-03 | Acceptance Test with 10 simulated failure scenarios |
| TEST-04 | Performance (Load) Test at a rate of 1000 records per second |

---

## 7. Requirements Prioritization Table

| Priority | Number of requirements |
|--------|-------------------|
| Essential (Must-have) | 22 |
| Desirable (Should-have) | 4 |
| Could-have | 2 |

---

If you need an **official document template for delivery to the client** or **conversion of this SRS to a Word/PDF file** with standard numbering, please let me know. I can also prepare **Use Case, Sequence and Activity diagrams** for this SRS using PlantUML or Draw.io code.
