# Componente Aplicación Flask

## Tipo
Servicio Web Python

## Ubicación
Servidor On-Premises

## Responsabilidad
Recibir las solicitudes HTTP de los usuarios y delegar las consultas a las funciones AWS Lambda.

## Dependencias
 Lambda1
 Lambda2
 Lambda3
 Lambda4

## Operaciones AWS
No utiliza boto3.
No accede directamente a AWS.

## Restricciones
Nunca consulta Athena directamente.
Nunca modifica infraestructura AWS.
No debe participar en remediaciones automáticas.
