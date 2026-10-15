# TP 6 - Observabilité, SRE et Chaos Engineering sur une architecture distribuée


## OpenTelemetry, Prometheus, Grafana, Loki, traces distribuées, SLI/SLO, alerting, incidents, chaos testing et résilience

**Niveau :** architecture distribuée avancée / SRE  
**Durée conseillée :** 2 h à 3 h  
**Architecture :** microservices distribués observables et testés sous panne  
**Technologies :** Python 3.12+, FastAPI, Docker, Kubernetes, OpenTelemetry, Prometheus, Grafana, Loki, Tempo ou Jaeger, Redis, PostgreSQL, Kafka, Locust  
**Systèmes :** Linux, macOS, Windows PowerShell

---

# 1. Objectifs du TP

À la fin de ce TP, vous serez capable de :

- comprendre la différence entre **logs, métriques et traces** ;
- instrumenter une API avec OpenTelemetry ;
- générer des traces distribuées ;
- propager un `trace_id` entre plusieurs services ;
- exporter des traces vers Jaeger ou Tempo ;
- exposer des métriques Prometheus ;
- construire des dashboards Grafana ;
- centraliser les logs avec Loki ;
- corréler logs et traces ;
- mesurer p50, p95 et p99 ;
- définir des SLI, SLO et SLA ;
- calculer un error budget ;
- créer des alertes ;
- distinguer symptômes et causes ;
- mesurer le consumer lag Kafka ;
- observer les pools PostgreSQL ;
- simuler une panne Redis ;
- simuler une panne PostgreSQL ;
- simuler une panne Kafka ;
- injecter de la latence réseau ;
- injecter de la perte de paquets ;
- provoquer de la saturation CPU ;
- provoquer de la pression mémoire ;
- simuler un pod crash ;
- observer la récupération automatique ;
- mesurer le Mean Time To Detect ;
- mesurer le Mean Time To Recover ;
- créer un runbook d'incident ;
- mener un mini exercice de chaos engineering complet.

---

# 2. Contexte

Nous reprenons l'architecture des TP précédents :

```text
ERP
CRM
Warehouse
Payment
PostgreSQL
Redis
Kafka
Kubernetes
```

Jusqu'ici, nous avons appris à :

```text
déployer
scaler
résister à certaines pannes
```

Dans ce TP, nous voulons répondre à une nouvelle question :

```text
Comment savoir ce qui se passe réellement dans le système ?
```

---

# 3. Les trois piliers de l'observabilité

```mermaid
flowchart LR
    O[Observabilité]

    L[Logs]
    M[Métriques]
    T[Traces]

    O --> L
    O --> M
    O --> T
```

---

# 4. Logs

Les logs répondent généralement à :

```text
Que s'est-il passé ?
```

Exemple :

```json
{
  "level": "ERROR",
  "service": "erp",
  "message": "warehouse timeout",
  "order_id": "abc",
  "trace_id": "xyz"
}
```

---

# 5. Métriques

Les métriques répondent plutôt à :

```text
Combien ?
A quelle fréquence ?
Pendant combien de temps ?
```

Exemples :

```text
http_requests_total
http_request_duration_seconds
process_cpu_seconds_total
kafka_consumer_lag
db_pool_in_use
```

---

# 6. Traces

Les traces répondent à :

```text
Quel chemin une requête a-t-elle suivi ?
```

Exemple :

```text
Client
  |
ERP
  |
CRM
  |
Warehouse
  |
PostgreSQL
```

---

# 7. Architecture d'observabilité

```mermaid
flowchart LR
    ERP[ERP]
    CRM[CRM]
    WH[Warehouse]

    OTEL[OpenTelemetry Collector]

    PROM[Prometheus]
    LOKI[Loki]
    TEMPO[Tempo / Jaeger]
    GRAF[Grafana]

    ERP --> OTEL
    CRM --> OTEL
    WH --> OTEL

    OTEL --> PROM
    OTEL --> TEMPO

    ERP --> LOKI
    CRM --> LOKI
    WH --> LOKI

    PROM --> GRAF
    LOKI --> GRAF
    TEMPO --> GRAF
```

---

# 8. Pourquoi OpenTelemetry ?

Sans standardisation :

```text
chaque bibliothèque
chaque format
chaque backend
```

peut utiliser une instrumentation différente.

OpenTelemetry propose un standard commun pour :

```text
traces
metrics
logs
context propagation
```

---

# 9. Arborescence

```text
tp6-observability-chaos/
|
|-- docker-compose.yml
|
|-- app/
|   |-- Dockerfile
|   |-- requirements.txt
|   |-- main.py
|
|-- otel/
|   |-- otel-collector-config.yaml
|
|-- prometheus/
|   |-- prometheus.yml
|
|-- grafana/
|   |-- provisioning/
|
|-- loki/
|   |-- loki-config.yaml
|
|-- load-tests/
|   |-- locustfile.py
|
|-- chaos/
|   |-- latency.sh
|   |-- packet-loss.sh
|   |-- cpu-stress.sh
|
|-- k8s/
|   |-- chaos-pod.yaml
|
|-- README.md
```

---

# 10. Docker Compose observabilité

Créer `docker-compose.yml`.

```yaml
services:

  # ------------------------------------------------------------------
  # Application FastAPI
  # ------------------------------------------------------------------
  app:
    build: ./app

    container_name: obs-app

    environment:
      OTEL_EXPORTER_OTLP_ENDPOINT: http://otel-collector:4317
      SERVICE_NAME: obs-api

    ports:
      - "8000:8000"

    depends_on:
      - otel-collector
      - redis
      - postgres


  # ------------------------------------------------------------------
  # PostgreSQL
  # ------------------------------------------------------------------
  postgres:
    image: postgres:16

    container_name: obs-postgres

    environment:
      POSTGRES_USER: obs
      POSTGRES_PASSWORD: obs
      POSTGRES_DB: obs

    ports:
      - "5432:5432"


  # ------------------------------------------------------------------
  # Redis
  # ------------------------------------------------------------------
  redis:
    image: redis:7

    container_name: obs-redis

    ports:
      - "6379:6379"


  # ------------------------------------------------------------------
  # OpenTelemetry Collector
  # ------------------------------------------------------------------
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest

    container_name: otel-collector

    command:
      - "--config=/etc/otelcol/config.yaml"

    volumes:
      - ./otel/otel-collector-config.yaml:/etc/otelcol/config.yaml

    ports:
      - "4317:4317"
      - "4318:4318"


  # ------------------------------------------------------------------
  # Jaeger
  # ------------------------------------------------------------------
  jaeger:
    image: jaegertracing/all-in-one:latest

    container_name: jaeger

    ports:
      - "16686:16686"
      - "4317"


  # ------------------------------------------------------------------
  # Prometheus
  # ------------------------------------------------------------------
  prometheus:
    image: prom/prometheus:latest

    container_name: prometheus

    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml

    ports:
      - "9090:9090"


  # ------------------------------------------------------------------
  # Loki
  # ------------------------------------------------------------------
  loki:
    image: grafana/loki:latest

    container_name: loki

    command:
      - "-config.file=/etc/loki/local-config.yaml"

    ports:
      - "3100:3100"


  # ------------------------------------------------------------------
  # Grafana
  # ------------------------------------------------------------------
  grafana:
    image: grafana/grafana:latest

    container_name: grafana

    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: admin

    ports:
      - "3000:3000"

    depends_on:
      - prometheus
      - jaeger
      - loki
```

---

# 11. Dépendances Python

Créer `app/requirements.txt`.

```txt
fastapi
uvicorn[standard]
httpx
redis
sqlalchemy
psycopg2-binary

prometheus-client
prometheus-fastapi-instrumentator

opentelemetry-api
opentelemetry-sdk
opentelemetry-exporter-otlp
opentelemetry-instrumentation-fastapi
opentelemetry-instrumentation-httpx
opentelemetry-instrumentation-sqlalchemy
```

---

# 12. Dockerfile application

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install \
    --no-cache-dir \
    -r requirements.txt

COPY main.py .

CMD [
  "uvicorn",
  "main:app",
  "--host",
  "0.0.0.0",
  "--port",
  "8000"
]
```

---

# 13. Application FastAPI

Créer `app/main.py`.

```python
import asyncio
import json
import logging
import os
import time
from uuid import uuid4

import httpx
import redis

from fastapi import (
    FastAPI,
    HTTPException,
    Request
)

from prometheus_client import (
    Counter,
    Gauge,
    Histogram
)

from prometheus_fastapi_instrumentator import (
    Instrumentator
)

from sqlalchemy import (
    create_engine,
    text
)

from opentelemetry import (
    trace
)

from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import (
    OTLPSpanExporter
)

from opentelemetry.instrumentation.fastapi import (
    FastAPIInstrumentor
)

from opentelemetry.instrumentation.httpx import (
    HTTPXClientInstrumentor
)

from opentelemetry.instrumentation.sqlalchemy import (
    SQLAlchemyInstrumentor
)

from opentelemetry.sdk.resources import (
    Resource
)

from opentelemetry.sdk.trace import (
    TracerProvider
)

from opentelemetry.sdk.trace.export import (
    BatchSpanProcessor
)


# -------------------------------------------------------------------
# Configuration générale
# -------------------------------------------------------------------

SERVICE_NAME = os.getenv(
    "SERVICE_NAME",
    "obs-api"
)

OTEL_ENDPOINT = os.getenv(
    "OTEL_EXPORTER_OTLP_ENDPOINT",
    "http://otel-collector:4317"
)


# -------------------------------------------------------------------
# Logging structuré
# -------------------------------------------------------------------

logging.basicConfig(
    level=logging.INFO,
    format="%(message)s"
)

logger = logging.getLogger(
    SERVICE_NAME
)


def log_json(
    level: str,
    message: str,
    **fields
):
    """
    Produit un log JSON structuré.

    Pourquoi du JSON ?

    Parce qu'un outil comme Loki, Elasticsearch ou Splunk
    peut plus facilement indexer des champs structurés.
    """

    payload = {
        "timestamp": time.time(),
        "level": level,
        "service": SERVICE_NAME,
        "message": message,
        **fields
    }

    logger.info(
        json.dumps(
            payload
        )
    )
```

---

# 14. Configuration OpenTelemetry

Ajouter :

```python
# -------------------------------------------------------------------
# OpenTelemetry
# -------------------------------------------------------------------

resource = Resource.create({
    "service.name": SERVICE_NAME
})


provider = TracerProvider(
    resource=resource
)


exporter = OTLPSpanExporter(
    endpoint=OTEL_ENDPOINT,
    insecure=True
)


processor = BatchSpanProcessor(
    exporter
)


provider.add_span_processor(
    processor
)


trace.set_tracer_provider(
    provider
)


tracer = trace.get_tracer(
    SERVICE_NAME
)
```

---

# 15. Pourquoi BatchSpanProcessor ?

Il n'envoie pas nécessairement chaque span immédiatement.

Il regroupe les exports.

Avantage :

```text
moins d'overhead réseau
```

---

# 16. FastAPI

Ajouter :

```python
app = FastAPI(
    title="Observability Lab"
)


# Instrumentation automatique FastAPI.
FastAPIInstrumentor.instrument_app(
    app
)


# Instrumentation automatique HTTPX.
HTTPXClientInstrumentor().instrument()
```

---

# 17. PostgreSQL

Ajouter :

```python
DATABASE_URL = (
    "postgresql://obs:obs@postgres:5432/obs"
)


engine = create_engine(
    DATABASE_URL,
    pool_pre_ping=True
)


# Instrumente automatiquement les appels SQLAlchemy.
SQLAlchemyInstrumentor().instrument(
    engine=engine
)
```

---

# 18. Redis

```python
redis_client = redis.Redis(
    host="redis",
    port=6379,
    decode_responses=True
)
```

---

# 19. Métriques métier

Ajouter :

```python
REQUESTS_TOTAL = Counter(
    "app_requests_total",
    "Nombre total de requêtes"
)


ERRORS_TOTAL = Counter(
    "app_errors_total",
    "Nombre total d'erreurs"
)


ACTIVE_REQUESTS = Gauge(
    "app_active_requests",
    "Nombre de requêtes en cours"
)


REQUEST_DURATION = Histogram(
    "app_request_duration_seconds",
    "Durée des requêtes",
    buckets=(
        0.01,
        0.05,
        0.1,
        0.25,
        0.5,
        1,
        2,
        5
    )
)
```

---

# 20. Pourquoi histogramme ?

Un Histogram Prometheus permet d'estimer :

```text
p50
p95
p99
```

avec `histogram_quantile`.

---

# 21. Middleware metrics + trace correlation

Ajouter :

```python
@app.middleware("http")
async def observability_middleware(
    request: Request,
    call_next
):

    REQUESTS_TOTAL.inc()

    ACTIVE_REQUESTS.inc()

    started = time.perf_counter()

    try:

        response = await call_next(
            request
        )

        return response

    except Exception:

        ERRORS_TOTAL.inc()

        raise

    finally:

        duration = (
            time.perf_counter()
            - started
        )

        REQUEST_DURATION.observe(
            duration
        )

        ACTIVE_REQUESTS.dec()

        # Récupération du span courant.
        span = trace.get_current_span()

        context = span.get_span_context()

        trace_id = format(
            context.trace_id,
            "032x"
        )

        log_json(
            "INFO",
            "request_completed",
            method=request.method,
            path=request.url.path,
            duration_ms=round(
                duration * 1000,
                2
            ),
            trace_id=trace_id
        )
```

---

# 22. Pourquoi injecter trace_id dans les logs ?

Cela permet de partir d'un log :

```text
ERROR
```

et retrouver immédiatement :

```text
la trace distribuée correspondante
```

---

# 23. Endpoint simple

```python
@app.get("/fast")
async def fast():

    return {
        "status": "ok"
    }
```

---

# 24. Endpoint avec span manuel

```python
@app.get("/work")
async def work():
    """
    Exemple de span manuel.

    Les spans automatiques sont très utiles,
    mais un span métier permet d'observer
    une étape fonctionnelle précise.
    """

    with tracer.start_as_current_span(
        "business_calculation"
    ):

        await asyncio.sleep(
            0.1
        )

    return {
        "status": "done"
    }
```

---

# 25. Span avec attributs

```python
@app.get("/orders/{order_id}")
async def order(
    order_id: str
):

    with tracer.start_as_current_span(
        "load_order"
    ) as span:

        span.set_attribute(
            "order.id",
            order_id
        )

        span.set_attribute(
            "business.domain",
            "erp"
        )

        await asyncio.sleep(
            0.05
        )

    return {
        "order_id": order_id
    }
```

---

# 26. Attention aux attributs

Ne jamais mettre dans les traces :

```text
password
token
carte bancaire
secret
données sensibles inutiles
```

---

# 27. Endpoint SQL

```python
@app.get("/db")
async def db_test():

    with engine.connect() as conn:

        result = conn.execute(
            text("SELECT pg_sleep(0.05), 1")
        )

        result.fetchone()

    return {
        "db": "ok"
    }
```

---

# 28. Endpoint Redis

```python
@app.get("/cache")
async def cache_test():

    key = "demo"

    value = redis_client.get(
        key
    )

    if value is None:

        redis_client.set(
            key,
            "cached-value",
            ex=30
        )

        value = "cached-value"

    return {
        "value": value
    }
```

---

# 29. Endpoint lent

```python
@app.get("/slow")
async def slow(
    delay_ms: int = 1000
):

    await asyncio.sleep(
        delay_ms / 1000
    )

    return {
        "delay_ms": delay_ms
    }
```

---

# 30. Endpoint erreur

```python
@app.get("/error")
async def error():

    ERRORS_TOTAL.inc()

    raise HTTPException(
        status_code=500,
        detail="Simulated error"
    )
```

---

# 31. Exposer /metrics

Ajouter :

```python
Instrumentator().instrument(
    app
).expose(
    app
)
```

---

# 32. Tester metrics

```bash
curl \
  http://localhost:8000/metrics
```

---

# 33. Prometheus configuration

Créer `prometheus/prometheus.yml`.

```yaml
global:

  # Prometheus collecte toutes les 5 secondes.
  scrape_interval: 5s


scrape_configs:

  - job_name: "app"

    static_configs:

      - targets:
          - "app:8000"
```

---

# 34. Démarrer l'environnement

```bash
docker compose up -d --build
```

---

# 35. Prometheus UI

Ouvrir :

```text
http://localhost:9090
```

---

# 36. Requête PromQL simple

```promql
app_requests_total
```

---

# 37. Requêtes par seconde

```promql
rate(app_requests_total[1m])
```

---

# 38. Error rate

```promql
rate(app_errors_total[5m])
```

---

# 39. p95

Avec histogramme :

```promql
histogram_quantile(
  0.95,
  sum(
    rate(
      app_request_duration_seconds_bucket[5m]
    )
  ) by (le)
)
```

---

# 40. p99

```promql
histogram_quantile(
  0.99,
  sum(
    rate(
      app_request_duration_seconds_bucket[5m]
    )
  ) by (le)
)
```

---

# 41. Pourquoi PromQL est puissant ?

Prometheus stocke des séries temporelles.

On peut donc calculer :

```text
rates
percentiles
agrégations
alertes
comparaisons
```

---

# 42. OpenTelemetry Collector

Créer `otel/otel-collector-config.yaml`.

```yaml
receivers:

  # Reçoit OTLP gRPC et HTTP.
  otlp:

    protocols:

      grpc:

      http:


processors:

  # Batch pour réduire le coût d'export.
  batch:


exporters:

  # Export des traces vers Jaeger en OTLP.
  otlp/jaeger:

    endpoint: jaeger:4317

    tls:
      insecure: true

  # Debug local utile pour voir les données dans les logs.
  debug:
    verbosity: basic


service:

  pipelines:

    traces:

      receivers:
        - otlp

      processors:
        - batch

      exporters:
        - otlp/jaeger
        - debug
```

---

# 43. Jaeger

Interface :

```text
http://localhost:16686
```

---

# 44. Générer une trace

```bash
curl \
  http://localhost:8000/orders/123
```

Puis chercher :

```text
service = obs-api
```

dans Jaeger.

---

# 45. Structure d'une trace

```text
trace
|
+-- span HTTP GET /orders/123
    |
    +-- span load_order
```

---

# 46. Traces distribuées

Ajoutons un second service.

Service A :

```text
ERP
```

appelle :

```text
CRM
```

OpenTelemetry propage automatiquement les headers de contexte.

---

# 47. Header traceparent

Exemple :

```text
traceparent:
00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
```

---

# 48. Pourquoi propager ?

Sans propagation :

```text
ERP trace
CRM trace
```

seraient séparées.

Avec propagation :

```text
une trace unique
```

---

# 49. Loki

Loki centralise les logs.

Approche conceptuelle :

```text
containers
   |
Promtail / Alloy
   |
Loki
   |
Grafana
```

---

# 50. Pourquoi Loki ?

Il permet de rechercher :

```text
service
pod
namespace
trace_id
level
```

---

# 51. Logs structurés

Exemple :

```json
{
  "level": "ERROR",
  "service": "warehouse",
  "trace_id": "abc",
  "order_id": "123",
  "message": "stock update failed"
}
```

---

# 52. Correlation ID vs Trace ID

`trace_id` :

```text
géré par le système de tracing
```

`correlation_id` :

```text
identifiant fonctionnel ajouté par l'application
```

Les deux peuvent coexister.

---

# 53. SLI

Service Level Indicator.

Exemples :

```text
latence p95
availability
error rate
successful orders ratio
consumer lag
```

---

# 54. SLO

Service Level Objective.

Exemple :

```text
99 % des requêtes API sous 500 ms
```

---

# 55. SLA

Accord contractuel.

Exemple :

```text
99,9 % de disponibilité mensuelle
```

---

# 56. Error budget

Avec SLO :

```text
99,9 %
```

budget d'erreur :

```text
0,1 %
```

---

# 57. Calcul mensuel

30 jours :

```text
43 200 minutes
```

0,1 % :

```text
43,2 minutes
```

---

# 58. Burn rate

Le burn rate mesure la vitesse à laquelle l'error budget est consommé.

---

# 59. Exemple

Si l'error budget mensuel est consommé en :

```text
2 heures
```

le burn rate est extrêmement élevé.

---

# 60. Alerting basé sur symptômes

Mauvaise alerte :

```text
CPU > 80 %
```

CPU élevé peut être normal.

Meilleure alerte :

```text
p95 très haut
et
error rate augmente
```

---

# 61. Symptôme vs cause

Symptôme :

```text
latence API élevée
```

Cause possible :

```text
PostgreSQL lock
Redis down
DNS lent
Kafka backlog
CPU throttling
```

---

# 62. Golden Signals

Les quatre signaux classiques SRE :

```text
latency
traffic
errors
saturation
```

---

# 63. Latency

```text
p50
p95
p99
```

---

# 64. Traffic

```text
RPS
messages/s
orders/min
```

---

# 65. Errors

```text
HTTP 5xx
timeouts
failed orders
DLQ
```

---

# 66. Saturation

```text
CPU
RAM
DB pool
connection pool
queue depth
Kafka lag
```

---

# 67. Dashboard Grafana recommandé

Créer au minimum :

```text
RPS
p50
p95
p99
error rate
active requests
CPU
RAM
PostgreSQL connections
Redis errors
Kafka lag
```

---

# 68. Locust

Créer `load-tests/locustfile.py`.

```python
from locust import (
    HttpUser,
    between,
    task
)


class ObsUser(HttpUser):

    wait_time = between(
        0.05,
        0.2
    )

    @task(5)
    def fast(self):

        self.client.get(
            "/fast"
        )

    @task(3)
    def work(self):

        self.client.get(
            "/work"
        )

    @task(1)
    def slow(self):

        self.client.get(
            "/slow?delay_ms=500"
        )
```

---

# 69. Lancer Locust

```bash
locust \
  -f load-tests/locustfile.py \
  --host http://localhost:8000
```

---

# 70. Baseline

Avant le chaos :

```text
50 utilisateurs
2 minutes
```

Noter :

```text
RPS
p95
p99
errors
CPU
RAM
```

---

# 71. Pourquoi baseline ?

Sans référence saine :

```text
impossible de mesurer l'impact d'une panne
```

---

# 72. Chaos Engineering

Le chaos engineering ne consiste pas à :

```text
casser au hasard
```

Il consiste à :

```text
formuler une hypothèse
injecter une panne contrôlée
observer
mesurer
apprendre
```

---

# 73. Méthode chaos

```text
1. définir steady state
2. formuler hypothèse
3. injecter panne
4. observer
5. arrêter si dangereux
6. analyser
7. corriger
8. rejouer
```

---

# 74. Steady State

Exemple :

```text
p95 < 500 ms
error rate < 1 %
RPS > 100
```

---

# 75. Hypothèse

Exemple :

```text
"Si Redis tombe pendant 30 secondes,
l'ERP continue de répondre
avec un taux d'erreur inférieur à 5 %."
```

---

# 76. Blast radius

Toujours limiter :

```text
nombre de services touchés
durée
utilisateurs
environnement
```

---

# 77. Chaos 1 - Redis down

```bash
docker compose stop redis
```

---

# 78. Observer

Mesurer :

```text
error rate
p95
logs
traces
recovery
```

---

# 79. Redémarrer

```bash
docker compose start redis
```

---

# 80. Question

Le système :

```text
échoue-t-il complètement ?
```

ou :

```text
dégrade-t-il seulement certaines fonctions ?
```

---

# 81. Chaos 2 - PostgreSQL down

```bash
docker compose stop postgres
```

---

# 82. Observer

Endpoints :

```text
/fast
```

peut continuer.

Mais :

```text
/db
```

doit échouer.

---

# 83. Conclusion pédagogique

Une panne partielle ne doit pas forcément devenir :

```text
panne totale
```

---

# 84. Chaos 3 - Latence réseau

Sous Linux, `tc netem` permet d'injecter de la latence.

Exemple :

```bash
tc qdisc add \
  dev eth0 \
  root \
  netem \
  delay 500ms
```

---

# 85. Supprimer latence

```bash
tc qdisc del \
  dev eth0 \
  root
```

---

# 86. Attention

Exécuter uniquement dans un environnement de test.

---

# 87. Docker netem

Dans un conteneur, il faut généralement la capability :

```text
NET_ADMIN
```

---

# 88. Exemple Compose

```yaml
cap_add:
  - NET_ADMIN
```

---

# 89. Chaos 4 - Packet loss

```bash
tc qdisc add \
  dev eth0 \
  root \
  netem \
  loss 10%
```

---

# 90. Effets possibles

```text
timeouts
retries
latence
erreurs intermittentes
```

---

# 91. Chaos 5 - CPU stress

Installer ou utiliser un conteneur de stress.

Exemple conceptuel :

```bash
stress-ng \
  --cpu 2 \
  --timeout 60s
```

---

# 92. Observer CPU saturation

```text
p95
p99
throughput
```

---

# 93. Chaos 6 - Memory pressure

```bash
stress-ng \
  --vm 1 \
  --vm-bytes 400M \
  --timeout 60s
```

---

# 94. Observer

```text
OOM
GC
latence
restarts
```

---

# 95. Chaos 7 - Kill application

```bash
docker compose kill app
```

---

# 96. Kubernetes equivalent

```bash
kubectl delete pod \
  <POD> \
  -n tp5
```

---

# 97. Mesurer recovery

```text
temps entre crash
et
retour du service
```

---

# 98. MTTD

Mean Time To Detect.

```text
temps moyen avant détection
```

---

# 99. MTTR

Mean Time To Recover.

```text
temps moyen avant récupération
```

---

# 100. Pourquoi les mesurer ?

Une architecture peut être résiliente techniquement mais :

```text
personne ne remarque le problème
```

ou :

```text
la récupération humaine prend trop longtemps
```

---

# 101. Alertmanager

Prometheus peut envoyer des alertes via Alertmanager.

---

# 102. Exemple règle Prometheus

```yaml
groups:

  - name: api-alerts

    rules:

      - alert: HighErrorRate

        expr: |
          rate(app_errors_total[5m])
          /
          rate(app_requests_total[5m])
          > 0.05

        for: 2m

        labels:
          severity: critical

        annotations:
          summary: "High API error rate"
```

---

# 103. Pourquoi `for: 2m` ?

Pour éviter une alerte sur un pic très court.

---

# 104. Alerte p95

```yaml
- alert: HighP95Latency

  expr: |
    histogram_quantile(
      0.95,
      sum(
        rate(
          app_request_duration_seconds_bucket[5m]
        )
      ) by (le)
    ) > 1

  for: 5m
```

---

# 105. Alert fatigue

Trop d'alertes inutiles :

```text
désensibilisent les équipes
```

---

# 106. Une bonne alerte doit être actionnable

La personne doit savoir :

```text
quoi vérifier
où regarder
quelle première action tenter
```

---

# 107. Runbook

Créer :

```text
runbooks/high-api-latency.md
```

---

# 108. Exemple runbook

```text
Incident :
p95 API > 1 seconde

Vérifications :

1. Grafana API dashboard
2. CPU / RAM
3. PostgreSQL connections
4. Redis errors
5. Kafka lag
6. traces lentes dans Jaeger

Actions possibles :

1. identifier dépendance lente
2. vérifier saturation pool
3. scaler API si pertinent
4. désactiver feature non critique
5. réduire trafic
```

---

# 109. Pourquoi un runbook ?

Pendant un incident :

```text
stress
temps limité
informations dispersées
```

Le runbook accélère le diagnostic.

---

# 110. RED Method

Pour les services HTTP :

```text
Rate
Errors
Duration
```

---

# 111. USE Method

Pour les ressources :

```text
Utilization
Saturation
Errors
```

---

# 112. Exemple PostgreSQL

USE :

```text
Utilization = connexions actives
Saturation = attente locks/pool
Errors = failed queries
```

---

# 113. PostgreSQL observabilité

Requête :

```sql
SELECT
    state,
    count(*)
FROM pg_stat_activity
GROUP BY state;
```

---

# 114. Locks

```sql
SELECT
    pid,
    locktype,
    mode,
    granted
FROM pg_locks;
```

---

# 115. Long queries

```sql
SELECT
    pid,
    now() - query_start AS duration,
    query
FROM pg_stat_activity
WHERE state <> 'idle'
ORDER BY duration DESC;
```

---

# 116. Redis observabilité

```bash
redis-cli INFO
```

---

# 117. Redis stats intéressantes

```text
used_memory
connected_clients
instantaneous_ops_per_sec
keyspace_hits
keyspace_misses
```

---

# 118. Cache hit ratio

```text
hits
/
(hits + misses)
```

---

# 119. Kafka observabilité

Métriques importantes :

```text
consumer lag
messages in/sec
bytes in/sec
under replicated partitions
offline partitions
```

---

# 120. Consumer lag

Si :

```text
lag augmente continuellement
```

alors :

```text
consommation < production
```

---

# 121. Chaos Kafka

Arrêter un broker dans un cluster pédagogique.

Observer :

```text
producers
consumers
lag
recovery
```

---

# 122. Un broker unique

Avec un seul broker :

```text
Kafka down = service event-driven indisponible
```

---

# 123. Multi-broker

En production :

```text
3 brokers ou plus
replication factor >= 3
```

selon contexte.

---

# 124. Trace sampling

Tracer 100 % des requêtes peut coûter cher.

---

# 125. Sampling

Exemple :

```text
10 %
```

des requêtes.

---

# 126. Head sampling

Décision prise :

```text
au début de la trace
```

---

# 127. Tail sampling

Décision prise :

```text
après observation de la trace
```

Permet par exemple de conserver :

```text
toutes les erreurs
toutes les traces > 2 s
```

---

# 128. Exemple tail sampling conceptuel

```text
trace normale rapide
=> drop

trace lente
=> keep

trace erreur
=> keep
```

---

# 129. Cardinalité métriques

Mauvaise idée :

```text
label order_id
```

dans Prometheus.

---

# 130. Pourquoi ?

Chaque `order_id` crée une nouvelle série temporelle.

Cela peut exploser :

```text
mémoire
stockage
coût
```

---

# 131. Labels adaptés

Bon :

```text
service
method
status_code
route
environment
```

---

# 132. Logs vs metrics pour haute cardinalité

Pour :

```text
order_id
user_id
trace_id
```

préférer :

```text
logs/traces
```

plutôt que métriques.

---

# 133. Observability overhead

L'observabilité elle-même consomme :

```text
CPU
RAM
réseau
stockage
```

---

# 134. Il faut mesurer son coût

Comparer :

```text
instrumentation désactivée
instrumentation activée
```

---

# 135. Test overhead

Mesurer :

```text
RPS
p95
CPU
```

avant/après instrumentation.

---

# 136. Baggage OpenTelemetry

Le baggage transporte des métadonnées entre services.

---

# 137. Attention baggage

Il est propagé sur de nombreux appels.

Eviter :

```text
gros payload
secret
PII
```

---

# 138. Trace d'un incident

Exemple :

```text
ERP 50 ms
CRM 40 ms
Warehouse 1800 ms
DB Warehouse 1750 ms
```

Conclusion :

```text
la latence se situe dans la DB Warehouse
```

---

# 139. Pourquoi trace distribuée ?

Sans trace :

```text
ERP lent
```

Avec trace :

```text
ERP attend Warehouse
Warehouse attend PostgreSQL
```

---

# 140. Profiling

Observabilité classique ne remplace pas toujours :

```text
profiling CPU
profiling mémoire
```

---

# 141. Outils Python

Exemples :

```text
cProfile
py-spy
memray
```

---

# 142. Continuous profiling

Outils possibles :

```text
Parca
Pyroscope
```

---

# 143. Exercice 1

Construire dashboard Grafana :

```text
RPS
p95
p99
errors
active requests
```

---

# 144. Exercice 2

Ajouter logs JSON avec :

```text
trace_id
correlation_id
order_id
```

---

# 145. Exercice 3

Créer une trace ERP -> CRM.

Vérifier qu'elle apparaît comme une seule trace distribuée.

---

# 146. Exercice 4

Créer span manuel :

```text
validate_customer
reserve_stock
persist_order
```

---

# 147. Exercice 5

Ajouter une métrique :

```text
orders_created_total
```

---

# 148. Exercice 6

Ajouter :

```text
orders_failed_total
```

---

# 149. Exercice 7

Calculer taux d'échec métier.

---

# 150. Exercice 8

Créer SLO :

```text
99 % des requêtes < 500 ms
```

---

# 151. Exercice 9

Calculer error budget mensuel.

---

# 152. Exercice 10

Créer alerte :

```text
error rate > 5 %
```

pendant :

```text
2 minutes
```

---

# 153. Exercice 11

Créer alerte :

```text
p95 > 1 s
```

pendant :

```text
5 minutes
```

---

# 154. Exercice 12

Arrêter Redis.

Mesurer :

```text
MTTD
MTTR
impact
```

---

# 155. Exercice 13

Arrêter PostgreSQL.

Identifier :

```text
symptômes
cause
trace
logs
metrics
```

---

# 156. Exercice 14

Injecter :

```text
500 ms réseau
```

entre ERP et CRM.

---

# 157. Exercice 15

Injecter :

```text
10 % packet loss
```

---

# 158. Exercice 16

Provoquer CPU saturation.

Mesurer p95/p99.

---

# 159. Exercice 17

Provoquer memory pressure.

Observer :

```text
OOM
restarts
latence
```

---

# 160. Exercice 18

Créer une hot row PostgreSQL.

Observer :

```text
locks
latence
traces
```

---

# 161. Exercice 19

Ralentir consumer Kafka.

Observer :

```text
lag
```

---

# 162. Exercice 20

Construire un runbook complet pour :

```text
High API Latency
```

---

# 163. Exercice 21

Construire runbook :

```text
Kafka Consumer Lag High
```

---

# 164. Exercice 22

Construire runbook :

```text
PostgreSQL Connections Exhausted
```

---

# 165. Exercice 23

Faire un test de chaos sous Locust.

---

# 166. Exercice 24

Comparer :

```text
steady state
chaos state
recovery state
```

---

# 167. Tableau d'expérience

| Phase | RPS | p95 | p99 | Errors | CPU | RAM |
|---|---:|---:|---:|---:|---:|---:|
| Baseline |  |  |  |  |  |  |
| Chaos |  |  |  |  |  |  |
| Recovery |  |  |  |  |  |  |

---

# 168. Exercice 25

Mesurer MTTD.

---

# 169. Exercice 26

Mesurer MTTR.

---

# 170. Exercice 27

Mesurer temps de retour aux SLO après incident.

---

# 171. Exercice 28

Simuler :

```text
Redis down + CRM latency
```

mais seulement après avoir testé chaque panne séparément.

---

# 172. Pourquoi combiner les pannes ?

Les incidents réels sont souvent :

```text
multi-facteurs
```

---

# 173. Mais attention

Toujours augmenter la complexité progressivement.

---

# 174. GameDay

Un GameDay est un exercice planifié où l'équipe simule des incidents.

---

# 175. Exemple scénario GameDay

```text
09:00 baseline
09:10 Redis down
09:20 recovery
09:30 Kafka lag
09:45 CRM latency
10:00 post-mortem
```

---

# 176. Post-mortem

Après incident :

```text
faits
timeline
impact
cause
détection
résolution
actions
```

---

# 177. Blameless post-mortem

On cherche :

```text
amélioration système
```

plutôt que :

```text
responsable individuel
```

---

# 178. Timeline

Exemple :

```text
10:01 incident commence
10:03 alerte déclenchée
10:05 diagnostic commence
10:09 cause identifiée
10:12 mitigation
10:15 service normal
```

---

# 179. MTTD

```text
10:03 - 10:01
= 2 min
```

---

# 180. MTTR

```text
10:15 - 10:01
= 14 min
```

---

# 181. Action item

Exemple :

```text
ajouter alerte sur DB pool saturation
```

---

# 182. Action item mesurable

Mieux :

```text
ajouter avant vendredi une alerte
DB pool > 90 %
pendant 5 min
```

---

# 183. Alert severity

Exemples :

```text
info
warning
critical
```

---

# 184. Critical

Doit généralement signifier :

```text
action humaine urgente
```

---

# 185. Warning

Peut signifier :

```text
dégradation
mais
pas d'action immédiate
```

---

# 186. Paging

Ne pager quelqu'un que si :

```text
il existe une action concrète
```

---

# 187. Logs sampling

Sous très forte charge :

```text
log par requête
```

peut coûter cher.

---

# 188. Solution

```text
sample logs normaux
garder toutes les erreurs
```

---

# 189. Structured context

Chaque log critique devrait inclure :

```text
service
environment
trace_id
correlation_id
operation
duration
result
```

---

# 190. Logs métier

Exemple :

```json
{
  "event": "order_confirmed",
  "order_id": "123",
  "duration_ms": 420
}
```

---

# 191. Logs techniques

Exemple :

```json
{
  "event": "db_timeout",
  "pool_in_use": 20,
  "pool_size": 20
}
```

---

# 192. Ne pas mélanger sans convention

Définir :

```text
event_type
category
severity
```

---

# 193. Dashboard métier

Exemples :

```text
orders/min
payment failure ratio
stock rejection ratio
```

---

# 194. Dashboard technique

Exemples :

```text
RPS
latency
CPU
RAM
pool usage
lag
```

---

# 195. Pourquoi les deux ?

Un système peut être techniquement sain mais métier cassé.

Exemple :

```text
200 OK partout
mais
100 % des paiements refusés
```

---

# 196. Synthetic monitoring

Un robot appelle régulièrement :

```text
health
login
checkout
order
```

---

# 197. Pourquoi ?

Pour détecter :

```text
expérience utilisateur cassée
```

même si les métriques internes semblent normales.

---

# 198. Black-box monitoring

Observe depuis l'extérieur.

---

# 199. White-box monitoring

Observe l'intérieur du système.

---

# 200. Les deux sont nécessaires

Externe :

```text
l'utilisateur voit une erreur
```

Interne :

```text
DB pool saturé
```

---

# 201. Exercice final de chaos

Hypothèse :

```text
Si PostgreSQL devient lent,
l'API dégrade sa latence
mais reste disponible à plus de 95 %.
```

---

# 202. Etapes

```text
1. baseline 2 min
2. injecter 1 s de latence DB
3. maintenir charge 5 min
4. observer dashboards
5. supprimer latence
6. mesurer recovery
```

---

# 203. Mesures obligatoires

```text
p50
p95
p99
error rate
RPS
active requests
DB connections
trace duration
```

---

# 204. Conclusion attendue

Exemple :

```text
Hypothèse non validée.
A 1 s de latence DB,
p99 dépasse 8 s
et error rate atteint 12 %.
```

---

# 205. Remédiation

Exemples :

```text
timeout plus court
bulkhead
load shedding
cache
pool tuning
circuit breaker
```

---

# 206. Rejouer l'expérience

Après correction :

```text
mêmes conditions
```

---

# 207. Comparer

| Métrique | Avant | Après |
|---|---:|---:|
| p95 |  |  |
| p99 |  |  |
| error rate |  |  |
| RPS |  |  |
| recovery time |  |  |

---

# 208. Questions de synthèse

1. Quelle différence entre logs, metrics et traces ?
2. A quoi sert OpenTelemetry ?
3. Pourquoi propager un trace ID ?
4. Pourquoi ajouter trace ID dans les logs ?
5. Qu'est-ce qu'un span ?
6. A quoi sert Prometheus ?
7. A quoi sert Grafana ?
8. A quoi sert Loki ?
9. Quelle différence entre p95 et moyenne ?
10. Qu'est-ce qu'un SLI ?
11. Qu'est-ce qu'un SLO ?
12. Qu'est-ce qu'un error budget ?
13. Qu'est-ce qu'un burn rate ?
14. Que sont les Golden Signals ?
15. Que signifie MTTD ?
16. Que signifie MTTR ?
17. Qu'est-ce qu'un runbook ?
18. Qu'est-ce que le chaos engineering ?
19. Pourquoi définir un blast radius ?
20. Pourquoi établir une baseline avant un test de panne ?

---

# 209. Corrections synthétiques

## 1. Logs / metrics / traces

Logs :

```text
événements détaillés
```

Metrics :

```text
mesures agrégées dans le temps
```

Traces :

```text
chemin d'une opération distribuée
```

---

## 2. OpenTelemetry

Standard d'instrumentation pour traces, metrics, logs et propagation de contexte.

---

## 3. Trace ID

Il relie les appels d'une même requête distribuée.

---

## 4. Trace ID dans logs

Il permet de naviguer d'un log vers une trace.

---

## 5. Span

Une étape d'une trace.

---

## 6. Prometheus

Collecte et interroge des métriques temporelles.

---

## 7. Grafana

Visualise les données et construit dashboards/alertes.

---

## 8. Loki

Centralise et recherche les logs.

---

## 9. p95

95 % des observations sont sous cette valeur.

---

## 10. SLI

Indicateur réel mesuré.

---

## 11. SLO

Objectif cible associé à un SLI.

---

## 12. Error budget

Marge d'erreur autorisée par le SLO.

---

## 13. Burn rate

Vitesse de consommation de l'error budget.

---

## 14. Golden Signals

```text
latency
traffic
errors
saturation
```

---

## 15. MTTD

Temps moyen avant détection.

---

## 16. MTTR

Temps moyen avant récupération.

---

## 17. Runbook

Procédure de diagnostic et d'action.

---

## 18. Chaos engineering

Expérimentation contrôlée sur des défaillances afin de valider la résilience.

---

## 19. Blast radius

Périmètre maximal volontairement impacté.

---

## 20. Baseline

Elle fournit l'état normal de référence.

---

# 210. Barème proposé

| Critère | Points |
|---|---:|
| OpenTelemetry | 2 |
| Traces distribuées | 2 |
| Prometheus | 2 |
| Grafana | 1.5 |
| Logs structurés | 1.5 |
| Loki | 1 |
| SLI / SLO / error budget | 2 |
| Alerting | 1 |
| Chaos Redis/PostgreSQL | 1.5 |
| Latence / packet loss | 1 |
| CPU / mémoire | 1 |
| MTTD / MTTR | 1 |
| Runbook | 1 |
| Rapport chaos final | 1.5 |
| **Total** | **20** |

---

# 211. Livrables

```text
tp6-observability-chaos/
|
|-- docker-compose.yml
|
|-- app/
|
|-- otel/
|-- prometheus/
|-- grafana/
|-- loki/
|
|-- chaos/
|-- load-tests/
|-- runbooks/
|
|-- README.md
```

---

# 212. README attendu

Le README doit contenir :

1. architecture ;
2. stack observabilité ;
3. démarrage ;
4. OpenTelemetry ;
5. Prometheus ;
6. Grafana ;
7. Loki ;
8. Jaeger/Tempo ;
9. trace distribuée ;
10. logs corrélés ;
11. p95/p99 ;
12. SLI ;
13. SLO ;
14. error budget ;
15. alertes ;
16. baseline ;
17. scénarios chaos ;
18. résultats ;
19. MTTD ;
20. MTTR ;
21. runbooks ;
22. post-mortem ;
23. limites ;
24. pistes d'amélioration.

---

# 213. Extensions expert++

Pour aller encore plus loin :

- Grafana Tempo ;
- Grafana Alloy ;
- Prometheus Operator ;
- ServiceMonitor ;
- Alertmanager ;
- OpenTelemetry Collector avancé ;
- tail sampling ;
- exemplar metrics ;
- exemplars Prometheus -> traces ;
- continuous profiling ;
- Pyroscope ;
- Parca ;
- eBPF ;
- Cilium Hubble ;
- Chaos Mesh ;
- LitmusChaos ;
- Toxiproxy ;
- Pumba ;
- tc/netem ;
- KEDA ;
- SLO tooling ;
- Sloth ;
- Pyrra ;
- OpenSLO ;
- multi-cluster observability ;
- synthetic monitoring ;
- blackbox exporter ;
- cloud managed observability ;
- SIEM integration ;
- incident automation ;
- PagerDuty ;
- Opsgenie ;
- GitOps runbooks ;
- automatic remediation.

---

# 214. Conclusion

Ce TP ajoute la couche indispensable qui manque souvent aux architectures distribuées :

```text
voir
comprendre
détecter
réagir
apprendre
```

Une architecture moderne ne doit pas seulement :

```text
fonctionner
```

Elle doit aussi permettre de répondre rapidement à :

```text
Pourquoi est-elle lente ?
Quel service échoue ?
Depuis quand ?
Combien d'utilisateurs sont impactés ?
Quel est le chemin exact de la requête ?
Le système récupère-t-il seul ?
Combien de temps faut-il pour revenir au SLO ?
```

Les notions essentielles abordées sont :

- logs ;
- métriques ;
- traces ;
- OpenTelemetry ;
- Prometheus ;
- Grafana ;
- Loki ;
- Jaeger / Tempo ;
- SLI ;
- SLO ;
- SLA ;
- error budget ;
- burn rate ;
- alerting ;
- MTTD ;
- MTTR ;
- chaos engineering ;
- runbooks ;
- post-mortems ;
- résilience expérimentale.

L'objectif final n'est pas d'avoir un système qui ne tombe jamais.

L'objectif est d'avoir un système :

```text
observable
prévisible
détectable
récupérable
et améliorable après chaque incident
```
