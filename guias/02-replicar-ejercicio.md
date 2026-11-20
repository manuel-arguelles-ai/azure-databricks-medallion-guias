# Guía 2 — Replicar el ejercicio de punta a punta

Prerrequisito: haber completado la [Guía 1](01-aprovisionar-azure.md). Resultado: un lakehouse Bronze → Silver → Gold con dashboard AI/BI y espacio Genie, y cifras que puedes comparar con las del ejercicio original. Tiempo: ~2 h (la mayor parte es esperar corridas serverless).

Repo del ejercicio: https://github.com/manuel-arguelles-ai/fmcg-medallion-lakehouse-databricks

## 0. Entorno
```bash
git clone https://github.com/manuel-arguelles-ai/fmcg-medallion-lakehouse-databricks && cd fmcg-medallion-lakehouse-databricks
python3 -m venv .venv && .venv/bin/pip install -r requirements.txt
export DATABRICKS_HOST=https://<workspace-url> DATABRICKS_AUTH_TYPE=azure-cli DATABRICKS_BUNDLE_ENGINE=direct
export BUNDLE_VAR_warehouse_id=<warehouse-id> DATABRICKS_WAREHOUSE_ID=<warehouse-id>
.venv/bin/pytest -q          # señal de éxito: todas las pruebas pasan (no necesitan Spark)
```
Solo si cambiaste el nombre del catálogo, pasa `--var catalog=<nombre>` al bundle.

## 1. Generar los datos sintéticos
```bash
(cd src && python -m datagen --out ../data/generated)      # --small para una prueba rápida de 3 meses
```
Es determinista (semilla 42). Señal de éxito: `data/generated/MANIFEST.json` y **1.606 archivos CSV**, ~1.005.642 líneas de pedido.

## 2. Subir a landing
```bash
bash infra/03-upload-landing.sh --dry-run
bash infra/03-upload-landing.sh
az storage blob list --account-name $SA -c landing --auth-mode login --query "length(@)"   # verifica el conteo aparte
```
Señal de éxito: 1.606 blobs (el progreso del upload usa `\r`; no te fíes de él para contar).

## 3. DDL de Silver y despliegue del bundle
```bash
python3 infra/run_sql.py src/silver/sql/01_silver_ddl.sql --warehouse $DATABRICKS_WAREHOUSE_ID
databricks bundle validate -t dev
databricks bundle deploy -t dev
```
Señal de éxito: `Validation OK!` y `Deployment complete!`; en el workspace aparecen los jobs `fmcg-bronze-ingest`, `fmcg-silver-transform`, `fmcg-gold-build` y `fmcg-medallion-pipeline`.

## 4. Primera corrida completa
```bash
databricks bundle run medallion_pipeline -t dev --params mode=full      # ~10 min en serverless
```
Encadena Bronze → Silver → Gold → reconciliación (que falla si las capas no cuadran).

### Cifras de referencia (corrida `full`, antes del lote incremental)
| Capa | Qué comprobar | Valor esperado |
|---|---|---|
| Bronze | tablas que reconcilian con el MANIFEST | 8 de 8, 0 filas rescatadas |
| Silver | pedidos madre / hija | 699.699 / 296.754 |
| Silver | cuarentena de pedidos (hija) / de precios | 4.633 / 24 |
| Gold | `fact_sales` | 996.453 filas |
| Gold | ingreso total | 37.450.346,40 PEN |

Si algo no coincide, mira `ops.pipeline_runs`, `ops.dq_results` y `ops.gold_runs` (conteos por capa y por regla) antes de tocar código.

## 5. Verificación independiente de Gold
```bash
.venv/bin/python scripts/verify_gold_kpis.py
```
Recalcula los KPI en Python puro desde los CSV y los compara con Gold. Señal de éxito: `VEREDICTO: Gold coincide con el calculo independiente` y código de salida 0.

## 6. Lote incremental
```bash
(cd src && python -m datagen.increment --root ../data/generated)   # escribe UN archivo nuevo y reproducible (25 filas de la hija) en hija/orders/ingest_date=<fecha nueva>/
```
Sube el archivo **nuevo** a landing (`az storage blob upload ... --overwrite false`, nunca sobrescribas), y corre:
```bash
databricks bundle run medallion_pipeline -t dev --params mode=incremental
```
Cifras esperadas tras el lote: Bronze hija/orders 305.968; Silver hija 296.776; cuarentena 4.634; `fact_sales` 996.475; ingreso 37.450.922,98 PEN. Una segunda corrida incremental sin lote nuevo **no debe cambiar nada**.

## 7. Dashboard y Genie
```bash
python3 infra/03-deploy-dashboard.py --warehouse $DATABRICKS_WAREHOUSE_ID --verify
python3 infra/04-deploy-genie.py --warehouse $DATABRICKS_WAREHOUSE_ID --ask "¿Cuáles son los ingresos totales?"
```
Señal de éxito: los 12 datasets del dashboard ejecutan; Genie responde con el mismo total que una consulta SQL directa a `fmcg_lakehouse.gold.fact_sales`.

## 8. Qué entregar / checklist de réplica
- [ ] pruebas locales en verde
- [ ] 1.606 archivos en landing
- [ ] 4 jobs desplegados y corrida `full` en SUCCESS
- [ ] cifras de referencia coinciden
- [ ] `verify_gold_kpis.py` con veredicto OK
- [ ] lote incremental + corrida repetida sin cambios
- [ ] dashboard publicado y Genie respondiendo
- [ ] captura del dashboard, de la corrida y de las tablas Gold

## 9. Trampas conocidas (cuestan tiempo)
- `databricks bundle deploy` sin `DATABRICKS_BUNDLE_ENGINE=direct` falla por clave GPG expirada de Terraform (CLI 0.288).
- En serverless **no existe `__file__`** en tareas Python: usa `inspect.currentframe().f_code.co_filename`.
- `F.try_to_date` no existe en el PySpark serverless (entorno 3): `F.call_function("try_to_date", ...)`.
- Las tablas Bronze tienen columnas distintas: no se pueden `UNION` entre sí.
- No hay Spark local: lo que no se puede probar en `pytest` se valida en el workspace.
- Si recreas el checkpoint de un stream de Auto Loader, sube su `--generation`.
- Landing es inmutable: un archivo = una carga; Auto Loader no recarga archivos ya vistos.
