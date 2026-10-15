# TP 4 - Performance, haute concurrence et résilience d'une architecture distribuée


## Charge, latence, saturation, pools, backpressure, load shedding, tuning, Kafka lag, PostgreSQL, Redis, Locust et chaos testing

**Niveau :** expert+  
**Durée conseillée :** 2 h à 3 h  
**Architecture :** distribuée sous forte charge  
**Technologies :** Python 3.12+, FastAPI, PostgreSQL, Redis, Kafka, Docker, Docker Compose, HTTPX, Locust, Pytest  
**Systèmes :** Linux, macOS, Windows PowerShell

---

# 1. Objectifs du TP

À la fin de ce TP, vous serez capable de :

- mesurer la performance réelle d'une API distribuée ;
- distinguer **débit**, **latence**, **concurrence** et **saturation** ;
- comprendre la différence entre moyenne, p50, p95 et p99 ;
- identifier un goulot d'étranglement ;
- configurer des pools HTTP et SQL ;
- comprendre le rôle du pool PostgreSQL ;
- limiter la concurrence avec des semaphores ;
- implémenter de la **backpressure** ;
- implémenter du **load shedding** ;
- détecter une saturation ;
- limiter les requêtes coûteuses ;
- tester différentes tailles de pools ;
- observer la contention SQL ;
- mesurer l'impact des locks ;
- mesurer le consumer lag Kafka ;
- simuler une dépendance lente ;
- tester le système à 10, 50, 100, 500 et 1000 utilisateurs ;
- analyser les effets d'un timeout trop long ;
- comprendre les retry storms ;
- configurer des retries raisonnables ;
- comprendre Little's Law ;
- suivre CPU, mémoire, réseau et I/O ;
- comparer les performances avant/après optimisation ;
- produire un rapport de performance reproductible.

---

# 2. Contexte

Nous reprenons l'architecture des TP précédents.

Le système contient :

```text
API ERP
CRM
Warehouse
Payment
PostgreSQL
Redis
Kafka
```

Cette fois, nous ne cherchons plus seulement à vérifier :

```text
"Est-ce que ça marche ?"
```

Nous allons répondre à :

```text
"Combien de charge le système supporte-t-il ?"
```

et surtout :

```text
"Que se passe-t-il quand il commence à saturer ?"
```

---

# 3. Architecture de test

```mermaid
flowchart LR
    L[Locust]
    G[ERP API]
    C[CRM]
    W[Warehouse]
    P[Payment]
    R[(Redis)]
    EDB[(PostgreSQL ERP)]
    K[Kafka]

    L --> G

    G --> C
    G --> EDB
    G --> R
    G --> K

    K --> W
    K --> P
```

---

# 4. Notions fondamentales

## 4.1 Débit

Le débit est souvent exprimé en :

```text
requêtes par seconde
```

ou :

```text
RPS
```

Exemple :

```text
500 RPS
```

signifie :

```text
500 requêtes traitées par seconde
```

---

## 4.2 Latence

La latence correspond au temps entre :

```text
envoi requête
et
réponse reçue
```

Exemple :

```text
120 ms
```

---

## 4.3 Concurrence

La concurrence représente le nombre de requêtes en cours simultanément.

Exemple :

```text
50 requêtes sont actuellement actives
```

---

## 4.4 Saturation

Un système sature lorsqu'une ressource atteint sa capacité pratique.

Exemples :

```text
CPU = 100 %
pool DB saturé
pool HTTP saturé
Kafka lag en hausse
mémoire proche de la limite
file d'attente qui augmente
```

---

# 5. Little's Law

Dans un système stable :

```text
L = λ × W
```

où :

```text
L = nombre moyen de requêtes dans le système
λ = débit moyen
W = temps moyen passé dans le système
```

Exemple :

```text
λ = 200 RPS
W = 0,25 s
```

Alors :

```text
L = 200 × 0,25
L = 50
```

Le système traite environ :

```text
50 requêtes simultanées
```

en moyenne.

---

# 6. Pourquoi cette formule est utile ?

Si la latence augmente mais que le débit reste stable :

```text
le nombre de requêtes simultanées augmente
```

Donc :

```text
plus de sockets
plus de mémoire
plus de connexions
plus de pression sur les pools
```

---

# 7. Moyenne vs percentiles

Une moyenne peut masquer de gros problèmes.

Exemple :

```text
moyenne = 120 ms
p50 = 70 ms
p95 = 500 ms
p99 = 2200 ms
```

Cela signifie :

```text
1 % des requêtes prennent plus de 2,2 secondes
```

---

# 8. Architecture Docker Compose

Créer :

```text
tp4-performance/
```

avec :

```text
tp4-performance/
|
|-- docker-compose.yml
|-- .env
|
|-- api/
|   |-- Dockerfile
|   |-- requirements.txt
|   |-- app/
|       |-- __init__.py
|       |-- main.py
|
|-- crm/
|-- load-tests/
|   |-- locustfile.py
|   |-- scenarios/
|
|-- tests/
|-- scripts/
|-- README.md
```

---

# 9. Docker Compose

Créer `docker-compose.yml`.

```yaml
services:

  # ------------------------------------------------------------------
  # PostgreSQL
  # ------------------------------------------------------------------
  postgres:
    image: postgres:16
    container_name: perf-postgres

    environment:
      POSTGRES_USER: perf
      POSTGRES_PASSWORD: perf
      POSTGRES_DB: perf

    ports:
      - "5432:5432"

    volumes:
      - pg_data:/var/lib/postgresql/data

    # Configuration volontairement petite au début.
    # Nous la modifierons pendant le TP.
    command:
      - "postgres"
      - "-c"
      - "max_connections=100"

    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U perf -d perf"]
      interval: 5s
      timeout: 3s
      retries: 20


  # ------------------------------------------------------------------
  # Redis
  # ------------------------------------------------------------------
  redis:
    image: redis:7
    container_name: perf-redis

    ports:
      - "6379:6379"


  # ------------------------------------------------------------------
  # Kafka
  # ------------------------------------------------------------------
  kafka:
    image: bitnami/kafka:latest
    container_name: perf-kafka

    environment:
      KAFKA_CFG_NODE_ID: 1
      KAFKA_CFG_PROCESS_ROLES: broker,controller
      KAFKA_CFG_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093
      KAFKA_CFG_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_CFG_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      KAFKA_CFG_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      ALLOW_PLAINTEXT_LISTENER: "yes"

    ports:
      - "9092:9092"


  # ------------------------------------------------------------------
  # CRM simulé
  # ------------------------------------------------------------------
  crm:
    build: ./crm
    container_name: perf-crm

    ports:
      - "8001:8000"


  # ------------------------------------------------------------------
  # API principale
  # ------------------------------------------------------------------
  api:
    build: ./api
    container_name: perf-api

    environment:
      DATABASE_URL: postgresql://perf:perf@postgres:5432/perf
      REDIS_URL: redis://redis:6379/0
      CRM_URL: http://crm:8000
      KAFKA_BOOTSTRAP_SERVERS: kafka:9092

      # Paramètres de pool.
      DB_POOL_SIZE: 10
      DB_MAX_OVERFLOW: 20

      # Limite concurrence vers CRM.
      CRM_MAX_CONCURRENCY: 20

    ports:
      - "8000:8000"

    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
      kafka:
        condition: service_started
      crm:
        condition: service_started


volumes:
  pg_data:
```

---

# 10. Pourquoi commencer avec des limites modestes ?

Parce qu'un TP de performance doit permettre d'observer :

```text
le point de saturation
```

Si tout est surdimensionné dès le départ :

```text
on apprend moins
```

---

# 11. Dépendances API

Créer `api/requirements.txt`.

```txt
fastapi
uvicorn[standard]
sqlalchemy
psycopg2-binary
httpx
redis
confluent-kafka
pydantic
```

---

# 12. Dockerfile API

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY app ./app

CMD [
  "uvicorn",
  "app.main:app",
  "--host",
  "0.0.0.0",
  "--port",
  "8000"
]
```

---

# 13. API FastAPI

Créer `api/app/main.py`.

```python
import asyncio
import os
import time
from contextlib import asynccontextmanager
from datetime import datetime, timezone
from uuid import uuid4

import httpx
import redis

from fastapi import (
    FastAPI,
    HTTPException,
    Request,
    Response
)

from pydantic import BaseModel

from sqlalchemy import (
    Column,
    DateTime,
    Integer,
    String,
    create_engine,
    text
)

from sqlalchemy.orm import (
    declarative_base,
    sessionmaker
)


# -------------------------------------------------------------------
# Configuration
# -------------------------------------------------------------------

DATABASE_URL = os.environ["DATABASE_URL"]
REDIS_URL = os.environ["REDIS_URL"]
CRM_URL = os.environ["CRM_URL"]

DB_POOL_SIZE = int(
    os.getenv(
        "DB_POOL_SIZE",
        "10"
    )
)

DB_MAX_OVERFLOW = int(
    os.getenv(
        "DB_MAX_OVERFLOW",
        "20"
    )
)

CRM_MAX_CONCURRENCY = int(
    os.getenv(
        "CRM_MAX_CONCURRENCY",
        "20"
    )
)


# -------------------------------------------------------------------
# PostgreSQL
# -------------------------------------------------------------------

engine = create_engine(
    DATABASE_URL,

    # Nombre de connexions conservées dans le pool.
    pool_size=DB_POOL_SIZE,

    # Connexions temporaires supplémentaires autorisées
    # lorsque le pool principal est plein.
    max_overflow=DB_MAX_OVERFLOW,

    # Temps maximum d'attente pour obtenir une connexion.
    # Si le pool reste saturé plus longtemps, SQLAlchemy lève une erreur.
    pool_timeout=2,

    # Vérifie qu'une connexion du pool est encore valide.
    pool_pre_ping=True
)


SessionLocal = sessionmaker(
    bind=engine,
    autocommit=False,
    autoflush=False
)


Base = declarative_base()


class RequestLogDB(Base):
    """
    Table utilisée pour simuler une écriture SQL par requête.

    Cela nous permet d'étudier :
    - le pool de connexions ;
    - la contention ;
    - la latence SQL ;
    - l'effet d'une forte concurrence.
    """

    __tablename__ = "request_logs"

    id = Column(
        String(36),
        primary_key=True
    )

    created_at = Column(
        DateTime(timezone=True),
        nullable=False
    )

    duration_ms = Column(
        Integer,
        nullable=False
    )


Base.metadata.create_all(engine)


# -------------------------------------------------------------------
# Redis
# -------------------------------------------------------------------

redis_client = redis.from_url(
    REDIS_URL,
    decode_responses=True
)


# -------------------------------------------------------------------
# Limitation de concurrence
# -------------------------------------------------------------------

# Ce sémaphore protège le CRM.
#
# Même si 1000 requêtes arrivent dans l'ERP,
# seulement 20 appels CRM seront exécutés simultanément.
crm_semaphore = asyncio.Semaphore(
    CRM_MAX_CONCURRENCY
)


# -------------------------------------------------------------------
# Client HTTP partagé
# -------------------------------------------------------------------

http_client: httpx.AsyncClient | None = None


@asynccontextmanager
async def lifespan(app: FastAPI):
    """
    Démarrage et arrêt de l'application.

    Nous créons un seul client HTTP partagé afin de bénéficier
    du connection pooling HTTP.

    C'est plus efficace que créer un nouveau client pour chaque requête.
    """

    global http_client

    http_client = httpx.AsyncClient(
        timeout=httpx.Timeout(
            connect=1.0,
            read=2.0,
            write=2.0,
            pool=1.0
        ),

        limits=httpx.Limits(
            # Nombre total de connexions simultanées.
            max_connections=100,

            # Nombre de connexions gardées ouvertes.
            max_keepalive_connections=20
        )
    )

    yield

    await http_client.aclose()


app = FastAPI(
    title="Performance Lab API",
    lifespan=lifespan
)
```

---

# 14. Pourquoi un client HTTP partagé ?

Créer ceci à chaque requête :

```python
async with httpx.AsyncClient() as client:
```

fonctionne.

Mais cela peut provoquer :

```text
plus de connexions TCP
plus de handshakes
moins de réutilisation
plus de latence
```

Un client partagé réutilise mieux les connexions.

---

# 15. Middleware de mesure

Ajouter :

```python
@app.middleware("http")
async def performance_middleware(
    request: Request,
    call_next
):
    """
    Mesure la durée totale de chaque requête HTTP.

    On utilise time.perf_counter() car il est adapté
    aux mesures de durée.
    """

    started = time.perf_counter()

    response = await call_next(
        request
    )

    duration_ms = (
        time.perf_counter()
        - started
    ) * 1000

    # Header très pratique pour les tests manuels.
    response.headers[
        "X-Response-Time-ms"
    ] = f"{duration_ms:.2f}"

    return response
```

---

# 16. Endpoint rapide

```python
@app.get("/fast")
async def fast():

    return {
        "status": "ok"
    }
```

Cet endpoint sert de référence.

Il permet de mesurer :

```text
latence FastAPI minimale
```

dans notre environnement.

---

# 17. Endpoint CPU

Ajouter :

```python
@app.get("/cpu")
async def cpu_work(
    iterations: int = 200_000
):
    """
    Simule un travail CPU.

    Attention :
    cette boucle Python bloque l'event loop.

    C'est volontaire pour montrer qu'asyncio n'améliore pas
    automatiquement les traitements CPU-bound.
    """

    total = 0

    for i in range(iterations):
        total += i * i

    return {
        "result": total
    }
```

---

# 18. CPU-bound vs I/O-bound

## I/O-bound

Exemples :

```text
HTTP
PostgreSQL
Redis
Kafka
filesystem
```

L'async est très utile.

## CPU-bound

Exemples :

```text
compression
cryptographie lourde
traitement image
calcul scientifique
```

L'event loop peut être bloquée.

---

# 19. Endpoint I/O simulé

```python
@app.get("/io")
async def io_wait(
    delay_ms: int = 200
):
    """
    Simule une attente non bloquante.
    """

    await asyncio.sleep(
        delay_ms / 1000
    )

    return {
        "delay_ms": delay_ms
    }
```

---

# 20. Comparer CPU et I/O

Sous 100 utilisateurs concurrents :

```text
/cpu
```

peut fortement dégrader les autres endpoints.

Alors que :

```text
/io
```

libère l'event loop pendant l'attente.

---

# 21. CRM simulé

Créer `crm/app/main.py`.

```python
import asyncio

from fastapi import (
    FastAPI,
    HTTPException,
    Query
)


app = FastAPI(
    title="Slow CRM Simulator"
)


@app.get("/customers/{customer_id}")
async def get_customer(
    customer_id: int,

    # Latence artificielle.
    delay_ms: int = Query(
        default=50,
        ge=0,
        le=10000
    ),

    # Permet de simuler des erreurs.
    fail: bool = False
):
    """
    Simule un CRM externe.

    asyncio.sleep ne bloque pas l'event loop.
    """

    await asyncio.sleep(
        delay_ms / 1000
    )

    if fail:
        raise HTTPException(
            status_code=500,
            detail="Simulated CRM failure"
        )

    return {
        "id": customer_id,
        "active": True
    }
```

---

# 22. Endpoint appel CRM

Dans l'API principale :

```python
@app.get("/customer/{customer_id}")
async def customer(
    customer_id: int
):
    """
    Appel CRM protégé par un sémaphore.

    Le sémaphore agit comme une limite de concurrence.
    """

    assert http_client is not None

    async with crm_semaphore:

        try:

            response = await http_client.get(
                f"{CRM_URL}/customers/{customer_id}"
            )

        except httpx.PoolTimeout:

            raise HTTPException(
                status_code=503,
                detail="HTTP pool saturated"
            )

        except httpx.TimeoutException:

            raise HTTPException(
                status_code=504,
                detail="CRM timeout"
            )

    response.raise_for_status()

    return response.json()
```

---

# 23. Pourquoi un sémaphore ?

Sans limite :

```text
1000 requêtes API
=> 1000 appels CRM potentiels
```

Cela peut détruire le CRM.

Avec :

```text
Semaphore(20)
```

nous imposons :

```text
maximum 20 appels CRM simultanés
```

---

# 24. Effet secondaire

Limiter la concurrence protège le CRM.

Mais :

```text
les autres requêtes attendent
```

Donc la latence peut augmenter.

---

# 25. Backpressure

La backpressure signifie :

```text
le système ralentit ou refuse une partie du trafic
pour éviter une saturation incontrôlée
```

---

# 26. Endpoint SQL

Ajouter :

```python
@app.post("/sql-write")
async def sql_write():

    started = time.perf_counter()

    db = SessionLocal()

    try:

        # Petite requête volontairement simple.
        db.execute(
            text("SELECT 1")
        )

        duration_ms = int(
            (
                time.perf_counter()
                - started
            )
            * 1000
        )

        row = RequestLogDB(
            id=str(uuid4()),
            created_at=datetime.now(
                timezone.utc
            ),
            duration_ms=duration_ms
        )

        db.add(row)

        db.commit()

        return {
            "duration_ms": duration_ms
        }

    except Exception:

        db.rollback()
        raise

    finally:

        db.close()
```

---

# 27. Pool PostgreSQL

Avec :

```text
pool_size = 10
max_overflow = 20
```

maximum pratique côté SQLAlchemy :

```text
30 connexions
```

simultanées.

---

# 28. Attention à max_connections

Si PostgreSQL autorise :

```text
100 connexions
```

et vous lancez :

```text
5 instances API
```

avec :

```text
30 connexions max chacune
```

le total théorique est :

```text
150
```

Donc :

```text
PostgreSQL peut saturer
```

---

# 29. Exercice de calcul

Supposons :

```text
PostgreSQL max_connections = 200
réserver 20 connexions admin/maintenance
10 pods API
```

Budget disponible :

```text
180 / 10 = 18
```

par pod environ.

---

# 30. Load shedding

Backpressure signifie souvent :

```text
attendre
```

Load shedding signifie :

```text
refuser volontairement
```

avant que le système ne s'écroule.

---

# 31. Compteur de requêtes actives

Ajouter :

```python
active_requests = 0
active_requests_lock = asyncio.Lock()

MAX_ACTIVE_REQUESTS = 100
```

Puis middleware :

```python
@app.middleware("http")
async def overload_protection(
    request: Request,
    call_next
):
    global active_requests

    async with active_requests_lock:

        if active_requests >= MAX_ACTIVE_REQUESTS:

            return Response(
                content=(
                    '{"detail":"Server overloaded"}'
                ),
                status_code=503,
                media_type="application/json"
            )

        active_requests += 1

    try:

        return await call_next(
            request
        )

    finally:

        async with active_requests_lock:
            active_requests -= 1
```

---

# 32. Pourquoi refuser des requêtes ?

Parce que :

```text
100 requêtes lentes
```

peuvent être préférables à :

```text
1000 requêtes toutes bloquées
```

---

# 33. Fail fast

Principe :

```text
si on sait qu'on ne peut pas traiter correctement,
répondre rapidement
```

plutôt que :

```text
laisser le client attendre 30 secondes
```

---

# 34. Rate limiting Redis

Créer :

```python
def rate_limit(
    client_id: str,
    limit: int = 100
) -> bool:

    key = (
        f"rate:"
        f"{client_id}:"
        f"{int(time.time() // 60)}"
    )

    count = redis_client.incr(
        key
    )

    if count == 1:
        redis_client.expire(
            key,
            65
        )

    return count <= limit
```

---

# 35. Pourquoi Redis ?

Parce que plusieurs instances API peuvent partager le même compteur.

---

# 36. Limite du rate limit fixe

La fenêtre :

```text
minute actuelle
```

peut créer un effet frontière.

Exemple :

```text
100 requêtes à 12:00:59
100 requêtes à 12:01:01
```

200 requêtes en 2 secondes.

---

# 37. Alternatives

Plus avancées :

```text
sliding window
token bucket
leaky bucket
```

---

# 38. Locust

Créer `load-tests/locustfile.py`.

```python
import random

from locust import (
    HttpUser,
    between,
    task
)


class PerformanceUser(HttpUser):
    """
    Utilisateur virtuel.

    Plusieurs tâches simulent différents comportements.
    """

    wait_time = between(
        0.05,
        0.2
    )


    @task(5)
    def fast_endpoint(self):

        self.client.get(
            "/fast"
        )


    @task(3)
    def crm_endpoint(self):

        customer_id = random.randint(
            1,
            10000
        )

        self.client.get(
            f"/customer/{customer_id}"
        )


    @task(2)
    def sql_endpoint(self):

        self.client.post(
            "/sql-write"
        )
```

---

# 39. Pourquoi pondérer les tâches ?

`@task(5)` signifie que cette tâche est choisie plus souvent que `@task(2)`.

Cela simule une charge plus réaliste.

---

# 40. Installer Locust

```bash
pip install locust
```

---

# 41. Démarrer Locust

```bash
locust \
  -f load-tests/locustfile.py \
  --host http://localhost:8000
```

Interface :

```text
http://localhost:8089
```

---

# 42. Scénario 1 - Baseline

Tester :

```text
10 utilisateurs
spawn rate = 2/s
durée = 60 s
```

Noter :

```text
RPS
mean
median
p95
p99
errors
```

---

# 43. Scénario 2 - Charge moyenne

Tester :

```text
50 utilisateurs
spawn rate = 10/s
```

---

# 44. Scénario 3 - Forte charge

Tester :

```text
200 utilisateurs
spawn rate = 20/s
```

---

# 45. Scénario 4 - Saturation

Tester :

```text
500 utilisateurs
spawn rate = 50/s
```

---

# 46. Scénario 5 - Stress test

Tester :

```text
1000 utilisateurs
spawn rate = 100/s
```

---

# 47. Test headless

```bash
locust \
  -f load-tests/locustfile.py \
  --host http://localhost:8000 \
  --headless \
  -u 200 \
  -r 20 \
  -t 2m
```

---

# 48. Export CSV

```bash
locust \
  -f load-tests/locustfile.py \
  --host http://localhost:8000 \
  --headless \
  -u 200 \
  -r 20 \
  -t 2m \
  --csv results/run-200
```

---

# 49. Pourquoi exporter ?

Pour comparer :

```text
avant optimisation
après optimisation
```

---

# 50. Docker stats

Pendant un test :

```bash
docker stats
```

Observer :

```text
CPU %
Memory
Network I/O
Block I/O
```

---

# 51. Interpréter CPU à 100 %

Si API :

```text
CPU = 100 %
```

et les p95/p99 montent :

```text
CPU peut être le bottleneck
```

---

# 52. Interpréter CPU faible + latence élevée

Possible cause :

```text
attente réseau
pool saturé
DB lente
lock
queue
```

---

# 53. Observer PostgreSQL

Entrer :

```bash
docker compose exec postgres \
  psql -U perf -d perf
```

---

# 54. Connexions PostgreSQL

```sql
SELECT
    state,
    COUNT(*)
FROM pg_stat_activity
GROUP BY state;
```

---

# 55. Requêtes actives

```sql
SELECT
    pid,
    state,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
WHERE state <> 'idle';
```

---

# 56. Pourquoi wait_event est utile ?

Il permet de voir si PostgreSQL attend :

```text
lock
I/O
client
transaction
```

---

# 57. Locks PostgreSQL

```sql
SELECT
    pid,
    locktype,
    mode,
    granted
FROM pg_locks
ORDER BY pid;
```

---

# 58. Simuler une contention SQL

Créer une table compteur.

```sql
CREATE TABLE counters (
    id INT PRIMARY KEY,
    value INT NOT NULL
);

INSERT INTO counters(id, value)
VALUES (1, 0);
```

---

# 59. Endpoint de contention

Ajouter :

```python
@app.post("/counter")
async def increment_counter():

    db = SessionLocal()

    try:

        # Cette requête verrouille la ligne.
        db.execute(
            text(
                """
                SELECT value
                FROM counters
                WHERE id = 1
                FOR UPDATE
                """
            )
        )

        db.execute(
            text(
                """
                UPDATE counters
                SET value = value + 1
                WHERE id = 1
                """
            )
        )

        db.commit()

        return {
            "status": "ok"
        }

    except:

        db.rollback()
        raise

    finally:

        db.close()
```

---

# 60. Pourquoi cet endpoint ralentira ?

Toutes les requêtes modifient :

```text
la même ligne
```

Donc elles se sérialisent.

---

# 61. Hot row

Une ligne fortement modifiée devient une :

```text
hot row
```

---

# 62. Solutions possibles

Selon le métier :

```text
sharding logique
compteurs distribués
batch
agrégation asynchrone
Redis INCR
Kafka
```

---

# 63. Test pool size

Tester successivement :

```text
DB_POOL_SIZE=5
DB_POOL_SIZE=10
DB_POOL_SIZE=20
DB_POOL_SIZE=40
```

---

# 64. Pourquoi plus grand n'est pas toujours meilleur ?

Trop de connexions :

```text
plus de context switching
plus de mémoire PostgreSQL
plus de contention
```

---

# 65. Courbe typique

```text
petit pool
=> attente pool

pool optimal
=> meilleur débit

pool trop grand
=> DB saturée
```

---

# 66. HTTP connection pool

Tester :

```python
max_connections=20
```

puis :

```python
max_connections=100
```

puis :

```python
max_connections=500
```

---

# 67. Pool timeout

Si :

```text
toutes les connexions HTTP sont utilisées
```

HTTPX attend une connexion disponible.

Si délai dépassé :

```text
PoolTimeout
```

---

# 68. Backpressure via pool

Un pool limité constitue déjà une forme de backpressure.

---

# 69. Timeout trop long

Configurer CRM :

```text
delay = 10 s
```

et timeout client :

```text
30 s
```

Sous forte charge :

```text
beaucoup de requêtes restent actives
```

---

# 70. Timeout plus court

Avec :

```text
read timeout = 2 s
```

le système libère plus vite les ressources.

---

# 71. Timeout budget

Supposons SLA API :

```text
2 secondes
```

Budget :

```text
API interne = 200 ms
CRM = 500 ms
DB = 300 ms
Kafka = 100 ms
marge = 900 ms
```

---

# 72. Retry storm

Si 500 requêtes échouent et chacune fait 3 retries :

```text
1500 appels supplémentaires
```

---

# 73. Retry avec jitter

```python
import random


async def retry_delay(
    attempt: int
):

    delay = (
        0.1
        * (2 ** attempt)
        + random.uniform(
            0,
            0.1
        )
    )

    await asyncio.sleep(
        delay
    )
```

---

# 74. Pourquoi jitter ?

Sans jitter :

```text
100 clients retry exactement ensemble
```

Avec jitter :

```text
les retries sont répartis dans le temps
```

---

# 75. Circuit breaker

Rappel :

```text
CLOSED
OPEN
HALF_OPEN
```

---

# 76. Pourquoi circuit breaker sous charge ?

Quand CRM est clairement en panne :

```text
continuer à lui envoyer des requêtes
ne sert à rien
```

---

# 77. Bulkhead pattern

Séparer les limites :

```text
CRM semaphore = 20
Payment semaphore = 10
Warehouse semaphore = 30
```

---

# 78. Pourquoi ?

Un service lent ne doit pas monopoliser toutes les ressources.

---

# 79. Kafka lag

Créer un consumer lent.

Exemple :

```python
time.sleep(0.5)
```

par message.

---

# 80. Observer le lag

```bash
docker compose exec kafka \
  kafka-consumer-groups.sh \
  --bootstrap-server kafka:9092 \
  --describe \
  --group warehouse-service
```

---

# 81. Interprétation

Si :

```text
production = 1000 msg/s
consommation = 700 msg/s
```

le lag augmente :

```text
300 msg/s
```

---

# 82. Temps de rattrapage

Si backlog :

```text
30 000 messages
```

et capacité supplémentaire :

```text
500 msg/s
```

temps minimum :

```text
30 000 / 500 = 60 secondes
```

---

# 83. Partitions Kafka et parallélisme

Avec :

```text
1 partition
```

un consumer group ne peut avoir qu'un consumer actif.

Avec :

```text
6 partitions
```

jusqu'à :

```text
6 consumers actifs
```

---

# 84. Mais plus de partitions a un coût

Plus de partitions signifie :

```text
plus de fichiers
plus de métadonnées
plus de réplication
plus de ressources broker
```

---

# 85. Cache Redis

Ajouter :

```python
async def get_customer_cached(
    customer_id: int
):

    key = f"customer:{customer_id}"

    cached = redis_client.get(
        key
    )

    if cached:
        return cached

    # Appel CRM...

    redis_client.set(
        key,
        "active",
        ex=30
    )
```

---

# 86. Mesurer l'effet du cache

Comparer :

```text
0 % hit
50 % hit
90 % hit
```

---

# 87. Cache hit ratio

```text
hits / total lookups
```

---

# 88. Cache stampede

Si une clé très populaire expire :

```text
1000 requêtes
=> 1000 appels CRM
```

---

# 89. Protection simple

Utiliser un lock Redis.

```python
lock = redis_client.lock(
    f"lock:customer:{customer_id}",
    timeout=5
)
```

---

# 90. Attention aux locks distribués

Ils doivent être utilisés avec prudence.

Problèmes :

```text
expiration
process crash
clock
long traitement
```

---

# 91. Stale-while-revalidate

Alternative :

```text
servir une ancienne valeur
et
rafraîchir en arrière-plan
```

---

# 92. Queueing

Quand un système approche la saturation :

```text
les requêtes commencent à attendre
```

La latence augmente souvent brutalement.

---

# 93. Knee point

Le point où :

```text
plus de charge
=> très peu de débit supplémentaire
=> énorme hausse de latence
```

est souvent appelé :

```text
knee point
```

---

# 94. Objectif d'un benchmark

Trouver :

```text
la zone stable avant le knee point
```

---

# 95. Throughput collapse

Au-delà d'une certaine charge :

```text
plus de concurrence
peut réduire le débit
```

à cause de :

```text
context switching
locks
timeouts
retries
GC
queues
```

---

# 96. Test progressif

Ne pas commencer directement à :

```text
1000 users
```

Faire :

```text
10
25
50
100
200
500
1000
```

---

# 97. Pourquoi ?

Pour identifier précisément :

```text
à quel niveau la dégradation commence
```

---

# 98. Soak test

Un soak test mesure la stabilité dans la durée.

Exemple :

```text
100 users pendant 1 heure
```

---

# 99. Ce qu'on recherche

```text
memory leak
pool leak
connexion non fermée
latence croissante
lag progressif
```

---

# 100. Spike test

Exemple :

```text
20 users
puis immédiatement 500 users
```

---

# 101. Stress test

On augmente jusqu'à casser le système.

---

# 102. Capacity test

On cherche la capacité soutenable.

---

# 103. Endurance test

On maintient une charge longtemps.

---

# 104. Test isolé vs test end-to-end

## Isolé

```text
/fast
```

mesure FastAPI.

## End-to-end

```text
ERP -> CRM -> DB -> Redis -> Kafka
```

mesure l'expérience réelle.

---

# 105. Ne jamais optimiser sans mesure

Mauvaise méthode :

```text
"je pense que Redis sera plus rapide"
```

Bonne méthode :

```text
mesurer
modifier
re-mesurer
comparer
```

---

# 106. Baseline

Avant toute optimisation, noter :

```text
RPS
p50
p95
p99
errors
CPU
RAM
DB connections
Kafka lag
```

---

# 107. Rapport expérimental

Créer un tableau :

| Test | Users | RPS | p50 | p95 | p99 | Errors | CPU |
|---|---:|---:|---:|---:|---:|---:|---:|
| Baseline | 10 |  |  |  |  |  |  |
| Medium | 50 |  |  |  |  |  |  |
| High | 200 |  |  |  |  |  |  |
| Stress | 500 |  |  |  |  |  |  |
| Max | 1000 |  |  |  |  |  |  |

---

# 108. Mesurer la concurrence active

Ajouter endpoint :

```python
@app.get("/debug/active")
async def debug_active():

    return {
        "active_requests":
            active_requests,

        "max_active_requests":
            MAX_ACTIVE_REQUESTS
    }
```

---

# 109. Métriques simples

Créer :

```python
metrics = {
    "requests": 0,
    "errors": 0,
    "overload_rejections": 0
}
```

---

# 110. Limite

Un dictionnaire mémoire :

```text
ne fonctionne que pour une instance
```

Avec plusieurs pods :

```text
chaque instance a ses propres chiffres
```

---

# 111. En production

On utilisera plutôt :

```text
Prometheus
OpenTelemetry
Grafana
```

---

# 112. Test multi-worker Uvicorn

Lancer :

```bash
uvicorn \
  app.main:app \
  --host 0.0.0.0 \
  --port 8000 \
  --workers 4
```

---

# 113. Effet des workers

Plusieurs workers :

```text
plusieurs processus Python
```

Ils peuvent exploiter plusieurs cœurs CPU.

---

# 114. Attention

Chaque worker possède :

```text
son propre pool DB
son propre client HTTP
sa propre mémoire
son propre sémaphore
```

---

# 115. Calcul pool avec 4 workers

Si :

```text
pool_size = 10
max_overflow = 20
workers = 4
```

maximum théorique :

```text
120 connexions
```

---

# 116. Ceci peut dépasser PostgreSQL

Donc :

```text
workers
×
(pool + overflow)
```

doit être contrôlé.

---

# 117. Gunicorn

Alternative production souvent utilisée :

```text
Gunicorn
+
Uvicorn workers
```

---

# 118. CPU quota Docker

Limiter API :

```yaml
deploy:
  resources:
    limits:
      cpus: "1.0"
      memory: 512M
```

---

# 119. Pourquoi limiter ?

Pour simuler :

```text
un pod Kubernetes réaliste
```

---

# 120. Test mémoire

Créer un endpoint pédagogique :

```python
@app.get("/memory")
async def memory(
    mb: int = 10
):

    # Allocation volontaire.
    data = bytearray(
        mb * 1024 * 1024
    )

    return {
        "allocated": len(data)
    }
```

---

# 121. Attention

Ne pas utiliser de valeurs trop élevées sur une petite machine.

---

# 122. OOM

Si un conteneur dépasse sa limite mémoire :

```text
OOM kill
```

---

# 123. Observer redémarrage

```bash
docker compose ps
```

---

# 124. Latence injectée

Ajouter CRM :

```text
delay_ms = 50
100
500
1000
2000
```

Comparer l'impact.

---

# 125. Propagation de latence

Si ERP fait 3 appels séquentiels :

```text
CRM = 500 ms
Pricing = 300 ms
Warehouse = 700 ms
```

latence minimale :

```text
~1500 ms
```

---

# 126. Parallélisme

Si appels indépendants :

```python
crm_result, pricing_result = await asyncio.gather(
    get_crm(),
    get_pricing()
)
```

latence devient environ :

```text
max(500, 300)
```

au lieu de :

```text
500 + 300
```

---

# 127. Mais attention

Paralléliser :

```text
augmente la concurrence aval
```

---

# 128. Fan-out amplification

Une requête utilisateur déclenche :

```text
5 appels internes
```

Alors :

```text
100 RPS externes
=> jusqu'à 500 RPS internes
```

---

# 129. Retry amplification

Si chacun retry 3 fois :

```text
500 × 3 = 1500
```

appels possibles.

---

# 130. Cascading failure

Scénario :

```text
CRM lent
ERP accumule requêtes
pool HTTP saturé
latence monte
clients retry
trafic augmente
DB reçoit plus de connexions
système entier se dégrade
```

---

# 131. Protection multi-niveaux

Utiliser :

```text
timeout
semaphore
circuit breaker
rate limit
load shedding
backpressure
```

ensemble.

---

# 132. Priorisation

Toutes les requêtes n'ont pas la même importance.

Exemple :

```text
GET dashboard
```

peut être moins prioritaire que :

```text
POST payment
```

---

# 133. Exercice priorités

Créer deux semaphores :

```python
critical_semaphore = asyncio.Semaphore(50)
reporting_semaphore = asyncio.Semaphore(10)
```

---

# 134. Endpoint reporting lent

```python
@app.get("/report")
async def report():

    async with reporting_semaphore:

        await asyncio.sleep(
            1
        )

        return {
            "status": "generated"
        }
```

---

# 135. Objectif

Sous forte charge reporting :

```text
les requêtes critiques restent disponibles
```

---

# 136. Queue interne

On peut utiliser :

```python
asyncio.Queue(maxsize=100)
```

---

# 137. Si queue pleine

Refuser :

```text
503
```

ou :

```text
429
```

selon le contexte.

---

# 138. 429 vs 503

## 429

```text
client dépasse une limite
```

## 503

```text
service temporairement incapable de traiter
```

---

# 139. SLI

Service Level Indicator.

Exemple :

```text
p95 latency
error rate
availability
```

---

# 140. SLO

Service Level Objective.

Exemple :

```text
99 % des requêtes < 500 ms
```

---

# 141. SLA

Accord contractuel.

Peut inclure :

```text
99,9 % disponibilité
```

---

# 142. Error budget

Si SLO :

```text
99,9 %
```

alors budget d'erreur :

```text
0,1 %
```

---

# 143. Exemple mensuel

30 jours :

```text
43 200 minutes
```

0,1 % :

```text
43,2 minutes
```

---

# 144. Exercice SLO

Définir :

```text
p95 < 500 ms
error rate < 1 %
```

Puis déterminer jusqu'à combien d'utilisateurs la plateforme respecte encore ces objectifs.

---

# 145. Test automatisé de performance

Créer `tests/test_perf_smoke.py`.

```python
import statistics
import time

import httpx


def test_fast_endpoint_latency():

    durations = []

    with httpx.Client(
        timeout=5
    ) as client:

        for _ in range(50):

            started = time.perf_counter()

            response = client.get(
                "http://localhost:8000/fast"
            )

            elapsed_ms = (
                time.perf_counter()
                - started
            ) * 1000

            assert response.status_code == 200

            durations.append(
                elapsed_ms
            )

    p50 = statistics.median(
        durations
    )

    assert p50 < 100
```

---

# 146. Attention aux assertions de performance

Elles dépendent :

```text
de la machine
de Docker
de la charge système
du réseau
```

---

# 147. Donc

Les seuils doivent être adaptés à l'environnement CI.

---

# 148. Warm-up

Avant un benchmark :

```text
envoyer quelques requêtes
```

pour :

```text
ouvrir pools
charger modules
initialiser caches
```

---

# 149. Cold start

Première requête souvent plus lente.

---

# 150. Docker cold start

Redémarrer le conteneur puis mesurer :

```text
première requête
```

---

# 151. Cache warm-up

Même logique :

```text
premiers appels CRM
=> cache vide
```

---

# 152. GC Python

Le Garbage Collector peut créer de petites pauses.

Sous charge :

```text
beaucoup d'objets
```

peut augmenter le coût mémoire.

---

# 153. Ne pas optimiser prématurément le GC

Mesurer avant toute modification.

---

# 154. Logging sous forte charge

Un log par requête peut devenir coûteux.

---

# 155. Logs synchrones

Si chaque log bloque :

```text
latence augmente
```

---

# 156. Bonnes pratiques

```text
logs structurés
sampling
niveaux adaptés
asynchronisme éventuel
```

---

# 157. Payload volumineux

Tester :

```text
1 KB
100 KB
1 MB
```

---

# 158. Effets

```text
CPU sérialisation
mémoire
réseau
latence
```

---

# 159. Compression

Peut réduire :

```text
réseau
```

mais augmenter :

```text
CPU
```

---

# 160. N+1 calls

Un endpoint :

```text
GET /orders
```

qui appelle le CRM une fois par commande peut provoquer :

```text
1 appel principal
+
N appels CRM
```

---

# 161. Exemple

100 commandes :

```text
101 appels
```

---

# 162. Solutions

```text
batch endpoint
cache
join local read model
projection Kafka
```

---

# 163. Test batch

Comparer :

```text
100 appels unitaires CRM
```

avec :

```text
1 appel batch CRM
```

---

# 164. Connection reuse

Observer avec :

```text
client partagé
```

vs :

```text
nouveau client par requête
```

---

# 165. Benchmark méthodique

Toujours changer :

```text
une variable à la fois
```

---

# 166. Exemple

Mauvais test :

```text
pool DB
workers
timeouts
cache
```

modifiés ensemble.

On ne sait plus ce qui a amélioré les résultats.

---

# 167. Test A/B technique

Run A :

```text
pool_size=10
```

Run B :

```text
pool_size=20
```

Même charge.

---

# 168. Tableau comparatif

| Config | RPS | p95 | p99 | Errors |
|---|---:|---:|---:|---:|
| Pool 5 |  |  |  |  |
| Pool 10 |  |  |  |  |
| Pool 20 |  |  |  |  |
| Pool 40 |  |  |  |  |

---

# 169. Scenario Kafka burst

Produire :

```text
50 000 messages
```

rapidement.

Mesurer :

```text
lag max
temps de rattrapage
CPU consumer
```

---

# 170. Scale consumer

Comparer :

```text
1 consumer
2 consumers
4 consumers
```

---

# 171. Limite partitions

Si topic = 3 partitions :

```text
4e consumer inutile
```

pour le parallélisme de ce groupe.

---

# 172. Test de panne pendant charge

Pendant Locust :

```bash
docker compose stop crm
```

Observer :

```text
timeouts
503/504
latence
retries
```

---

# 173. Puis redémarrer

```bash
docker compose start crm
```

Observer le retour à la normale.

---

# 174. Recovery time

Mesurer :

```text
temps entre retour du CRM
et retour des métriques à la normale
```

---

# 175. Recovery storm

Quand une dépendance revient :

```text
beaucoup de requêtes en attente
```

peuvent la surcharger immédiatement.

---

# 176. Solution

```text
jitter
circuit half-open
limites concurrence
ramp-up
```

---

# 177. Test memory leak pédagogique

Créer temporairement :

```python
LEAK = []


@app.get("/leak")
async def leak():

    LEAK.append(
        bytearray(
            1024 * 1024
        )
    )

    return {
        "items": len(LEAK)
    }
```

---

# 178. Observer

Appeler 100 fois :

```text
~100 MB retenus
```

---

# 179. Important

Supprimer cet endpoint après l'expérience.

---

# 180. Soak test pour fuite mémoire

```text
100 users
30 minutes
```

Observer :

```text
mémoire stable ?
```

---

# 181. Queue depth

Pour tout worker asynchrone, surveiller :

```text
longueur queue
```

---

# 182. Pourquoi ?

Une queue qui augmente continuellement signifie :

```text
débit entrant > débit sortant
```

---

# 183. Autoscaling

Un bon signal d'autoscaling peut être :

```text
CPU
RPS
queue depth
Kafka consumer lag
```

---

# 184. CPU seul n'est pas toujours suffisant

Un consumer Kafka peut avoir :

```text
CPU 30 %
lag énorme
```

parce qu'il attend une base lente.

---

# 185. Load test distribué

Locust peut être distribué :

```text
master
workers
```

---

# 186. Pourquoi ?

Une seule machine Locust peut elle-même devenir le bottleneck.

---

# 187. Toujours monitorer le générateur de charge

Si Locust CPU = 100 % :

```text
vous ne mesurez plus correctement le serveur
```

---

# 188. Exercice 1

Trouver la capacité maximale respectant :

```text
p95 < 500 ms
errors < 1 %
```

---

# 189. Exercice 2

Comparer :

```text
pool DB 5 / 10 / 20 / 40
```

---

# 190. Exercice 3

Comparer :

```text
HTTP pool 20 / 100 / 500
```

---

# 191. Exercice 4

Comparer :

```text
CRM semaphore 5 / 20 / 100
```

---

# 192. Exercice 5

Activer load shedding à :

```text
50
100
200
```

requêtes actives.

---

# 193. Exercice 6

Mesurer l'effet d'un CRM :

```text
50 ms
500 ms
2 s
```

---

# 194. Exercice 7

Ajouter cache Redis et mesurer :

```text
hit ratio
RPS
p95
```

---

# 195. Exercice 8

Créer un endpoint batch CRM.

Comparer avec N appels unitaires.

---

# 196. Exercice 9

Créer une hot row SQL.

Mesurer la contention.

---

# 197. Exercice 10

Remplacer la hot row par :

```text
Redis INCR
```

Comparer.

---

# 198. Exercice 11

Simuler une panne CRM sous charge.

Mesurer :

```text
erreur max
recovery time
```

---

# 199. Exercice 12

Créer un retry storm volontaire.

Puis corriger avec :

```text
backoff
jitter
circuit breaker
```

---

# 200. Exercice 13

Limiter l'API à :

```text
1 CPU
512 MB
```

et comparer.

---

# 201. Exercice 14

Tester :

```text
1 worker
2 workers
4 workers
```

---

# 202. Exercice 15

Calculer le nombre maximum de connexions PostgreSQL potentielles pour chaque configuration.

---

# 203. Exercice 16

Mesurer Kafka lag avec :

```text
1
2
4 consumers
```

---

# 204. Exercice 17

Créer un reporting bulkhead.

Saturer reporting sans impacter les endpoints critiques.

---

# 205. Exercice 18

Définir un SLO :

```text
p95 < 500 ms
availability > 99 %
```

et vérifier à quelle charge il est dépassé.

---

# 206. Exercice 19

Faire un soak test.

Durée conseillée :

```text
20 à 30 minutes
```

---

# 207. Exercice 20

Créer un rapport final avec :

```text
baseline
bottleneck
optimisations
résultats
limites
```

---

# 208. Méthode d'analyse recommandée

Pour chaque anomalie :

```text
1. observer
2. formuler une hypothèse
3. mesurer
4. modifier une seule variable
5. re-tester
6. conclure
```

---

# 209. Exemple

Observation :

```text
p95 monte à 3 s à 200 users
```

Hypothèse :

```text
pool DB saturé
```

Mesure :

```text
pg_stat_activity
```

Modification :

```text
pool 10 -> 20
```

Re-test :

```text
même charge
```

---

# 210. Ne pas confondre corrélation et causalité

Exemple :

```text
CPU monte
latence monte
```

Cela ne prouve pas automatiquement :

```text
CPU est la cause
```

---

# 211. Table finale à remplir

| Expérience | Hypothèse | Mesure | Modification | Résultat | Conclusion |
|---|---|---|---|---|---|
| Pool DB |  |  |  |  |  |
| CRM semaphore |  |  |  |  |  |
| Cache Redis |  |  |  |  |  |
| Workers |  |  |  |  |  |
| Kafka scale |  |  |  |  |  |
| Load shedding |  |  |  |  |  |

---

# 212. Questions de synthèse

1. Quelle différence entre concurrence et débit ?
2. Pourquoi plus de connexions DB n'est pas toujours meilleur ?
3. À quoi sert un semaphore ?
4. Qu'est-ce que la backpressure ?
5. Qu'est-ce que le load shedding ?
6. Pourquoi un timeout trop long est-il dangereux ?
7. Qu'est-ce qu'un retry storm ?
8. Pourquoi ajouter du jitter ?
9. Qu'est-ce qu'un bulkhead ?
10. Pourquoi p99 est-il important ?
11. Qu'est-ce qu'une hot row ?
12. Qu'est-ce qu'un knee point ?
13. Pourquoi un système peut-il perdre du débit quand la charge augmente ?
14. Qu'est-ce qu'un soak test ?
15. Quelle différence entre stress test et capacity test ?
16. Pourquoi surveiller Kafka consumer lag ?
17. Pourquoi CPU n'est-il pas toujours un bon signal d'autoscaling ?
18. Pourquoi réutiliser un client HTTP ?
19. Pourquoi faut-il mesurer avant d'optimiser ?
20. Pourquoi changer une variable à la fois ?

---

# 213. Corrections synthétiques

## 1. Concurrence vs débit

Concurrence :

```text
nombre de requêtes simultanées
```

Débit :

```text
nombre de requêtes terminées par unité de temps
```

---

## 2. Pool DB

Un pool trop grand peut surcharger PostgreSQL.

---

## 3. Semaphore

Il limite le nombre d'opérations concurrentes.

---

## 4. Backpressure

Elle ralentit ou met en attente le trafic afin de protéger une ressource.

---

## 5. Load shedding

Il refuse volontairement une partie du trafic.

---

## 6. Timeout long

Il immobilise les ressources plus longtemps pendant une panne.

---

## 7. Retry storm

Les retries multiplient la charge sur un service déjà défaillant.

---

## 8. Jitter

Il évite que tous les clients retry au même moment.

---

## 9. Bulkhead

Il isole des ressources entre plusieurs dépendances ou catégories de trafic.

---

## 10. p99

Il représente la latence des requêtes les plus lentes.

---

## 11. Hot row

Ligne SQL très fortement contendue.

---

## 12. Knee point

Zone où une légère augmentation de charge provoque une forte dégradation de latence.

---

## 13. Throughput collapse

La contention, les retries et les files d'attente peuvent faire baisser le débit utile.

---

## 14. Soak test

Test long visant à détecter les problèmes progressifs.

---

## 15. Stress vs capacity

Stress :

```text
aller jusqu'à casser
```

Capacity :

```text
trouver la charge soutenable
```

---

## 16. Consumer lag

Il mesure le retard de traitement.

---

## 17. Autoscaling

Une application peut être I/O-bound avec CPU faible mais backlog élevé.

---

## 18. Client HTTP partagé

Il réutilise les connexions et réduit le coût réseau.

---

## 19. Mesure

Sans baseline, impossible de savoir si l'optimisation aide réellement.

---

## 20. Une variable à la fois

Pour isoler l'effet causal d'un changement.

---

# 214. Barème proposé

| Critère | Points |
|---|---:|
| Environnement Docker | 1 |
| Instrumentation latence | 1 |
| Locust | 2 |
| Analyse p50/p95/p99 | 2 |
| Pool PostgreSQL | 2 |
| Pool HTTP | 1 |
| Backpressure | 1.5 |
| Load shedding | 1.5 |
| Rate limiting | 1 |
| Concurrence / semaphore | 1 |
| Kafka lag | 1 |
| Tests de panne | 1 |
| Soak / stress / capacity | 1 |
| Rapport expérimental | 2 |
| Conclusion technique | 1 |
| **Total** | **20** |

---

# 215. Livrables

Le rendu doit contenir :

```text
tp4-performance/
|
|-- docker-compose.yml
|-- .env.example
|
|-- api/
|-- crm/
|
|-- load-tests/
|-- tests/
|-- scripts/
|
|-- results/
|
|-- README.md
```

---

# 216. README attendu

Le README doit contenir :

1. architecture ;
2. prérequis ;
3. lancement ;
4. endpoints ;
5. baseline ;
6. scénarios Locust ;
7. mesures PostgreSQL ;
8. mesures Docker ;
9. analyse pools ;
10. analyse concurrence ;
11. analyse p95/p99 ;
12. saturation ;
13. load shedding ;
14. rate limiting ;
15. panne CRM ;
16. Kafka lag ;
17. cache ;
18. optimisations ;
19. limites ;
20. conclusion.

---

# 217. Extensions expert++

Pour aller encore plus loin :

- Prometheus ;
- Grafana ;
- OpenTelemetry ;
- Jaeger ;
- k6 ;
- Vegeta ;
- wrk ;
- asyncpg ;
- PgBouncer ;
- Redis Cluster ;
- Kafka multi-brokers ;
- partitionnement PostgreSQL ;
- read replicas ;
- Kubernetes HPA ;
- KEDA sur Kafka lag ;
- service mesh ;
- Envoy ;
- circuit breaker proxy ;
- adaptive concurrency ;
- distributed load testing ;
- chaos engineering ;
- TC/netem ;
- packet loss ;
- latency injection ;
- CPU throttling ;
- autoscaling ;
- SLO/error budgets ;
- performance regression CI.

---

# 218. Conclusion

Ce TP montre qu'une architecture performante n'est pas simplement une architecture :

```text
"qui répond vite sur mon PC"
```

Il faut savoir :

- mesurer ;
- définir une baseline ;
- augmenter progressivement la charge ;
- identifier les bottlenecks ;
- comprendre les pools ;
- contrôler la concurrence ;
- protéger les dépendances ;
- appliquer backpressure et load shedding ;
- surveiller les percentiles ;
- surveiller les queues ;
- surveiller Kafka lag ;
- observer PostgreSQL ;
- tester les pannes ;
- mesurer le recovery ;
- comparer avant/après.

Une architecture distribuée bien conçue doit non seulement fonctionner en conditions normales, mais aussi rester **prévisible sous saturation, panne partielle et forte concurrence**.
