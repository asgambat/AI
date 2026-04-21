# Onboarding: AdminCore EHR Integration

> Data: 2026-04-10
> Generato da: GitHub Copilot (CodeOnborder Agent)
> Versione Progetto: 0.0.1-SNAPSHOT

---

## 📋 Quick Start (10 minuti)

### Prerequisiti

- **Java**: 21+ (Eclipse Temurin 24 consigliato)
- **Maven**: 3.8+
- **MongoDB**: locale o remoto (connection string necessaria)
- **Kafka**: local o Azure Event Hubs (bootstrap server necessario)
- **Node.js**: **NON richiesto** per il build Maven (scaricato automaticamente in `frontend/node/`)

### Setup Iniziale

1. **Clonare il repository**

   ```bash
   git clone <repo-url>
   cd admincore-ehr-integration
   ```

2. **Configurare le variabili di ambiente** (creare `.env` nella root del progetto):

   ```bash
   # Database
   MONGO_PROTOCOL=mongodb
   MONGO_USER=<username>
   MONGO_PASSWORD=<password>
   MONGO_URI=<host>:<port>/<database>
   
   # Kafka
   KAFKA_SERVER=<bootstrap-server>:9092
   KAFKA_API_KEY=<api-key>
   KAFKA_API_SECRET=<api-secret>
   KAFKA_SECURITY_PROTOCOL=SASL_SSL
   KAFKA_OFFSET_RESET=latest
   
   # EHR API
   EHR_API_BASE_URL=<ehr-endpoint>
   EHR_API_SUBSCRIPTION_KEY=<ehr-key>
   
   # Applicazione
   PORT=8080
   SPRING_APPLICATION_NAME=admincore-ehr-integration
   ```

3. **Build backend + frontend completo**

   ```bash
   mvn clean package -Dskip.frontend=false
   ```

   Oppure, **backend solo** (più veloce durante lo sviluppo):

   ```bash
   mvn clean package -Dskip.frontend=true
   ```

4. **Avviare l'applicazione**

   ```bash
   java -jar target/admincore-ehr-integration-0.0.1-SNAPSHOT.jar \
     --spring.profiles.active=dev
   ```

5. **Verificare il funzionamento**
   - API: `curl http://localhost:8080/api/resources`
   - GUI: Apri browser → `http://localhost:8080`
   - Health: `curl http://localhost:8080/actuator/health`

---

## 🏗️ Mappa di Sistema (Bird's-Eye View)

### Cosa fa il sistema?

**AdminCore-EHR Integration** è un servizio di sincronizzazione FHIR che:

- Consuma eventi FHIR (Practitioner, Organization, Practice, Location, PractitionerRole) da Kafka
- Persiste le risorse in MongoDB
- Fornisce API REST per la consultazione e la gestione
- Esegue job di "feeder" per validare, processare e sincronizzare i dati verso l'EHR v17
- Espone una GUI React per il monitoraggio real-time, il controllo batch e la gestione della configurazione

### Componenti Principali

```
┌─────────────────────────────────────────────────────────────┐
│                   KAFKA (Message Broker)                    │
│ Topics: fhir-practitioner, fhir-organization, fhir-practice │
│         fhir-location, fhir-practitioner-role               │
└──────────────────────┬──────────────────────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
┌────────────────────────────────────────────┐
│ Kafka Consumers (Event Ingestion)          │
│ ├─ LocationEventConsumer                   │
│ ├─ OrganizationEventConsumer               │
│ ├─ PractitionerEventConsumer               │
│ ├─ PractitionerRoleEventConsumer           │
└────────────────────┬───────────────────────┘
                     │
                     ▼
        ┌────────────────────────┐
        │  MONGODB (Persistence) │
        │  Collections:          │
        │  - Resources (FHIR)    │
        │  - Batch Jobs          │
        │  - Configuration       │
        └───────────┬────────────┘
                    │
     ┌──────────────┼────────────────────┐
     ▼               ▼                   ▼
┌─────────────┐ ┌────────────────┐ ┌────────────────────────┐
│  REST API   │ │ Schedulers     │ │ Feeder Jobs            │
│ /api/       │ │ (Periodic)     │ │ (Process & Send)       │
│ resources   │ │                │ │                        │
│ /api/feeder │ │ RetryScheduler │ ├─ EhrFeeder             │
│ /api/config │ │ FeederScheduler│ ├─ PracticeFeeder        │
│ /api/stats  │ │                │ ├─ PractitionerRoleFeeder│
└─────────────┘ └────────────────┘ └────────────────────────┘
     ▲            (Every N secs)         │
     │                                   │
     │                                   ▼
     │                       ┌──────────────────┐
     │                       │  EHR API v17     │
     └───────────────────────│  (HTTP Client)   │
       GUI Monitoring        └──────────────────┘
       /api/batch
       /api/config
       /api/stats
```

---

## 🔄 Architettura e Dipendenze

### Stack Tecnologico

| Layer | Tecnologia | Versione | Note |
| ------- | ----------- | ---------- | ------ |
| **Backend** | Java + Spring Boot | 21 + 4.0.3 | Spring Data MongoDB, Spring Kafka |
| **Frontend** | React + TypeScript + Vite | 19 + TS 5.8 | ESLint, Vitest, React Router |
| **Database** | MongoDB | latest | Spring Data MongoDB (auto-config) |
| **Broker** | Kafka | 4.2.0 | Supporta Azure Event Hubs |
| **FHIR** | HAPI FHIR | 8.6.0 | R4 structures + base |
| **Logging** | Logback + Logstash | 8.0 | JSON structured logging |
| **Rate Limiter** | Bucket4j | 8.10.1 | Request throttling |

### Dipendenze Interne (Moduli)

```AdmincoreEhrIntegrationApplication (Entry Point)
│
├─ Kafka Consumers (Ingestion Layer)
│  ├─ OrganizationEventConsumer
│  ├─ LocationEventConsumer
│  ├─ PractitionerEventConsumer
│  └─ PractitionerRoleEventConsumer
│
├─ Service Layer (Business Logic)
│  ├─ feeder/
│  │  ├─ EhrFeeder (Strategy pattern)
│  │  ├─ PracticeFeeder
│  │  ├─ PractitionerRoleFeeder
│  │  └─ FeederJobProperties (Config)
│  │
│  ├─ processors/ (Data processing)
│  │
│  ├─ schedulers/ (Periodic jobs)
│  │  ├─ FeederScheduler
│  │  └─ RetryScheduler
│  │
│  ├─ evaluator/ (Validation logic)
│  │
│  └─ ehr/ (EHR integration)
│
├─ Controller Layer (REST API)
│  ├─ ResourceController (/api/resources)
│  ├─ FeederController (/api/feeder)
│  ├─ (UI Controllers in ui.controller.*)
│  │  ├─ BatchController (/api/batch)
│  │  ├─ ConfigController (/api/config)
│  │  ├─ StatsController (/api/stats)
│  │  └─ KafkaListenerController
│
├─ Repository Layer (Data Access)
│  └─ MongoHelper (MongoDB queries)
│
├─ Config Layer
│  ├─ ApplicationProperties
│  ├─ EhrProperties
│  ├─ FeederJobProperties
│  ├─ FhirProperties
│  ├─ LogstashProperties
│  ├─ KafkaClusterGenericConfig
│  ├─ RetryConfig
│  └─ MongoIndexConfiguration
│
└─ Frontend (React SPA)
   ├─ App.tsx (Root)
   ├─ components/
   │  ├─ monitoring/ (Real-time dashboards)
   │  └─ config/ (Config management UI)
   ├─ pages/ (Router pages)
   ├─ hooks/ (Custom React hooks)
   └─ api/ (HTTP client)
```

---

## 🔄 Top 3 Business Flows

### Flow 1: Event Ingestion & Persistence

**Trigger**: Evento FHIR pubblicato su Kafka topic

**Sequenza**:

1. Kafka consumer (`PractitionerEventConsumer`, etc.) riceve il messaggio
2. Valida il formato (FHIR HAPI parser)
3. Estrae l'ID risorsa dalla payload
4. Persiste/aggiorna in MongoDB collection (es. `practitioners`, `organizations`)
5. Registra il batch job per il processing successivo
6. Commit dell'offset Kafka (BATCH ack mode)

**Dati interessati**:

- **Input**: JSON FHIR da Kafka
- **Storage**: MongoDB collections per tipo risorsa
- **Metadata**: Timestamp, source, processing status

**Gestione errori**:

- Parse error → Log + skip (no retry)
- DB error → Rollback offset, retry next consumer poll
- Network timeout → Kafka auto-retry handling

**File pertinenti**:

- `src/main/java/com/acme/admincore_ehr_integration/service/consumers/`
- `AbstractConsumer.java` (base logic)
- `OrganizationEventConsumer.java`, `LocationEventConsumer.java`, etc.

---

### Flow 2: Feeder Job Processing

**Trigger**: Scheduler `FeederScheduler` (every N seconds, configurable)

**Sequenza**:

1. FeederScheduler sceglie un batch di risorse non-elaborate da MongoDB
2. Per ogni risorsa, invoca il `Feeder` appropriato (PracticeFeeder, PractitionerRoleFeeder, etc.)
3. Feeder applica business logic:
   - Trasformazione FHIR → formato EHR
   - Validazione dati
   - Enrichment (lookup esterni, se necessario)
4. Invia la risorsa all'EHR API via HTTP (RestClient)
5. Su successo: marca la risorsa come "processed" in MongoDB
6. Su errore (retryable): incrementa attempt counter, schedule for retry
7. Su error permanente: marca come "failed", log dettagliato

**Schedulatore di Retry**:

- `RetryScheduler` esegue periodicamente (configurabile)
- Prende risorse con `status=pending_retry` e `attempt < maxAttempts`
- Reinvia tramite Feeder
- Dopo `maxAttempts`, passa a "failed"

**Configurazione (hot-reloadable)**:

- `application.feeder.processing-max-number` (batch size per run)
- `application.feeder.processing-max-attempts` (max retry count)
- `application.feeder.lease-seconds` (lock duration per batch)

**File pertinenti**:

- `src/main/java/com/acme/admincore_ehr_integration/service/feeder/`
- `EhrFeeder.java` (base interface)
- `PracticeFeeder.java`, `PractitionerRoleFeeder.java`
- `src/main/java/com/acme/admincore_ehr_integration/service/schedulers/`

---

### Flow 3: Configuration Management (Hot-Reload)

**Trigger**: Operatore invia `PUT /api/config/{propertyPath}` via GUI

**Sequenza**:

1. `ConfigController` riceve richiesta con nuovo valore
2. Valida il path e il tipo di valore
3. Salva in mappa in-memory (override)
4. Chiama `RuntimeConfigApplicator` per determinare se applicabile a runtime
5. Se **hot-reloadable** (4 proprietà specifiche):
   - Invalida la cache (se presente)
   - Chiama il setter sul bean `@ConfigurationProperties`
   - Durante il prossimo ciclo scheduler/request, il nuovo valore è già attivo
   - Ritorna `hotReloaded=true` al cliente
6. Se **restart-required** (2 proprietà, es. scheduled delays):
   - Memorizza override per il prossimo startup
   - Ritorna `requiresRestart=true` al cliente
7. GUI notifica operatore: "Applied at runtime" o "Restart needed"

**Hot-Reloadable Properties**:

- `application.feeder.lease-seconds`
- `application.feeder.processing-max-number`
- `application.feeder.processing-max-attempts`
- `ui.stats.default-error-rate-threshold`

**Restart-Required Properties**:

- `application.feeder.schedulers.user-feeder-fixed-delay`
- `application.feeder.schedulers.store-feeder-fixed-delay`

**File pertinenti**:

- `src/main/java/com/acme/admincore_ehr_integration/ui/controller/ConfigController.java`
- `src/main/java/com/acme/admincore_ehr_integration/ui/service/` (ConfigService, RuntimeConfigApplicator)
- `docs/ADRs/adr-001-runtime-config-refresh.md` (Decision rationale)

---

## ⚙️ Configuration & Environments

### File di Configurazione Principali

| File | Scopo | Nota |
| ------ | ------- | ------ |
| `src/main/resources/config/application.yaml` | Configurazione principale | Leggere per capire tutti i parametri |
| `src/main/resources/config/application-dev.yaml` | Override per dev locale | Profile: `spring.profiles.active=dev` |
| `.env` (root project) | Variabili d'ambiente locali | **NON committare** |

### Gestione Segreti

- **Per lo sviluppo locale**: Crea `.env` nella root; caricato da `scripts/run.ps1`
- **Per staging/prod**: Usa variabili d'ambiente del container/orchestrator (non committare in repo)
- **Secrets non devono mai essere nel codice** ❌

### Profili Spring Boot

```bash
# Development (locale with hot-reload)
mvn spring-boot:run -Dspring-boot.run.arguments=--spring.profiles.active=dev

# Staging
java -jar app.jar --spring.profiles.active=staging

# Production
java -jar app.jar --spring.profiles.active=prod
```

---

## 🧪 Testing & Quality Gates

### Eseguire i Test

**Backend**:

```bash
# Tutti i test
mvn test

# Test specifico
mvn test -Dtest=FeederControllerTest

# Con coverage
mvn test jacoco:report
```

---

## 📍 "Dove Cambiare Cosa" — Guida per Contribuenti

### Se devi modificare l'**API REST** (aggiungere endpoint, DTO)

**Inizia qui:**

1. `src/main/java/com/acme/admincore_ehr_integration/controller/` (entry REST)
2. Crea DTO request/response in `domain/` (separato da model internal)
3. Aggiungi business logic in `service/`
4. Espandi test in `src/test/`

**Convenzioni**:

- `@RestController` → `@RequestMapping("/api/...")`
- Request/response → `@GetMapping`, `@PostMapping`, etc.
- DTO separati da domain models (SOLID)

---

### Se devi modificare la **logica di processing Feeder** (trasformazione, validazione FHIR)

**Inizia qui:**

1. `src/main/java/com/acme/admincore_ehr_integration/service/feeder/`
2. Implementa `IEhrFeederStrategy` per nuove risorse
3. Aggiungi in `FeederJobProperties` se serve nuova config
4. Test in `src/test/java/com/acme/admincore_ehr_integration/service/feeder/`

**Convenzioni**:

- Strategy pattern per diversi tipi di risorse
- Thread-safe per execution parallela
- Logging dettagliato per debug

---

### Se devi aggiungere un **nuovo Kafka Consumer** (nuovo tipo evento FHIR)

**Inizia qui:**

1. `src/main/java/com/acme/admincore_ehr_integration/service/consumers/`
2. Estendi `AbstractConsumer` (base logic comuni)
3. Implementa `@KafkaListener(topics = "...")`
4. Registra il bean in `AdmincoreEhrIntegrationApplication`
5. Test in `src/test/java/com/acme/admincore_ehr_integration/service/consumers/`

**Convenzioni**:

- Graceful shutdown via `GracefulKafkaConsumerLifecycle`
- Errori non-retryable → log + skip
- Errori retryable → rollback offset, next poll riprova

---

### Se devi modificare la **UI** (React components, pages)

**Inizia qui:**

1. `frontend/src/components/` (shared components)
2. `frontend/src/pages/` (page/route specifiche)
3. `frontend/src/api/` (HTTP client wrapper)
4. Test in `frontend/src/test/`

**Convenzioni**:

- React hooks per state management
- Separazione presentazione ↔ logica (container patterns)
- TypeScript strict mode
- ESLint compliance

**Dev/Hot-Reload**:

```bash
# Terminal 1: Backend API
mvn spring-boot:run \
  -Dspring-boot.run.arguments=--spring.profiles.active=dev \
  -Dskip.frontend=true

# Terminal 2: Vite dev server (frontend hot-reload)
cd frontend/
npm run dev   # Serve on http://localhost:5173
```

---

### Se devi aggiungere **configurazione** (nuova property application.yaml)

**Inizia qui:**

1. Aggiungi property in `src/main/resources/config/application.yaml`
2. Crea `@ConfigurationProperties` bean in `config/` se classe non esiste
3. Inietta in servizio che la usa
4. Determina se hot-reloadable o restart-required
5. Se hot-reloadable, aggiorna `RuntimeConfigApplicator` in `ui/service/`

**Convenzioni**:

- Usare env var per override (es. `${KAFKA_SERVER:default}`)
- Documentare in questo ONBOARDING

---

### Se devi modificare la **persistenza** (MongoDB schema, query)

**Inizia qui:**

1. `src/main/java/com/acme/admincore_ehr_integration/repository/`
2. `MongoHelper` → query e mapping
3. `config/MongoIndexConfiguration.java` → indici per performance
4. Test in `src/test/java/com/acme/admincore_ehr_integration/repository/`

**Convenzioni**:

- Query parametrizzate (no string concat → injection risk)
- Indici su campi frequentemente cercati
- Validazione dati in input

---

## 🏗️ Entry Points per Sviluppatore Nuovo

### Per capire il flusso globale (30 minuti)

1. Leggi `README.md` (questo doc, sezioni overview)
2. Leggi `AdmincoreEhrIntegrationApplication.java`
3. Leggi `application.yaml` (prima 50 linee)
4. Scegli un consumer di tuo interesse (es. `OrganizationEventConsumer.java`) e traccia fino a MongoDB

### Per debuggare un'issue (15-30 minuti)

1. Leggi il messaggio d'errore e lo stack trace
2. Identifica il layer (consumer? feeder? controller? DB?)
3. Se Kafka:
   - Verifica config bootstrap server: `application.yaml`
   - Verifica consumer topic in `@KafkaListener`
   - Test config con `kafka-console-consumer` (tool esterno)
4. Se DB:
   - Leggi query in `MongoHelper`
   - Usa MongoDB Compass per ispezionare le collections
   - Verifica indici in config
5. Se feeder:
   - Abilita debug logging: `application-dev.yaml` con `LOG_LEVEL=DEBUG`
   - Ispeziona trasformazione FHIR in `*Feeder.java`
6. Se REST API:
   - Test endpoint con Postman o `curl`
   - Abilita logging controller

### Per eseguire il progetto localmente (10 minuti)

```bash
# 1. Start MongoDB (Docker)
docker run -d --name mongo -p 27017:27017 \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=admin \
  mongo:latest

# 2. Start Kafka (Docker) — o usa broker remoto
docker run -d --name kafka -p 9092:9092 \
  -e KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092 \
  -e KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=PLAINTEXT:PLAINTEXT \
  -e KAFKA_CFG_ZOOKEEPER_CONNECT=zookeeper:2181 \
  bitnami/kafka:latest

# 3. Start app backend
mvn spring-boot:run \
  -Dspring-boot.run.arguments=--spring.profiles.active=dev \
  -Dskip.frontend=true

# 4. In another terminal, start frontend (optional)
cd frontend/ && npm run dev

# 5. Accedi alla GUI
# Open browser → http://localhost:8080 (API) or http://localhost:5173 (Vite if dev)
```

---

## 📚 Glossario Terminologico

| Termine | Significato | Contesto |
| --------- | ------------ | --------- |
| **FHIR** | Fast Healthcare Interoperability Resources | Standard per scambio dati sanitari |
| **EHR** | Electronic Health Record | Sistema di destinazione (v17) |
| **AdminCore** | Sistema sorgente | Produce eventi |
| **Feeder** | Componente di processing/invio | Legge MongoDB, trasforma, invia EHR |
| **Consumer** | Kafka consumer | Ascolta topic Kafka, persiste in MongoDB |
| **Batch** | Lotto risorse per processing | Identificato da lease-time, retry-count |
| **Hot-reload** | Configurazione applicabile a runtime | Senza restart JVM |
| **Strategy Pattern** | Design pattern (IEhrFeederStrategy) | Estensibilità per diversi tipi risorse |
| **HAPI** | HL7® FHIR® parser library | `ca.uhn.hapi.fhir` |
| **Logstash** | Log collector/processor | Output JSON structured logs |
| **RestClient** | Spring HTTP client (5.0+) | Invocazioni API EHR |

---

## ⚠️ Zone Critiche & Considerazioni

### Zone ad Alto Rischio (Cambiamenti richiedono estrema attenzione)

1. **Feeder Logic** (PracticeFeeder, PractitionerRoleFeeder)
   - Trasformazione FHIR → EHR determina correttezza dei dati
   - Bug qui → dati corrotti nel sistema target
   - ✅ Risolvibile con test coverage 100% su edge cases

2. **Kafka Consumer Lifecycle** (GracefulKafkaConsumerLifecycle, batch ack mode)
   - Gestione offset determina idempotenza e consistency
   - Configurazione scorretta → duplicati or data loss
   - ✅ Controllare ADR-001 e test di lifecycle

3. **MongoDB Indexes** (MongoIndexConfiguration)
   - Assenza di indici → query lenissime in produzione
   - ✅ Verificare performance con `mongostat` prima deploy

---

## 🔧 Comandi Utili per lo Sviluppo

```bash
# Build
mvn clean package -Dskip.frontend=true

# Test
mvn test
mvn test -Dtest=FeederControllerTest  # test specifico

# Run backend dev
mvn spring-boot:run \
  -Dspring-boot.run.arguments=--spring.profiles.active=dev \
  -Dskip.frontend=true

# Run backend debug
mvn spring-boot:run \
  -Dspring-boot.run.arguments=--spring.profiles.active=dev \
  -Dspring-boot.run.jvmArguments="-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005" \
  -Dskip.frontend=true

# Run frontend dev (separate terminal)
cd frontend/ && npm run dev   # http://localhost:5173

# Lint
cd frontend/ && npm run lint
npm run lint:fix

# Type check
cd frontend/ && npm run typecheck

# Coverage frontend
cd frontend/ && npm run test:coverage

# MongoDB (inspect)
# Use MongoDB Compass → mongodb://admin:admin@localhost:27017

# Kafka (inspect topics)
kafka-topics.sh --bootstrap-server localhost:9092 --list
kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic fhir-organization

# Clean up Docker
docker container stop mongo kafka zookeeper
docker container rm mongo kafka zookeeper
```

---

## 📖 Documentazione Aggiuntiva

- `README.md` — Overview funzionale, setup, build
- `docs/ADRs/adr-001-runtime-config-refresh.md` — Decision rationale per config hot-reload
- `docs/engineering/code-review-guidelines.md` — Code review process
- `.github/instructions/backend.instructions.md` — Backend coding standards (Java, Spring Boot)
- `.github/instructions/frontend.instructions.md` — Frontend coding standards (React, TS)

---

## 🎯 Prossimi Passi per Nuovi Contributori

1. ✅ Leggi questo ONBOARDING.md (completo)
2. ✅ Clona il repo e esegui Quick Start (10 min)
3. ✅ Leggi ADR-001 se interessato a feeder/config logic
4. ✅ Scegli un area (API, Feeder, Consumer, UI) e leggi i file pertinenti
5. ✅ Esegui `mvn test` per assicurarti che l'ambiente funziona
6. ✅ Apri una PR (segui code-review-guidelines.md) per il tuo primo contributo

---

**Data Ultima Revisione**: 2026-04-10  
**Autore**: GitHub Copilot (CodeOnborder Agent)
