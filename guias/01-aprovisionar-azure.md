# Guía 1 — Aprovisionar en Azure todo lo necesario

Objetivo: dejar listo el entorno donde correrá el ejercicio [FMCG Medallion Lakehouse](https://github.com/manuel-arguelles-ai/fmcg-medallion-lakehouse-databricks): grupo de recursos, ADLS Gen2, Access Connector, workspace Databricks premium y los objetos de Unity Catalog. Tiempo: ~30 min. Costo: sin NAT gateway ni cómputo permanente; ver [Costos y limpieza](#7-costos-y-limpieza).

> **Regla de oro:** el login (`az login`) lo hace una persona en su terminal. Un asistente (Copilot, Claude) **nunca** inicia sesión por ti ni imprime tokens. Antes de cualquier `create`, `update` o `delete`, revisa el comando exacto.

## 0. Requisitos
| Qué | Detalle |
|---|---|
| Suscripción Azure | con permiso de **Owner**, o Contributor + User Access Administrator (hay que crear una asignación de rol) |
| Herramientas | Azure CLI, Databricks CLI ≥ 0.288, Python 3.11+ (`az --version`, `databricks --version`) |
| Resource providers | `Microsoft.Databricks`, `Microsoft.Storage` registrados (`az provider register -n Microsoft.Databricks`) |
| Unity Catalog | quien ejecute el paso 5 debe ser **metastore admin** o tener `CREATE STORAGE CREDENTIAL`, `CREATE EXTERNAL LOCATION` y `CREATE CATALOG` |

## 1. Variables y convención de nombres
Los nombres de storage son únicos a nivel global (3–24 caracteres, minúsculas y números). Cambia el sufijo `001` o añade tus iniciales si ya está tomado.

```bash
export RG=rg-fmcg-lakehouse-dev-cus-001
export LOC=centralus
export SA=stfmcglakehousedevcus001          # único global
export AC=ac-fmcg-lakehouse-dev-cus-001
export DBW=dbw-fmcg-lakehouse-dev-cus-001
export TAGS="project=fmcg-lakehouse env=dev"
```
Señal de éxito: `az account show --query "{sub:name,user:user.name}" -o table` muestra tu suscripción y tu usuario.

## 2. Grupo de recursos y ADLS Gen2
```bash
az group create -n $RG -l $LOC --tags $TAGS -o none
az storage account create -n $SA -g $RG -l $LOC --sku Standard_LRS --kind StorageV2 --hns true \
  --min-tls-version TLS1_2 --allow-blob-public-access false --tags $TAGS -o none
for c in landing bronze silver gold uc-managed; do
  az storage container create --account-name $SA -n $c --auth-mode login -o none
done
```
`--hns true` activa el namespace jerárquico (ADLS Gen2). Tu usuario necesita el rol *Storage Blob Data Contributor* sobre la cuenta para usar `--auth-mode login`.
Señal de éxito: `az storage container list --account-name $SA --auth-mode login -o table` lista los 5 contenedores.

## 3. Access Connector con identidad administrada
Unity Catalog accede al storage con esta identidad, sin claves.
```bash
az databricks access-connector create -g $RG -n $AC -l $LOC --identity-type SystemAssigned --tags $TAGS -o none
PRINCIPAL=$(az databricks access-connector show -g $RG -n $AC --query identity.principalId -o tsv)
az role assignment create --assignee-object-id "$PRINCIPAL" --assignee-principal-type ServicePrincipal \
  --role "Storage Blob Data Contributor" --scope "$(az storage account show -n $SA -g $RG --query id -o tsv)" -o none
```
Señal de éxito: `az role assignment list --assignee $PRINCIPAL --all -o table` muestra *Storage Blob Data Contributor*.

## 4. Workspace de Azure Databricks (premium)
```bash
az databricks workspace create -g $RG -n $DBW -l $LOC --sku premium --enable-no-public-ip false --tags $TAGS -o none
az databricks workspace show -g $RG -n $DBW --query "{url:workspaceUrl,state:provisioningState}" -o table
```
- `premium` es obligatorio para Unity Catalog.
- `--enable-no-public-ip false`: los nodos usan IP pública y **no se crea NAT gateway** (evita un cargo fijo por hora). Para producción se recomienda lo contrario.
- Señal de éxito: `provisioningState = Succeeded` y una URL `adb-<id>.<n>.azuredatabricks.net`.

```bash
export DATABRICKS_HOST=https://<workspace-url> DATABRICKS_AUTH_TYPE=azure-cli
databricks current-user me      # confirma que el CLI se autentica con tu sesión de az
```

## 5. Unity Catalog
Importante: el metastore regional puede estar **compartido** con otros trabajos. Crea solo objetos con prefijo propio (`cred_fmcg_*`, `loc_fmcg_*`, `fmcg_lakehouse`) y no toques catálogos ajenos.

1. **Credencial de almacenamiento** (no se crea por SQL):
   ```bash
   ACID=$(az databricks access-connector show -g $RG -n $AC --query id -o tsv)
   databricks storage-credentials create --json "{\"name\": \"cred_fmcg_lakehouse\",
     \"azure_managed_identity\": {\"access_connector_id\": \"$ACID\"}}"
   databricks storage-credentials get cred_fmcg_lakehouse
   ```
   Trampa: con `--json` el `name` va **dentro** del JSON, no como argumento.
2. **Warehouse SQL:** en el workspace, crea (o usa) un SQL Warehouse serverless con auto-stop de 10 min y anota su id (`databricks warehouses list`).
3. **Locations, catálogo, esquemas, volumes y tablas `ops`:** ejecuta el SQL del repo del ejercicio.
   ```bash
   git clone https://github.com/manuel-arguelles-ai/fmcg-medallion-lakehouse-databricks && cd fmcg-medallion-lakehouse-databricks
   python3 infra/run_sql.py infra/02-unity-catalog.sql --warehouse <warehouse-id> --dry-run   # revisa
   python3 infra/run_sql.py infra/02-unity-catalog.sql --warehouse <warehouse-id>
   python3 infra/run_sql.py infra/02b-seed-source-registry.sql --warehouse <warehouse-id>
   ```
   Si cambiaste los nombres del paso 1, ajusta `infra/02-unity-catalog.sql` (cuenta de storage en las `abfss://...`) antes de ejecutarlo.

Señal de éxito (SQL): `SHOW EXTERNAL LOCATIONS` lista 5 `loc_fmcg_*`; `SHOW SCHEMAS IN fmcg_lakehouse` muestra bronze, silver, gold, quarantine, ops; `SELECT count(*) FROM fmcg_lakehouse.ops.source_registry` = 8.

## 6. Lista de verificación final
- [ ] RG, storage (5 contenedores), Access Connector con rol, workspace premium en `Succeeded`
- [ ] `databricks current-user me` responde
- [ ] credencial y 5 external locations creadas
- [ ] catálogo `fmcg_lakehouse` con 5 esquemas y 2 volumes (`bronze.landing`, `ops.checkpoints`)
- [ ] `ops.source_registry` con 8 filas
- [ ] id del warehouse anotado

## 7. Costos y limpieza
Sin NAT gateway, cómputo solo serverless y bajo demanda, warehouse con auto-stop. Para borrar todo lo creado en la guía:
```bash
az group delete --name $RG --yes --no-wait     # el catálogo y la credencial de UC se borran aparte en el workspace
```

## 8. Problemas frecuentes
| Síntoma | Causa y arreglo |
|---|---|
| `StorageAccountAlreadyTaken` | el nombre es global: cambia `$SA` |
| `AuthorizationFailed` en `role assignment create` | falta rol Owner o User Access Administrator |
| `bundle deploy` falla con *openpgp: key expired* | usa `DATABRICKS_BUNDLE_ENGINE=direct` (se usa en la Guía 2) |
| `PERMISSION_DENIED` creando credencial o locations | tu usuario no es metastore admin: pide el privilegio |
| `az storage ... --auth-mode login` da 403 | asígnate *Storage Blob Data Contributor* sobre la cuenta |
