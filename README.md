# DynaLox

Ready-made Dynatrace dashboards for Loxone Miniserver metrics, exported via [`lox otel`](https://github.com/discostu105/lox).

This is an experiment combining the OpenTelemetry export capabilities of the [`lox` CLI](https://github.com/discostu105/lox) with Dynatrace dashboards for smart home observability.

## Setup

### Option 1: CLI

1. Install [`lox`](https://github.com/discostu105/lox) and configure it for your Miniserver.

2. Start the OpenTelemetry metric exporter:

```bash
lox otel serve --interval 10s \
  --endpoint https://<tenant>.live.dynatrace.com/api/v2/otlp \
  --header "Authorization=Api-Token <YOUR_TOKEN>" \
  --delta
```

### Option 2: Docker

The `lox` CLI is available as a Docker image at `ghcr.io/discostu105/lox`.

1. Create a `lox` config file (`config.yaml`):

```yaml
host: https://<MINISERVER_HOST>
user: <MINISERVER_USER>
pass: <MINISERVER_PASS>
serial: ""
```

2. Run the container:

```bash
docker run -d --name lox-otel \
  -v $(pwd)/config.yaml:/root/.lox/config.yaml:ro \
  ghcr.io/discostu105/lox:0.6.2 \
  otel serve \
    --interval 60s \
    --endpoint https://<TENANT>.live.dynatrace.com/api/v2/otlp \
    --header "Authorization=Api-Token <YOUR_TOKEN>" \
    --delta
```

### Option 3: Kubernetes

1. Create a Secret for the `lox` config (Miniserver connection):

```yaml
# lox-config-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: lox-config
type: Opaque
stringData:
  config.yaml: |
    host: https://<MINISERVER_HOST>
    user: <MINISERVER_USER>
    pass: "<MINISERVER_PASS>"
    serial: ""
```

2. Create a Secret for the Dynatrace OTLP headers:

```yaml
# lox-otel-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: lox-otel
type: Opaque
stringData:
  headers: "Authorization=Api-Token <YOUR_TOKEN>"
```

3. Create the Deployment:

```yaml
# lox-otel-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lox-otel
  labels:
    app: lox-otel
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: lox-otel
  template:
    metadata:
      labels:
        app: lox-otel
    spec:
      containers:
        - name: lox
          image: ghcr.io/discostu105/lox:0.6.2
          args:
            - otel
            - serve
            - --interval
            - "60s"
            - --endpoint
            - "$(OTEL_EXPORTER_OTLP_ENDPOINT)"
            - --header
            - "$(OTEL_EXPORTER_OTLP_HEADERS)"
            - --delta
          env:
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: "https://<TENANT>.live.dynatrace.com/api/v2/otlp"
            - name: OTEL_EXPORTER_OTLP_HEADERS
              valueFrom:
                secretKeyRef:
                  name: lox-otel
                  key: headers
          volumeMounts:
            - name: config
              mountPath: /root/.lox
              readOnly: true
          resources:
            requests:
              memory: "32Mi"
              cpu: "10m"
            limits:
              memory: "128Mi"
              cpu: "100m"
      volumes:
        - name: config
          secret:
            secretName: lox-config
            items:
              - key: config.yaml
                path: config.yaml
```

4. Apply the manifests:

```bash
kubectl apply -f lox-config-secret.yaml
kubectl apply -f lox-otel-secret.yaml
kubectl apply -f lox-otel-deployment.yaml
```

### Deploy Dashboards

Deploy the dashboards to your Dynatrace environment using [`dtctl`](https://github.com/dynatrace-oss/dtctl):

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
