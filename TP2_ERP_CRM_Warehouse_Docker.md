# TP 2 - Intégration multi-systèmes : ERP, CRM et Warehouse avec Docker

**Version fortement commentée - édition pédagogique**

## APIs distribuées, concurrence, latence, résilience, idempotence, messaging et observabilité

**Niveau :** avancé à très avancé  
**Durée conseillée :** 6 h à 10 h  
**Architecture :** microservices pédagogiques conteneurisés  
**Technologies :** Python 3.11+, FastAPI, PostgreSQL, Redis, RabbitMQ, Docker, Docker Compose, HTTPX, Pytest, Locust  
**Systèmes :** Linux, macOS, Windows PowerShell et Windows CMD

---

# 1. Objectifs du TP

**À la fin de ce TP, vous serez capable de** :

- construire une architecture composée de plusieurs systèmes métiers ;
- simuler un **ERP**, un **CRM** et un **Warehouse Management System** ;
- faire communiquer plusieurs services par **API REST** ;
- mettre en place des échanges **asynchrones** avec **RabbitMQ** ;
- comprendre les problèmes de **latence réseau** ;
- simuler des erreurs et des pannes ;
- utiliser des **timeouts** ;
- appliquer des **retries avec backoff** ;
- implémenter une forme simple de **circuit breaker** ;
- comprendre la **concurrence** entre plusieurs requêtes simultanées ;
- prévenir les doubles traitements avec une **clé d'idempotence** ;
- gérer les risques de **race condition** ;
- introduire Redis pour certaines coordinations ;
- mesurer les performances d'une architecture distribuée ;
- effectuer des **tests de charge** ;
- centraliser des métriques simples ;
- comprendre les différences entre appels synchrones et événements asynchrones ;
- orchestrer tous les composants avec **Docker Compose**.

---

# 2. Contexte métier

**Une entreprise possède trois systèmes principaux** :

## CRM

**Le CRM gère** :

- les clients ;
- leurs informations ;
- leur statut commercial.

## ERP

**L'ERP gère** :

- les commandes ;
- la facturation ;
- le prix des produits ;
- l'état métier de la commande.

## Warehouse

**Le Warehouse Management System gère** :

- le stock ;
- la réservation des produits ;
- la préparation logistique.

---

# 3. Scénario métier principal

Lorsqu'un client passe une commande :

1. le client est vérifié dans le CRM ;
2. l'ERP crée la commande ;
3. l'ERP demande au Warehouse de réserver le stock ;
4. le Warehouse répond ;
5. l'ERP confirme ou refuse la commande ;
6. un événement est publié pour signaler la création ou le refus.

---

# 4. Architecture cible

```mermaid
flowchart LR
    U[Client]
    G[API Gateway / ERP]
    C[CRM API]
    W[Warehouse API]
    R[(Redis)]
    E[(ERP PostgreSQL)]
    CDB[(CRM PostgreSQL)]
    WDB[(Warehouse PostgreSQL)]
    MQ[RabbitMQ]
    N[Notification Worker]

    U --> G
    G --> C
    G --> W
    G --> E
    G --> R
    C --> CDB
    W --> WDB
    G --> MQ
    MQ --> N
```

---

# 4.1. Pourquoi trois bases PostgreSQL ?

Dans ce TP, chaque système possède sa propre base :

```text
CRM       -> crm-db
ERP       -> erp-db
Warehouse -> warehouse-db
```

Ce choix est volontaire.

Il permet de reproduire une architecture où chaque domaine métier est propriétaire de ses données.

### Avantage

Le CRM peut évoluer indépendamment du Warehouse.

### Inconvénient

Une transaction SQL unique ne peut plus englober tout le processus métier.

Par exemple :

```text
ERP commit OK
Warehouse commit OK
RabbitMQ KO
```

Le système peut se retrouver dans un état partiellement cohérent.

C'est précisément pour étudier ce problème que le TP introduit ensuite :

- Saga ;
- Outbox Pattern ;
- idempotence ;
- compensation ;
- événements.

---

# 5. Flux métier principal

```mermaid
sequenceDiagram
    participant Client
    participant ERP
    participant CRM
    participant Warehouse
    participant RabbitMQ
    participant Worker

    Client->>ERP: POST /orders
    ERP->>CRM: GET /customers/{id}
    CRM-->>ERP: Customer
    ERP->>Warehouse: POST /reservations
    Warehouse-->>ERP: Stock reserved
    ERP->>ERP: Persist order
    ERP->>RabbitMQ: order.created
    ERP-->>Client: 201 Created
    RabbitMQ->>Worker: order.created
    Worker->>Worker: Process event
```

---

# 6. Problèmes étudiés

Ce TP ne se limite pas au CRUD.

Nous allons volontairement introduire des problèmes typiques des systèmes distribués :

- latence réseau ;
- service indisponible ;
- timeout ;
- erreurs HTTP 500 ;
- requêtes concurrentes ;
- double clic utilisateur ;
- doublon de message ;
- race condition sur le stock ;
- retry excessif ;
- surcharge ;
- dépendance lente ;
- événements asynchrones ;
- cohérence éventuelle.


---

# 6.1. Comment lire le code de ce TP

Dans ce TP, les commentaires de code utilisent quatre niveaux d'explication :

```text
1. CE QUE FAIT LE CODE
2. POURQUOI ON LE FAIT
3. CE QUI PEUT MAL SE PASSER
4. COMMENT ON LE FERAIT EN PRODUCTION
```

Exemple :

```python
# 1. On ouvre une session SQL.
db = SessionLocal()

# 2. Une session par requête évite de partager le même contexte transactionnel.

# 3. Si on oublie de fermer la session, le pool de connexions peut se saturer.

# 4. En production, on encapsule souvent cela dans une dépendance FastAPI.
```

> **Objectif pédagogique :** ne pas simplement recopier les commandes, mais comprendre les conséquences de chaque choix d'architecture.

---

# 7. Arborescence du projet

```text
tp2-distributed/
|
|-- docker-compose.yml
|-- .env
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
|   |-- app/
|       |-- __init__.py
|       |-- main.py
|
|-- erp/
|   |-- Dockerfile
|   |-- requirements.txt
|   |-- app/
|       |-- __init__.py
|       |-- main.py
|
|-- worker/
|   |-- Dockerfile
|   |-- requirements.txt
|   |-- worker.py
|
|-- load-tests/
|   |-- locustfile.py
|
|-- tests/
|   |-- test_scenarios.py
|
|-- README.md
```

---

# 8. Création des dossiers

## Linux / macOS

```bash
mkdir -p tp2-distributed/{crm/app,warehouse/app,erp/app,worker,load-tests,tests}
cd tp2-distributed

touch crm/app/__init__.py
touch warehouse/app/__init__.py
touch erp/app/__init__.py
```

## Windows PowerShell

```powershell
mkdir tp2-distributed
cd tp2-distributed

mkdir crm
mkdir crm\app

mkdir warehouse
mkdir warehouse\app

mkdir erp
mkdir erp\app

mkdir worker
mkdir load-tests
mkdir tests

New-Item crm\app\__init__.py -ItemType File
New-Item warehouse\app\__init__.py -ItemType File
New-Item erp\app\__init__.py -ItemType File
```

---

# 9. Fichier .env

Créer :

```env
CRM_DB_URL=postgresql://crm:crm@crm-db:5432/crm
WAREHOUSE_DB_URL=postgresql://warehouse:warehouse@warehouse-db:5432/warehouse
ERP_DB_URL=postgresql://erp:erp@erp-db:5432/erp

CRM_URL=http://crm:8000
WAREHOUSE_URL=http://warehouse:8000

REDIS_URL=redis://redis:6379/0

RABBITMQ_URL=amqp://guest:guest@rabbitmq:5672/
```

---

# 10. Docker Compose complet

Créer `docker-compose.yml` :

```yaml
# docker-compose.yml
#
# Ce fichier orchestre l'ensemble du système distribué.
# Chaque bloc sous `services:` représente un conteneur.
#
# Important :
# - les services communiquent entre eux via le réseau Docker interne ;
# - le nom du service devient automatiquement un nom DNS ;
# - par exemple l'ERP peut joindre le CRM via http://crm:8000 ;
# - les ports exposés vers localhost servent surtout aux tests depuis la machine hôte.

services:

  # ------------------------------------------------------------------
  # Base de données du CRM
  # ------------------------------------------------------------------
  crm-db:
    # Image officielle PostgreSQL.
    image: postgres:16

    # Nom explicite du conteneur, utile pour `docker ps` et `docker logs`.
    container_name: crm-db

    # Variables utilisées par l'image PostgreSQL lors de l'initialisation.
    environment:
      POSTGRES_USER: crm
      POSTGRES_PASSWORD: crm
      POSTGRES_DB: crm

    # Le port 5432 est le port PostgreSQL DANS le conteneur.
    # Nous le publions sur 5433 côté hôte pour éviter les conflits
    # avec une éventuelle installation PostgreSQL locale.
    ports:
      - "5433:5432"

    # Le volume rend les données persistantes :
    # elles survivent à un simple `docker compose down`.
    volumes:
      - crm_data:/var/lib/postgresql/data

    # Le healthcheck évite de démarrer trop tôt les services dépendants.
    # `pg_isready` teste si PostgreSQL accepte réellement les connexions.
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U crm -d crm"]
      interval: 5s
      timeout: 3s
      retries: 10

  # ------------------------------------------------------------------
  # Base de données du Warehouse
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
      retries: 10

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
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U erp -d erp"]
      interval: 5s
      timeout: 3s
      retries: 10

  redis:
    image: redis:7
    container_name: redis
    ports:
      - "6379:6379"

  rabbitmq:
    image: rabbitmq:3-management
    container_name: rabbitmq
    ports:
      - "5672:5672"
      - "15672:15672"

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

  warehouse:
    build: ./warehouse
    container_name: warehouse
    environment:
      DATABASE_URL: ${WAREHOUSE_DB_URL}
    ports:
      - "8002:8000"
    depends_on:
      warehouse-db:
        condition: service_healthy

  erp:
    build: ./erp
    container_name: erp
    environment:
      DATABASE_URL: ${ERP_DB_URL}
      CRM_URL: ${CRM_URL}
      WAREHOUSE_URL: ${WAREHOUSE_URL}
      REDIS_URL: ${REDIS_URL}
      RABBITMQ_URL: ${RABBITMQ_URL}
    ports:
      - "8000:8000"
    depends_on:
      erp-db:
        condition: service_healthy
      redis:
        condition: service_started
      rabbitmq:
        condition: service_started
      crm:
        condition: service_started
      warehouse:
        condition: service_started

  worker:
    build: ./worker
    container_name: notification-worker
    environment:
      RABBITMQ_URL: ${RABBITMQ_URL}
    depends_on:
      rabbitmq:
        condition: service_started

volumes:
  crm_data:
  warehouse_data:
  erp_data:
```

---

# 11. Dockerfile générique

Le même principe sera utilisé dans les trois services.

Exemple `crm/Dockerfile` :

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY app ./app

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Explication ligne par ligne

- `FROM python:3.12-slim` : utilise une image Python légère.
- `WORKDIR /app` : définit `/app` comme répertoire courant dans le conteneur.
- `COPY requirements.txt .` : copie uniquement les dépendances avant le code afin de profiter du cache Docker.
- `RUN pip install ...` : installe les bibliothèques à la construction de l'image.
- `COPY app ./app` : copie ensuite le code source.
- `--host 0.0.0.0` : indispensable dans Docker ; avec `127.0.0.1`, Uvicorn ne serait accessible que depuis l'intérieur du conteneur.
- `--port 8000` : tous les microservices peuvent utiliser le port `8000` à l'intérieur de leur propre conteneur car ils possèdent chacun leur propre espace réseau.

> **Point important :** le port interne peut être identique pour CRM, ERP et Warehouse. Ce sont les ports publiés sur la machine hôte qui doivent être différents.


Créer la même structure pour :

```text
warehouse/Dockerfile
erp/Dockerfile
```

---

# 12. Dépendances CRM

Créer `crm/requirements.txt` :

```txt
fastapi
uvicorn[standard]
sqlalchemy
psycopg2-binary
pydantic
email-validator
```

---

# 13. Service CRM

Créer `crm/app/main.py` :

```python
# crm/app/main.py

import os
import time
from typing import Generator

from fastapi import FastAPI, HTTPException, Query
from pydantic import BaseModel, EmailStr
from sqlalchemy import Boolean, Column, Integer, String, create_engine
from sqlalchemy.orm import Session, declarative_base, sessionmaker


# -------------------------------------------------------------------
# Configuration
# -------------------------------------------------------------------

# La chaîne de connexion n'est volontairement pas écrite en dur dans le code.
# Elle est injectée par Docker Compose via la variable DATABASE_URL.
#
# Exemple dans Docker :
# postgresql://crm:crm@crm-db:5432/crm
#
# `crm-db` fonctionne comme hostname grâce au DNS interne de Docker.
DATABASE_URL = os.environ["DATABASE_URL"]


# -------------------------------------------------------------------
# Moteur SQLAlchemy
# -------------------------------------------------------------------

engine = create_engine(
    DATABASE_URL,

    # pool_pre_ping=True demande à SQLAlchemy de vérifier qu'une connexion
    # récupérée dans le pool est encore valide avant de la réutiliser.
    # C'est très utile dans des environnements conteneurisés où une base
    # peut être redémarrée.
    pool_pre_ping=True
)


# SessionLocal est une "fabrique" de sessions SQLAlchemy.
# Chaque requête HTTP devrait utiliser sa propre session et la fermer ensuite.
SessionLocal = sessionmaker(
    bind=engine,

    # Nous contrôlons nous-mêmes les transactions.
    autocommit=False,

    # Evite que SQLAlchemy pousse automatiquement des changements
    # vers la base avant certains SELECT.
    autoflush=False
)


# Classe de base utilisée pour déclarer nos modèles ORM.
Base = declarative_base()


class CustomerDB(Base):
    """
    Représentation SQLAlchemy d'un client stocké en base.

    Attention à la différence :
    - CustomerDB = modèle de persistance SQL ;
    - CustomerCreate / CustomerOut = contrats d'API Pydantic.
    """

    __tablename__ = "customers"

    # Clé primaire auto-incrémentée par PostgreSQL.
    id = Column(Integer, primary_key=True)

    # nullable=False signifie que la valeur est obligatoire en base.
    name = Column(String(100), nullable=False)

    # unique=True ajoute une contrainte d'unicité.
    # Deux clients ne doivent pas utiliser exactement le même email.
    email = Column(String(150), nullable=False, unique=True)

    # Permet de désactiver un client sans le supprimer.
    active = Column(Boolean, default=True, nullable=False)


# Pour simplifier le TP, nous créons les tables au démarrage.
# En production, on préférera des migrations avec Alembic.
Base.metadata.create_all(engine)


class CustomerCreate(BaseModel):
    name: str
    email: EmailStr
    active: bool = True


class CustomerOut(CustomerCreate):
    id: int


app = FastAPI(
    title="CRM API",
    version="1.0.0"
)


def get_db() -> Generator:
    db = SessionLocal()

    try:
        yield db
    finally:
        db.close()


@app.get("/health")
def health():
    return {
        "service": "crm",
        "status": "ok"
    }


@app.post("/customers", response_model=CustomerOut, status_code=201)
def create_customer(payload: CustomerCreate):

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


@app.get("/customers/{customer_id}", response_model=CustomerOut)
def get_customer(
    customer_id: int,

    # Paramètre pédagogique permettant d'introduire artificiellement
    # une latence de 0 à 10 secondes.
    #
    # Exemple :
    # GET /customers/1?delay_ms=2500
    delay_ms: int = Query(default=0, ge=0, le=10000),

    # Permet de forcer une erreur HTTP 500.
    # Exemple :
    # GET /customers/1?fail=true
    fail: bool = False
):
    """
    Retourne un client.

    Cet endpoint contient volontairement des options de simulation
    afin d'étudier les effets de la latence et des pannes en cascade.
    """

    if delay_ms:
        # time.sleep bloque le worker pendant la durée demandée.
        # Dans un TP de résilience, cela permet d'observer ce qui se passe
        # lorsqu'une dépendance devient lente.
        time.sleep(delay_ms / 1000)

    if fail:
        # Cette erreur sera vue par l'ERP comme une erreur "upstream".
        raise HTTPException(
            status_code=500,
            detail="CRM simulated failure"
        )

    db = SessionLocal()

    try:
        customer = db.get(CustomerDB, customer_id)

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

# 14. Dépendances Warehouse

Créer `warehouse/requirements.txt` :

```txt
fastapi
uvicorn[standard]
sqlalchemy
psycopg2-binary
pydantic
```

---

# 15. Service Warehouse

Créer `warehouse/app/main.py` :

```python
import os
import time

from fastapi import FastAPI, HTTPException, Query
from pydantic import BaseModel, Field
from sqlalchemy import Column, Integer, String, create_engine, select
from sqlalchemy.orm import declarative_base, sessionmaker


DATABASE_URL = os.environ["DATABASE_URL"]

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

    product_id = Column(String(100), primary_key=True)
    quantity = Column(Integer, nullable=False)


Base.metadata.create_all(engine)


class StockCreate(BaseModel):
    product_id: str
    quantity: int = Field(ge=0)


class ReservationRequest(BaseModel):
    product_id: str
    quantity: int = Field(gt=0)


class StockOut(BaseModel):
    product_id: str
    quantity: int


class ReservationOut(BaseModel):
    product_id: str
    reserved_quantity: int
    remaining_quantity: int


app = FastAPI(
    title="Warehouse API",
    version="1.0.0"
)


@app.get("/health")
def health():
    return {
        "service": "warehouse",
        "status": "ok"
    }


@app.post("/stocks", response_model=StockOut, status_code=201)
def create_stock(payload: StockCreate):

    db = SessionLocal()

    try:
        current = db.get(StockDB, payload.product_id)

        if current:
            current.quantity = payload.quantity
        else:
            current = StockDB(
                product_id=payload.product_id,
                quantity=payload.quantity
            )
            db.add(current)

        db.commit()
        db.refresh(current)

        return StockOut(
            product_id=current.product_id,
            quantity=current.quantity
        )

    finally:
        db.close()


@app.get("/stocks/{product_id}", response_model=StockOut)
def get_stock(product_id: str):

    db = SessionLocal()

    try:
        stock = db.get(StockDB, product_id)

        if stock is None:
            raise HTTPException(
                status_code=404,
                detail="Product not found"
            )

        return StockOut(
            product_id=stock.product_id,
            quantity=stock.quantity
        )

    finally:
        db.close()


@app.post("/reservations", response_model=ReservationOut)
def reserve_stock(
    payload: ReservationRequest,
    delay_ms: int = Query(default=0, ge=0, le=10000),
    fail: bool = False
):
    """
    Réservation avec verrouillage pessimiste SQL.

    Le SELECT ... FOR UPDATE évite qu'une autre transaction
    modifie le même stock au même moment.
    """

    if delay_ms:
        time.sleep(delay_ms / 1000)

    if fail:
        raise HTTPException(
            status_code=500,
            detail="Warehouse simulated failure"
        )

    db = SessionLocal()

    try:
        # Construction d'un SELECT ... FOR UPDATE.
        #
        # Le verrou pessimiste empêche deux transactions concurrentes
        # de réserver simultanément le même stock.
        #
        # Sans ce verrou :
        # - requête A lit quantité=1 ;
        # - requête B lit quantité=1 ;
        # - les deux peuvent croire que l'article est encore disponible.
        stmt = (
            select(StockDB)
            .where(StockDB.product_id == payload.product_id)
            .with_for_update()
        )

        # Exécution du SELECT.
        # Si une autre transaction détient déjà le verrou sur cette ligne,
        # cette requête attendra la libération du verrou.
        stock = db.execute(stmt).scalar_one_or_none()

        if stock is None:
            raise HTTPException(
                status_code=404,
                detail="Product not found"
            )

        if stock.quantity < payload.quantity:
            raise HTTPException(
                status_code=409,
                detail="Insufficient stock"
            )

        # La quantité est décrémentée dans la transaction courante.
        stock.quantity -= payload.quantity

        # COMMIT rend la modification durable et libère le verrou SQL.
        db.commit()

        # Recharge l'objet avec les valeurs réellement persistées.
        db.refresh(stock)

        return ReservationOut(
            product_id=stock.product_id,
            reserved_quantity=payload.quantity,
            remaining_quantity=stock.quantity
        )

    except:
        db.rollback()
        raise

    finally:
        db.close()
```

---

# 15.1. Ce que protège réellement la transaction

Dans le Warehouse, la partie critique est :

```text
LIRE LE STOCK
VERIFIER LE STOCK
DECREMENTER LE STOCK
COMMIT
```

Ces quatre opérations doivent être vues comme une seule unité logique.

Sans transaction cohérente :

```text
stock lu = 1
autre requête modifie stock
première requête continue avec une valeur devenue obsolète
```

`SELECT ... FOR UPDATE` transforme la ligne concernée en ressource protégée jusqu'au `COMMIT` ou au `ROLLBACK`.

---

# 16. Pourquoi utiliser SELECT FOR UPDATE ?

Sans verrouillage, deux requêtes peuvent lire :

```text
stock = 1
```

au même moment.

Les deux pensent pouvoir réserver le dernier article.

Cela produit une **race condition**.

```mermaid
sequenceDiagram
    participant R1 as Requête A
    participant DB as Base
    participant R2 as Requête B

    R1->>DB: Lire stock = 1
    R2->>DB: Lire stock = 1

    R1->>DB: Réserver 1
    R2->>DB: Réserver 1

    Note over DB: Risque de double réservation
```

Avec `FOR UPDATE` :

```mermaid
sequenceDiagram
    participant R1 as Requête A
    participant DB as Base
    participant R2 as Requête B

    R1->>DB: SELECT FOR UPDATE
    DB-->>R1: verrou obtenu

    R2->>DB: SELECT FOR UPDATE
    Note over R2,DB: attente du verrou

    R1->>DB: UPDATE puis COMMIT
    DB-->>R2: verrou disponible
    R2->>DB: stock insuffisant
```

---

# 17. Dépendances ERP

Créer `erp/requirements.txt` :

```txt
fastapi
uvicorn[standard]
sqlalchemy
psycopg2-binary
pydantic
httpx
redis
pika
```

---

# 18. Service ERP

Créer `erp/app/main.py` :

```python
import json
import os
import time
from datetime import datetime, timezone
from enum import Enum
from uuid import UUID, uuid4

import httpx
import pika
import redis
from fastapi import FastAPI, Header, HTTPException, Response, status
from pydantic import BaseModel, Field
from sqlalchemy import Column, DateTime, Integer, String, create_engine
from sqlalchemy.orm import declarative_base, sessionmaker


# -------------------------------------------------------------------
# Configuration injectée par Docker
# -------------------------------------------------------------------
#
# Aucun hostname localhost ici :
# à l'intérieur du réseau Docker, les services se contactent via leur nom.
#
# Exemple :
# CRM_URL=http://crm:8000
# WAREHOUSE_URL=http://warehouse:8000

DATABASE_URL = os.environ["DATABASE_URL"]
CRM_URL = os.environ["CRM_URL"]
WAREHOUSE_URL = os.environ["WAREHOUSE_URL"]
REDIS_URL = os.environ["REDIS_URL"]
RABBITMQ_URL = os.environ["RABBITMQ_URL"]


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


class OrderDB(Base):
    __tablename__ = "orders"

    id = Column(String(36), primary_key=True)
    customer_id = Column(Integer, nullable=False)
    product_id = Column(String(100), nullable=False)
    quantity = Column(Integer, nullable=False)
    status = Column(String(50), nullable=False)
    created_at = Column(DateTime(timezone=True), nullable=False)


Base.metadata.create_all(engine)


# Client Redis partagé.
#
# Redis sera utilisé ici pour :
# - mémoriser les clés d'idempotence ;
# - partager l'état du circuit breaker entre plusieurs instances ERP.
#
# decode_responses=True demande au client de retourner des chaînes Python
# plutôt que des bytes.
redis_client = redis.from_url(
    REDIS_URL,
    decode_responses=True
)


class OrderStatus(str, Enum):
    CREATED = "CREATED"
    CONFIRMED = "CONFIRMED"
    REJECTED = "REJECTED"


class OrderCreate(BaseModel):
    customer_id: int
    product_id: str
    quantity: int = Field(gt=0)


class OrderOut(BaseModel):
    id: UUID
    customer_id: int
    product_id: str
    quantity: int
    status: OrderStatus
    created_at: datetime


app = FastAPI(
    title="ERP API",
    version="1.0.0"
)


# ---------------------------------------------------------
# Circuit breaker pédagogique
# ---------------------------------------------------------

# -------------------------------------------------------------------
# Paramètres du circuit breaker
# -------------------------------------------------------------------

# Compteur partagé du nombre d'échecs récents vers le Warehouse.
FAILURE_KEY = "warehouse_failures"

# Timestamp indiquant jusqu'à quand le circuit reste ouvert.
OPEN_KEY = "warehouse_circuit_open_until"

# Après 3 échecs, nous cessons temporairement d'appeler le Warehouse.
FAILURE_THRESHOLD = 3

# Durée de protection.
CIRCUIT_OPEN_SECONDS = 20


def circuit_is_open() -> bool:
    open_until = redis_client.get(OPEN_KEY)

    if not open_until:
        return False

    return time.time() < float(open_until)


def record_warehouse_success() -> None:
    redis_client.delete(FAILURE_KEY)
    redis_client.delete(OPEN_KEY)


def record_warehouse_failure() -> None:
    failures = redis_client.incr(FAILURE_KEY)

    if failures >= FAILURE_THRESHOLD:
        redis_client.set(
            OPEN_KEY,
            time.time() + CIRCUIT_OPEN_SECONDS,
            ex=CIRCUIT_OPEN_SECONDS
        )


# ---------------------------------------------------------
# RabbitMQ
# ---------------------------------------------------------

def publish_event(event_name: str, payload: dict) -> None:
    """
    Publie un événement métier dans RabbitMQ.

    Exemple d'événement :
        routing key = order.created
        payload     = informations de commande

    Cette fonction est volontairement simple pour le TP.

    Dans une architecture de production, il faudrait aussi étudier :
    - publisher confirms ;
    - Outbox Pattern ;
    - DLQ ;
    - retry queues ;
    - versionnement des événements ;
    - observabilité ;
    - idempotence des consommateurs.
    """

    # Conversion de l'URL AMQP en paramètres Pika.
    params = pika.URLParameters(RABBITMQ_URL)

    # Ouverture d'une connexion TCP/AMQP vers RabbitMQ.
    # Cette API est bloquante.
    connection = pika.BlockingConnection(params)

    try:
        channel = connection.channel()

        # Un exchange "topic" permet de router selon une clé telle que :
        # - order.created
        # - order.failed
        # - customer.updated
        channel.exchange_declare(
            exchange="business.events",
            exchange_type="topic",

            # durable=True signifie que la définition de l'exchange
            # survit à un redémarrage du broker.
            durable=True
        )

        # Publication du message.
        channel.basic_publish(
            exchange="business.events",

            # La routing key détermine quelles queues recevront le message.
            routing_key=event_name,

            # RabbitMQ transporte des bytes :
            # on sérialise donc notre dictionnaire en JSON puis en UTF-8.
            body=json.dumps(payload).encode(),

            properties=pika.BasicProperties(
                content_type="application/json",

                # delivery_mode=2 demande un message persistant.
                # Cela améliore la durabilité mais ne remplace pas
                # les publisher confirms ni l'Outbox Pattern.
                delivery_mode=2
            )
        )

    finally:
        connection.close()


# ---------------------------------------------------------
# Appels HTTP résilients
# ---------------------------------------------------------

async def get_customer(customer_id: int) -> dict:
    """
    Appel CRM avec timeout.
    """

    # Les timeouts sont séparés par phase :
    #
    # connect : temps maximum pour établir la connexion ;
    # read    : temps maximum d'attente de données en lecture ;
    # write   : temps maximum d'écriture de la requête ;
    # pool    : temps maximum pour obtenir une connexion disponible
    #           dans le pool HTTP.
    #
    # Le but est d'éviter qu'une dépendance lente immobilise
    # indéfiniment les ressources de l'ERP.
    timeout = httpx.Timeout(
        connect=1.0,
        read=2.0,
        write=2.0,
        pool=1.0
    )

    async with httpx.AsyncClient(timeout=timeout) as client:

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

    if response.status_code >= 500:
        raise HTTPException(
            status_code=502,
            detail="CRM upstream error"
        )

    response.raise_for_status()

    return response.json()


async def reserve_stock_with_retry(
    product_id: str,
    quantity: int
) -> dict:
    """
    Appel Warehouse avec :
    - timeout ;
    - retry ;
    - backoff exponentiel ;
    - circuit breaker.
    """

    if circuit_is_open():
        raise HTTPException(
            status_code=503,
            detail="Warehouse circuit breaker is open"
        )

    # Les timeouts sont séparés par phase :
    #
    # connect : temps maximum pour établir la connexion ;
    # read    : temps maximum d'attente de données en lecture ;
    # write   : temps maximum d'écriture de la requête ;
    # pool    : temps maximum pour obtenir une connexion disponible
    #           dans le pool HTTP.
    #
    # Le but est d'éviter qu'une dépendance lente immobilise
    # indéfiniment les ressources de l'ERP.
    timeout = httpx.Timeout(
        connect=1.0,
        read=2.0,
        write=2.0,
        pool=1.0
    )

    max_attempts = 3

    # Chaque tentative représente un nouvel appel réseau.
    # Attention : un retry n'est sûr que si l'opération distante
    # est idempotente ou protégée contre les doublons.
    for attempt in range(1, max_attempts + 1):

        try:
            # AsyncClient évite de bloquer l'event loop pendant l'attente réseau.
            async with httpx.AsyncClient(timeout=timeout) as client:

                response = await client.post(
                    f"{WAREHOUSE_URL}/reservations",
                    json={
                        "product_id": product_id,
                        "quantity": quantity
                    }
                )

            if response.status_code == 409:
                record_warehouse_success()

                raise HTTPException(
                    status_code=409,
                    detail="Insufficient stock"
                )

            if response.status_code >= 500:
                raise httpx.HTTPStatusError(
                    "Warehouse 5xx",
                    request=response.request,
                    response=response
                )

            response.raise_for_status()

            record_warehouse_success()

            return response.json()

        except (
            httpx.TimeoutException,
            httpx.RequestError,
            httpx.HTTPStatusError
        ) as exc:

            record_warehouse_failure()

            if attempt == max_attempts:
                raise HTTPException(
                    status_code=503,
                    detail=f"Warehouse unavailable after retries: {type(exc).__name__}"
                )

            # Backoff exponentiel :
            # tentative 1 -> 0.2 s
            # tentative 2 -> 0.4 s
            # tentative 3 -> 0.8 s
            #
            # On évite ainsi de marteler immédiatement un service
            # qui est déjà en difficulté.
            backoff_seconds = 0.2 * (2 ** (attempt - 1))

            # asyncio.sleep ne bloque pas le thread :
            # d'autres requêtes peuvent être traitées pendant l'attente.
            await __import__("asyncio").sleep(backoff_seconds)

    raise RuntimeError("Unreachable")


# ---------------------------------------------------------
# Endpoints
# ---------------------------------------------------------

@app.get("/health")
def health():
    return {
        "service": "erp",
        "status": "ok"
    }


@app.post(
    "/orders",
    response_model=OrderOut,
    status_code=status.HTTP_201_CREATED
)
async def create_order(
    payload: OrderCreate,
    response: Response,
    idempotency_key: str | None = Header(
        default=None,
        alias="Idempotency-Key"
    )
):
    """
    Création d'une commande avec idempotence.
    """

    if not idempotency_key:
        raise HTTPException(
            status_code=400,
            detail="Idempotency-Key header is required"
        )

    # On transforme la clé fournie par le client en clé Redis.
    #
    # Exemple :
    # Idempotency-Key: order-test-001
    # devient :
    # idempotency:order-test-001
    cache_key = f"idempotency:{idempotency_key}"

    # Si Redis contient déjà un résultat, cette opération a déjà été traitée.
    existing_order_id = redis_client.get(cache_key)

    if existing_order_id:
        db = SessionLocal()

        try:
            existing = db.get(OrderDB, existing_order_id)

            if existing is None:
                raise HTTPException(
                    status_code=500,
                    detail="Broken idempotency reference"
                )

            response.headers["X-Idempotent-Replay"] = "true"

            return OrderOut(
                id=UUID(existing.id),
                customer_id=existing.customer_id,
                product_id=existing.product_id,
                quantity=existing.quantity,
                status=OrderStatus(existing.status),
                created_at=existing.created_at
            )

        finally:
            db.close()

    # -----------------------------------------------------
    # 1. Vérification CRM
    # -----------------------------------------------------

    customer = await get_customer(payload.customer_id)

    if not customer["active"]:
        raise HTTPException(
            status_code=409,
            detail="Customer is inactive"
        )

    # -----------------------------------------------------
    # 2. Réservation Warehouse
    # -----------------------------------------------------

    await reserve_stock_with_retry(
        payload.product_id,
        payload.quantity
    )

    # -----------------------------------------------------
    # 3. Création ERP
    # -----------------------------------------------------

    order_id = uuid4()

    created_at = datetime.now(timezone.utc)

    order = OrderDB(
        id=str(order_id),
        customer_id=payload.customer_id,
        product_id=payload.product_id,
        quantity=payload.quantity,
        status=OrderStatus.CONFIRMED.value,
        created_at=created_at
    )

    db = SessionLocal()

    try:
        db.add(order)
        db.commit()
    finally:
        db.close()

    # -----------------------------------------------------
    # 4. Sauvegarde idempotence
    # -----------------------------------------------------

    # Mémorise le résultat pendant 1 heure.
    #
    # Une répétition de la même requête avec la même clé ne créera donc
    # pas une deuxième commande ERP.
    #
    # Attention : dans cette version pédagogique, la réservation Warehouse
    # a déjà eu lieu avant l'enregistrement de cette clé.
    # C'est précisément pour cela qu'une vraie idempotence distribuée
    # doit également exister côté Warehouse.
    redis_client.set(
        cache_key,
        str(order_id),
        ex=3600
    )

    # -----------------------------------------------------
    # 5. Publication d'un événement
    # -----------------------------------------------------

    try:
        publish_event(
            "order.created",
            {
                "order_id": str(order_id),
                "customer_id": payload.customer_id,
                "product_id": payload.product_id,
                "quantity": payload.quantity,
                "status": OrderStatus.CONFIRMED.value,
                "created_at": created_at.isoformat()
            }
        )

    except Exception as exc:
        print(
            "WARNING: event publication failed:",
            repr(exc)
        )

    response.headers["Location"] = f"/orders/{order_id}"

    return OrderOut(
        id=order_id,
        customer_id=payload.customer_id,
        product_id=payload.product_id,
        quantity=payload.quantity,
        status=OrderStatus.CONFIRMED,
        created_at=created_at
    )


@app.get("/orders/{order_id}", response_model=OrderOut)
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
            status=OrderStatus(order.status),
            created_at=order.created_at
        )

    finally:
        db.close()
```

---


---

# 18.1. Limites volontaires de cette implémentation

Le TP privilégie la compréhension. Plusieurs éléments sont donc simplifiés.

### Création des tables

Nous utilisons :

```python
Base.metadata.create_all(engine)
```

En production, on préférera généralement **Alembic** afin de versionner les migrations SQL.

### Connexions RabbitMQ

La fonction pédagogique ouvre une connexion lors de la publication.

En production, on cherchera plutôt à :

- réutiliser les connexions ;
- confirmer les publications ;
- surveiller les erreurs de broker.

### Idempotence

La clé ERP seule ne suffit pas à garantir l'idempotence de bout en bout.

Le Warehouse doit lui aussi reconnaître un identifiant de réservation unique.

### Publication d'événement

La commande peut être validée dans PostgreSQL alors que RabbitMQ est indisponible.

C'est pourquoi le TP introduit ensuite l'**Outbox Pattern**.


# 19. Attention au retry

Un retry peut être utile.

Mais il peut aussi être dangereux.

Exemple :

```text
ERP -> Warehouse
```

Si la requête est effectivement exécutée côté Warehouse mais que la réponse est perdue :

```text
Warehouse réserve le stock
réponse perdue
ERP pense que l'appel a échoué
ERP retry
Warehouse réserve une deuxième fois
```

Cela peut provoquer un double traitement.

---

# 20. Solution réelle : idempotence côté Warehouse

Créer une clé de réservation unique.

Modifier le modèle :

```python
class ReservationRequest(BaseModel):
    request_id: str
    product_id: str
    quantity: int = Field(gt=0)
```

Le Warehouse doit enregistrer chaque `request_id`.

Ainsi, si la même requête revient :

```text
request_id déjà connu
```

il retourne le résultat précédent sans réserver une seconde fois.

---

# 21. Pattern d'idempotence distribué

```mermaid
sequenceDiagram
    participant ERP
    participant Warehouse
    participant DB

    ERP->>Warehouse: POST reservation request_id=ABC
    Warehouse->>DB: Chercher ABC

    alt ABC inconnu
        Warehouse->>DB: Réserver stock
        Warehouse->>DB: Sauvegarder résultat ABC
        Warehouse-->>ERP: 200
    else ABC déjà traité
        Warehouse->>DB: Lire résultat précédent
        Warehouse-->>ERP: 200 replay
    end
```

---

# 22. Worker RabbitMQ

Créer `worker/requirements.txt` :

```txt
pika
```

Créer `worker/Dockerfile` :

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY worker.py .

CMD ["python", "worker.py"]
```

Créer `worker/worker.py` :

```python
import json
import os
import time

import pika


RABBITMQ_URL = os.environ["RABBITMQ_URL"]


def main():

    while True:

        try:
            params = pika.URLParameters(
                RABBITMQ_URL
            )

            connection = pika.BlockingConnection(
                params
            )

            channel = connection.channel()

            channel.exchange_declare(
                exchange="business.events",
                exchange_type="topic",
                durable=True
            )

            channel.queue_declare(
                queue="notifications",
                durable=True
            )

            channel.queue_bind(
                queue="notifications",
                exchange="business.events",
                routing_key="order.*"
            )

            def callback(
                ch,
                method,
                properties,
                body
            ):
                # `body` arrive sous forme de bytes.
                # Nous décodons ici le JSON en dictionnaire Python.
                data = json.loads(body)

                print(
                    "EVENT RECEIVED:",
                    data
                )

                # Simulation d'un traitement externe :
                # envoi d'email, appel SMS, génération de facture, etc.
                time.sleep(0.5)

                print(
                    "Notification processed for order:",
                    data.get("order_id")
                )

                # ACK manuel :
                # RabbitMQ peut considérer le message comme traité.
                #
                # Si le worker tombe AVANT cet ACK, le message peut être
                # redistribué à un autre consommateur.
                # D'où l'importance d'un consommateur idempotent.
                ch.basic_ack(
                    delivery_tag=method.delivery_tag
                )

            # prefetch_count=10 limite le nombre de messages non acquittés
            # envoyés simultanément à ce worker.
            #
            # Cela constitue une forme de backpressure :
            # le broker ne surcharge pas indéfiniment un consommateur lent.
            channel.basic_qos(
                prefetch_count=10
            )

            channel.basic_consume(
                queue="notifications",
                on_message_callback=callback
            )

            print(
                "Worker waiting for messages..."
            )

            channel.start_consuming()

        except Exception as exc:

            print(
                "Worker connection error:",
                repr(exc)
            )

            time.sleep(3)


if __name__ == "__main__":
    main()
```

---

# 23. Pourquoi utiliser RabbitMQ ?

Sans RabbitMQ :

```text
ERP -> service email -> attente
```

Le temps de réponse ERP dépend du service email.

Avec RabbitMQ :

```text
ERP -> RabbitMQ -> réponse client
                |
                +-> worker
```

L'ERP ne doit plus attendre le traitement complet.

---

# 24. Démarrer toute l'architecture

## Linux / macOS

```bash
docker compose up --build
```

## Windows PowerShell

```powershell
docker compose up --build
```

Mode détaché :

```bash
docker compose up -d --build
```

---

# 25. Vérifier les conteneurs

```bash
docker compose ps
```

Résultat attendu :

```text
crm
crm-db
warehouse
warehouse-db
erp
erp-db
redis
rabbitmq
notification-worker
```

---

# 26. Vérifier les logs

```bash
docker compose logs
```

Logs ERP :

```bash
docker compose logs erp
```

Logs en continu :

```bash
docker compose logs -f erp
```

---

# 27. Interfaces disponibles

ERP :

```text
http://localhost:8000/docs
```

CRM :

```text
http://localhost:8001/docs
```

Warehouse :

```text
http://localhost:8002/docs
```

RabbitMQ Management :

```text
http://localhost:15672
```

Identifiants pédagogiques :

```text
guest
guest
```

---

# 28. Initialiser le CRM

Créer un client :

## Linux / macOS

```bash
curl -i \
  -X POST \
  http://localhost:8001/customers \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Alice Martin",
    "email": "alice@example.com",
    "active": true
  }'
```

## Windows PowerShell

```powershell
$body = @{
    name = "Alice Martin"
    email = "alice@example.com"
    active = $true
} | ConvertTo-Json

Invoke-RestMethod `
    -Method Post `
    -Uri "http://localhost:8001/customers" `
    -ContentType "application/json" `
    -Body $body
```

---

# 29. Initialiser le stock

```bash
curl -i \
  -X POST \
  http://localhost:8002/stocks \
  -H "Content-Type: application/json" \
  -d '{
    "product_id": "LAPTOP-001",
    "quantity": 10
  }'
```

---

# 30. Créer une commande

```bash
curl -i \
  -X POST \
  http://localhost:8000/orders \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: order-test-001" \
  -d '{
    "customer_id": 1,
    "product_id": "LAPTOP-001",
    "quantity": 2
  }'
```

Réponse attendue :

```text
HTTP/1.1 201 Created
```

Avec :

```json
{
  "id": "...",
  "customer_id": 1,
  "product_id": "LAPTOP-001",
  "quantity": 2,
  "status": "CONFIRMED",
  "created_at": "..."
}
```

---

# 31. Vérifier le stock

```bash
curl \
  http://localhost:8002/stocks/LAPTOP-001
```

Résultat :

```json
{
  "product_id": "LAPTOP-001",
  "quantity": 8
}
```

---

# 32. Tester l'idempotence

Renvoyer exactement la même commande :

```bash
curl -i \
  -X POST \
  http://localhost:8000/orders \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: order-test-001" \
  -d '{
    "customer_id": 1,
    "product_id": "LAPTOP-001",
    "quantity": 2
  }'
```

Le header devrait contenir :

```text
X-Idempotent-Replay: true
```

Le stock doit rester :

```text
8
```

et non :

```text
6
```

---

# 33. Mesurer la latence

## curl Linux / macOS

```bash
curl -s \
  -o /dev/null \
  -w "status=%{http_code} total=%{time_total}s\n" \
  http://localhost:8001/customers/1
```

---

# 34. Simuler 2 secondes de latence CRM

```bash
curl -s \
  -o /dev/null \
  -w "status=%{http_code} total=%{time_total}s\n" \
  "http://localhost:8001/customers/1?delay_ms=2000"
```

---

# 35. Notions de latence

La latence totale peut être approximée par :

```text
T_total =
T_client_erp
+ T_erp_crm
+ T_crm
+ T_erp_warehouse
+ T_warehouse
+ T_database
+ T_serialization
```

Si les appels CRM et Warehouse sont séquentiels :

```text
T_total ≈ T_CRM + T_Warehouse + T_ERP
```

---

# 36. Appels parallèles

Lorsque deux appels sont indépendants, il peut être intéressant de les paralléliser.

Exemple :

```python
import asyncio


customer_result, pricing_result = await asyncio.gather(
    get_customer(customer_id),
    get_pricing(product_id)
)
```

Si :

```text
CRM = 500 ms
Pricing = 700 ms
```

En séquentiel :

```text
≈ 1200 ms
```

En parallèle :

```text
≈ 700 ms
```

plus un léger overhead.

---

# 37. Attention à la concurrence

Paralléliser toutes les requêtes n'est pas toujours bénéfique.

Trop de concurrence peut provoquer :

- saturation CPU ;
- trop de connexions PostgreSQL ;
- saturation du pool HTTP ;
- surcharge réseau ;
- amplification d'erreurs ;
- augmentation des files d'attente ;
- hausse de la latence globale.

---

# 38. Little's Law

Dans un système stable :

```text
L = λ × W
```

avec :

- `L` = nombre moyen de requêtes simultanément dans le système ;
- `λ` = débit moyen ;
- `W` = temps moyen passé dans le système.

Exemple :

```text
100 requêtes / seconde
latence moyenne = 0,2 seconde
```

Donc :

```text
L = 100 × 0,2 = 20
```

Le système doit supporter environ :

```text
20 requêtes concurrentes
```

en moyenne.

---

# 39. Test de concurrence Warehouse

Supposons :

```text
stock = 1
```

Envoyer 10 réservations simultanées.

Créer `tests/concurrency_test.py` :

```python
import asyncio

import httpx


URL = "http://localhost:8002/reservations"


async def reserve(index: int):

    async with httpx.AsyncClient(
        timeout=5.0
    ) as client:

        response = await client.post(
            URL,
            json={
                "product_id": "LAST-ITEM",
                "quantity": 1
            }
        )

        return {
            "request": index,
            "status": response.status_code,
            "body": response.text
        }


async def main():

    # On prépare 10 coroutines qui vont essayer de réserver
    # exactement le même article.
    tasks = [
        reserve(i)
        for i in range(10)
    ]

    # asyncio.gather lance les opérations de manière concurrente.
    # L'objectif est de provoquer volontairement une compétition
    # sur la même ligne SQL.
    results = await asyncio.gather(
        *tasks
    )

    # Avec SELECT FOR UPDATE et un stock initial de 1 :
    # - 1 requête doit réussir ;
    # - 9 requêtes doivent recevoir HTTP 409.
    for result in results:
        print(result)


asyncio.run(main())
```

---

# 40. Initialiser un seul article

```bash
curl \
  -X POST \
  http://localhost:8002/stocks \
  -H "Content-Type: application/json" \
  -d '{
    "product_id": "LAST-ITEM",
    "quantity": 1
  }'
```

Puis :

```bash
python tests/concurrency_test.py
```

Résultat attendu :

```text
1 réservation réussie
9 réponses 409
```

---

# 41. Test sans verrouillage

Pour l'exercice :

1. retirez `.with_for_update()` ;
2. ajoutez une latence avant l'UPDATE ;
3. recommencez le test concurrent.

Observer les anomalies possibles.

---

# 42. Timeout ERP vers Warehouse

Actuellement :

```python
read=2.0
```

Pour simuler un Warehouse lent :

```text
POST /reservations?delay_ms=5000
```

Le client ERP expirera avant les 5 secondes.

---

# 43. Pourquoi ne pas mettre un timeout de 60 secondes ?

Parce qu'une dépendance lente peut monopoliser :

- threads ;
- workers ;
- sockets ;
- connexions ;
- mémoire.

Sous forte charge, cela peut provoquer une **cascade de saturation**.

---

# 44. Timeout budget

Dans une architecture professionnelle, on peut définir :

```text
budget total client = 2 secondes
```

Puis répartir :

```text
ERP interne = 200 ms
CRM = 500 ms
Warehouse = 700 ms
marge = 600 ms
```

Le but est d'éviter :

```text
chaque service possède un timeout supérieur au timeout du service appelant
```

---

# 45. Retry avec backoff exponentiel

Notre ERP utilise :

```text
0,2 s
0,4 s
0,8 s
```

Le principe :

```text
backoff = base × 2^attempt
```

---

# 46. Ajouter du jitter

Pour éviter que 100 clients ne réessaient exactement au même moment :

```python
import random


backoff = (
    0.2 * (2 ** attempt)
    + random.uniform(0, 0.2)
)
```

C'est le **jitter**.

---

# 47. Retry storm

Scénario :

```text
1000 requêtes arrivent
Warehouse tombe
chaque ERP retry 3 fois
```

Résultat potentiel :

```text
3000 appels supplémentaires
```

Le retry peut empirer une panne.

C'est un **retry storm**.

---

# 48. Circuit Breaker

Le circuit breaker possède généralement trois états :

```text
CLOSED
OPEN
HALF_OPEN
```

```mermaid
stateDiagram-v2
    [*] --> CLOSED

    CLOSED --> OPEN: trop d'échecs
    OPEN --> HALF_OPEN: délai écoulé
    HALF_OPEN --> CLOSED: test réussi
    HALF_OPEN --> OPEN: nouvel échec
```

Notre implémentation pédagogique gère surtout :

```text
CLOSED
OPEN
```

Un exercice permettra d'ajouter `HALF_OPEN`.

---

# 49. Pourquoi Redis pour le circuit breaker ?

Sans stockage partagé :

```text
ERP instance 1 connaît les erreurs
ERP instance 2 ne les connaît pas
ERP instance 3 ne les connaît pas
```

Avec Redis :

```text
toutes les instances partagent l'état
```

---

# 50. Rate limiting

Un endpoint peut être limité par exemple à :

```text
100 requêtes/minute/client
```

Une implémentation simple Redis peut utiliser :

```text
INCR rate:user:123
EXPIRE rate:user:123 60
```

Pseudo-code :

```python
key = f"rate:{client_id}"

count = redis_client.incr(key)

if count == 1:
    redis_client.expire(key, 60)

if count > 100:
    raise HTTPException(
        status_code=429,
        detail="Too many requests"
    )
```

---

# 51. Bulkhead pattern

Le pattern Bulkhead consiste à isoler les ressources.

Exemple :

```text
pool CRM = 20 connexions
pool Warehouse = 20 connexions
```

Ainsi, un Warehouse lent ne consomme pas toutes les ressources disponibles.

---

# 52. Synchronisation vs asynchronisme

## Synchrone

```text
ERP -> Warehouse
ERP attend la réponse
```

Avantage :

- résultat immédiat.

Inconvénient :

- couplage temporel.

---

## Asynchrone

```text
ERP -> RabbitMQ
Warehouse Worker traite plus tard
```

Avantage :

- découplage ;
- absorption des pics ;
- meilleure résilience.

Inconvénient :

- cohérence éventuelle ;
- logique métier plus complexe.

---

# 53. Cohérence éventuelle

Exemple :

```text
ERP : order = CREATED
Warehouse : reservation pas encore exécutée
```

Puis quelques secondes plus tard :

```text
ERP : order = CONFIRMED
```

Pendant cet intervalle, les systèmes n'ont pas exactement le même état.

C'est de la **cohérence éventuelle**.

---

# 54. Saga pattern

Dans un système distribué, il n'existe pas toujours une transaction SQL unique couvrant :

```text
ERP DB
CRM DB
Warehouse DB
```

On peut utiliser une saga.

Exemple :

```text
1. créer commande ERP
2. réserver stock
3. facturer
```

Si la facturation échoue :

```text
annuler réservation stock
annuler commande
```

---

# 55. Saga orchestrée

```mermaid
sequenceDiagram
    participant ERP
    participant Warehouse
    participant Billing

    ERP->>ERP: Create order
    ERP->>Warehouse: Reserve stock
    Warehouse-->>ERP: OK

    ERP->>Billing: Charge customer

    alt Payment OK
        Billing-->>ERP: OK
        ERP->>ERP: Confirm order
    else Payment Failed
        Billing-->>ERP: FAILED
        ERP->>Warehouse: Cancel reservation
        ERP->>ERP: Cancel order
    end
```

---

# 56. Outbox Pattern

Problème :

```text
ERP commit commande
RabbitMQ publication échoue
```

La commande existe mais aucun événement n'a été envoyé.

Solution classique :

```text
transaction SQL :
  INSERT order
  INSERT outbox_event
```

Puis un worker lit la table outbox et publie dans RabbitMQ.

---

# 57. Outbox simplifié

Table :

```sql
CREATE TABLE outbox_events (
    id UUID PRIMARY KEY,
    event_type VARCHAR(100) NOT NULL,
    payload JSONB NOT NULL,
    published BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

Flux :

```mermaid
flowchart LR
    ERP[ERP Transaction]
    DB[(ERP DB)]
    O[Outbox Worker]
    MQ[RabbitMQ]

    ERP --> DB
    DB --> O
    O --> MQ
```

---

# 58. Poison Message

Un message provoque toujours une erreur.

Exemple :

```text
payload invalide
```

Si le worker retry indéfiniment :

```text
boucle infinie
```

Solution :

```text
Dead Letter Queue
```

ou :

```text
DLQ
```

---

# 59. Architecture avec DLQ

```mermaid
flowchart LR
    MQ[Queue principale]
    W[Worker]
    DLQ[Dead Letter Queue]

    MQ --> W
    W -->|success| ACK[ACK]
    W -->|permanent failure| DLQ
```

---

# 60. Observabilité

Un système distribué nécessite au minimum :

- logs ;
- métriques ;
- traces ;
- identifiants de corrélation.

---

# 61. Correlation ID

Le client envoie :

```http
X-Correlation-ID: 31b9-...
```

L'ERP doit propager cette valeur au CRM et au Warehouse.

Ainsi :

```text
Client log
ERP log
CRM log
Warehouse log
```

peuvent être associés au même échange.

---

# 62. Middleware Correlation ID

Exemple :

```python
from uuid import uuid4

from fastapi import Request


@app.middleware("http")
async def correlation_middleware(
    request: Request,
    call_next
):
    correlation_id = request.headers.get(
        "X-Correlation-ID",
        str(uuid4())
    )

    request.state.correlation_id = correlation_id

    response = await call_next(request)

    response.headers[
        "X-Correlation-ID"
    ] = correlation_id

    return response
```

---

# 63. Propagation du header

```python
headers = {
    "X-Correlation-ID": correlation_id
}

response = await client.get(
    url,
    headers=headers
)
```

---

# 64. Logs structurés

Au lieu de :

```text
Warehouse error
```

préférer :

```json
{
  "level": "ERROR",
  "service": "erp",
  "event": "warehouse_request_failed",
  "correlation_id": "abc-123",
  "attempt": 2,
  "duration_ms": 2012
}
```

---

# 65. Mesures importantes

Pour chaque endpoint :

```text
request_count
error_count
latency_ms
p50
p95
p99
```

---

# 66. Percentiles

Supposons :

```text
p50 = 80 ms
p95 = 600 ms
p99 = 2500 ms
```

Même si la moyenne est faible, 1 % des utilisateurs attendent plus de 2,5 secondes.

C'est pourquoi :

```text
moyenne seule != performance réelle
```

---

# 67. Test de charge avec Locust

Créer `load-tests/locustfile.py` :

```python
import uuid

from locust import HttpUser, between, task


class ERPUser(HttpUser):
    """
    Représente un utilisateur virtuel Locust.

    Chaque instance exécute régulièrement la méthode décorée avec @task.
    """

    # Pause aléatoire entre deux actions d'un utilisateur virtuel.
    # Cela évite un trafic totalement artificiel "sans respiration".
    wait_time = between(
        0.1,
        0.5
    )

    @task
    def create_order(self):

        # Chaque commande possède une nouvelle clé d'idempotence.
        # On mesure ici la création normale de commandes, pas le replay.
        idempotency_key = str(
            uuid.uuid4()
        )

        self.client.post(
            "/orders",
            headers={
                "Idempotency-Key": idempotency_key
            },
            json={
                "customer_id": 1,
                "product_id": "LAPTOP-001",
                "quantity": 1
            },
            name="/orders"
        )
```

---

# 68. Installer Locust

```bash
pip install locust
```

---

# 69. Lancer Locust

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

# 70. Scénario de test de charge

Commencer par :

```text
10 utilisateurs
spawn rate = 2/s
```

Puis :

```text
50 utilisateurs
spawn rate = 10/s
```

Puis :

```text
200 utilisateurs
spawn rate = 20/s
```

Observer :

- nombre de requêtes/s ;
- latence moyenne ;
- p95 ;
- p99 ;
- erreurs ;
- saturation.

---

# 71. Test headless

```bash
locust \
  -f load-tests/locustfile.py \
  --host http://localhost:8000 \
  --headless \
  -u 50 \
  -r 10 \
  -t 60s
```

---

# 72. Scénario de panne

Arrêter Warehouse :

```bash
docker compose stop warehouse
```

Puis envoyer une commande.

Observer :

```text
retry
circuit breaker
503
```

---

# 73. Redémarrer Warehouse

```bash
docker compose start warehouse
```

---

# 74. Simuler panne RabbitMQ

```bash
docker compose stop rabbitmq
```

Créer une commande.

Dans notre implémentation actuelle :

```text
la commande est créée
mais l'événement peut être perdu
```

C'est volontaire.

Cette observation justifie l'étude du :

```text
Outbox Pattern
```

---

# 75. Scénario CRM indisponible

```bash
docker compose stop crm
```

Puis :

```bash
curl -i \
  -X POST \
  http://localhost:8000/orders \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: crm-down-test" \
  -d '{
    "customer_id": 1,
    "product_id": "LAPTOP-001",
    "quantity": 1
  }'
```

Résultat attendu :

```text
503 CRM unavailable
```

---

# 76. Docker Stats

Observer les ressources :

```bash
docker stats
```

À analyser :

- CPU ;
- mémoire ;
- réseau ;
- Block I/O.

---

# 77. Inspecter le réseau Docker

```bash
docker network ls
```

Puis :

```bash
docker network inspect tp2-distributed_default
```

---

# 78. Résolution DNS Docker

Dans Docker Compose :

```text
crm
warehouse
redis
rabbitmq
```

sont des noms DNS internes.

Depuis ERP :

```text
http://crm:8000
http://warehouse:8000
```

fonctionnent.

Depuis l'hôte :

```text
http://localhost:8001
http://localhost:8002
```

---

# 79. Tester depuis un conteneur

```bash
docker compose exec erp python -c \
"import socket; print(socket.gethostbyname('crm'))"
```

---

# 80. Inspecter Redis

```bash
docker compose exec redis redis-cli
```

Puis :

```text
KEYS *
```

Voir l'idempotence :

```text
GET idempotency:order-test-001
```

---

# 81. Inspecter PostgreSQL ERP

```bash
docker compose exec erp-db \
  psql -U erp -d erp
```

Puis :

```sql
SELECT * FROM orders;
```

---

# 82. Inspecter le stock

```bash
docker compose exec warehouse-db \
  psql -U warehouse -d warehouse
```

Puis :

```sql
SELECT * FROM stocks;
```

---

# 83. Tester une latence volontaire

Modifier temporairement l'appel ERP vers Warehouse :

```python
f"{WAREHOUSE_URL}/reservations?delay_ms=1500"
```

Relancer :

```bash
docker compose up -d --build erp
```

Mesurer le temps total.

---

# 84. Cascade de latence

Supposons :

```text
CRM = 800 ms
Warehouse = 1200 ms
ERP traitement = 100 ms
```

En séquentiel :

```text
≈ 2100 ms
```

Si chaque service appelle encore plusieurs dépendances :

```text
la latence s'accumule
```

---

# 85. Fan-out

Exemple :

```text
ERP appelle :
CRM
Warehouse
Pricing
Fraud
Delivery
```

Si un seul service est lent, tout le temps de réponse peut être affecté.

---

# 86. Tail latency

Plus le nombre de dépendances augmente, plus il devient probable qu'au moins une soit lente.

Exemple :

```text
5 appels réseau
```

Même si chaque service est rapide 95 % du temps, la probabilité que tous soient rapides est plus faible.

---

# 87. Connection pooling

Il est inefficace de créer une nouvelle connexion réseau à chaque requête.

HTTPX permet de réutiliser les connexions.

Dans une application réelle, on peut créer un client partagé.

Exemple conceptuel :

```python
client = httpx.AsyncClient(
    limits=httpx.Limits(
        max_connections=100,
        max_keepalive_connections=20
    )
)
```

---

# 88. Pourquoi limiter le pool ?

Sans limite :

```text
forte charge
-> trop de connexions
-> saturation service distant
```

Avec limite :

```text
backpressure
```

---

# 89. Backpressure

La backpressure consiste à éviter d'accepter davantage de travail que ce que le système peut traiter.

Techniques :

- queue limitée ;
- rate limiting ;
- semaphore ;
- pool limité ;
- HTTP 429 ;
- broker ;
- autoscaling.

---

# 90. Semaphore asyncio

Exemple :

```python
import asyncio


warehouse_semaphore = asyncio.Semaphore(20)


async def guarded_call():
    async with warehouse_semaphore:
        return await call_warehouse()
```

Ainsi :

```text
maximum 20 appels Warehouse simultanés
```

---

# 91. Test de saturation

Dans ERP, limiter volontairement :

```text
Warehouse concurrence = 5
```

Puis lancer :

```text
100 utilisateurs Locust
```

Observer la file d'attente et la hausse de latence.

---

# 92. Défaillances partielles

Dans un monolithe :

```text
application fonctionne
ou
application tombe
```

Dans un système distribué :

```text
ERP fonctionne
CRM fonctionne
Warehouse lent
RabbitMQ indisponible
Redis fonctionne
```

Le système peut être **partiellement défaillant**.

---

# 93. Health check vs readiness

## Liveness

Question :

```text
Le processus est-il vivant ?
```

## Readiness

Question :

```text
Le service est-il réellement prêt à recevoir du trafic ?
```

Une readiness peut vérifier :

- DB ;
- Redis ;
- configuration minimale.

---

# 94. Endpoint readiness

Exemple :

```python
@app.get("/ready")
def ready():

    checks = {
        "database": True,
        "redis": True
    }

    # Ajouter ici les vraies vérifications.

    if not all(checks.values()):
        raise HTTPException(
            status_code=503,
            detail=checks
        )

    return {
        "status": "ready",
        "checks": checks
    }
```

---

# 95. Exercice 1 - Ajouter une API Pricing

Créer un quatrième service :

```text
pricing
```

Endpoint :

```text
GET /prices/{product_id}
```

Réponse :

```json
{
  "product_id": "LAPTOP-001",
  "unit_price": 1200.0,
  "currency": "EUR"
}
```

L'ERP doit appeler :

```text
CRM
Pricing
Warehouse
```

---

# 96. Solution partielle Pricing

```python
from fastapi import FastAPI, HTTPException


app = FastAPI()


PRICES = {
    "LAPTOP-001": 1200.0,
    "PHONE-001": 700.0
}


@app.get("/prices/{product_id}")
def get_price(product_id: str):

    if product_id not in PRICES:
        raise HTTPException(
            status_code=404,
            detail="Unknown product"
        )

    return {
        "product_id": product_id,
        "unit_price": PRICES[product_id],
        "currency": "EUR"
    }
```

---

# 97. Exercice 2 - Paralléliser CRM et Pricing

CRM et Pricing sont indépendants.

Les lancer en parallèle :

```python
customer, price = await asyncio.gather(
    get_customer(payload.customer_id),
    get_price(payload.product_id)
)
```

Comparer la latence avant et après.

---

# 98. Exercice 3 - Ajouter un semaphore Warehouse

Limiter à :

```text
10 appels Warehouse simultanés
```

Solution :

```python
warehouse_semaphore = asyncio.Semaphore(10)


async def guarded_reserve(
    product_id: str,
    quantity: int
):
    async with warehouse_semaphore:
        return await reserve_stock_with_retry(
            product_id,
            quantity
        )
```

---

# 99. Exercice 4 - Ajouter un vrai HALF_OPEN

Lorsque le circuit est ouvert :

1. attendre 20 secondes ;
2. autoriser une requête test ;
3. si succès :
   - repasser CLOSED ;
4. si échec :
   - repasser OPEN.

---

# 100. Exercice 5 - Implémenter Idempotency-Key Warehouse

Ajouter une table :

```sql
reservation_requests
```

avec :

```text
request_id
product_id
quantity
result
```

La même clé doit toujours produire le même résultat.

---

# 101. Exercice 6 - Ajouter une Dead Letter Queue

Créer :

```text
notifications.dlq
```

Après 3 erreurs permanentes :

```text
message -> DLQ
```

---

# 102. Exercice 7 - Ajouter l'Outbox Pattern

Modifier ERP :

```text
commande + événement
```

doivent être écrits dans la même transaction PostgreSQL.

Un worker doit publier les événements non publiés.

---

# 103. Exercice 8 - Ajouter une métrique de latence

Pour chaque requête ERP :

```text
duration_ms
```

Logger :

```json
{
  "method": "POST",
  "path": "/orders",
  "duration_ms": 418
}
```

---

# 104. Exercice 9 - Ajouter un endpoint de métriques

Créer :

```text
GET /metrics-simple
```

Réponse :

```json
{
  "requests": 1200,
  "errors": 14,
  "warehouse_failures": 3
}
```

---

# 105. Exercice 10 - Ajouter du cache CRM

Les données client changent peu.

L'ERP peut mettre en cache :

```text
customer:{id}
```

durant :

```text
30 secondes
```

Pseudo-code :

```python
cached = redis_client.get(
    f"customer:{customer_id}"
)

if cached:
    return json.loads(cached)
```

---

# 106. Attention au cache

Le cache introduit des problèmes :

- données obsolètes ;
- invalidation ;
- stampede ;
- incohérences.

---

# 107. Cache Stampede

Une clé expire.

1000 requêtes arrivent.

Toutes appellent le CRM au même moment.

C'est un :

```text
cache stampede
```

Solutions :

- lock distribué ;
- TTL aléatoire ;
- stale-while-revalidate ;
- single-flight.

---

# 108. Exercice 11 - Dégradation contrôlée

Si CRM est temporairement indisponible mais qu'une copie récente est en cache :

```text
utiliser la valeur stale
```

avec un header :

```text
X-Data-Stale: true
```

---

# 109. Exercice 12 - Rejouer un événement

Envoyer volontairement deux fois le même événement RabbitMQ.

Le worker doit être idempotent.

Exemple :

```text
processed_event:{event_id}
```

dans Redis.

---

# 110. Exactly-once ?

Dans les systèmes distribués, obtenir un vrai :

```text
exactly once
```

de bout en bout est difficile.

On privilégie souvent :

```text
at-least-once delivery
+
idempotent consumer
```

---

# 111. At-most-once

Un message peut être perdu mais n'est pas traité deux fois.

---

# 112. At-least-once

Un message n'est normalement pas perdu mais peut être livré plusieurs fois.

Il faut donc :

```text
consumer idempotent
```

---

# 113. Tests d'intégration

Créer un scénario complet :

```text
1. créer client
2. créer stock
3. créer commande
4. vérifier commande ERP
5. vérifier stock Warehouse
6. rejouer commande avec même Idempotency-Key
7. vérifier que le stock n'a pas changé
```

---

# 114. Exemple de test Python

```python
import uuid

import httpx


def test_complete_order_flow():

    customer = httpx.post(
        "http://localhost:8001/customers",
        json={
            "name": "Test Integration",
            "email": f"{uuid.uuid4()}@example.com",
            "active": True
        }
    )

    assert customer.status_code == 201

    customer_id = customer.json()["id"]

    product_id = f"PROD-{uuid.uuid4()}"

    stock = httpx.post(
        "http://localhost:8002/stocks",
        json={
            "product_id": product_id,
            "quantity": 5
        }
    )

    assert stock.status_code == 201

    key = str(uuid.uuid4())

    first_order = httpx.post(
        "http://localhost:8000/orders",
        headers={
            "Idempotency-Key": key
        },
        json={
            "customer_id": customer_id,
            "product_id": product_id,
            "quantity": 2
        }
    )

    assert first_order.status_code == 201

    second_order = httpx.post(
        "http://localhost:8000/orders",
        headers={
            "Idempotency-Key": key
        },
        json={
            "customer_id": customer_id,
            "product_id": product_id,
            "quantity": 2
        }
    )

    assert second_order.status_code == 201

    assert (
        first_order.json()["id"]
        ==
        second_order.json()["id"]
    )

    remaining_stock = httpx.get(
        f"http://localhost:8002/stocks/{product_id}"
    )

    assert remaining_stock.json()["quantity"] == 3
```

---

# 115. Stopper l'environnement

```bash
docker compose down
```

Conserver les données :

```bash
docker compose down
```

Supprimer également les volumes :

```bash
docker compose down -v
```

---

# 116. Rebuild d'un seul service

```bash
docker compose up -d --build erp
```

---

# 117. Entrer dans un conteneur

```bash
docker compose exec erp sh
```

---

# 118. Voir les variables d'environnement

```bash
docker compose exec erp env
```

---

# 119. Erreurs courantes

## Erreur : connection refused

Vérifier :

```bash
docker compose ps
```

Puis :

```bash
docker compose logs warehouse
```

---

## Erreur : service DNS introuvable

Vérifier que le service se trouve dans le même réseau Compose.

---

## Erreur PostgreSQL not ready

Vérifier le healthcheck.

---

## Erreur RabbitMQ connection refused

RabbitMQ peut prendre quelques secondes à démarrer.

Le worker possède volontairement une boucle de reconnexion.

---

# 119.1. Tableau des expériences à réaliser

| Expérience | Manipulation | Résultat attendu | Concept |
|---|---|---|---|
| CRM lent | `delay_ms=2000` | Latence ERP augmente | propagation de latence |
| Warehouse arrêté | `docker compose stop warehouse` | retries puis 503 | résilience |
| même `Idempotency-Key` | rejouer la commande | même order ID | idempotence |
| stock = 1 + 10 requêtes | test concurrent | 1 succès, 9 conflits | concurrence |
| RabbitMQ arrêté | créer commande | commande créée mais event perdu | dual write problem |
| charge Locust | 50 puis 200 users | p95/p99 augmentent | saturation |
| circuit ouvert | provoquer 3 échecs | appels bloqués temporairement | circuit breaker |
| worker lent | augmenter `sleep` | queue augmente | backpressure |

> **Important :** le but n'est pas uniquement d'obtenir un code `200` ou `201`. Il faut observer le comportement du système sous dégradation.

---

# 120. Analyse attendue

À la fin du TP, vous devez être capable d'expliquer :

1. pourquoi une API distribuée est plus complexe qu'un CRUD local ;
2. comment la latence s'accumule ;
3. pourquoi les timeouts sont obligatoires ;
4. pourquoi un retry peut empirer une panne ;
5. à quoi sert le backoff ;
6. à quoi sert le jitter ;
7. pourquoi l'idempotence est indispensable ;
8. ce qu'est une race condition ;
9. comment `SELECT FOR UPDATE` protège le stock ;
10. pourquoi un circuit breaker existe ;
11. comment RabbitMQ découple les services ;
12. ce qu'est la cohérence éventuelle ;
13. ce qu'est une saga ;
14. pourquoi l'Outbox Pattern existe ;
15. ce qu'est une DLQ ;
16. pourquoi les logs doivent utiliser des correlation IDs ;
17. pourquoi les p95 et p99 sont plus utiles qu'une moyenne seule ;
18. ce qu'est la backpressure ;
19. pourquoi il faut limiter la concurrence ;
20. pourquoi une architecture distribuée doit être testée sous charge.

---

# 121. Questions de synthèse

## Q1

Pourquoi ne faut-il pas faire :

```text
timeout = 60 secondes
retry = 10
```

sur tous les appels ?

### Réponse

Parce qu'en cas de panne, chaque requête peut :

- rester bloquée longtemps ;
- consommer des connexions ;
- déclencher de nombreux retries ;
- amplifier la surcharge ;
- provoquer une panne en cascade.

---

## Q2

Pourquoi une commande ERP utilise-t-elle une `Idempotency-Key` ?

### Réponse

Pour empêcher qu'un double clic, un retry réseau ou une répétition involontaire crée plusieurs commandes identiques.

---

## Q3

Pourquoi le Warehouse doit lui aussi être idempotent ?

### Réponse

Parce que même si l'ERP est idempotent, un retry interne ERP vers Warehouse peut provoquer plusieurs réservations.

---

## Q4

Pourquoi RabbitMQ améliore-t-il la résilience ?

### Réponse

Parce qu'il découple temporellement le producteur et le consommateur.

L'ERP peut produire un événement même si le traitement métier suivant n'est pas instantané.

---

## Q5

Quel est le principal inconvénient de l'asynchronisme ?

### Réponse

Le système devient plus complexe et peut temporairement présenter des états différents entre plusieurs services.

---

## Q6

Pourquoi utiliser un circuit breaker ?

### Réponse

Pour éviter de continuer à envoyer du trafic vers un service déjà considéré comme défaillant.

---

## Q7

Pourquoi mesurer p95 et p99 ?

### Réponse

Parce qu'ils montrent l'expérience des requêtes les plus lentes, souvent invisible avec une simple moyenne.

---

## Q8

Pourquoi une transaction SQL unique ne suffit-elle pas ?

### Réponse

Parce qu'une transaction PostgreSQL locale ne peut pas garantir automatiquement la cohérence atomique entre plusieurs bases et plusieurs services indépendants.

---

# 122. Livrables attendus

Le rendu doit contenir :

```text
tp2-distributed/
|
|-- docker-compose.yml
|-- .env.example
|
|-- crm/
|-- warehouse/
|-- erp/
|-- worker/
|
|-- load-tests/
|-- tests/
|
|-- README.md
```

---

# 123. README attendu

Le README doit contenir :

1. architecture ;
2. diagramme ;
3. prérequis ;
4. démarrage Docker ;
5. ports ;
6. création d'un client ;
7. création du stock ;
8. création d'une commande ;
9. test d'idempotence ;
10. test de concurrence ;
11. test de panne ;
12. test de latence ;
13. test Locust ;
14. analyse des résultats.

---

# 124. Barème proposé

| Critère | Points |
|---|---:|
| Docker Compose fonctionnel | 2 |
| CRM | 1 |
| Warehouse | 2 |
| ERP | 3 |
| PostgreSQL | 1 |
| Appels REST inter-services | 2 |
| Idempotence | 2 |
| Concurrence / verrouillage | 2 |
| Timeout / retry / backoff | 1.5 |
| Circuit breaker | 1 |
| RabbitMQ | 1 |
| Tests de charge | 1 |
| Analyse et réponses | 0.5 |
| **Total** | **20** |

---

# 125. Extensions très avancées

Pour aller encore plus loin :

- remplacer RabbitMQ par Kafka ;
- ajouter un Schema Registry ;
- ajouter PostgreSQL Outbox ;
- ajouter Debezium CDC ;
- ajouter OpenTelemetry ;
- ajouter Jaeger ;
- ajouter Prometheus ;
- ajouter Grafana ;
- ajouter Traefik ou Nginx ;
- ajouter une API Gateway ;
- ajouter OAuth2 / OpenID Connect ;
- ajouter Keycloak ;
- déployer sur Kubernetes ;
- ajouter Horizontal Pod Autoscaler ;
- ajouter readiness et liveness probes ;
- ajouter Kubernetes NetworkPolicy ;
- ajouter chaos testing ;
- ajouter Redis Sentinel ;
- ajouter PostgreSQL replication ;
- ajouter Pact pour les contract tests ;
- ajouter Testcontainers ;
- ajouter une stratégie Canary ;
- ajouter rate limiting centralisé ;
- ajouter service mesh ;
- ajouter gRPC pour certains échanges internes.

---

# 126. Conclusion

Ce TP montre qu'un échange API entre plusieurs systèmes métiers ne se résume pas à :

```text
POST -> JSON -> 200 OK
```

Dans une architecture réelle, il faut raisonner sur :

- la **latence** ;
- la **concurrence** ;
- les **verrous** ;
- la **résilience** ;
- les **timeouts** ;
- les **retries** ;
- le **backoff** ;
- le **jitter** ;
- l'**idempotence** ;
- le **circuit breaker** ;
- la **backpressure** ;
- les **transactions distribuées** ;
- la **cohérence éventuelle** ;
- les **événements** ;
- les **queues** ;
- l'**observabilité** ;
- les **tests de charge** ;
- les **pannes partielles**.

Ces mécanismes constituent les bases des architectures distribuées modernes et des intégrations entre ERP, CRM, WMS, systèmes de paiement, outils logistiques et plateformes de données.
