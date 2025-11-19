
# Capítulo 4. Gestión de PDBs de Aplicación

<br/><br/>

## **Práctica 4.1 Creación de Application PDBs y Sincronización**

En esta práctica crearás un **Application Container Root** y múltiples **Application PDBs** a partir del **Application Seed**.
Instalarás una **aplicación base (v1.0)** en el Application Root y **sincronizarás** los objetos de aplicación hacia las Application PDBs.
Además, dejarás puntos de restauración para revertir cambios si es necesario y validarás el estado con vistas de diccionario.

[Práctica 4.1](Practica_4-1.MD)

<br/><br/>

## **Práctica 4.2 Configurar Unified Auditing**

En esta práctica aprenderás a **verificar, habilitar y validar la auditoría unificada (Unified Auditing)** en Oracle Database 19c.
Explorarás cómo comprobar el modo de auditoría actual, habilitar el modo *Pure* mediante el relink del binario Oracle y crear políticas personalizadas para auditar operaciones sobre objetos sensibles dentro de una PDB.

También generarás reportes a partir de la vista `UNIFIED_AUDIT_TRAIL` para analizar los eventos registrados, comprendiendo el alcance de la auditoría en entornos multitenant y cómo identificar desde qué contenedor (`APP_PDB1`, `APP_PDB2`) se originan los eventos mediante las vistas `UNIFIED_AUDIT_TRAIL` y `CDB_UNIFIED_AUDIT_TRAIL`.

[Práctica 4.2](Practica_4-2.MD)

