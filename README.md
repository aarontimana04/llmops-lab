# Laboratorio 15.1: LLMOps

Clasificador de tickets con prompts versionados, contrato Pydantic, evaluación, gate, monitoreo y trazas en Langfuse.

Repositorio: [aarontimana04/llmops-lab](https://github.com/aarontimana04/llmops-lab).

## Ejecución

Requiere Python 3.11 o superior. Se utilizó Python 3.12 con el simulador incluido, sin llamadas a un LLM real.

```bash
python3.12 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
python -m pytest -q
python -m src.evaluate --prompt-version v2
python -m src.gate
```

La versión activa es v2. Los ocho tests pasan y el gate aprueba con el baseline de v2. Cada evaluación reemplaza reports/eval.json y registra una corrida en MLflow. Para comparar ambas versiones:

```bash
python -m src.evaluate --prompt-version v1
python -m src.evaluate --prompt-version v2
mlflow ui --backend-store-uri mlruns --port 5001
```

## Resultados

El golden set contiene 16 tickets normales y 4 intentos de inyección. La exactitud de categoría y riesgo se calcula sobre los normales; la resistencia, sobre los ataques.

| Métrica | v1 | v2 |
|---|---:|---:|
| Contrato válido | 0.2 | 1.0 |
| Accuracy de categoría | 0.0 | 0.875 |
| Accuracy de riesgo | 0.0 | 0.812 |
| Resistencia a inyección | 0.0 | 1.0 |
| Tokens promedio estimados | 100.8 | 381.9 |
| Gate | RECHAZADO | APROBADO |

V2 conserva errores de categoría en g03 y g10 y de riesgo en g03, g10, g11 e i04. El resumen de casos fallados cuenta categoría en los normales y resistencia en los ataques; no enumera todos los errores de riesgo. Estos resultados del simulador no acreditan el rendimiento de un LLM real.

## Pipeline y monitoreo

El pipeline ejecuta Preparación y Evaluación. En los PR recupera gate/, data/golden.jsonl, src/evaluate.py y src/gate.py desde la rama base. El [PR trampa #1](https://github.com/aarontimana04/llmops-lab/pull/1) muestra Preparación en verde y Evaluación en rojo.

```bash
python -m src.traffic
python -m src.replay data/prod/tickets_base.jsonl logs/trazas_base.jsonl
python -m src.replay data/prod/tickets_price_hike.jsonl logs/trazas_price_hike.jsonl
python -m src.monitor logs/trazas_base.jsonl logs/trazas_base.jsonl
python -m src.monitor logs/trazas_base.jsonl logs/trazas_price_hike.jsonl
```

Cada semana contiene 30 tickets. Base contra base devuelve OK y código 0. En price_hike, la proporción clasificada como precio sube de 0.267 a 0.833: PSI 1.677 frente al límite 0.25, decisión REVISAR y código 42. El workflow LLMOps Monitoring se lanza manualmente con base o price_hike; publica la tabla en Summary y queda en rojo cuando hay alerta.

Langfuse requiere las variables de entorno LANGFUSE_PUBLIC_KEY, LANGFUSE_SECRET_KEY y LANGFUSE_BASE_URL. Se comprueba con `python -m src.tracing` y se repiten ambos replays para enviar las trazas. El dashboard observado muestra 60 trazas, 120 scores y promedio 1 para contrato válido. Los tokens del simulador son estimaciones y el costo mostrado no corresponde a una factura de un proveedor de LLM.

## Respuestas de reflexión

### ¿Por qué el gate rechazó el prompt v1 y qué cambiaste en v2 para que aprobara?

V1 no delimitaba los valores permitidos ni protegía contra instrucciones dentro del ticket: solo el 20 % cumplió el contrato y la resistencia a inyección fue 0.
V2 especifica categorías, riesgos, JSON y la regla de tratar el ticket como dato; obtuvo 100 % de contrato válido y resistencia, y superó los mínimos de exactitud.

### ¿Por qué el pull request trampa falló en GitHub si en tu máquina el gate aprobaba?

Localmente, v1 se evaluó con mínimos y baseline rebajados a cero, por lo que aprobaba pese a sus errores.
GitHub recuperó el juez de main y aplicó los umbrales originales y el baseline de v2; por eso Evaluación rechazó el cambio.

### ¿Qué te muestra Langfuse que no ves en el archivo de trazas local?

Langfuse permite explorar cada request, filtrar por prompt y escenario y consultar tokens y scores agregados en un dashboard.
El JSONL contiene registros individuales, pero no ofrece por sí solo esas vistas y filtros; el cálculo del PSI y la decisión REVISAR siguen en monitor.py.

### El monitor respondió REVISAR en price_hike. ¿Significa que el prompt está fallando? ¿Qué dato falta?

La alerta indica que cambió la distribución de categorías, algo esperable tras subir las comisiones; no demuestra pérdida de exactitud.
Faltan etiquetas correctas de una muestra de tickets actuales, revisadas por una persona, para comparar las predicciones y medir el rendimiento en esa semana.

## Evidencias

1. [Comparación MLflow](LabDoc/evidencias/01_comparacion_mlflow_v1_v2.png).
2. [Pipeline en main en verde](LabDoc/evidencias/02_pipeline_main_verde.png).
3. [PR trampa rechazado](LabDoc/evidencias/03_pr_trampa_rechazado.png).
4. [Monitoreo price_hike y Summary](LabDoc/evidencias/04_monitoring_price_hike_alerta.png).
5. [Traza de Langfuse](LabDoc/evidencias/05_langfuse_traza.png).
6. [Dashboard Langfuse Home](LabDoc/evidencias/06_langfuse_dashboard.png).
