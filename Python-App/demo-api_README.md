# demo-api - sample Python application for the Observability lab

A small Flask application instrumented with `prometheus_client`. It produces the
metrics you need for the **PromQL** and **custom-exporter** exercises (Prometheus
exercises 8-10 in the lab).

## Files

| File               | Purpose                                             |
| ------------------ | --------------------------------------------------- |
| `app.py`           | The Flask application                               |
| `requirements.txt` | Python deps (`flask`, `prometheus_client`)          |
| `Dockerfile`       | Builds `demo-api:1.0`                               |
| `prometheus.yml`   | Used by the compose stack; already scrapes demo-api |

## Exposed metrics

| Metric                                                    | Type      | Description                    |
| --------------------------------------------------------- | --------- | ------------------------------ |
| `demo_http_requests_total{method, endpoint, status}`      | counter   | All HTTP requests              |
| `demo_http_request_duration_seconds_bucket{endpoint, le}` | histogram | Request latency                |
| `demo_http_requests_in_flight`                            | gauge     | In-flight requests             |
| `demo_active_users`                                       | gauge     | Simulated KPI (random 50..200) |

`/api/orders` returns HTTP 500 about 10% of the time on purpose so you have
errors to alert on.

## Quick start with Docker Compose (recommended)

```bash
docker compose up -d --build
# wait ~10 seconds for traffic to accumulate, then:
open http://localhost:9090   # Prometheus
open http://localhost:3000   # Grafana (admin / admin)
```

The `traffic` service generates 2 requests/sec automatically.

## Run only the app with Docker

```bash
docker build -t demo-api:1.0 .
docker run -d --name demo-api -p 8000:8000 demo-api:1.0
curl http://localhost:8000/metrics
```

## Run on Kubernetes

```bash
docker build -t demo-api:1.0 .
# Load the image into the local cluster:
#   kind:     kind load docker-image demo-api:1.0
#   minikube: minikube image load demo-api:1.0
kubectl apply -f k8s.yaml
kubectl port-forward svc/demo-api 8000:8000
curl http://localhost:8000/metrics
```

## Generate traffic manually

```bash
./traffic.sh                          # against localhost:8000
./traffic.sh http://demo-api:8000     # from inside another container
```

## Run locally without containers

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
python3 app.py
```
