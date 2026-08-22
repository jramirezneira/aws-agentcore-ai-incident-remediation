# Incident Schema

Cada incidente representa un único tipo de fallo observado.

## fingerprint
Identificador único del incidente.

## caller_name
Componente que ejecutó la operación que produjo el error.
Ejemplo:
- lambda2
- glue-job-name
- analytics-web

## caller_type
Tipo del componente que originó el incidente.
Ejemplos:
- AWS Lambda
- Flask
- glue-job


## excecution_context
campo opcional que incluye información adicional del contexto de ejecución y configuraciones.
Ejemplos:
- Jobs arguments
- RetryAttempts
- NumberOfWorkers


## layer
Capa arquitectónica del componente origen.
Valores:
- CLIENT
- BACKEND
- DATA


## target_operation
Operación ejecutada que generó el incidente.
Ejemplos:
- Athena.StartQueryExecution
- Glue.UpdateTable
- S3.PutObject
- HTTP

## count
Número de veces que se observó el incidente.

## sample
Fragmento representativo del log.

Debe utilizarse como evidencia principal para determinar la causa raíz.

---
