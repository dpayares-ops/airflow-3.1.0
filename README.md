# Flujo de Airflow 3.10

Este documento describe el flujo de trabajo y la arquitectura de **Apache Airflow 3.10** en nuestro entorno.

---

## 1. Arquitectura General

Airflow 3.10 está basado en una arquitectura **modular y escalable**, utilizando principalmente los siguientes componentes:

| Componente         | Descripción                                                                 |
|-------------------|-----------------------------------------------------------------------------|
| **Scheduler**      | Monitorea los DAGs y planifica las tareas pendientes en la base de datos.  |
| **Worker**         | Ejecuta las tareas en paralelo según lo programado por el Scheduler.       |
| **API Server**     | Exposición RESTful para integración con otros servicios y ejecución remota de DAGs. |
| **Dag Processor**  | Procesa los DAGs, analiza los archivos Python y los registra en la DB.    |
| **Triggerer**      | Maneja triggers asíncronos como sensores y tareas dependientes de eventos.|
| **Database**       | Almacena el estado de las tareas, DAGs, conexiones y variables.           |
| **Webserver**      | UI de Airflow para monitoreo, administración y ejecución de DAGs.         |

---

## 2. Flujo de ejecución

```
flowchart TD
    A[DAG Folder] --> B[Dag Processor]
    B --> C[Database]
    C --> D[Scheduler]
    D --> E[Worker]
    E --> C
    D --> F[Triggerer]
    F --> E
    G[API Server] --> H[Worker / Scheduler]
    I[Webserver] --> C


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
                    ┌──────────────────────────────┐
                    │  API SERVER (Deployment)     │
                    │  airflow-api-server          │
                    │  [green] envFrom:            │
                    │   - airflow-config (ConfigMap)│
                    │   - airflow-secrets (Secret) │
                    └─────────────┬────────────────┘
                                  │
            ┌─────────────────────┴─────────────────────┐
            │                                           │
 ┌──────────▼──────────┐                    ┌───────────▼───────────┐
 │ Scheduler (Deployment)│                  │ DAG Processor (Deployment) │
 │ airflow-scheduler     │                  │ airflow-dag-processor      │
 │ [green] envFrom:      │                  │ [green] envFrom:          │
 │  - airflow-config     │                  │  - airflow-config         │
 │  - airflow-secrets    │                  │  - airflow-secrets        │
 └──────────┬───────────┘                  └───────────┬────────────┘
            │                                            │
            │                                            │
     ┌──────▼───────┐                           ┌────────▼────────┐
     │  Worker      │                           │ Logs/DAG PVC    │
     │ StatefulSet  │                           │ (PersistentVol) │
     │ airflow-worker│                           │ [purple]        │
     │ [blue] envFrom:│                          │                │
     │ - airflow-config│                          └────────────────┘
     │ - airflow-secrets│
     │ REDIS_PASSWORD (Secret) │
     │ AIRFLOW__CELERY__BROKER_URL │
     │ AIRFLOW__CELERY__RESULT_BACKEND │
     └──────┬───────┘
            │
     ┌──────▼────────┐
     │ Triggerer      │
     │ StatefulSet    │
     │ airflow-triggerer │
     │ [blue] envFrom:   │
     │ - airflow-config  │
     │ - airflow-secrets │
     └──────┬───────────┘
            │
     ┌──────▼───────────┐
     │ Redis Broker     │
     │ StatefulSet      │
     │ airflow-redis-0  │
     │ [blue] Password: airflow-redis-password (Secret) │
     └──────┬───────────┘
            │
     ┌──────▼───────────┐
     │ PostgreSQL       │
     │ StatefulSet      │
     │ airflow-postgresql-0 │
     │ [blue] Username/Password: airflow-secrets │
     │ [purple] PVC: postgresql-data            │
     └───────────────────┘

 ---               
