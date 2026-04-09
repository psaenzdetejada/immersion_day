# 🚚 Guia paso a paso para Immersion Day: Logistics & Fleet Copilot
## Paso a paso con prompts para Cortex Code

**Tiempo total:** 90 minutos
**Herramienta:** Snowflake Cortex Code (CoCo) — todo se hace con prompts en el IDE
**Objetivo:** Construir un pipeline end-to-end Bronze > Silver > Gold + 3 casos de uso AI + Semantic View + Cortex Agent
**Industria:** Transporte, Logistica y Travel
**Base de datos:** LOGISTICS_IMMERSION

---

## FASE 0: CREAR CUENTA TRIAL DE SNOWFLAKE (10 minutos)

> **Objetivo:** Cada participante crea su propia cuenta trial de Snowflake para el hackathon.

### Paso 0.1 — Registro en Snowflake

1. Abre el navegador y ve a: **https://signup.snowflake.com/**
2. Rellena el formulario con tu email corporativo
3. En la pantalla de configuracion, selecciona:
   - **Cloud Provider:** AWS
   - **Edition:** Enterprise
   - **Region:** EU Frankfurt
4. Acepta los terminos y haz clic en **"Get Started"**
5. Revisa tu email y haz clic en el enlace de activacion
6. Establece tu usuario y contrasena

> **Importante:** La cuenta trial incluye **$400 en creditos** y dura **30 dias**.

### Paso 0.2 — Primer acceso y configuracion

1. Accede a tu cuenta Snowflake desde la URL que recibiste por email
2. Verifica que tienes el rol **ACCOUNTADMIN**
3. Comprueba que el warehouse **COMPUTE_WH** esta disponible

```
Ejecuta: SELECT CURRENT_ACCOUNT(), CURRENT_ROLE(), CURRENT_WAREHOUSE(), CURRENT_REGION();
```

### Paso 0.3 — Activar Cortex Code

1. En Snowsight, ve a **Projects > Cortex Code**
2. Acepta los terminos de uso de Cortex AI
3. Abre una nueva sesion de Cortex Code
4. Verifica con un prompt simple:

```
Dime que version de Snowflake estoy usando con SELECT CURRENT_VERSION();
```

> **Checkpoint:** Cuenta trial activa con Cortex Code funcionando.

---

### Datos disponibles

Los 4 ficheros JSON estan en un repositorio Git de GitHub:

```
https://github.com/psaenzdetejada/immersion_day/
  └── data/
        ├── fleet_vehicles.json            (350 vehiculos de flota)
        ├── route_events.json              (10,000 eventos de ruta)
        ├── delivery_tickets.json          (750 tickets de incidencias)
        └── shipments.json                 (2,000 envios)
```

### Arquitectura objetivo

```
GitHub Repo (JSON) --> Git Integration --> Snowflake
         │
   ┌─────▼─────────────────────────────────────────────┐
   │  BRONZE (raw JSON en tablas VARIANT)               │
   │  ↓                                                 │
   │  SILVER (normalizado, flatten, cast, dedup)        │
   │  ↓                                                 │
   │  GOLD (KPIs, metricas, vistas analiticas)          │
   │  ↓                                                 │
   │  AI (3 casos de uso con Cortex)                    │
   │  ↓                                                 │
   │  SEMANTIC VIEW (modelo semantico para NL2SQL)      │
   │  ↓                                                 │
   │  AGENT (Copilot en Snowflake Intelligence)         │
   └────────────────────────────────────────────────────┘
```

### Roles sugeridos en el equipo (5 personas)

| Rol | Que hace | Cuando |
|-----|----------|--------|
| **Ingeniero de ingesta** | Conecta el repo Git y carga los 4 JSON a Bronze | Fase 1 |
| **Ingeniero dbt 1** | Modelos Silver (flatten + cast) | Fase 2 |
| **Ingeniero dbt 2** | Modelos Gold (KPIs + metricas) | Fase 2 |
| **Ingeniero AI** | 3 prompts Cortex + Semantic View + Agent | Fase 3-5 |
| **Demo lead** | Prepara la presentacion desde el minuto 60 | Todo |

---

## FASE 1: BRONZE — Ingesta de JSON via Git Integration (25 minutos)

> **Objetivo:** Conectar el repositorio Git de GitHub a Snowflake y cargar los 4 ficheros JSON como tablas raw en Bronze.

### Paso 1.1 — Crear la base de datos y esquemas

```
Crea una base de datos llamada LOGISTICS_IMMERSION con cuatro esquemas: BRONZE, SILVER, GOLD y AI.
Usa el warehouse COMPUTE_WH.
```

### Paso 1.2 — Crear la integracion con el repositorio Git

```
Necesito conectar Snowflake a un repositorio publico de GitHub para cargar datos JSON.
El repositorio es: https://github.com/psaenzdetejada/immersion_day/

Crea una API Integration de tipo git_https_api que permita acceder a
https://github.com/Snowflake-Spain-SE-Demos-Sandbox
y luego crea un Git Repository en LOGISTICS_IMMERSION.PUBLIC llamado IMMERSION_REPO
que apunte a ese repositorio.

No necesita autenticacion porque es un repo publico.
```

> **SQL directo:**

```sql
CREATE OR REPLACE API INTEGRATION GIT_HACKATHON_INTEGRATION
  API_PROVIDER = git_https_api
  API_ALLOWED_PREFIXES = ('https://github.com/psaenzdetejada/immersion_day/')
  ENABLED = TRUE;

CREATE OR REPLACE GIT REPOSITORY LOGISTICS_IMMERSION.PUBLIC.HACKATHON_REPO
  API_INTEGRATION = GIT_HACKATHON_INTEGRATION
  ORIGIN = 'https://github.com/psaenzdetejada/immersion_day/.git';
```

### Paso 1.3 — Sincronizar y verificar

```sql
ALTER GIT REPOSITORY LOGISTICS_IMMERSION.PUBLIC.HACKATHON_REPO FETCH;
LIST @LOGISTICS_IMMERSION.PUBLIC.HACKATHON_REPO/branches/main/data/logistics/;
```

> **Resultado esperado:** Los 4 ficheros JSON listados.

### Paso 1.4 — Crear el file format

```
En LOGISTICS_IMMERSION.BRONZE, crea un file format JSON llamado JSON_FORMAT con STRIP_OUTER_ARRAY = TRUE.
```

### Paso 1.5 — Crear tablas Bronze y cargar datos

> **SQL directo:**

```sql
USE DATABASE LOGISTICS_IMMERSION;
USE SCHEMA BRONZE;

CREATE OR REPLACE TABLE RAW_FLEET (
    SRC VARIANT,
    LOAD_TS TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    FILENAME VARCHAR DEFAULT METADATA$FILENAME
);
COPY INTO RAW_FLEET (SRC, LOAD_TS, FILENAME)
FROM (SELECT $1, CURRENT_TIMESTAMP(), METADATA$FILENAME FROM @LOGISTICS_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/logistics/fleet_vehicles.json)
FILE_FORMAT = (TYPE = 'JSON', STRIP_OUTER_ARRAY = TRUE);

CREATE OR REPLACE TABLE RAW_ROUTE_EVENTS (
    SRC VARIANT,
    LOAD_TS TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    FILENAME VARCHAR DEFAULT METADATA$FILENAME
);
COPY INTO RAW_ROUTE_EVENTS (SRC, LOAD_TS, FILENAME)
FROM (SELECT $1, CURRENT_TIMESTAMP(), METADATA$FILENAME FROM @LOGISTICS_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/logistics/route_events.json)
FILE_FORMAT = (TYPE = 'JSON', STRIP_OUTER_ARRAY = TRUE);

CREATE OR REPLACE TABLE RAW_DELIVERY_TICKETS (
    SRC VARIANT,
    LOAD_TS TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    FILENAME VARCHAR DEFAULT METADATA$FILENAME
);
COPY INTO RAW_DELIVERY_TICKETS (SRC, LOAD_TS, FILENAME)
FROM (SELECT $1, CURRENT_TIMESTAMP(), METADATA$FILENAME FROM @LOGISTICS_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/logistics/delivery_tickets.json)
FILE_FORMAT = (TYPE = 'JSON', STRIP_OUTER_ARRAY = TRUE);

CREATE OR REPLACE TABLE RAW_SHIPMENTS (
    SRC VARIANT,
    LOAD_TS TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    FILENAME VARCHAR DEFAULT METADATA$FILENAME
);
COPY INTO RAW_SHIPMENTS (SRC, LOAD_TS, FILENAME)
FROM (SELECT $1, CURRENT_TIMESTAMP(), METADATA$FILENAME FROM @LOGISTICS_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/logistics/shipments.json)
FILE_FORMAT = (TYPE = 'JSON', STRIP_OUTER_ARRAY = TRUE);
```

### Paso 1.6 — Validar la carga

```
Haz un SELECT COUNT(*) de cada una de las 4 tablas Bronze en LOGISTICS_IMMERSION.BRONZE
y muestra una fila de ejemplo de cada una.
```

> **Checkpoint:** RAW_FLEET: 350 filas / RAW_ROUTE_EVENTS: 10,000 filas / RAW_DELIVERY_TICKETS: 750 filas / RAW_SHIPMENTS: 2,000 filas

---

## FASE 2: SILVER + GOLD — Transformacion con dbt (35 minutos)

> **Objetivo:** Crear modelos dbt que transformen el JSON raw en tablas normalizadas (Silver) y vistas analiticas (Gold).

### Paso 2.1 — Inicializar el proyecto dbt

```
Crea un proyecto dbt llamado LOGISTICS_DBT que conecte a la base de datos LOGISTICS_IMMERSION.
El proyecto debe tener:
- models/staging/ (modelos Silver)
- models/marts/ (modelos Gold)

El profile debe usar warehouse COMPUTE_WH, base de datos LOGISTICS_IMMERSION,
esquema SILVER para staging y GOLD para marts.
```

### Paso 2.2 — Sources

```
Crea models/staging/sources.yml con las 4 tablas Bronze como sources:
- database: LOGISTICS_IMMERSION
- schema: BRONZE
- tables: RAW_FLEET, RAW_ROUTE_EVENTS, RAW_DELIVERY_TICKETS, RAW_SHIPMENTS
```


### Modelo Silver: STG_FLEET

```
Crea un modelo dbt models/staging/stg_fleet.sql que haga flatten del JSON
de la tabla RAW_FLEET (source Bronze). Extrae todos los campos relevantes del JSON:
vehiculos de flota (vehicle_id, plate_number, type, brand, base_location, max_payload_kg, fuel_type, km_total, status)

Materializa como table en el esquema SILVER.
```


### Modelo Silver: STG_ROUTE_EVENTS

```
Crea un modelo dbt models/staging/stg_route_events.sql que haga flatten del JSON
de la tabla RAW_ROUTE_EVENTS (source Bronze). Extrae todos los campos relevantes del JSON:
eventos de ruta (event_id, vehicle_id, shipment_id, event_type, location, gps_lat, gps_lon, speed_kmh)

Materializa como table en el esquema SILVER.
```


### Modelo Silver: STG_DELIVERY_TICKETS

```
Crea un modelo dbt models/staging/stg_delivery_tickets.sql que haga flatten del JSON
de la tabla RAW_DELIVERY_TICKETS (source Bronze). Extrae todos los campos relevantes del JSON:
tickets de incidencias (ticket_id, shipment_id, vehicle_id, category, priority, description, sla_breached, estimated_cost_eur)

Materializa como table en el esquema SILVER.
```


### Modelo Silver: STG_SHIPMENTS

```
Crea un modelo dbt models/staging/stg_shipments.sql que haga flatten del JSON
de la tabla RAW_SHIPMENTS (source Bronze). Extrae todos los campos relevantes del JSON:
envios (shipment_id, client_name, origin, destination, service_type, amount_eur, weight_kg, status)

Materializa como table en el esquema SILVER.
```



### Ejecutar y validar Silver

```
Ejecuta dbt run --select staging y luego dbt test.
```

> **Checkpoint:** 4 tablas en LOGISTICS_IMMERSION.SILVER con datos tipados.


### Modelo Gold: VEHICLE_360

```
Crea un modelo dbt models/marts/vehicle_360.sql:
Vista 360 de vehiculo: datos basicos + total_shipments + total_route_events + incidents_count + total_km_last_90d + total_revenue_eur + avg_delivery_time_hours + days_until_maintenance. JOIN de todas las Silver.
Materializa como table en el esquema GOLD.
```


### Modelo Gold: ROUTE_HEALTH

```
Crea un modelo dbt models/marts/route_health.sql:
Salud de rutas por origin, destination y dia: total_shipments, total_events, delay_events, avg_delivery_hours, on_time_rate, total_revenue_eur, distinct_vehicles.
Materializa como table en el esquema GOLD.
```


### Modelo Gold: DELIVERY_RISK

```
Crea un modelo dbt models/marts/delivery_risk.sql:
Riesgo de incidencias por vehiculo/ruta: pending_tickets + sla_breaches_last_90d + delay_count + damage_claims + km_since_maintenance + delivery_risk_score (0-100).
Materializa como table en el esquema GOLD.
```



### Ejecutar y validar Gold

```
Ejecuta dbt run y dbt test. Muestra conteo de filas de las tablas Gold.
```

> **Checkpoint:** 3 tablas en LOGISTICS_IMMERSION.GOLD listas para AI.

---

## FASE 3: AI — 3 Casos de Uso con Cortex (20 minutos)

> **Objetivo:** Implementar 3 casos de uso AI usando SNOWFLAKE.CORTEX.COMPLETE sobre Silver y Gold.


### Caso de Uso 1: Clasificacion de incidencias de entrega

> **Funcion Cortex:** SNOWFLAKE.CORTEX.COMPLETE

```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
para clasificacion de incidencias de entrega de la tabla LOGISTICS_IMMERSION.SILVER.STG_DELIVERY_TICKETS.

El LLM debe analizar el campo "description" y devolver un JSON con: tipo_incidencia (retraso|dano|direccion|vehiculo|documentacion|rechazo|temperatura|peso), urgencia (critica|alta|media|baja), departamento (operaciones|flota|calidad|atencion_cliente|legal), impacto_servicio (alto|medio|bajo), resumen_corto (max 15 palabras)

El prompt del LLM debe decir:
"Eres un sistema de clasificacion de incidencias de una empresa de transporte y logistica."

Crea una vista en el esquema LOGISTICS_IMMERSION.AI llamada VW_DELIVERY_CLASSIFICATION.
Limita a 50 registros.
```


### Caso de Uso 2: Resumen ejecutivo de vehiculo/flota

> **Funcion Cortex:** SNOWFLAKE.CORTEX.COMPLETE

```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
para resumen ejecutivo de vehiculo/flota de la tabla LOGISTICS_IMMERSION.GOLD.VEHICLE_360.

El LLM debe analizar los datos del registro y devolver un resumen ejecutivo en espanol (3-4 frases) usando estos datos: plate_number, type, base_location, km_total, total_shipments, incidents_count, total_revenue_eur, days_until_maintenance

El prompt del LLM debe decir:
"Eres un gestor de flota de transporte. Genera un resumen ejecutivo en espanol (3-4 frases) del estado de este vehiculo."

Crea una vista en el esquema LOGISTICS_IMMERSION.AI llamada VW_FLEET_EXECUTIVE_SUMMARY.
Limita a 20 registros.
```


### Caso de Uso 3: Recomendacion de optimizacion de ruta

> **Funcion Cortex:** SNOWFLAKE.CORTEX.COMPLETE

```
Escribe una query SQL que use SNOWFLAKE.CORTEX.COMPLETE con el modelo 'llama3.1-70b'
para recomendacion de optimizacion de ruta de la tabla LOGISTICS_IMMERSION.GOLD.DELIVERY_RISK.

El LLM debe analizar los datos del registro y devolver un JSON con: nivel_riesgo (alto|medio|bajo), accion_principal (cambio_ruta|mantenimiento_vehiculo|refuerzo_capacidad|formacion_conductor), mejora_estimada_pct, plazo_implementacion_dias, impacto_coste_eur, recomendacion_detallada

El prompt del LLM debe decir:
"Eres un experto en optimizacion logistica. Analiza esta ruta/vehiculo y devuelve SOLO un JSON valido."

Crea una vista en el esquema LOGISTICS_IMMERSION.AI llamada VW_ROUTE_OPTIMIZATION.
Limita a 25 registros con filtro delivery_risk_score >= 30.
```



### Validar los 3 casos AI

```
Ejecuta las 3 vistas AI y muestra 3 filas de ejemplo de cada una:
- SELECT * FROM LOGISTICS_IMMERSION.AI.VW_DELIVERY_CLASSIFICATION LIMIT 3;
- SELECT * FROM LOGISTICS_IMMERSION.AI.VW_FLEET_EXECUTIVE_SUMMARY LIMIT 3;
- SELECT * FROM LOGISTICS_IMMERSION.AI.VW_ROUTE_OPTIMIZATION LIMIT 3;
```

> **Checkpoint:** 3 vistas AI en LOGISTICS_COPILOT.AI con resultados coherentes.

---

## FASE 4: SEMANTIC VIEW — Vista Semantica para NL2SQL (5 minutos)

> **Objetivo:** Crear una Semantic View sobre las tablas Gold y Silver para consultas en lenguaje natural.

```
Crea una Semantic View en LOGISTICS_IMMERSION.GOLD llamada SV_LOGISTICS_COPILOT
que conecte las tablas Silver y Gold para consultas en lenguaje natural.

Tablas logicas:
- stg_fleet: LOGISTICS_IMMERSION.SILVER.STG_FLEET (PK: vehicle_id)
- stg_route_events: LOGISTICS_IMMERSION.SILVER.STG_ROUTE_EVENTS (PK: event_id)
- stg_delivery_tickets: LOGISTICS_IMMERSION.SILVER.STG_DELIVERY_TICKETS (PK: ticket_id)
- stg_shipments: LOGISTICS_IMMERSION.SILVER.STG_SHIPMENTS (PK: shipment_id)
- vehicle_360: LOGISTICS_IMMERSION.GOLD.VEHICLE_360
- route_health: LOGISTICS_IMMERSION.GOLD.ROUTE_HEALTH
- delivery_risk: LOGISTICS_IMMERSION.GOLD.DELIVERY_RISK

Relaciones: todas las tablas se relacionan por vehicle_id con la tabla principal.

Anade dimensiones con comentarios en espanol, metricas con sinonimos,
e instrucciones AI_SQL_GENERATION explicando que es una empresa de transporte y logistica en Espana.
```

> **Checkpoint:** Semantic View creada. Prueba con queries tipo:
> `SELECT * FROM SEMANTIC VIEW (LOGISTICS_IMMERSION.GOLD.SV_LOGISTICS_COPILOT DIMENSIONS (...) METRICS (...));`

---

## FASE 5: CORTEX AGENT — Agente para Snowflake Intelligence (5 minutos)

> **Objetivo:** Crear un Cortex Agent conectado a la Semantic View.

```sql
CREATE SCHEMA IF NOT EXISTS LOGISTICS_IMMERSION.AGENTS;

CREATE OR REPLACE AGENT LOGISTICS_IMMERSION.AGENTS.LOGISTICS_COPILOT_AGENT
  COMMENT = 'Agente de transporte, logistica y travel para consultas en lenguaje natural'
  PROFILE = '{"display_name": "Logistics & Fleet Copilot"}'
  FROM SPECIFICATION $$
  {
    "models": {
      "orchestration": "claude-4-sonnet"
    },
    "instructions": {
      "orchestration": "Eres un asistente de operaciones de una empresa de transporte y logistica en Espana. Usa la herramienta de analisis para responder preguntas sobre los datos. Responde siempre en espanol.",
      "response": "Responde de forma clara y concisa en espanol. Usa formato tabla cuando sea apropiado."
    },
    "tools": [
      {
        "tool_spec": {
          "type": "cortex_analyst_text_to_sql",
          "name": "logistics_analyst",
          "description": "Consulta datos operativos: Los datos cubren vehiculos de flota, eventos de ruta, tickets de incidencias de entrega y envios. Empresa de transporte/logistica en Espana."
        }
      }
    ],
    "tool_resources": {
      "logistics_analyst": {
        "semantic_view": "LOGISTICS_IMMERSION.GOLD.SV_LOGISTICS_COPILOT",
        "execution_environment": {
          "type": "warehouse",
          "warehouse": "COMPUTE_WH"
        },
        "query_timeout": 60
      }
    }
  }
  $$;

GRANT USAGE ON AGENT LOGISTICS_IMMERSION.AGENTS.LOGISTICS_COPILOT_AGENT TO ROLE PUBLIC;
GRANT SELECT ON SEMANTIC VIEW LOGISTICS_IMMERSION.GOLD.SV_LOGISTICS_COPILOT TO ROLE PUBLIC;
```

### Probar el agente

Ve a **Snowsight > AI & ML > Snowflake Intelligence** y busca **"Logistics & Fleet Copilot"**.

> **Checkpoint:** El agente responde preguntas en espanol con datos reales.

---

## BONUS: Streamlit App (si os sobra tiempo)

```
Crea una aplicacion Streamlit en Snowflake que se conecte a LOGISTICS_COPILOT y muestre:
1. KPIs principales con st.metric
2. Tabla interactiva de VEHICLE_360 con filtros
3. Para cada registro seleccionado, mostrar el resumen AI
4. Grafico con los top 10 por risk/score
```

---

## Tips para la Demo (5 minutos)

| Minuto | Contenido |
|--------|-----------|
| 0:00 - 0:30 | Presentar al equipo y enfoque |
| 0:30 - 1:00 | Pipeline Bronze > Silver > Gold |
| 1:00 - 2:30 | Demo 3 casos AI en vivo |
| 2:30 - 3:30 | Agente en Snowflake Intelligence |
| 3:30 - 4:30 | Streamlit o valor de negocio |
| 4:30 - 5:00 | Conclusiones |

---

## FASE FINAL: REVISION DE COSTES DEL HACKATHON

```sql
SELECT service_type, ROUND(SUM(credits_used), 2) AS total_credits
FROM SNOWFLAKE.ACCOUNT_USAGE.METERING_HISTORY
WHERE start_time >= '2026-05-28' AND start_time < '2026-05-29'
GROUP BY service_type ORDER BY total_credits DESC;

SELECT function_name, model_name, SUM(tokens) AS total_tokens, ROUND(SUM(token_credits), 4) AS total_credits
FROM SNOWFLAKE.ACCOUNT_USAGE.CORTEX_FUNCTIONS_USAGE_HISTORY
WHERE start_time >= '2026-05-28' AND start_time < '2026-05-29'
GROUP BY function_name, model_name ORDER BY total_credits DESC;

SELECT username, ROUND(SUM(credits), 4) AS total_credits, SUM(request_count) AS total_requests
FROM SNOWFLAKE.ACCOUNT_USAGE.CORTEX_ANALYST_USAGE_HISTORY
WHERE start_time >= '2026-05-28' AND start_time < '2026-05-29'
GROUP BY username ORDER BY total_credits DESC;

SELECT warehouse_name, ROUND(SUM(credits_used), 2) AS total_credits, ROUND(SUM(credits_used) * 3.00, 2) AS estimated_cost_usd
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE start_time >= '2026-05-28' AND start_time < '2026-05-29'
GROUP BY warehouse_name ORDER BY total_credits DESC;
```

> **Nota:** Las vistas de ACCOUNT_USAGE tienen un retraso de hasta 2 horas.
