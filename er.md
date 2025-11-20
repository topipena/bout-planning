```mermaid
erDiagram
  USERS {
    text id PK
    text email
  }

  OPERATORS {
    uuid id PK
    text name
    text size "ENUM: small|large"
  }

  COUNTRIES {
    text code PK
    text name
  }

  AREAS {
    uuid id PK
    text name
    text country_code FK
  }

  MEMBERSHIPS {
    text user_id FK
    text role
    uuid operator_id FK
    uuid area_id FK
  }

  VESSELS {
    uuid id PK
    uuid operator_id FK
    uuid area_id FK
    uuid category_id FK
    text name
    numeric horsepower
    text propulsion_type
    int capacity
  }

  VESSEL_POSITIONS {
    uuid id PK
    uuid vessel_id FK
    timestamptz at
    jsonb location "lat/lng"
  }

  VESSEL_CATEGORIES {
    uuid id PK
    text code
    jsonb name_i18n
  }

  ROUTES {
    uuid id PK
    jsonb name_i18n
    jsonb limits "speed/capacity/weather windows"
  }

  SERVICE_POINTS {
    uuid id PK
    uuid area_id FK
    jsonb name_i18n
  }

  TRIPS {
    uuid id PK
    uuid vessel_id FK
    uuid route_id FK
    uuid operator_id FK
    uuid area_id FK
    text captain_user_id FK
    timestamptz start_time
    timestamptz end_time
  }

  AREA_SETTINGS {
    uuid area_id PK
    numeric commission_pct
    text payment_timing "ENUM: on_acceptance|after_trip"
    uuid pricing_model_id FK
  }

  PRICING_MODELS {
    uuid id PK
    text scope "ENUM: area|route|operator"
    text type "ENUM: fixed|distance_based|time_based|dynamic"
    jsonb config
  }

  PRICING_RULES {
    uuid id PK
    uuid pricing_model_id FK
    bool include "true = inclusion, false = exclusion"
    uuid operator_id FK "nullable"
    uuid vessel_category_id FK "nullable"
    uuid route_id FK "nullable"
    jsonb conditions "optional extra filters"
  }


  USERS ||--o{ MEMBERSHIPS : has
  OPERATORS ||--o{ MEMBERSHIPS : scopes
  AREAS ||--o{ MEMBERSHIPS : scopes

  OPERATORS ||--o{ VESSELS : owns
  AREAS ||--o{ VESSELS : contains
  VESSEL_CATEGORIES ||--o{ VESSELS : classifies

  VESSELS ||--o{ TRIPS : uses
  ROUTES ||--o{ TRIPS : scheduled_on
  USERS  ||--o{ TRIPS : captains

  ROUTES  ||--o{ SERVICE_POINTS : includes
  AREAS ||--o{ SERVICE_POINTS : contains

  AREAS ||--|| AREA_SETTINGS : has_settings
  PRICING_MODELS ||--o{ AREA_SETTINGS : used_by

  PRICING_MODELS ||--o{ PRICING_RULES : has_rules
  COUNTRIES ||--o{ AREAS : contains

```
