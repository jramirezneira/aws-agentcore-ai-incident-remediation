# Remediaciones automáticas soportadas

## Objetivo
Este documento define las operaciones automáticas que puede ejecutar el motor de remediación. El agente únicamente puede construir el `execution_plan` utilizando las estructuras descritas aquí. Si una remediación requiere una operación o parámetro distinto, debe devolver obligatoriamente `execution_mode=MANUAL`.
Los marcadores de posición entre corchetes (ej: `[VARIABLE]`) definen valores dinámicos que el agente debe reemplazar por datos reales extraídos de la evidencia.


# Reglas de Parámetros y Variables Globales
Al construir los comandos en `technical_solutions` o los parámetros en `execution_plan`, debes aplicar estrictamente las siguientes variables de entorno globales del sistema:

## Variables Globales de la Infraestructura
- **Región AWS:** `us-west-2`
- **GLUE_RECOVERY_STATE_MACHINE_ARN:** `arn:aws:states:us-west-2:156581257326:stateMachine:workflow-remedicion-crawler-glue`


---
## Amazon S3

### create_bucket
- **Servicio:** s3
- **Operación:** create_bucket
- **Uso:** Crear el bucket de resultados cuando Athena falle porque el recurso no existe.
- **Estructura obligatoria:** (Es mandatorio incluir `CreateBucketConfiguration` con la región global para evitar fallos de endpoint).
```json
{
  "step": "Crear el bucket S3 [NOMBRE_DEL_BUCKET_A_CREAR]",
  "service": "s3",
  "operation": "create_bucket",
  "parameters": {
    "Bucket": "[NOMBRE_DEL_BUCKET_A_CREAR]",
    "CreateBucketConfiguration": {
      "LocationConstraint": "[REGION_AWS]
    }
  }
}
```





---


## AWS Lambda

### update_function_configuration
- **Servicio:** lambda
- **Operación:** update_function_configuration
- **Uso:** Incrementar dinámicamente recursos de cómputo ante fallos confirmados de rendimiento.
- **Estructura:**
```json
{
  "step": "Actualizar la configuración de la función Lambda [NOMBRE_DE_LA_LAMBDA]",
  "service": "lambda",
  "operation": "update_function_configuration",
  "parameters": {
    "FunctionName": "[NOMBRE_DE_LA_LAMBDA_REAL]"
  }
}
```
- **Regla Estricta de Parámetros:** Añade dentro del objeto `parameters` únicamente la llave necesaria según la evidencia:
  - Si la evidencia reporta **`OutOfMemory`**, añade `"MemorySize": [NUEVO_VALOR_ENTERO]` (ej: `1024`, sin comillas).
  - Si la evidencia reporta **`Task timed out`**, añade `"Timeout": [NUEVO_VALOR_ENTERO]` (ej: `30`, sin comillas).
- **Prohibido Modificar:** Environment, Layers, Runtime, Handler, Role, VPC, Image, Architectures. Cualquier intento devuelve `MANUAL`.

### put_function_concurrency
- **Servicio:** lambda
- **Operación:** put_function_concurrency
- **Uso:** Restaurar la concurrencia reservada cuando haya sido configurada accidentalmente en cero (0).
- **Estructura:**
```json
{
  "step": "Restaurar la concurrencia reservada para la función Lambda [NOMBRE_DE_LA_LAMBDA]",
  "service": "lambda",
  "operation": "put_function_concurrency",
  "parameters": {
    "FunctionName": "[NOMBRE_DE_LA_LAMBDA]",
    "ReservedConcurrentExecutions": [VALOR_NUMERICO_ENTERO]
  }
}
```
---

## AWS Glue — Remediación mediante Step Function
### start_execution — Glue Schema / Data Catalog Recovery

**Step Function autorizada:**
  `arn:aws:states:us-west-2:156581257326:stateMachine:workflow-remedicion-crawler-glue`
- **Uso:** Ejecutar el workflow de recuperación de Glue cuando el error corresponda específicamente a una discrepancia de esquema entre los datos y el Glue Data Catalog.
- El agente debe identificar desde la evidencia el **nombre de la base de datos (`database`)** y la **tabla (`table`)** afectada.
- Si no puede identificarlos de forma inequívoca, debe devolver obligatoriamente `execution_mode=MANUAL`.

### Excepciones que habilitan esta remediación:
- `INSERT_COLUMN_ARITY_MISMATCH.TOO_MANY_DATA_COLUMNS`
- `INSERT_COLUMN_ARITY_MISMATCH.NOT_ENOUGH_DATA_COLUMNS`

- **Estructura:**
```json
{
  "step": "Ejecutar workflow de recuperación Glue para actualizar Data Catalog y reejecutar el Glue Job",
  "service": "stepfunctions",
  "operation": "start_execution",
  "parameters": {
    	"stateMachineArn": "[GLUE_RECOVERY_STATE_MACHINE_ARN]",
    	"database": "[DATABASE]",
	"table": "[TABLE]"
  }
}
```


