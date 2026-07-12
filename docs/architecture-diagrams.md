# Architecture diagram sources (Mermaid)

Editable source for the rendered diagrams in `SYSTEM_ARCHITECTURE.md`. Re-export to `/diagrams/<name>.svg` after editing.

## High-level pipeline (Figure 1)

`diagrams/01-high-level-pipeline.svg`

```mermaid
flowchart TD
    subgraph M["Monitored machine (one per employee)"]
        L[pta_launcher] --> P[process monitor]
        L --> K[keyboard service]
        L --> MO[mouse agent + backend]
        L --> F[focus service]
        L --> A[activities syncer]
    end

    P --> DB[(PostgreSQL<br/>raw event tables)]
    K --> DB
    MO --> DB
    F --> DB
    A --> DB

    DB --> AGG[Aggregation functions<br/>UTS · CDM · EGF]
    ID[Canonical identity layer] -.-> AGG
    AGG --> FEAT[(Feature tables<br/>3 engineered tables)]
    FEAT --> ML[ML models<br/>PSM · CDM · GPRM]
    ML --> PRED[(Prediction tables)]
    PRED --> API[HTTP API]
    API --> UI[Dashboard]
```

## Activity derivation (Figure 2)

`diagrams/02-activity-derivation.svg`

```mermaid
flowchart LR
    PE[(process_events<br/>this host only)] --> SY[Activity syncer<br/>enrich + classify]
    SY --> ACT[(activities)]
```

## Data model layers (Figure 3)

`diagrams/03-data-model-layers.svg`

```mermaid
flowchart TD
    RAW["RAW LAYER — 20 event tables<br/>process · keyboard · mouse · focus · activities"]
    ID["IDENTITY LAYER<br/>employee_identity · employee_identity_override"]
    FEAT["FEATURE LAYER — 3 engineered tables<br/>unified_time_series · cheat_detection_features · employee_growth_features"]
    WM["WATERMARKS — 3 tables<br/>incremental aggregation cursors"]
    PRED["PREDICTION LAYER — 4 tables<br/>productivity · cheat · cheat_alerts · growth"]

    RAW --> FEAT
    ID -.->|canonical user/device| FEAT
    WM -.->|how far processed| FEAT
    FEAT --> PRED
```

## Feature aggregation (Figure 4)

`diagrams/04-feature-aggregation.svg`

```mermaid
flowchart LR
    RAW[(raw event tables)] --> STG[stg_* per-device slice]
    STG --> FN["aggregation function<br/>(1 row per window × device)"]
    WM[(watermark)] <-->|read/advance| FN
    FN --> OUT[(feature table)]
```

## Infrastructure &amp; scaling (Figure 5)

`diagrams/05-infrastructure-scaling.svg`

```mermaid
flowchart TD
    subgraph AG["N monitored machines"]
        direction LR
        A1[machine: 5 services]
        A2[machine: 5 services]
        A3[machine: 5 services]
    end
    AG -->|pooled connections| PB[Connection pooler]
    PB --> PG[(PostgreSQL primary)]
    PG -.->|streaming replication| RR[(read replica*)]
    PG --> AGGJ[Aggregation + scoring jobs]
    subgraph APIT["Stateless API tier"]
        direction LR
        I1[API instance]
        I2[API instance]
        I3[API instance]
    end
    RR --> APIT
    PG --> APIT
    LB[Load balancer*] --> APIT
    APIT --> UI[Dashboard]
```

