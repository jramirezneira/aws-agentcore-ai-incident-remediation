# Componente: AWS Glue Data Catalog

## Tipo
Catálogo de metadatos

## Responsabilidad
Gestionar el esquema utilizado por Amazon Athena.

## Operaciones utilizadas
* GetTable
* UpdateTable

## Errores frecuentes
* ParamValidationError

## Restricciones
No recrear tablas automáticamente.
Las actualizaciones deben preservar el esquema existente.
