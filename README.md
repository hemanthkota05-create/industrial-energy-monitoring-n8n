# Agentic AI – Real-Time Industrial Energy Optimization and Fault Monitoring

An n8n workflow that monitors an industrial electrical and environmental setup in real time. It finds faults at the edge, uses an AI agent to explain them and suggest fixes, sends alerts, and gives operators a chatbot they can ask about the live state of the system and its energy use.

## Overview

The system tracks five sensor signals, from an ESP32 microcontroller or a built-in simulator:

| Sensor | Normal range |
|---|---|
| Light voltage | 207 – 253 V (nominal 230 V) |
| Fan voltage | 207 – 253 V (nominal 230 V) |
| Temperature | below 45 °C |
| Leakage gas | below 300 ppm |
| Load current | below 10 A |

It is built as two flows.

### Flow 1 – Monitoring, fault detection and AI analysis

1. **Data input**
   - `ESP32 HTTP Telemetry`: a webhook (`POST /webhook/sensor-data`) that receives real readings from an ESP32.
   - `Every Minute (Simulated ESP32)`: a schedule trigger that makes realistic simulated readings (normal operation, low or high voltage, power interruption, overcurrent, overheating, gas leak), so you can demo the system without hardware.
2. **Edge processing and fault detection** (rule-based, no AI cost)
   - Finds `LOW_VOLTAGE`, `HIGH_VOLTAGE`, `POWER_INTERRUPTION`, `POWER_DISTURBANCE` (sudden voltage swings), `ABNORMAL_CURRENT`, `HIGH_TEMPERATURE` and `GAS_LEAK`.
   - Gives each fault a severity (medium, high or critical) and raises the severity when several faults happen together.
   - Works out power (kW) and keeps a rolling energy summary: average and peak power, energy used in the window (kWh), and how long the load has run without a break.
   - Saves the latest 120 readings and 50 fault events in workflow static data.
3. **Agentic AI analysis** (runs only when a fault is found)
   - An AI agent using OpenAI `gpt-4o-mini`, with memory of past faults, checks the electrical and environmental signals against each other to cut false positives. It then names the fault type and severity, lists likely root causes, explains the problem in plain language, ranks corrective actions, states any immediate safety action, and gives an energy-saving tip.
4. **Alerting and logging**
   - Builds a readable alert message.
   - Sends it to Telegram (optional).
   - Adds the event to a Google Sheets fault log (optional).

### Flow 2 – Operator chatbot

A chat interface where operators can ask questions such as:
- "What is the system status?"
- "Why did the last fault happen?"
- "What should I check first?"
- "How can we reduce energy consumption?"

The chatbot loads the live system state (latest reading, recent faults, last 10 readings, energy summary and thresholds) and answers only from that data. It puts safety-critical warnings first.

## Architecture

```
ESP32 / Simulator ──► Read Sensors ──► Edge Fault Detection ──► Abnormal?
                                                              │
                                     ┌────────── yes ─────────┴──── no ──► Keep Monitoring
                                     ▼
                         Agentic AI Analysis (LLM + memory)
                                     ▼
                           Format Alert ──► Telegram / Google Sheets

Operator Chat ──► Load Live System State ──► Chatbot Agent (LLM + memory) ──► Reply
```

## Tech stack

- **n8n**: workflow orchestration
- **OpenAI gpt-4o-mini**: AI agents for fault analysis and the chatbot
- **ESP32**: sensor data source over HTTP
- **Telegram**: alerts (optional)
- **Google Sheets**: event log (optional)

## Setup

1. Import `workflow/agentic-ai-industrial-energy-monitoring.json` into n8n (**Workflows → Import from file**).
2. Add an **OpenAI** credential to both chat model nodes.
3. (Optional) Add a **Telegram** credential and replace `REPLACE_WITH_YOUR_TELEGRAM_CHAT_ID` with your chat ID.
4. (Optional) Add a **Google Sheets** credential, replace `REPLACE_WITH_GOOGLE_SHEET_ID`, and create a sheet tab named `FaultLog`.
5. Activate the workflow so readings build up and the chatbot has history to use.

### Sending data from an ESP32

POST JSON to `https://<your-n8n-host>/webhook/sensor-data`:

```json
{
  "light_voltage": 229.4,
  "fan_voltage": 231.2,
  "temperature_c": 34.1,
  "gas_ppm": 42,
  "current_a": 3.6
}
```

If no real data arrives, the simulator keeps the system running for demos.

## Use cases

- Predictive maintenance and early fault warning in small industrial units
- Energy audits and load-scheduling optimization
- Safety monitoring for gas leaks and overheating
- Academic and IoT + AI demonstration projects

## Author

Hemanth Kota
