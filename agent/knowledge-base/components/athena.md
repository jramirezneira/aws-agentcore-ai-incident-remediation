# Componente: Amazon Athena

## Tipo
Motor de consultas SQL

## Responsabilidad
Ejecutar consultas sobre los datos almacenados en Amazon S3.

## Base de datos
test

## Tabla principal
TrafficDator_Madrid_output

## Dependencias
* AWS Glue Data Catalog
* Amazon S3

## Operaciones utilizadas
* StartQueryExecution
* GetQueryExecution
* GetQueryResults

## Errores frecuentes
* InvalidRequestException
* AccessDeniedException

## Restricciones
Las consultas son únicamente de lectura.
Athena requiere un bucket válido para OutputLocation.
