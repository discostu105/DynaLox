# DynaLox Dashboard Plan

## Available Data Summary

**261 metric series** ingested via OpenTelemetry from a Loxone Miniserver at `loxone.int.neumueller.net`.

All control data uses a single metric key `loxone.control.value`, differentiated by dimensions:
- `control.category` — Energie, Fühler, Beleuchtung, Klima, Lüftung, Temperatur, Heizung, Sicherheit
- `control.name` — individual control name
- `control.room` — 12 rooms across 3 floors (UG/EG/OG) + Zentral + Garten
- `control.type` — Meter, InfoOnlyAnalog, IRoomControllerV2, LightControllerV2, Switch, etc.
- `state.name` — actual, total, totalDay, totalWeek, totalMonth, totalYear, value, etc.
- `unit` — kW, °C, °, %, ppm

Additionally, 15 system/network health metrics:
- `loxone.system.*` — heap, heap_bytes, tasks, tasks_count
- `loxone.lan.*` — rx_packets, tx_packets, tx_errors, tx_collisions, rx_overflow, eof_errors
- `loxone.can.*` — packets_sent, packets_received, receive_errors, frame_errors, overruns

---

## Proposed Dashboards

### Dashboard 1: Energy Overview

**Purpose:** Central view of household energy production, consumption, storage, and grid interaction.

| Tile | Visualization | Query logic |
|------|--------------|-------------|
| Current Power Consumption | singleValue | `control.name == "Stromverbrauch"`, `state.name == "actual"` |
| Current Grid Draw | singleValue | `control.name == "Netzbezug"`, `state.name == "actual"` |
| Current PV/Battery | singleValue | `control.name == "Speicher"`, `state.name == "actual"` + storage |
| Energy Flow (Gpwr/Spwr/Ppwr) | singleValue x3 | `control.name == "Energiefluss"`, Grid/Solar/Storage power |
| Self-Consumption Rate | singleValue | `control.name == "Energiefluss"`, `state.name == "selfConsumption"` |
| Power over Time | lineChart | Stromverbrauch, Netzbezug, Speicher actual values as timeseries |
| Daily Consumption Breakdown | barChart | All Energie meters, `state.name == "totalDay"` |
| Per-Consumer Breakdown | pieChart | Individual meters (Heizung, Büro, Netzwerkschrank, etc.) totalDay |
| Weekly/Monthly Trend | lineChart | Stromverbrauch totalWeek/totalMonth over time |

### Dashboard 2: Climate & Indoor Environment

**Purpose:** Room-by-room temperature, humidity, CO2, and HVAC status.

| Tile | Visualization | Query logic |
|------|--------------|-------------|
| Outdoor Temperature | singleValue | `control.name == "Intelligente Raumregelung"`, `state.name == "actualOutdoorTemp"` |
| Average Outdoor Temp | singleValue | `state.name == "averageOutdoorTemp"` |
| Room Temperatures | table | All `control.category == "Temperatur"`, `state.name == "value"` |
| Temperature Over Time | lineChart | All temperature sensors as timeseries, by room |
| Humidity by Room | barChart | `control.category == "Fühler"`, `control.name == "Luftfeuchte"` |
| Humidity Over Time | lineChart | All Luftfeuchte sensors as timeseries |
| CO2 Levels by Room | barChart | `control.name == "CO2"`, by room |
| CO2 Over Time | lineChart | CO2 sensors timeseries (Küche, Kinderzimmer Bastian, Sophie, Schlafzimmer) |
| Open Windows | singleValue | `state.name == "openWindow"` |
| Heating/Cooling Mode | singleValue | IRCV2Daytimer `state.name == "mode"` |
| Heating Supply Temp (Vorlauf) | lineChart | `control.name == "Heizung Vorlauf [BT2]"` |
| Hot Water Temp (BT7) | lineChart | `control.name == "Warmwasser [BT7]"` |

### Dashboard 3: Lighting & Controls

**Purpose:** Lighting status and energy consumption per zone.

| Tile | Visualization | Query logic |
|------|--------------|-------------|
| Active Lights | singleValue | Count of Switch controls with `state.name == "active"` and value > 0 |
| Light Energy Today | table | All `Zähler Licht*` and `Zähler Wohnzimmer Licht*` meters, totalDay |
| Wohnzimmer Light Zones | barChart | Mitte/Ost/West/Lightstrip totalDay |
| Light Power Over Time | lineChart | Wohnzimmer light meters `state.name == "actual"` |
| Active Moods | table | LightControllerV2 `state.name == "activeMoodsNum"` by zone |
| Operating Mode | singleValue | GlobalState `state.name == "operatingMode"` |
| Sunrise / Sunset | singleValue x2 | GlobalState sunrise/sunset |

### Dashboard 4: Miniserver Health

**Purpose:** Technical health monitoring of the Loxone Miniserver hardware.

| Tile | Visualization | Query logic |
|------|--------------|-------------|
| System Heap (bytes) | lineChart | `loxone.system.heap_bytes` |
| System Tasks | lineChart | `loxone.system.tasks_count` |
| LAN Traffic | lineChart | `loxone.lan.rx_packets` + `loxone.lan.tx_packets` |
| LAN Errors | lineChart | `loxone.lan.tx_errors` + `loxone.lan.tx_collisions` + `loxone.lan.rx_overflow` + `loxone.lan.eof_errors` |
| CAN Bus Traffic | lineChart | `loxone.can.packets_sent` + `loxone.can.packets_received` |
| CAN Bus Errors | lineChart | `loxone.can.receive_errors` + `loxone.can.frame_errors` + `loxone.can.overruns` |
| Error Rate Summary | singleValue | Total error counts across LAN + CAN |

---

## Implementation Order

1. **Energy Overview** — highest value, most data points (196 series), core use case
2. **Climate & Indoor Environment** — actionable comfort/health insights
3. **Miniserver Health** — infrastructure monitoring, important for reliability
4. **Lighting & Controls** — operational visibility, lower priority

## DQL Patterns

All control metrics use `timeseries` with the single metric key and dimension filters:

```dql
timeseries avg(loxone.control.value),
  filter: { control.category == "Energie" AND control.name == "Stromverbrauch" AND state.name == "actual" },
  by: { control.name }
```

System metrics use their specific metric keys:

```dql
timeseries avg(loxone.system.heap_bytes)
```
