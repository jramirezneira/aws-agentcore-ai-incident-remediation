# Componente: Lambda1

## Tipo
AWS Lambda

## Responsabilidad
Ejecutar consultas SQL sobre Amazon Athena y devolver resultados a la aplicación Flask.

## Dependencias
* Amazon Athena
* AWS Glue Data Catalog
* Amazon S3 (OutputLocation)

## Operaciones AWS utilizadas
* StartQueryExecution
* GetQueryExecution
* GetQueryResults

## Restricciones
No modifica infraestructura AWS.
No actualiza configuración de otros servicios.
Su función es exclusivamente consultar datos.
