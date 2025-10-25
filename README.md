# airflow-3.1.0
# Airflow 3.1.0 Kubernetes Architecture
Diagrama conceptual del stack desplegado en el namespace `airflow`.
---
Namespace: airflow
                     ┌──────────────────────┐
                     │  Airflow API Server  │
                     │  (UI + REST API)     │
                     │  airflow-api-server  │
                     └─────────┬──────────┘
                               │
                ┌──────────────┴──────────────┐
                │                             │
      ┌─────────▼─────────┐         ┌─────────▼─────────┐
      │  Airflow Scheduler │         │  DAG Processor     │
      │  airflow-scheduler │         │  airflow-dag-processor │
      └─────────┬─────────┘         └─────────┬─────────┘
                │                             │
                │                             │
          ┌─────▼─────┐                 ┌─────▼─────┐
          │ Celery    │                 │ Logs /    │
          │ Workers   │                 │ DAGs PVC  │
          │ airflow-  │                 │ (Shared)  │
          │ worker-0  │                 └───────────┘
          └─────┬─────┘
                │
                │
       ┌────────▼────────┐
       │  Triggerer       │
       │ airflow-triggerer│
       └────────┬────────┘
                │
        ┌───────▼────────┐
        │ Redis Broker    │
        │ airflow-redis-0 │
        └────────┬────────┘
                 │
          ┌──────▼───────┐
          │ PostgreSQL    │
          │ airflow-postgresql-0 │
          └──────────────┘


---

## 🔹 Leyenda

| Símbolo | Representa |
|---------|------------|
| Deployment (verde) | Pods gestionados por Deployment (Scheduler, API Server, DAG Processor) |
| StatefulSet (azul) | Pods con estado persistente (Worker, Triggerer, Redis, PostgreSQL) |
| Secret (amarillo) | Claves sensibles: `airflow-secrets`, `airflow-redis-password` |
| ConfigMap (naranja) | Configuración de Airflow: `airflow-config` |
| PVC (morado) | Volúmenes persistentes compartidos: `/opt/airflow/dags`, `/opt/airflow/logs`, `postgresql-data` |

---

## 🔹 Flujo de tareas

1. **Scheduler → Worker:** envía tareas a través de **Redis**.  
2. **Worker:** ejecuta las tareas, escribe logs en **PVC** y resultados en **PostgreSQL**.  
3. **Triggerer:** maneja sensores/triggers asíncronos usando **Redis** y **PostgreSQL**.  
4. **API Server/UI:** se conecta a **PostgreSQL** para estado de DAGs y tareas, permite loguearse.  
5. **DAG Processor:** sincroniza DAGs desde **PVC** para todos los pods.

---

                
