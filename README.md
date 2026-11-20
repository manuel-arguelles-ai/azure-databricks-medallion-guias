# Azure Databricks Medallion — guías de aprovisionamiento y réplica

Dos guías paso a paso para montar y replicar el ejercicio [FMCG Medallion Lakehouse](https://github.com/manuel-arguelles-ai/fmcg-medallion-lakehouse-databricks) (Bronze → Silver → Gold → dashboard AI/BI + Genie) sobre Azure Databricks y Unity Catalog.

| # | Guía | Para qué |
|---|---|---|
| 1 | [Aprovisionar en Azure](guias/01-aprovisionar-azure.md) | Dejar listo el entorno: grupo de recursos, ADLS Gen2, Access Connector, workspace premium, Unity Catalog |
| 2 | [Replicar el ejercicio](guias/02-replicar-ejercicio.md) | Generar datos, correr el pipeline, validar con cifras de referencia, lote incremental, dashboard y Genie |

Cada paso trae su **señal de éxito**, y la guía 2 incluye las **cifras de referencia** para comprobar que tu réplica coincide.

## Usarlas con un copiloto
[`prompts/copiloto-claude.md`](prompts/copiloto-claude.md) trae un prompt por guía para GitHub Copilot o Claude, con las reglas del ejercicio: el login lo haces tú, y todo comando que cree, cambie o borre se muestra y se confirma antes de ejecutarse.

## Reglas del ejercicio
- El `az login` lo hace una persona; nadie más inicia sesión ni imprime tokens.
- Comando exacto y confirmación antes de cualquier `create`, `update` o `delete`.
- El metastore de Unity Catalog puede ser compartido: solo se tocan objetos con prefijo propio.

Licencia: MIT.
