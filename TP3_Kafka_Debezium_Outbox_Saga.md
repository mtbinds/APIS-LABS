# TP 3 - Architecture Event-Driven ERP / CRM / Warehouse avec Kafka, CDC, Outbox et Saga


## Kafka, Debezium, Schema Registry, PostgreSQL, Redis, Saga, DLQ, idempotence, observabilité et chaos testing

**Niveau :** intermédiaire

**Durée conseillée :** 2 h à 3 h  

**Architecture :** event-driven distribuée  

**Technologies :** Python 3.12+, FastAPI, PostgreSQL, Kafka, Kafka Connect, Debezium, Redis, Docker, Docker Compose, HTTPX, Pytest  

**Systèmes :** Linux, macOS, Windows PowerShell

---

# 1. Objectifs du TP

**À la fin de ce TP, vous serez capable de** :

- construire une architecture **event-driven** réaliste ;
- comprendre le rôle d'un **broker Kafka** ;
- produire et consommer des événements ;
- utiliser plusieurs **topics** ;
- comprendre les **partitions** ;
- utiliser des **consumer groups** ;
- comprendre les offsets Kafka ;
- garantir l'ordre sur une même clé métier ;
- gérer les doublons ;
- rendre les consommateurs **idempotents** ;
- implémenter l'**Outbox Pattern** ;
- utiliser **Debezium CDC** ;
- éviter le problème du **dual write** ;
- construire une **Saga** ;
- implémenter des actions de compensation ;
- gérer les erreurs avec une **Dead Letter Topic** ;
- mettre en place des retries ;
- comprendre `at-most-once`, `at-least-once` et `exactly-once` ;
- propager des `Correlation-ID` ;
- mesurer la latence de bout en bout ;
- simuler des pannes ;
- observer la cohérence éventuelle ;
- tester la concurrence ;
- analyser le comportement du système sous charge.

---

# 2. Contexte métier

Nous simulons une entreprise qui possède quatre domaines principaux :

```text
CRM
ERP
Warehouse
Payment
```

Le flux métier principal est une commande client.

Lorsqu'une commande est créée :

```text
1. ERP crée la commande
2. ERP écrit un événement dans sa table Outbox
3. Debezium détecte l'INSERT
4. Kafka reçoit l'événement
5. Warehouse réserve le stock
6. Payment traite le paiement
7. ERP reçoit les événements de résultat
8. ERP confirme ou annule la commande
```

---

# 3. Pourquoi passer à une architecture event-driven ?

Dans le TP 2, l'ERP appelait directement :

```text
CRM
Warehouse
```

avec des appels HTTP synchrones.

Cela crée un couplage temporel :

```text
ERP dépend du fait que Warehouse soit disponible maintenant
```

Avec Kafka :

```text
ERP publie un événement
Warehouse le traite lorsqu'il est disponible
```

---

# 4. Architecture cible

```mermaid
flowchart LR
    C[Client]
    ERP[ERP API]
    CRM[CRM API]
    W[Warehouse Consumer]
    P[Payment Consumer]

    EDB[(ERP PostgreSQL)]
    CDB[(CRM PostgreSQL)]
    WDB[(Warehouse PostgreSQL)]
    PDB[(Payment PostgreSQL)]

    O[(Outbox)]
    DBZ[Debezium]
    KC[Kafka Connect]
    K[Kafka]
    R[(Redis)]

    C --> ERP
    ERP --> CRM
    ERP --> EDB
    EDB --> O
    O --> DBZ
    DBZ --> KC
    KC --> K

    K --> W
    K --> P

    W --> WDB
    P --> PDB

    W --> K
    P --> K

    K --> ERP

    ERP --> R
```

---

# 5. Flux métier détaillé

```mermaid
sequenceDiagram
    participant Client
    participant ERP
    participant DB as ERP DB
    participant Debezium
    participant Kafka
    participant Warehouse
    participant Payment

    Client->>ERP: POST /orders
    ERP->>DB: INSERT order
    ERP->>DB: INSERT outbox_event
    ERP->>DB: COMMIT
    ERP-->>Client: 202 Accepted

    Debezium->>DB: CDC WAL
    Debezium->>Kafka: order.created

    Kafka->>Warehouse: order.created
    Kafka->>Payment: order.created

    Warehouse->>Kafka: stock.reserved
    Payment->>Kafka: payment.completed

    Kafka->>ERP: results
    ERP->>DB: order = CONFIRMED
```

---

# 6. Le problème du Dual Write

Imaginez :

```text
ERP écrit la commande en base
puis
ERP publie dans Kafka
```

Deux opérations différentes.

Scénario problématique :

```text
1. COMMIT PostgreSQL réussi
2. crash ERP
3. publication Kafka jamais exécutée
```

Résultat :

```text
commande existe
événement absent
Warehouse ne sait jamais qu'il doit agir
```

C'est le **dual write problem**.

---

# 7. Solution : Outbox Pattern

Au lieu de :

```text
DB puis Kafka
```

on fait :

```text
transaction SQL :
    INSERT order
    INSERT outbox_event
COMMIT
```

Puis **Debezium** lit les changements.

---

# 8. Pourquoi l'Outbox Pattern est important

Avec l'Outbox :

```text
commande + événement
```

sont écrits dans **la même transaction PostgreSQL**.

Donc :

```text
soit les deux existent
soit aucun n'existe
```

---

# 9. Arborescence du projet

```text
tp3-event-driven/
|
|-- docker-compose.yml
|-- .env
|
|-- erp/
|   |-- Dockerfile
|   |-- requirements.txt
|   |-- app/
|       |-- __init__.py
|       |-- main.py
|       |-- consumer.py
|
|-- crm/
|   |-- Dockerfile
|   |-- requirements.txt
|   |-- app/
|       |-- __init__.py
|       |-- main.py
|
|-- warehouse/
|   |-- Dockerfile
|   |-- requirements.txt
|   |-- consumer.py
|
|-- payment/
|   |-- Dockerfile
|   |-- requirements.txt
|   |-- consumer.py
|
|-- tests/
|   |-- test_order_flow.py
|
|-- scripts/
|   |-- create-topics.sh
|   |-- register-connector.sh
|
|-- README.md
```

---

# 10. Création des dossiers

## Linux / macOS

```bash
mkdir -p tp3-event-driven/{erp/app,crm/app,warehouse,payment,tests,scripts}
cd tp3-event-driven

touch erp/app/__init__.py
touch crm/app/__init__.py
```

## Windows PowerShell

```powershell
mkdir tp3-event-driven
cd tp3-event-driven

mkdir erp
mkdir erp\app
mkdir crm
mkdir crm\app
mkdir warehouse
mkdir payment
mkdir tests
mkdir scripts

New-Item erp\app\__init__.py -ItemType File
New-Item crm\app\__init__.py -ItemType File
```

---

# 11. Fichier .env

```env
ERP_DB_URL=postgresql://erp:erp@erp-db:5432/erp
CRM_DB_URL=postgresql://crm:crm@crm-db:5432/crm
WAREHOUSE_DB_URL=postgresql://warehouse:warehouse@warehouse-db:5432/warehouse
PAYMENT_DB_URL=postgresql://payment:payment@payment-db:5432/payment

KAFKA_BOOTSTRAP_SERVERS=kafka:9092
REDIS_URL=redis://redis:6379/0
CRM_URL=http://crm:8000
```

---

# 12. Docker Compose

Créer `docker-compose.yml`.

```yaml
services:

  # ------------------------------------------------------------------
  # PostgreSQL ERP
  # ------------------------------------------------------------------
  erp-db:
    image: postgres:16
    container_name: erp-db

    environment:
      POSTGRES_USER: erp
      POSTGRES_PASSWORD: erp
      POSTGRES_DB: erp

    ports:
      - "5435:5432"

    volumes:
      - erp_data:/var/lib/postgresql/data

    # Debezium lit le WAL PostgreSQL.
    # PostgreSQL doit donc utiliser le mode logical replication.
    command:
      - "postgres"
      - "-c"
      - "wal_level=logical"

    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U erp -d erp"]
      interval: 5s
      timeout: 3s
      retries: 20


  # ------------------------------------------------------------------
  # PostgreSQL CRM
  # ------------------------------------------------------------------
  crm-db:
    image: postgres:16
    container_name: crm-db

    environment:
      POSTGRES_USER: crm
      POSTGRES_PASSWORD: crm
      POSTGRES_DB: crm

    ports:
      - "5433:5432"

    volumes:
      - crm_data:/var/lib/postgresql/data

    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U crm -d crm"]
      interval: 5s
      timeout: 3s
      retries: 20


  # ------------------------------------------------------------------
  # PostgreSQL Warehouse
  # ------------------------------------------------------------------
  warehouse-db:
    image: postgres:16
    container_name: warehouse-db

    environment:
      POSTGRES_USER: warehouse
      POSTGRES_PASSWORD: warehouse
      POSTGRES_DB: warehouse

    ports:
      - "5434:5432"

    volumes:
      - warehouse_data:/var/lib/postgresql/data

    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U warehouse -d warehouse"]
      interval: 5s
      timeout: 3s
      retries: 20


  # ------------------------------------------------------------------
  # PostgreSQL Payment
  # ------------------------------------------------------------------
  payment-db:
    image: postgres:16
    container_name: payment-db

    environment:
      POSTGRES_USER: payment
      POSTGRES_PASSWORD: payment
      POSTGRES_DB: payment

    ports:
      - "5436:5432"

    volumes:
      - payment_data:/var/lib/postgresql/data

    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U payment -d payment"]
      interval: 5s
      timeout: 3s
      retries: 20


  # ------------------------------------------------------------------
  # Kafka
  # ------------------------------------------------------------------
  kafka:
    image: bitnami/kafka:latest
    container_name: kafka

    environment:
      # Mode KRaft :
      # Kafka fonctionne ici sans ZooKeeper.
      KAFKA_CFG_NODE_ID: 1
      KAFKA_CFG_PROCESS_ROLES: broker,controller

      # Listener interne utilisé par les conteneurs.
      KAFKA_CFG_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093

      KAFKA_CFG_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092

      KAFKA_CFG_CONTROLLER_LISTENER_NAMES: CONTROLLER

      KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT

      KAFKA_CFG_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093

      ALLOW_PLAINTEXT_LISTENER: "yes"

    ports:
      - "9092:9092"

    volumes:
      - kafka_data:/bitnami/kafka


  # ------------------------------------------------------------------
  # Kafka Connect + Debezium
  # ------------------------------------------------------------------
  connect:
    image: debezium/connect:2.7
    container_name: connect

    environment:
      BOOTSTRAP_SERVERS: kafka:9092

      # Groupe Kafka utilisé par Kafka Connect.
      GROUP_ID: 1

      # Topics internes utilisés par Kafka Connect.
      CONFIG_STORAGE_TOPIC: connect_configs
      OFFSET_STORAGE_TOPIC: connect_offsets
      STATUS_STORAGE_TOPIC: connect_statuses

      # Facteur de réplication = 1 car TP local avec un seul broker.
      CONFIG_STORAGE_REPLICATION_FACTOR: 1
      OFFSET_STORAGE_REPLICATION_FACTOR: 1
      STATUS_STORAGE_REPLICATION_FACTOR: 1

    ports:
      - "8083:8083"

    depends_on:
      - kafka
      - erp-db


  # ------------------------------------------------------------------
  # Redis
  # ------------------------------------------------------------------
  redis:
    image: redis:7
    container_name: redis

    ports:
      - "6379:6379"


  # ------------------------------------------------------------------
  # CRM API
  # ------------------------------------------------------------------
  crm:
    build: ./crm
    container_name: crm

    environment:
      DATABASE_URL: ${CRM_DB_URL}

    ports:
      - "8001:8000"

    depends_on:
      crm-db:
        condition: service_healthy


  # ------------------------------------------------------------------
  # ERP API
  # ------------------------------------------------------------------
  erp:
    build: ./erp
    container_name: erp

    environment:
      DATABASE_URL: ${ERP_DB_URL}
      CRM_URL: ${CRM_URL}
      KAFKA_BOOTSTRAP_SERVERS: ${KAFKA_BOOTSTRAP_SERVERS}
      REDIS_URL: ${REDIS_URL}

    ports:
      - "8000:8000"

    depends_on:
      erp-db:
        condition: service_healthy
      kafka:
        condition: service_started
      redis:
        condition: service_started
      crm:
        condition: service_started


  # ------------------------------------------------------------------
  # ERP Consumer
  # ------------------------------------------------------------------
  erp-consumer:
    build: ./erp
    container_name: erp-consumer

    command:
      - "python"
      - "-m"
      - "app.consumer"

    environment:
      DATABASE_URL: ${ERP_DB_URL}
      KAFKA_BOOTSTRAP_SERVERS: ${KAFKA_BOOTSTRAP_SERVERS}
      REDIS_URL: ${REDIS_URL}

    depends_on:
      erp-db:
        condition: service_healthy
      kafka:
        condition: service_started


  # ------------------------------------------------------------------
  # Warehouse Consumer
  # ------------------------------------------------------------------
  warehouse:
    build: ./warehouse
    container_name: warehouse

    environment:
      DATABASE_URL: ${WAREHOUSE_DB_URL}
      KAFKA_BOOTSTRAP_SERVERS: ${KAFKA_BOOTSTRAP_SERVERS}
      REDIS_URL: ${REDIS_URL}

    depends_on:
      warehouse-db:
        condition: service_healthy
      kafka:
        condition: service_started


  # ------------------------------------------------------------------
  # Payment Consumer
  # ------------------------------------------------------------------
  payment:
    build: ./payment
    container_name: payment

    environment:
      DATABASE_URL: ${PAYMENT_DB_URL}
      KAFKA_BOOTSTRAP_SERVERS: ${KAFKA_BOOTSTRAP_SERVERS}
      REDIS_URL: ${REDIS_URL}

    depends_on:
      payment-db:
        condition: service_healthy
      kafka:
        condition: service_started


volumes:
  erp_data:
  crm_data:
  warehouse_data:
  payment_data:
  kafka_data:
```

---

# 13. Explication des rôles Docker

Cette architecture contient plusieurs types de composants.

## APIs synchrones

```text
ERP API
CRM API
```

Ils répondent directement à des requêtes HTTP.

## Consumers

```text
ERP Consumer
Warehouse Consumer
Payment Consumer
```

Ils restent connectés à Kafka et attendent des messages.

## Infrastructure

```text
PostgreSQL
Kafka
Kafka Connect
Debezium
Redis
```

---

# 14. Kafka : concepts essentiels

Kafka n'est pas simplement une file.

Il stocke les événements dans un journal ordonné.

```text
Topic
  |
  +-- Partition 0
  +-- Partition 1
  +-- Partition 2
```

---

# 15. Topic

Un topic représente un flux logique.

Nous utiliserons :

```text
erp.public.outbox_events
warehouse.events
payment.events
order.dlq
```

---

# 16. Partition

Chaque topic Kafka peut être divisé en plusieurs partitions.

```mermaid
flowchart LR
    T[order-events]
    P0[Partition 0]
    P1[Partition 1]
    P2[Partition 2]

    T --> P0
    T --> P1
    T --> P2
```

L'ordre est garanti :

```text
dans une partition
```

mais pas :

```text
entre plusieurs partitions
```

---

# 17. Clé Kafka

Pour conserver l'ordre des événements d'une commande :

```text
key = order_id
```

Ainsi, tous les événements du même `order_id` vont normalement vers la même partition.

---

# 18. Consumer Group

Un consumer group permet de répartir les partitions entre plusieurs instances.

```mermaid
flowchart LR
    K[Kafka Topic]
    P0[P0]
    P1[P1]
    P2[P2]

    C1[Warehouse consumer 1]
    C2[Warehouse consumer 2]

    K --> P0
    K --> P1
    K --> P2

    P0 --> C1
    P1 --> C2
    P2 --> C1
```

---

# 19. Offset

Kafka numérote chaque message dans une partition.

Exemple :

```text
offset 0
offset 1
offset 2
offset 3
```

Le consumer mémorise jusqu'où il a lu.

---

# 20. Dépendances ERP

Créer `erp/requirements.txt` :

```txt
fastapi
uvicorn[standard]
sqlalchemy
psycopg2-binary
pydantic
httpx
redis
confluent-kafka
```

---

# 21. Dockerfile ERP

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# Copie des dépendances avant le code.
# Cela permet à Docker de réutiliser le cache si seul le code change.
COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY app ./app

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

# 22. Modèle ERP

Créer `erp/app/main.py`.

```python
import json
import os
from datetime import datetime, timezone
from uuid import UUID, uuid4

import httpx
import redis

from fastapi import FastAPI, Header, HTTPException
from pydantic import BaseModel, Field

from sqlalchemy import (
    Column,
    DateTime,
    Integer,
    JSON,
    String,
    create_engine,
)

from sqlalchemy.orm import declarative_base, sessionmaker


# -------------------------------------------------------------------
# Configuration
# -------------------------------------------------------------------

DATABASE_URL = os.environ["DATABASE_URL"]
CRM_URL = os.environ["CRM_URL"]
REDIS_URL = os.environ["REDIS_URL"]


# -------------------------------------------------------------------
# SQLAlchemy
# -------------------------------------------------------------------

engine = create_engine(
    DATABASE_URL,

    # Vérifie la validité des connexions récupérées dans le pool.
    pool_pre_ping=True
)


SessionLocal = sessionmaker(
    bind=engine,
    autocommit=False,
    autoflush=False
)


Base = declarative_base()


# -------------------------------------------------------------------
# Modèle Order
# -------------------------------------------------------------------

class OrderDB(Base):
    """
    Commande ERP.

    Le statut évoluera au cours de la Saga.
    """

    __tablename__ = "orders"

    id = Column(
        String(36),
        primary_key=True
    )

    customer_id = Column(
        Integer,
        nullable=False
    )

    product_id = Column(
        String(100),
        nullable=False
    )

    quantity = Column(
        Integer,
        nullable=False
    )

    status = Column(
        String(50),
        nullable=False
    )

    created_at = Column(
        DateTime(timezone=True),
        nullable=False
    )


# -------------------------------------------------------------------
# Table Outbox
# -------------------------------------------------------------------

class OutboxEventDB(Base):
    """
    Table Outbox.

    Chaque événement métier est écrit ici
    dans la même transaction que la commande.

    Debezium observera ensuite cette table.
    """

    __tablename__ = "outbox_events"

    id = Column(
        String(36),
        primary_key=True
    )

    aggregate_id = Column(
        String(36),
        nullable=False
    )

    aggregate_type = Column(
        String(100),
        nullable=False
    )

    event_type = Column(
        String(100),
        nullable=False
    )

    payload = Column(
        JSON,
        nullable=False
    )

    created_at = Column(
        DateTime(timezone=True),
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
# API models
# -------------------------------------------------------------------

class OrderCreate(BaseModel):
    customer_id: int
    product_id: str
    quantity: int = Field(gt=0)


class OrderOut(BaseModel):
    id: UUID
    customer_id: int
    product_id: str
    quantity: int
    status: str
    created_at: datetime


# -------------------------------------------------------------------
# FastAPI
# -------------------------------------------------------------------

app = FastAPI(
    title="ERP Event-Driven API",
    version="1.0.0"
)
```

---

# 23. Vérification CRM

Toujours dans `erp/app/main.py` :

```python
async def get_customer(customer_id: int) -> dict:
    """
    Vérifie que le client existe avant de créer la commande.

    Cet appel reste synchrone car l'ERP a besoin de la réponse
    immédiatement pour savoir s'il peut accepter la demande.
    """

    timeout = httpx.Timeout(
        connect=1.0,
        read=2.0,
        write=2.0,
        pool=1.0
    )

    async with httpx.AsyncClient(
        timeout=timeout
    ) as client:

        try:
            response = await client.get(
                f"{CRM_URL}/customers/{customer_id}"
            )

        except httpx.TimeoutException:
            raise HTTPException(
                status_code=504,
                detail="CRM timeout"
            )

        except httpx.RequestError:
            raise HTTPException(
                status_code=503,
                detail="CRM unavailable"
            )

    if response.status_code == 404:
        raise HTTPException(
            status_code=400,
            detail="Unknown customer"
        )

    response.raise_for_status()

    return response.json()
```

---

# 24. Création de commande + Outbox

Ajouter :

```python
@app.post(
    "/orders",
    response_model=OrderOut,
    status_code=202
)
async def create_order(
    payload: OrderCreate,

    # Cette clé protège contre les doubles clics
    # et les retries HTTP du client.
    idempotency_key: str | None = Header(
        default=None,
        alias="Idempotency-Key"
    )
):
    """
    Crée une commande ERP.

    Important :
    nous ne confirmons pas encore la commande.

    Le résultat HTTP 202 signifie :
    "requête acceptée, traitement asynchrone en cours".
    """

    if not idempotency_key:
        raise HTTPException(
            status_code=400,
            detail="Idempotency-Key is required"
        )

    # ----------------------------------------------------------------
    # Vérification idempotence côté API
    # ----------------------------------------------------------------

    redis_key = f"erp:idempotency:{idempotency_key}"

    existing_order_id = redis_client.get(
        redis_key
    )

    if existing_order_id:

        db = SessionLocal()

        try:
            order = db.get(
                OrderDB,
                existing_order_id
            )

            if order is None:
                raise HTTPException(
                    status_code=500,
                    detail="Broken idempotency reference"
                )

            return OrderOut(
                id=UUID(order.id),
                customer_id=order.customer_id,
                product_id=order.product_id,
                quantity=order.quantity,
                status=order.status,
                created_at=order.created_at
            )

        finally:
            db.close()


    # ----------------------------------------------------------------
    # Vérification CRM
    # ----------------------------------------------------------------

    customer = await get_customer(
        payload.customer_id
    )

    if not customer["active"]:
        raise HTTPException(
            status_code=409,
            detail="Customer inactive"
        )


    # ----------------------------------------------------------------
    # Création de l'identité de la commande
    # ----------------------------------------------------------------

    order_id = uuid4()

    created_at = datetime.now(
        timezone.utc
    )


    # ----------------------------------------------------------------
    # Transaction atomique ERP + Outbox
    # ----------------------------------------------------------------

    db = SessionLocal()

    try:

        order = OrderDB(
            id=str(order_id),
            customer_id=payload.customer_id,
            product_id=payload.product_id,
            quantity=payload.quantity,

            # Le traitement distribué n'est pas encore terminé.
            status="PENDING",

            created_at=created_at
        )

        # L'événement est enregistré comme une ligne SQL.
        # On ne contacte PAS Kafka ici.
        event = OutboxEventDB(
            id=str(uuid4()),
            aggregate_id=str(order_id),
            aggregate_type="Order",
            event_type="order.created",
            payload={
                "order_id": str(order_id),
                "customer_id": payload.customer_id,
                "product_id": payload.product_id,
                "quantity": payload.quantity,
                "created_at": created_at.isoformat()
            },
            created_at=created_at
        )

        db.add(order)
        db.add(event)

        # Ce COMMIT est l'opération critique.
        #
        # PostgreSQL garantit :
        # - soit order + outbox sont écrits ;
        # - soit aucun des deux ne l'est.
        db.commit()

    except:
        db.rollback()
        raise

    finally:
        db.close()


    # ----------------------------------------------------------------
    # Idempotence HTTP
    # ----------------------------------------------------------------

    redis_client.set(
        redis_key,
        str(order_id),
        ex=3600
    )


    return OrderOut(
        id=order_id,
        customer_id=payload.customer_id,
        product_id=payload.product_id,
        quantity=payload.quantity,
        status="PENDING",
        created_at=created_at
    )
```

---

# 25. Pourquoi retourner 202 et non 201 ?

La commande existe bien dans l'ERP.

Mais le processus métier n'est pas terminé.

Le système attend encore :

```text
Warehouse
Payment
```

Donc :

```text
202 Accepted
```

est pédagogiquement plus cohérent ici.

---

# 26. Endpoint GET commande

```python
@app.get(
    "/orders/{order_id}",
    response_model=OrderOut
)
def get_order(order_id: UUID):

    db = SessionLocal()

    try:

        order = db.get(
            OrderDB,
            str(order_id)
        )

        if order is None:
            raise HTTPException(
                status_code=404,
                detail="Order not found"
            )

        return OrderOut(
            id=UUID(order.id),
            customer_id=order.customer_id,
            product_id=order.product_id,
            quantity=order.quantity,
            status=order.status,
            created_at=order.created_at
        )

    finally:
        db.close()
```

---

# 27. CRM minimal

Le CRM peut reprendre celui du TP 2.

`crm/requirements.txt` :

```txt
fastapi
uvicorn[standard]
sqlalchemy
psycopg2-binary
pydantic
email-validator
```

---

# 28. CRM Dockerfile

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY app ./app

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

# 29. CRM code

Créer `crm/app/main.py`.

```python
import os

from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, EmailStr

from sqlalchemy import (
    Boolean,
    Column,
    Integer,
    String,
    create_engine
)

from sqlalchemy.orm import (
    declarative_base,
    sessionmaker
)


DATABASE_URL = os.environ[
    "DATABASE_URL"
]


engine = create_engine(
    DATABASE_URL,
    pool_pre_ping=True
)


SessionLocal = sessionmaker(
    bind=engine,
    autocommit=False,
    autoflush=False
)


Base = declarative_base()


class CustomerDB(Base):
    __tablename__ = "customers"

    id = Column(
        Integer,
        primary_key=True
    )

    name = Column(
        String(100),
        nullable=False
    )

    email = Column(
        String(150),
        nullable=False,
        unique=True
    )

    active = Column(
        Boolean,
        nullable=False,
        default=True
    )


Base.metadata.create_all(engine)


class CustomerCreate(BaseModel):
    name: str
    email: EmailStr
    active: bool = True


class CustomerOut(CustomerCreate):
    id: int


app = FastAPI(
    title="CRM API"
)


@app.post(
    "/customers",
    response_model=CustomerOut,
    status_code=201
)
def create_customer(
    payload: CustomerCreate
):

    db = SessionLocal()

    try:

        customer = CustomerDB(
            name=payload.name,
            email=str(payload.email),
            active=payload.active
        )

        db.add(customer)
        db.commit()
        db.refresh(customer)

        return CustomerOut(
            id=customer.id,
            name=customer.name,
            email=customer.email,
            active=customer.active
        )

    finally:
        db.close()


@app.get(
    "/customers/{customer_id}",
    response_model=CustomerOut
)
def get_customer(
    customer_id: int
):

    db = SessionLocal()

    try:

        customer = db.get(
            CustomerDB,
            customer_id
        )

        if customer is None:
            raise HTTPException(
                status_code=404,
                detail="Customer not found"
            )

        return CustomerOut(
            id=customer.id,
            name=customer.name,
            email=customer.email,
            active=customer.active
        )

    finally:
        db.close()
```

---

# 30. Configurer Debezium

Créer `scripts/register-connector.sh`.

```bash
#!/usr/bin/env bash

# Ce script enregistre un connecteur PostgreSQL Debezium
# via l'API REST de Kafka Connect.

curl -X POST \
  http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d '{
    "name": "erp-outbox-connector",

    "config": {
      "connector.class": "io.debezium.connector.postgresql.PostgresConnector",

      "database.hostname": "erp-db",
      "database.port": "5432",
      "database.user": "erp",
      "database.password": "erp",
      "database.dbname": "erp",

      "topic.prefix": "erp",

      "plugin.name": "pgoutput",

      "table.include.list": "public.outbox_events",

      "slot.name": "erp_outbox_slot",
      "publication.autocreate.mode": "filtered",

      "tombstones.on.delete": "false"
    }
  }'
```

---

# 31. Que fait Debezium ?

Debezium ne fait pas :

```text
SELECT * FROM outbox_events toutes les secondes
```

Il lit les changements depuis le mécanisme de réplication PostgreSQL.

C'est du **CDC** :

```text
Change Data Capture
```

---

# 32. Flux CDC

```mermaid
flowchart LR
    ERP[ERP Transaction]
    PG[(PostgreSQL)]
    WAL[WAL]
    DBZ[Debezium]
    K[Kafka]

    ERP --> PG
    PG --> WAL
    WAL --> DBZ
    DBZ --> K
```

---

# 33. PostgreSQL WAL

WAL signifie :

```text
Write-Ahead Log
```

PostgreSQL écrit les changements dans un journal avant de les considérer comme durables.

Debezium s'appuie sur ce journal.

---

# 34. Topic généré par Debezium

Avec cette configuration, Debezium générera typiquement un topic proche de :

```text
erp.public.outbox_events
```

---

# 35. Format d'un événement Debezium

Un message CDC contient davantage que notre payload métier.

Exemple conceptuel :

```json
{
  "before": null,
  "after": {
    "id": "...",
    "aggregate_id": "...",
    "event_type": "order.created",
    "payload": {
      "order_id": "..."
    }
  },
  "op": "c",
  "source": {
    "table": "outbox_events"
  }
}
```

---

# 36. Pourquoi cela peut être gênant ?

Warehouse ne devrait pas avoir besoin de connaître :

```text
before
after
source
op
```

Il veut surtout :

```text
order.created
```

Dans une architecture plus avancée, on ajoutera une transformation Debezium Outbox Event Router.

---

# 37. Warehouse dependencies

Créer `warehouse/requirements.txt`.

```txt
sqlalchemy
psycopg2-binary
confluent-kafka
redis
```

---

# 38. Warehouse Dockerfile

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY consumer.py .

CMD ["python", "consumer.py"]
```

---

# 39. Warehouse Consumer

Créer `warehouse/consumer.py`.

```python
import json
import os
import time

import redis

from confluent_kafka import (
    Consumer,
    Producer
)

from sqlalchemy import (
    Column,
    Integer,
    String,
    create_engine,
    select
)

from sqlalchemy.orm import (
    declarative_base,
    sessionmaker
)


# -------------------------------------------------------------------
# Configuration
# -------------------------------------------------------------------

DATABASE_URL = os.environ[
    "DATABASE_URL"
]

KAFKA_BOOTSTRAP_SERVERS = os.environ[
    "KAFKA_BOOTSTRAP_SERVERS"
]

REDIS_URL = os.environ[
    "REDIS_URL"
]


# -------------------------------------------------------------------
# SQLAlchemy
# -------------------------------------------------------------------

engine = create_engine(
    DATABASE_URL,
    pool_pre_ping=True
)

SessionLocal = sessionmaker(
    bind=engine,
    autocommit=False,
    autoflush=False
)

Base = declarative_base()


class StockDB(Base):
    __tablename__ = "stocks"

    product_id = Column(
        String(100),
        primary_key=True
    )

    quantity = Column(
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
# Kafka Consumer
# -------------------------------------------------------------------

consumer = Consumer({
    "bootstrap.servers": KAFKA_BOOTSTRAP_SERVERS,

    # Toutes les instances Warehouse appartenant au même group.id
    # se répartiront les partitions.
    "group.id": "warehouse-service",

    # Si aucun offset n'existe encore, on démarre au plus ancien.
    "auto.offset.reset": "earliest",

    # Très important :
    # nous désactivons le commit automatique.
    #
    # Sinon Kafka pourrait considérer un message comme traité
    # avant la fin de notre transaction SQL.
    "enable.auto.commit": False
})


producer = Producer({
    "bootstrap.servers": KAFKA_BOOTSTRAP_SERVERS
})


consumer.subscribe([
    "erp.public.outbox_events"
])
```

---

# 40. Extraire le payload métier

Ajouter :

```python
def extract_outbox_event(message_value: bytes) -> dict:
    """
    Extrait la ligne Outbox depuis l'enveloppe Debezium.

    Le message Kafka contient une enveloppe CDC.
    """

    raw = json.loads(
        message_value.decode()
    )

    # Selon la configuration Debezium,
    # la structure exacte peut varier.
    payload = raw.get(
        "payload",
        raw
    )

    after = payload.get(
        "after"
    )

    if not after:
        raise ValueError(
            "No 'after' section in CDC event"
        )

    return after
```

---

# 41. Idempotence du consumer

Kafka peut livrer un événement plusieurs fois.

Donc :

```text
consumer doit être idempotent
```

Ajouter :

```python
def already_processed(event_id: str) -> bool:
    """
    Vérifie dans Redis si cet événement a déjà été traité.
    """

    return redis_client.exists(
        f"warehouse:processed:{event_id}"
    ) == 1


def mark_processed(event_id: str) -> None:
    """
    Mémorise le traitement pendant 24 heures.
    """

    redis_client.set(
        f"warehouse:processed:{event_id}",
        "1",
        ex=86400
    )
```

---

# 42. Pourquoi Redis n'est pas parfait ici ?

Il existe encore une petite fenêtre de panne.

Exemple :

```text
1. stock modifié en PostgreSQL
2. crash avant mark_processed Redis
3. message rejoué
4. stock potentiellement modifié deux fois
```

Le mécanisme réellement robuste doit idéalement conserver :

```text
effet métier + event_id traité
```

dans **la même transaction SQL**.

---

# 43. Table processed_events

Une meilleure solution :

```sql
CREATE TABLE processed_events (
    event_id VARCHAR(36) PRIMARY KEY,
    processed_at TIMESTAMPTZ NOT NULL
);
```

Ainsi :

```text
UPDATE stock
INSERT processed_event
COMMIT
```

sont atomiques.

---

# 44. Traitement Warehouse

Ajouter :

```python
def process_order_created(event: dict):
    """
    Réserve le stock.

    Retourne ensuite :
    - stock.reserved
    ou
    - stock.rejected
    """

    event_id = event["id"]

    # Protection simple contre les doublons.
    if already_processed(event_id):
        print(
            "Duplicate ignored:",
            event_id
        )
        return

    event_type = event["event_type"]

    if event_type != "order.created":
        return

    payload = event["payload"]

    order_id = payload["order_id"]
    product_id = payload["product_id"]
    quantity = payload["quantity"]

    db = SessionLocal()

    try:

        # Verrou pessimiste sur la ligne stock.
        stmt = (
            select(StockDB)
            .where(
                StockDB.product_id
                == product_id
            )
            .with_for_update()
        )

        stock = db.execute(
            stmt
        ).scalar_one_or_none()

        if (
            stock is None
            or stock.quantity < quantity
        ):

            result = {
                "event_id": event_id,
                "event_type": "stock.rejected",
                "order_id": order_id,
                "reason": "INSUFFICIENT_STOCK"
            }

        else:

            stock.quantity -= quantity

            result = {
                "event_id": event_id,
                "event_type": "stock.reserved",
                "order_id": order_id,
                "product_id": product_id,
                "quantity": quantity
            }

        db.commit()

        # Marquage idempotence.
        mark_processed(event_id)

        # Publication d'un nouvel événement.
        producer.produce(
            "warehouse.events",

            # La clé order_id maintient l'ordre
            # des événements d'une même commande.
            key=order_id,

            value=json.dumps(
                result
            ).encode()
        )

        producer.flush()

    except:

        db.rollback()
        raise

    finally:

        db.close()
```

---

# 45. Boucle principale Warehouse

Ajouter :

```python
def main():

    print(
        "Warehouse consumer started"
    )

    while True:

        # poll attend jusqu'à 1 seconde.
        msg = consumer.poll(
            1.0
        )

        if msg is None:
            continue

        if msg.error():
            print(
                "Kafka error:",
                msg.error()
            )
            continue

        try:

            event = extract_outbox_event(
                msg.value()
            )

            process_order_created(
                event
            )

            # Commit manuel uniquement après
            # un traitement réussi.
            consumer.commit(
                message=msg,
                asynchronous=False
            )

        except Exception as exc:

            print(
                "Processing failed:",
                repr(exc)
            )

            # Pas de commit.
            #
            # Au redémarrage, Kafka pourra
            # redélivrer le message.

            time.sleep(1)


if __name__ == "__main__":
    main()
```

---

# 46. At-least-once delivery

Avec cette stratégie :

```text
traitement réussi
puis
commit offset
```

Si le consumer tombe après le traitement mais avant le commit :

```text
message sera rejoué
```

Donc :

```text
at-least-once
```

---

# 47. Pourquoi l'idempotence est indispensable ?

Parce que :

```text
at-least-once
```

implique :

```text
un message peut être livré plusieurs fois
```

---

# 48. Payment Service

Créer `payment/requirements.txt`.

```txt
sqlalchemy
psycopg2-binary
confluent-kafka
redis
```

---

# 49. Payment Dockerfile

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY consumer.py .

CMD ["python", "consumer.py"]
```

---

# 50. Payment Consumer

Créer `payment/consumer.py`.

```python
import json
import os
import random
import time

from confluent_kafka import (
    Consumer,
    Producer
)


KAFKA_BOOTSTRAP_SERVERS = os.environ[
    "KAFKA_BOOTSTRAP_SERVERS"
]


consumer = Consumer({
    "bootstrap.servers": KAFKA_BOOTSTRAP_SERVERS,
    "group.id": "payment-service",
    "auto.offset.reset": "earliest",
    "enable.auto.commit": False
})


producer = Producer({
    "bootstrap.servers": KAFKA_BOOTSTRAP_SERVERS
})


consumer.subscribe([
    "erp.public.outbox_events"
])


def extract_event(value: bytes) -> dict:

    raw = json.loads(
        value.decode()
    )

    payload = raw.get(
        "payload",
        raw
    )

    return payload["after"]


def process_payment(event: dict):

    if event["event_type"] != "order.created":
        return

    payload = event["payload"]

    order_id = payload[
        "order_id"
    ]

    # Simulation pédagogique :
    # 80 % succès, 20 % échec.
    success = random.random() < 0.8

    if success:

        result = {
            "event_type": "payment.completed",
            "order_id": order_id
        }

    else:

        result = {
            "event_type": "payment.failed",
            "order_id": order_id,
            "reason": "PAYMENT_DECLINED"
        }

    producer.produce(
        "payment.events",
        key=order_id,
        value=json.dumps(
            result
        ).encode()
    )

    producer.flush()


def main():

    while True:

        msg = consumer.poll(
            1.0
        )

        if msg is None:
            continue

        if msg.error():
            print(msg.error())
            continue

        try:

            event = extract_event(
                msg.value()
            )

            process_payment(
                event
            )

            consumer.commit(
                message=msg,
                asynchronous=False
            )

        except Exception as exc:

            print(
                "Payment error:",
                repr(exc)
            )

            time.sleep(1)


if __name__ == "__main__":
    main()
```

---

# 51. Saga

La commande nécessite maintenant deux résultats :

```text
stock.reserved
payment.completed
```

La commande est confirmée seulement si les deux sont positifs.

---

# 52. États de Saga

```mermaid
stateDiagram-v2
    [*] --> PENDING

    PENDING --> STOCK_OK
    PENDING --> PAYMENT_OK

    STOCK_OK --> CONFIRMED: payment.completed
    PAYMENT_OK --> CONFIRMED: stock.reserved

    PENDING --> CANCELLED: stock.rejected
    PENDING --> CANCELLED: payment.failed

    STOCK_OK --> COMPENSATING: payment.failed
    COMPENSATING --> CANCELLED
```

---

# 53. ERP Consumer

Créer `erp/app/consumer.py`.

```python
import json
import os

from confluent_kafka import Consumer

from sqlalchemy import (
    create_engine
)

from sqlalchemy.orm import (
    sessionmaker
)

from app.main import OrderDB


DATABASE_URL = os.environ[
    "DATABASE_URL"
]

KAFKA_BOOTSTRAP_SERVERS = os.environ[
    "KAFKA_BOOTSTRAP_SERVERS"
]


engine = create_engine(
    DATABASE_URL,
    pool_pre_ping=True
)


SessionLocal = sessionmaker(
    bind=engine,
    autocommit=False,
    autoflush=False
)


consumer = Consumer({
    "bootstrap.servers": KAFKA_BOOTSTRAP_SERVERS,
    "group.id": "erp-saga",
    "auto.offset.reset": "earliest",
    "enable.auto.commit": False
})


consumer.subscribe([
    "warehouse.events",
    "payment.events"
])


# Etat pédagogique conservé en mémoire.
#
# En production, cet état doit être durable :
# PostgreSQL, Redis ou une table dédiée.
saga_state: dict[str, dict] = {}
```

---

# 54. Agréger les résultats

Ajouter :

```python
def handle_event(event: dict):

    order_id = event[
        "order_id"
    ]

    event_type = event[
        "event_type"
    ]

    state = saga_state.setdefault(
        order_id,
        {
            "stock": None,
            "payment": None
        }
    )


    # -----------------------------------------------------
    # Mise à jour de l'état Saga
    # -----------------------------------------------------

    if event_type == "stock.reserved":
        state["stock"] = True

    elif event_type == "stock.rejected":
        state["stock"] = False

    elif event_type == "payment.completed":
        state["payment"] = True

    elif event_type == "payment.failed":
        state["payment"] = False


    # -----------------------------------------------------
    # Calcul du statut final
    # -----------------------------------------------------

    if (
        state["stock"] is True
        and state["payment"] is True
    ):

        final_status = "CONFIRMED"

    elif (
        state["stock"] is False
        or state["payment"] is False
    ):

        final_status = "CANCELLED"

    else:

        # Il manque encore un résultat.
        return


    # -----------------------------------------------------
    # Mise à jour ERP
    # -----------------------------------------------------

    db = SessionLocal()

    try:

        order = db.get(
            OrderDB,
            order_id
        )

        if order is None:
            print(
                "Unknown order:",
                order_id
            )
            return

        order.status = final_status

        db.commit()

        print(
            "Order",
            order_id,
            "->",
            final_status
        )

    except:

        db.rollback()
        raise

    finally:

        db.close()
```

---

# 55. Boucle ERP Consumer

```python
def main():

    print(
        "ERP saga consumer started"
    )

    while True:

        msg = consumer.poll(
            1.0
        )

        if msg is None:
            continue

        if msg.error():
            print(
                msg.error()
            )
            continue

        try:

            event = json.loads(
                msg.value().decode()
            )

            handle_event(
                event
            )

            consumer.commit(
                message=msg,
                asynchronous=False
            )

        except Exception as exc:

            print(
                "ERP saga error:",
                repr(exc)
            )


if __name__ == "__main__":
    main()
```

---

# 56. Limite de cette Saga

L'état Saga est stocké ici :

```python
saga_state = {}
```

Donc si le conteneur ERP Consumer redémarre :

```text
état perdu
```

C'est volontaire.

L'exercice suivant consiste à rendre la Saga durable.

---

# 57. Exercice : Saga durable

Créer une table :

```sql
CREATE TABLE order_saga (
    order_id UUID PRIMARY KEY,
    stock_status VARCHAR(20),
    payment_status VARCHAR(20),
    updated_at TIMESTAMPTZ NOT NULL
);
```

L'état devient persistant.

---

# 58. Compensation

Scénario :

```text
stock.reserved
payment.failed
```

Il faut annuler la réservation.

ERP publie alors :

```text
stock.release.requested
```

Warehouse devra remettre le stock.

---

# 59. Saga avec compensation

```mermaid
sequenceDiagram
    participant ERP
    participant Kafka
    participant Warehouse
    participant Payment

    ERP->>Kafka: order.created

    Kafka->>Warehouse: order.created
    Kafka->>Payment: order.created

    Warehouse->>Kafka: stock.reserved
    Payment->>Kafka: payment.failed

    Kafka->>ERP: payment.failed

    ERP->>Kafka: stock.release.requested

    Kafka->>Warehouse: stock.release.requested

    Warehouse->>Kafka: stock.released
```

---

# 60. Créer les topics manuellement

Créer `scripts/create-topics.sh`.

```bash
#!/usr/bin/env bash

docker compose exec kafka \
  kafka-topics.sh \
  --bootstrap-server kafka:9092 \
  --create \
  --if-not-exists \
  --topic warehouse.events \
  --partitions 3 \
  --replication-factor 1


docker compose exec kafka \
  kafka-topics.sh \
  --bootstrap-server kafka:9092 \
  --create \
  --if-not-exists \
  --topic payment.events \
  --partitions 3 \
  --replication-factor 1


docker compose exec kafka \
  kafka-topics.sh \
  --bootstrap-server kafka:9092 \
  --create \
  --if-not-exists \
  --topic order.dlq \
  --partitions 1 \
  --replication-factor 1
```

---

# 61. Pourquoi trois partitions ?

Parce que plusieurs instances d'un même consumer group peuvent travailler en parallèle.

Avec :

```text
3 partitions
```

au maximum :

```text
3 consommateurs actifs
```

peuvent recevoir chacun une partition.

---

# 62. Que se passe-t-il avec 5 consumers ?

Si topic :

```text
3 partitions
```

et consumer group :

```text
5 instances
```

alors :

```text
3 travaillent
2 restent idle
```

---

# 63. Lancer l'architecture

```bash
docker compose up -d --build
```

---

# 64. Vérifier les conteneurs

```bash
docker compose ps
```

---

# 65. Voir les logs Kafka Connect

```bash
docker compose logs -f connect
```

---

# 66. Enregistrer Debezium

Linux/macOS :

```bash
bash scripts/register-connector.sh
```

Windows PowerShell :

```powershell
curl.exe -X POST `
  http://localhost:8083/connectors `
  -H "Content-Type: application/json" `
  --data-binary "@connector.json"
```

---

# 67. Vérifier le connecteur

```bash
curl \
  http://localhost:8083/connectors/erp-outbox-connector/status
```

---

# 68. Initialiser un client CRM

```bash
curl -X POST \
  http://localhost:8001/customers \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Alice Martin",
    "email": "alice@example.com",
    "active": true
  }'
```

---

# 69. Initialiser le stock

Comme Warehouse n'a pas d'API dans cette version, utiliser PostgreSQL directement.

```bash
docker compose exec warehouse-db \
  psql -U warehouse -d warehouse
```

Puis :

```sql
INSERT INTO stocks(product_id, quantity)
VALUES ('LAPTOP-001', 10);
```

---

# 70. Créer une commande

```bash
curl -i \
  -X POST \
  http://localhost:8000/orders \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: order-001" \
  -d '{
    "customer_id": 1,
    "product_id": "LAPTOP-001",
    "quantity": 2
  }'
```

---

# 71. Réponse attendue

```text
HTTP/1.1 202 Accepted
```

Body :

```json
{
  "id": "...",
  "customer_id": 1,
  "product_id": "LAPTOP-001",
  "quantity": 2,
  "status": "PENDING",
  "created_at": "..."
}
```

---

# 72. Observer l'Outbox

```bash
docker compose exec erp-db \
  psql -U erp -d erp
```

Puis :

```sql
SELECT * FROM outbox_events;
```

---

# 73. Observer les topics Kafka

```bash
docker compose exec kafka \
  kafka-topics.sh \
  --bootstrap-server kafka:9092 \
  --list
```

---

# 74. Lire un topic

```bash
docker compose exec kafka \
  kafka-console-consumer.sh \
  --bootstrap-server kafka:9092 \
  --topic warehouse.events \
  --from-beginning
```

---

# 75. Vérifier la commande

```bash
curl \
  http://localhost:8000/orders/<ORDER_ID>
```

Après quelques instants :

```text
CONFIRMED
```

ou :

```text
CANCELLED
```

---

# 76. Cohérence éventuelle

Juste après le POST :

```text
PENDING
```

Puis plus tard :

```text
CONFIRMED
```

Le système n'est pas immédiatement cohérent.

C'est de la :

```text
cohérence éventuelle
```

---

# 77. Dead Letter Queue

Un message peut être impossible à traiter.

Exemple :

```text
JSON invalide
champ obligatoire absent
version incompatible
```

Au lieu de retry indéfiniment :

```text
message -> DLQ
```

---

# 78. Fonction DLQ

Exemple :

```python
def send_to_dlq(
    producer,
    original_message,
    reason: str
):

    payload = {
        "reason": reason,
        "original_value": (
            original_message
            .value()
            .decode(
                errors="replace"
            )
        )
    }

    producer.produce(
        "order.dlq",
        value=json.dumps(
            payload
        ).encode()
    )

    producer.flush()
```

---

# 79. Retry Topic

Une stratégie plus propre :

```text
topic principal
   |
échec
   v
retry-5s
   |
retry
   v
retry-30s
   |
retry
   v
DLQ
```

---

# 80. Pourquoi ne pas retry immédiatement ?

Parce qu'une erreur externe peut durer plusieurs secondes.

Retry immédiat :

```text
augmente la charge
```

---

# 81. Poison Message

Un message peut toujours échouer.

Exemple :

```text
payload cassé
```

Même après 100 retries :

```text
il restera cassé
```

C'est un **poison message**.

---

# 82. Exactly Once

Kafka possède certaines garanties transactionnelles.

Mais :

```text
exactly-once de bout en bout
```

entre :

```text
Kafka
PostgreSQL
API externe
Redis
```

reste complexe.

---

# 83. Stratégie réaliste

On privilégie souvent :

```text
at-least-once
+
idempotence
```

---

# 84. Idempotence SQL

Créer :

```sql
CREATE TABLE processed_events (
    event_id VARCHAR(36) PRIMARY KEY,
    processed_at TIMESTAMPTZ NOT NULL
);
```

Puis :

```text
BEGIN

SELECT event_id

si absent :
    effet métier
    INSERT processed_events

COMMIT
```

---

# 85. Concurrence Kafka

Si plusieurs événements d'une commande sont consommés en parallèle sur différentes partitions :

```text
ordre potentiellement incorrect
```

Solution :

```text
key = order_id
```

---

# 86. Test d'ordre

Produire :

```text
order.created
stock.reserved
payment.completed
order.confirmed
```

avec la même clé Kafka.

Observer les offsets.

---

# 87. Consumer Lag

Le lag mesure :

```text
messages produits
-
messages consommés
```

Exemple :

```text
latest offset = 10000
consumer offset = 9500
lag = 500
```

---

# 88. Pourquoi surveiller le lag ?

Un lag élevé peut signifier :

- consumer trop lent ;
- trop peu de partitions ;
- traitement coûteux ;
- base lente ;
- panne partielle.

---

# 89. Voir les consumer groups

```bash
docker compose exec kafka \
  kafka-consumer-groups.sh \
  --bootstrap-server kafka:9092 \
  --list
```

---

# 90. Voir le lag

```bash
docker compose exec kafka \
  kafka-consumer-groups.sh \
  --bootstrap-server kafka:9092 \
  --describe \
  --group warehouse-service
```

---

# 91. Test de panne Warehouse

```bash
docker compose stop warehouse
```

Créer plusieurs commandes.

Observer :

```text
Kafka continue de stocker les événements
```

Puis :

```bash
docker compose start warehouse
```

Warehouse rattrape son retard.

---

# 92. Différence avec REST

Avec REST synchrone :

```text
Warehouse down
=> commande bloquée
```

Avec Kafka :

```text
Warehouse down
=> événements restent dans Kafka
=> traitement reprend ensuite
```

---

# 93. Test de panne ERP Consumer

```bash
docker compose stop erp-consumer
```

Warehouse et Payment continuent.

La commande reste temporairement :

```text
PENDING
```

Après redémarrage :

```bash
docker compose start erp-consumer
```

le consumer rattrape les événements.

---

# 94. Test de duplication

Rejouer manuellement un événement Kafka.

Le résultat métier ne doit pas être exécuté deux fois.

---

# 95. Pourquoi les consumers doivent être idempotents ?

Kafka peut redélivrer un message après :

- crash ;
- rebalance ;
- timeout ;
- commit offset tardif.

---

# 96. Rebalance

Lorsqu'un consumer entre ou quitte un groupe :

```text
Kafka redistribue les partitions
```

C'est un :

```text
rebalance
```

---

# 97. Scénario rebalance

Lancer :

```text
warehouse consumer 1
warehouse consumer 2
```

Puis stopper l'un des deux.

Les partitions sont redistribuées.

---

# 98. Scale Warehouse

```bash
docker compose up -d \
  --scale warehouse=3
```

---

# 99. Limite du container_name

Avec :

```yaml
container_name: warehouse
```

Docker Compose ne peut pas facilement créer plusieurs instances.

Pour tester le scaling :

```text
retirer container_name
```

---

# 100. Backpressure

Kafka permet naturellement d'absorber un pic.

Exemple :

```text
10000 commandes arrivent rapidement
Warehouse traite 500/s
```

Kafka garde le retard.

---

# 101. Attention : Kafka n'est pas magique

Si le débit entrant reste durablement supérieur au débit de traitement :

```text
lag augmente sans fin
```

Il faut :

- augmenter partitions ;
- scaler consumers ;
- optimiser traitement ;
- réduire trafic ;
- appliquer rate limiting.

---

# 102. Schema Registry

Dans une architecture réelle, les événements doivent respecter un schéma.

Exemple :

```json
{
  "event_type": "order.created",
  "version": 1,
  "order_id": "uuid"
}
```

---

# 103. Pourquoi versionner les événements ?

Aujourd'hui :

```json
{
  "order_id": "123",
  "quantity": 2
}
```

Demain :

```json
{
  "order_id": "123",
  "quantity": 2,
  "currency": "EUR"
}
```

Les anciens consumers doivent continuer à fonctionner.

---

# 104. Backward Compatibility

Un nouveau schéma est backward compatible si :

```text
nouveau consumer
peut lire
anciens messages
```

---

# 105. Forward Compatibility

Un ancien consumer peut continuer à lire un message produit avec le nouveau schéma.

---

# 106. Version dans le payload

Approche simple :

```json
{
  "event_type": "order.created",
  "schema_version": 1,
  "payload": {}
}
```

---

# 107. Correlation ID

Chaque commande doit posséder un identifiant permettant de suivre tout le parcours.

Exemple :

```text
X-Correlation-ID
```

---

# 108. Propagation dans Kafka

Un événement peut contenir :

```json
{
  "correlation_id": "...",
  "event_id": "...",
  "order_id": "..."
}
```

---

# 109. Pourquoi event_id et correlation_id sont différents ?

`event_id` :

```text
identifie un événement unique
```

`correlation_id` :

```text
relie plusieurs événements du même flux métier
```

---

# 110. Exemple

```text
correlation_id = C1

event_id E1 -> order.created
event_id E2 -> stock.reserved
event_id E3 -> payment.completed
event_id E4 -> order.confirmed
```

---

# 111. Logs structurés

Exemple :

```json
{
  "service": "warehouse",
  "level": "INFO",
  "event": "stock_reserved",
  "order_id": "...",
  "correlation_id": "...",
  "duration_ms": 32
}
```

---

# 112. Mesurer la latence de bout en bout

On veut mesurer :

```text
T_confirmed - T_order_created
```

Cela donne le temps réel du workflow distribué.

---

# 113. p50 / p95 / p99

Sur 1000 commandes :

```text
p50 = 180 ms
p95 = 900 ms
p99 = 3100 ms
```

Cela signifie :

```text
99 % des commandes terminent en moins de 3,1 secondes
```

---

# 114. Exercice : ajouter latency_ms

Lors de la confirmation :

```python
latency_ms = (
    datetime.now(timezone.utc)
    - order.created_at
).total_seconds() * 1000
```

Logger la valeur.

---

# 115. Chaos Testing

Nous allons volontairement casser des composants.

Objectif :

```text
observer le comportement
```

et non :

```text
espérer qu'il ne se passe rien
```

---

# 116. Chaos 1 : arrêter Kafka

```bash
docker compose stop kafka
```

Observer :

- erreurs producers ;
- erreurs consumers ;
- comportement Debezium ;
- reconnexion.

---

# 117. Chaos 2 : arrêter PostgreSQL ERP

```bash
docker compose stop erp-db
```

Observer :

- API ERP ;
- Debezium ;
- consumers.

---

# 118. Chaos 3 : ralentir Payment

Ajouter :

```python
time.sleep(5)
```

dans `process_payment()`.

Observer :

```text
latence Saga
```

---

# 119. Chaos 4 : produire beaucoup de commandes

Créer un script concurrent.

```python
import asyncio
import uuid

import httpx


async def create_order(index: int):

    async with httpx.AsyncClient(
        timeout=10
    ) as client:

        response = await client.post(
            "http://localhost:8000/orders",
            headers={
                "Idempotency-Key": str(
                    uuid.uuid4()
                )
            },
            json={
                "customer_id": 1,
                "product_id": "LAPTOP-001",
                "quantity": 1
            }
        )

        return (
            index,
            response.status_code
        )


async def main():

    tasks = [
        create_order(i)
        for i in range(100)
    ]

    results = await asyncio.gather(
        *tasks
    )

    print(results)


asyncio.run(
    main()
)
```

---

# 120. Observer le lag après charge

```bash
docker compose exec kafka \
  kafka-consumer-groups.sh \
  --bootstrap-server kafka:9092 \
  --describe \
  --group warehouse-service
```

---

# 121. Exercice : Dead Letter Queue réelle

Modifier Warehouse :

```text
après 3 erreurs
=> publier dans order.dlq
=> commit offset original
```

---

# 122. Exercice : Retry Topics

Créer :

```text
warehouse.retry.5s
warehouse.retry.30s
warehouse.dlq
```

---

# 123. Exercice : Compensation complète

Si :

```text
stock réservé
paiement échoué
```

ERP publie :

```text
stock.release.requested
```

Warehouse remet la quantité.

---

# 124. Exercice : Consumer idempotent SQL

Remplacer Redis par une table :

```text
processed_events
```

et inclure cette écriture dans la transaction SQL métier.

---

# 125. Exercice : Optimistic Locking

Ajouter :

```text
version
```

dans la table `orders`.

Lors d'une mise à jour :

```sql
UPDATE orders
SET status = 'CONFIRMED',
    version = version + 1
WHERE id = ?
AND version = ?;
```

---

# 126. Optimistic vs Pessimistic Locking

## Optimistic

On suppose :

```text
conflits rares
```

et on détecte le conflit à l'écriture.

## Pessimistic

On verrouille avant modification.

Exemple :

```sql
SELECT ... FOR UPDATE
```

---

# 127. Isolation SQL

PostgreSQL propose plusieurs niveaux.

```text
READ COMMITTED
REPEATABLE READ
SERIALIZABLE
```

---

# 128. READ COMMITTED

Chaque requête voit uniquement les données déjà validées.

Mais deux SELECT dans une même transaction peuvent voir des résultats différents.

---

# 129. REPEATABLE READ

Une transaction conserve une vue cohérente des données.

---

# 130. SERIALIZABLE

PostgreSQL tente de garantir un comportement équivalent à des transactions exécutées en série.

Mais :

```text
des transactions peuvent être annulées
```

et doivent être rejouées.

---

# 131. Exercice : conflit SERIALIZABLE

Lancer deux transactions concurrentes modifiant le même stock.

Observer :

```text
serialization failure
```

---

# 132. Event Sourcing

Kafka ressemble à un journal d'événements.

Mais utiliser Kafka ne signifie pas automatiquement faire de l'Event Sourcing.

---

# 133. Event Sourcing

Dans l'Event Sourcing :

```text
l'état courant
=
reconstruction depuis les événements
```

Exemple :

```text
OrderCreated
StockReserved
PaymentCompleted
OrderConfirmed
```

---

# 134. CQRS

CQRS sépare :

```text
Command
Query
```

Écriture :

```text
commande métier
```

Lecture :

```text
modèle optimisé de consultation
```

---

# 135. Projection

Un consumer peut construire une table de lecture.

Exemple :

```text
order_dashboard
```

alimentée par les événements Kafka.

---

# 136. Exercice : projection Analytics

Créer un consumer :

```text
analytics
```

qui maintient :

```text
nombre commandes
commandes confirmées
commandes annulées
CA simulé
```

---

# 137. Pourquoi une projection est intéressante ?

Elle évite d'interroger plusieurs microservices en temps réel pour construire un dashboard.

---

# 138. Relecture des événements

Kafka conserve les messages pendant une durée configurée.

On peut créer un nouveau consumer et relire l'historique.

---

# 139. Cas d'usage

Créer après coup :

```text
analytics-service
```

et reconstruire ses données à partir de Kafka.

---

# 140. Attention aux événements anciens

Si le schéma a changé :

```text
consumer moderne
doit pouvoir comprendre
anciens événements
```

D'où :

```text
versionnement
Schema Registry
compatibilité
```

---

# 141. Test d'intégration

Créer `tests/test_order_flow.py`.

```python
import time
import uuid

import httpx


def test_order_event_driven_flow():

    # -----------------------------------------------------
    # Création client
    # -----------------------------------------------------

    customer = httpx.post(
        "http://localhost:8001/customers",
        json={
            "name": "Integration User",
            "email": f"{uuid.uuid4()}@example.com",
            "active": True
        }
    )

    assert customer.status_code == 201

    customer_id = customer.json()[
        "id"
    ]


    # -----------------------------------------------------
    # Création commande
    # -----------------------------------------------------

    order = httpx.post(
        "http://localhost:8000/orders",
        headers={
            "Idempotency-Key": str(
                uuid.uuid4()
            )
        },
        json={
            "customer_id": customer_id,
            "product_id": "LAPTOP-001",
            "quantity": 1
        }
    )

    assert order.status_code == 202

    order_id = order.json()[
        "id"
    ]


    # -----------------------------------------------------
    # Polling du statut
    # -----------------------------------------------------

    deadline = time.time() + 15

    final_status = None

    while time.time() < deadline:

        response = httpx.get(
            f"http://localhost:8000/orders/{order_id}"
        )

        assert response.status_code == 200

        current_status = response.json()[
            "status"
        ]

        if current_status in {
            "CONFIRMED",
            "CANCELLED"
        }:
            final_status = current_status
            break

        time.sleep(0.5)


    assert final_status in {
        "CONFIRMED",
        "CANCELLED"
    }
```

---

# 142. Pourquoi le test utilise du polling ?

Parce que le workflow est asynchrone.

On ne sait pas exactement quand :

```text
Warehouse
Payment
ERP Consumer
```

auront fini.

---

# 143. Timeout de test

Un test distribué ne doit jamais attendre indéfiniment.

On définit :

```text
deadline = maintenant + 15 secondes
```

---

# 144. Debug Kafka

Lister topics :

```bash
docker compose exec kafka \
  kafka-topics.sh \
  --bootstrap-server kafka:9092 \
  --list
```

---

# 145. Décrire un topic

```bash
docker compose exec kafka \
  kafka-topics.sh \
  --bootstrap-server kafka:9092 \
  --describe \
  --topic warehouse.events
```

---

# 146. Consumer console

```bash
docker compose exec kafka \
  kafka-console-consumer.sh \
  --bootstrap-server kafka:9092 \
  --topic payment.events \
  --from-beginning
```

---

# 147. Producer console

```bash
docker compose exec -it kafka \
  kafka-console-producer.sh \
  --bootstrap-server kafka:9092 \
  --topic payment.events
```

---

# 148. Voir les logs Warehouse

```bash
docker compose logs -f warehouse
```

---

# 149. Voir les logs ERP Consumer

```bash
docker compose logs -f erp-consumer
```

---

# 150. Voir Debezium

```bash
docker compose logs -f connect
```

---

# 151. Voir PostgreSQL ERP

```bash
docker compose exec erp-db \
  psql -U erp -d erp
```

---

# 152. Inspecter les commandes

```sql
SELECT
    id,
    status,
    customer_id,
    product_id,
    quantity,
    created_at
FROM orders;
```

---

# 153. Inspecter l'Outbox

```sql
SELECT
    id,
    aggregate_id,
    event_type,
    created_at
FROM outbox_events
ORDER BY created_at;
```

---

# 154. Analyse de panne : Debezium arrêté

```bash
docker compose stop connect
```

Créer une commande.

Observer :

```text
commande PENDING
outbox créée
aucun event Kafka
```

Puis :

```bash
docker compose start connect
```

Debezium reprend depuis son offset CDC.

---

# 155. Pourquoi c'est puissant ?

Parce que l'événement n'est pas perdu.

Il était durablement stocké dans PostgreSQL.

---

# 156. Analyse de panne : Kafka arrêté après COMMIT

Avec Outbox :

```text
commande + outbox restent en DB
```

Après retour de Kafka :

```text
Debezium peut continuer
```

---

# 157. Limites volontaires du TP

Cette version est avancée mais reste pédagogique.

Elle simplifie plusieurs éléments :

- pas de TLS Kafka ;
- pas de SASL ;
- un seul broker ;
- réplication Kafka = 1 ;
- Saga temporairement en mémoire ;
- pas de vrai Schema Registry intégré ;
- peu de métriques ;
- pas d'OpenTelemetry ;
- pas de Kubernetes ;
- pas de véritables services de paiement externes.

---

# 158. Production : ce qu'il faudrait ajouter

Dans une vraie architecture :

- cluster Kafka multi-brokers ;
- TLS ;
- authentification SASL ;
- ACL ;
- Schema Registry ;
- Avro / Protobuf ;
- monitoring Kafka ;
- consumer lag alerts ;
- OpenTelemetry ;
- Prometheus ;
- Grafana ;
- traces distribuées ;
- secrets manager ;
- migrations Alembic ;
- tests contractuels ;
- autoscaling ;
- DLQ opérationnelle ;
- replay tooling ;
- runbooks.

---

# 159. Tableau des expériences à réaliser

| Expérience | Manipulation | Résultat attendu | Concept |
|---|---|---|---|
| créer commande | `POST /orders` | `PENDING` puis final | event-driven |
| stopper Warehouse | `docker compose stop warehouse` | lag augmente | découplage |
| relancer Warehouse | `docker compose start warehouse` | backlog rattrapé | replay |
| stopper Debezium | `stop connect` | outbox s'accumule | CDC |
| relancer Debezium | `start connect` | événements repris | durabilité |
| même Idempotency-Key | rejouer POST | même commande | idempotence |
| message dupliqué | rejouer event | effet unique | consumer idempotent |
| payment lent | `sleep(5)` | Saga plus lente | latence |
| plusieurs consumers | scale | partitions réparties | consumer group |
| poison message | JSON cassé | DLQ | gestion erreurs |

---

# 160. Questions de synthèse

1. Pourquoi Kafka ne remplace-t-il pas automatiquement une base de données ?
2. Pourquoi l'Outbox Pattern évite-t-il le dual write ?
3. Que signifie `at-least-once` ?
4. Pourquoi faut-il des consumers idempotents ?
5. Quelle différence entre `event_id` et `correlation_id` ?
6. À quoi sert une clé Kafka ?
7. Pourquoi l'ordre n'est-il garanti qu'à l'intérieur d'une partition ?
8. Qu'est-ce qu'un consumer group ?
9. Pourquoi le lag est-il une métrique importante ?
10. Qu'est-ce qu'une Saga ?
11. Qu'est-ce qu'une compensation ?
12. Pourquoi utiliser une DLQ ?
13. Qu'est-ce qu'un poison message ?
14. Pourquoi versionner les événements ?
15. Qu'est-ce que le CDC ?
16. Pourquoi Debezium est-il préférable à un polling SQL naïf ?
17. Quelle différence entre event-driven et event sourcing ?
18. Qu'est-ce que CQRS ?
19. Pourquoi la cohérence est-elle éventuelle ?
20. Pourquoi tester les pannes est-il indispensable ?

---

# 161. Corrections synthétiques

## 1. Kafka vs base de données

Kafka est principalement un journal distribué d'événements.

Une base relationnelle reste mieux adaptée à de nombreux besoins transactionnels et requêtes complexes.

---

## 2. Outbox Pattern

Commande et événement Outbox sont écrits dans la même transaction.

On évite donc :

```text
DB réussie
Kafka échoué
```

---

## 3. At-least-once

Un message doit être traité au moins une fois mais peut être redélivré.

---

## 4. Consumer idempotent

Parce qu'un événement dupliqué ne doit pas produire un double effet métier.

---

## 5. Event ID vs Correlation ID

`event_id` identifie un message unique.

`correlation_id` relie plusieurs événements d'un même processus.

---

## 6. Clé Kafka

Elle influence la partition.

Une même clé permet de conserver l'ordre relatif des événements du même agrégat.

---

## 7. Ordre

Kafka maintient un ordre dans une partition, pas entre toutes les partitions d'un topic.

---

## 8. Consumer Group

Plusieurs instances d'un même service se partagent les partitions.

---

## 9. Lag

Il indique le retard du consumer.

---

## 10. Saga

Une Saga coordonne plusieurs transactions locales réparties entre plusieurs services.

---

## 11. Compensation

Une compensation annule logiquement une étape déjà effectuée.

---

## 12. DLQ

Elle isole les messages impossibles à traiter.

---

## 13. Poison Message

Message qui échoue toujours quelle que soit la quantité de retries.

---

## 14. Versionnement

Il permet de faire évoluer le contrat des événements sans casser les consommateurs existants.

---

## 15. CDC

Capture des changements de données.

---

## 16. Debezium vs polling

Debezium lit le flux de changements PostgreSQL au lieu d'interroger régulièrement les tables.

---

## 17. Event-driven vs Event Sourcing

Event-driven décrit une architecture pilotée par événements.

Event Sourcing stocke les événements comme source principale de vérité métier.

---

## 18. CQRS

Séparation des modèles d'écriture et de lecture.

---

## 19. Cohérence éventuelle

Les services ne sont pas tous mis à jour au même instant.

---

## 20. Chaos testing

Parce qu'un système distribué doit être conçu pour fonctionner malgré les pannes partielles.

---

# 162. Barème proposé

| Critère | Points |
|---|---:|
| Docker Compose | 2 |
| Kafka | 2 |
| Debezium CDC | 2 |
| Outbox Pattern | 3 |
| Warehouse Consumer | 2 |
| Payment Consumer | 1 |
| Saga ERP | 3 |
| Idempotence | 2 |
| DLQ / retries | 1 |
| Tests de panne | 1 |
| Analyse | 1 |
| **Total** | **20** |

---

# 163. Livrables

Le rendu doit contenir :

```text
tp3-event-driven/
|
|-- docker-compose.yml
|-- .env.example
|
|-- erp/
|-- crm/
|-- warehouse/
|-- payment/
|
|-- scripts/
|-- tests/
|
|-- README.md
```

---

# 164. README attendu

Le README doit expliquer :

1. architecture globale ;
2. rôle de chaque service ;
3. rôle de Kafka ;
4. rôle de Debezium ;
5. rôle de l'Outbox ;
6. démarrage Docker ;
7. enregistrement du connecteur ;
8. création des topics ;
9. création d'une commande ;
10. observation du workflow ;
11. tests de panne ;
12. test d'idempotence ;
13. consumer lag ;
14. scénario de Saga ;
15. limitations.

---

# 165. Extensions expert+

Pour aller encore plus loin :

- Schema Registry réel ;
- Avro ;
- Protobuf ;
- Kafka transactions ;
- exactly-once semantics ;
- Kafka Streams ;
- ksqlDB ;
- Debezium Outbox Event Router ;
- OpenTelemetry ;
- Prometheus ;
- Grafana ;
- Jaeger ;
- Redpanda ;
- Testcontainers ;
- Pact ;
- Chaos Mesh ;
- Kubernetes ;
- Strimzi Kafka Operator ;
- autoscaling basé sur consumer lag ;
- Event Sourcing complet ;
- CQRS complet ;
- materialized views ;
- CDC multi-systèmes ;
- lakehouse ;
- Apache Flink ;
- Spark Structured Streaming.

---

# 166. Conclusion

Ce TP introduit un changement important de raisonnement.

Dans une architecture synchrone :

```text
service A appelle service B
```

Dans une architecture event-driven :

```text
service A publie un fait
service B réagit à ce fait
```

Le système devient plus résilient et plus découplé, mais aussi plus complexe.

Il faut alors maîtriser :

- Kafka ;
- topics ;
- partitions ;
- consumer groups ;
- offsets ;
- CDC ;
- Outbox Pattern ;
- idempotence ;
- Saga ;
- compensation ;
- DLQ ;
- retries ;
- ordering ;
- consumer lag ;
- schémas ;
- cohérence éventuelle ;
- observabilité ;
- chaos testing.

Ces concepts constituent une base solide pour travailler sur des architectures SI distribuées modernes, des plateformes de données temps réel et des systèmes d'intégration ERP / CRM / Warehouse à grande échelle.
