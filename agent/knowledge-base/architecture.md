# Arquitectura del Sistema de Consulta Histórica de Tráfico de Madrid

## Objetivo
Este sistema proporciona información histórica del tráfico de la ciudad de Madrid a una aplicación web.
La información se almacena en Amazon S3, se consulta mediante Amazon Athena y el catálogo de metadatos es gestionado por AWS Glue Data Catalog.
Las consultas SQL son ejecutadas por un conjunto de funciones AWS Lambda. Una aplicación web desarrollada con Flask y desplegada en un servidor on-premises recibe las peticiones de los usuarios e invoca las funciones Lambda correspondientes.

---

# Flujo funcional
Usuario
↓
Aplicación Web Flask (On-Premises)
↓
AWS Lambda
↓
Amazon Athena
↓
AWS Glue Data Catalog
↓
Amazon S3

---

# Componentes
## Aplicación Flask

**Responsabilidad**
* Recibir solicitudes HTTP de los usuarios.
* Invocar las funciones Lambda correspondientes.
* No realiza consultas directas sobre Athena.
* No almacena datos.

---

## Lambda1, Lambda2, Lambda3 y Lambda4
**Responsabilidad**
* Ejecutar consultas SQL sobre Amazon Athena.
* Procesar los resultados.
* Devolver la información a la aplicación Flask.

**Dependencias**
* Amazon Athena
* Amazon S3 (OutputLocation)
* AWS Glue Data Catalog

**Operaciones AWS utilizadas**
* StartQueryExecution
* GetQueryExecution
* GetQueryResults

Estas funciones no modifican infraestructura AWS.

---

## Amazon Athena
**Responsabilidad**

Ejecutar consultas SQL sobre el histórico de tráfico.

**Base de datos**
test

**Tabla principal**
TrafficDator_Madrid_output

**Dependencias**
* AWS Glue Data Catalog
* Bucket S3 de resultados

Las funciones Lambda utilizan Athena exclusivamente para consultas de lectura.

---

## AWS Glue Data Catalog
**Responsabilidad**
Gestionar el esquema de la tabla utilizada por Athena.
Las funciones Lambda no crean tablas nuevas.
Únicamente consultan o actualizan metadatos cuando es necesario.

---

## Amazon S3
**Bucket de datos**
s3://variosjavierramirez/TrafficDator_Madrid_output/

**Responsabilidad**
* Almacenar el histórico de tráfico.
* Servir como origen de datos para Athena.

Athena utiliza además un bucket independiente para almacenar los resultados temporales de las consultas (OutputLocation).

---

# Dependencias funcionales
La aplicación Flask depende de las funciones Lambda.
Las funciones Lambda dependen de Athena.
Athena depende del catálogo de Glue y de los buckets S3.
Un fallo en Athena puede impedir que la aplicación Flask devuelva información al usuario.

---

# Restricciones importantes
Las funciones Lambda no deben modificar automáticamente su propia configuración como mecanismo de remediación, salvo que exista evidencia explícita de que dicha configuración sea la causa raíz del incidente.
Cuando se detecten errores relacionados con Amazon Athena, las remediaciones deben priorizar:
1. Verificar la existencia y accesibilidad de los buckets S3.
2. Verificar el catálogo de Glue.
3. Verificar permisos de acceso.
4. Sólo modificar configuraciones cuando exista evidencia suficiente.

La reejecución automática únicamente es recomendable para procesos batch o cuando exista un mecanismo seguro de reintento.

---


