# DynaLox

Ready-made Dynatrace dashboards for Loxone Miniserver metrics, exported via [`lox otel`](https://github.com/discostu105/lox).

This is an experiment combining the OpenTelemetry export capabilities of the [`lox` CLI](https://github.com/discostu105/lox) with Dynatrace dashboards for smart home observability.

## Setup

1. Install [`lox`](https://github.com/discostu105/lox) and configure it for your Miniserver.

2. Start the OpenTelemetry metric exporter:

```bash
lox otel serve --interval 10s \
  --endpoint https://<tenant>.live.dynatrace.com/api/v2/otlp \
  --header "Authorization=Api-Token dt0c01.xxxx" \
  --delta
```

3. Deploy the dashboards to your Dynatrace environment using [`dtctl`](https://github.com/dynatrace-oss/dtctl):

```bash
dtctl apply -f dashboards/energy-overview.yaml
dtctl apply -f dashboards/climate-environment.yaml
dtctl apply -f dashboards/miniserver-health.yaml
dtctl apply -f dashboards/lighting-controls.yaml
```

## Dashboards

| Dashboard | Description |
|-----------|-------------|
| **Energy Overview** | Power consumption, grid draw, PV production, battery storage, self-consumption rate, per-consumer breakdown |
| **Climate & Environment** | Room temperatures, humidity, CO₂ levels, outdoor temperature, heating supply & hot water temps |
| **Miniserver Health** | System heap/tasks, LAN and CAN bus traffic and error rates |
| **Lighting & Controls** | Light switch states, active moods, per-zone energy, operating mode |

## License

Apache 2.0
