 
# Práctica 3.1 Exploración y Configuración de RMAN en CDB (Respaldo y Duplicación)

<br/><br/>

## Objetivos

* Analizar las diferencias de configuración RMAN entre modo CDB y bases no-CDB.
* Configurar políticas de retención, compresión y paralelización de backups en entornos multitenant.
* Habilitar **Archivelog Mode** y configurar la **Fast Recovery Area (FRA)** con parámetros óptimos.
* Explorar metadata de respaldos mediante vistas dinámicas (`V$RMAN_`, `CDB_`) del diccionario de datos.

<br/><br/>


## Tiempo estimado

- 60 minutos

<br/><br/>

## Objetivo visual

![diagrama1](../images/img4-1.png)

<br/><br/>

## Tabla de ayuda


| **Elemento**                 | **Descripción**                        | **Ejemplo**                                                                |
| ---------------------------- | ------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| **ORACLE_SID**               | Define la instancia CDB activa.             | `ORCL`                                                                                            |
| **ORACLE_HOME**              | Ruta de instalación de Oracle.              | `/u01/app/oracle/product/19c/dbhome_1`                                                            |
| **RMAN**                     | Herramienta para respaldos y duplicaciones. | `rman target /`                                                                                   |
| **SQL*Plus**                 | Ejecuta comandos SQL administrativos.       | `sqlplus / as sysdba`                                                                             |
| **Modo ARCHIVELOG**          | Permite respaldos en línea y recuperación.  | `ALTER DATABASE ARCHIVELOG;`                                                                      |
| **Fast Recovery Area (FRA)** | Almacena respaldos y archivelogs.           | `/u01/app/oracle/fast_recovery_area`                                                              |
| **Política de retención**    | Define cuánto tiempo conservar respaldos.   | `RECOVERY WINDOW OF 7 DAYS`                                                                       |
| **Compresión / Paralelismo** | Optimiza rendimiento y espacio de backup.   | `COMPRESSION 'MEDIUM'`, `PARALLELISM 2`                                                           |
| **Vistas de validación**     | Consultan configuraciones y estado de RMAN. | `V$RMAN_CONFIGURATION`, `V$RMAN_BACKUP_JOB_DETAILS`                                               |
| **Backup de prueba**         | Verifica la configuración del entorno.      | `BACKUP CURRENT CONTROLFILE FORMAT '/u01/app/oracle/fast_recovery_area/controlfile_test_%U.bkp';` |


<br/><br/>



## Instrucciones

### Tarea 1. Validar estado del CDB y entorno de trabajo

#### **Paso 1.** Establecer variables de entorno en la sesión actual:

```bash
export ORACLE_SID=ORCL
export ORACLE_HOME=/u01/app/oracle/product/19c/dbhome_1
export PATH=$ORACLE_HOME/bin:$PATH
```

#### **Paso 2.** Verificar que el CDB se encuentra activo:

```sql
sqlplus / as sysdba <<EOF
SELECT instance_name, status, database_role FROM v$instance;
SELECT name, open_mode, cdb FROM v$database;
EXIT;
EOF
```

#### **Paso 3.** Confirmar espacio disponible en disco para la FRA:

```bash
df -h /u01
```

<br/><br/>

### Tarea 2. Conectarse a RMAN y revisar configuración actual

#### **Paso 1.** Conectarse a RMAN en modo CDB:

```bash
rman target /
```

#### **Paso 2.** Mostrar toda la configuración RMAN actual:

```rman
RMAN> SHOW ALL;
```

#### **Paso 3.** Documentar diferencias CDB vs. non-CDB:

```sql
sqlplus / as sysdba <<EOF
SELECT name, cdb, con_id, con_dbid FROM v$database;
SELECT con_id, name, open_mode, restricted, total_size/1024/1024 AS size_mb
FROM v$containers ORDER BY con_id;
EXIT;
EOF
```

<br/><br/>

### Tarea 3. Habilitar Archivelog Mode

#### **Paso 1.** Verificar estado actual:

```sql
sqlplus / as sysdba <<EOF
SELECT name, log_mode, force_logging, flashback_on FROM v$database;
EXIT;
EOF
```

#### **Paso 2.** Habilitar Archivelog Mode si está desactivado:

```sql
sqlplus / as sysdba <<EOF
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE ARCHIVELOG;
ALTER DATABASE OPEN;
ALTER PLUGGABLE DATABASE ALL OPEN;
EXIT;
EOF
```

#### **Paso 3.** Validar destino de los archivelogs:

```sql
sqlplus / as sysdba <<EOF
SELECT name, value FROM v$parameter
WHERE name LIKE 'log_archive_dest%' OR name = 'log_archive_format';
EXIT;
EOF
```

<br/><br/>

### Tarea 4. Configurar Fast Recovery Area (FRA)

#### **Paso 1.** Establecer ubicación y tamaño:

```sql
sqlplus / as sysdba <<EOF
ALTER SYSTEM SET db_recovery_file_dest='/u01/app/oracle/fast_recovery_area' SCOPE=BOTH;
ALTER SYSTEM SET db_recovery_file_dest_size=20G SCOPE=BOTH;
SHOW PARAMETER db_recovery_file_dest;
EXIT;
EOF
```

#### **Paso 2.** Crear directorio con permisos adecuados:

```bash
mkdir -p /u01/app/oracle/fast_recovery_area
chown oracle:oinstall /u01/app/oracle/fast_recovery_area
chmod 755 /u01/app/oracle/fast_recovery_area
ls -ld /u01/app/oracle/fast_recovery_area
```

#### **Paso 3.** Consultar uso de espacio en FRA:

```sql
sqlplus / as sysdba <<EOF
SELECT file_type, percent_space_used, percent_space_reclaimable, number_of_files
FROM v$flash_recovery_area_usage ORDER BY percent_space_used DESC;
SELECT name, space_limit/1024/1024/1024 AS space_limit_gb,
       space_used/1024/1024/1024 AS space_used_gb,
       space_reclaimable/1024/1024/1024 AS space_reclaimable_gb,
       number_of_files
FROM v$recovery_file_dest;
EXIT;
EOF
```

<br/><br/>

### Tarea 5. Configurar políticas RMAN

#### **Paso 1.** Configurar política de retención:

```rman
CONFIGURE RETENTION POLICY TO RECOVERY WINDOW OF 7 DAYS;
CONFIGURE CONTROLFILE AUTOBACKUP ON;
CONFIGURE CONTROLFILE AUTOBACKUP FORMAT FOR DEVICE TYPE DISK TO '/u01/app/oracle/fast_recovery_area/%F';
SHOW RETENTION POLICY;
```

#### **Paso 2.** Activar compresión y paralelismo:

```rman
CONFIGURE COMPRESSION ALGORITHM 'MEDIUM';
CONFIGURE DEVICE TYPE DISK PARALLELISM 2 BACKUP TYPE TO BACKUPSET;
CONFIGURE BACKUP OPTIMIZATION ON;
SHOW ALL;
```

#### **Paso 3.** Definir política de eliminación de Archivelogs:

```rman
CONFIGURE ARCHIVELOG DELETION POLICY TO BACKED UP 1 TIMES TO DISK;
SHOW ARCHIVELOG DELETION POLICY;
```

<br/><br/>

### Tarea 6. Verificar metadatos y realizar backup de prueba

#### **Paso 1.** Consultar configuración RMAN vía SQL:

```sql
sqlplus / as sysdba <<EOF
SELECT name, value FROM v$rman_configuration ORDER BY conf#;
EXIT;
EOF
```

#### **Paso 2.** Realizar backup de controlfile de prueba:

```rman
BACKUP CURRENT CONTROLFILE FORMAT '/u01/app/oracle/fast_recovery_area/controlfile_test_%U.bkp';
LIST BACKUP SUMMARY;
```

#### **Paso 3.** Revisar metadatos del backup:

```sql
sqlplus / as sysdba <<EOF
SELECT session_key, input_type, status,
       TO_CHAR(start_time, 'YYYY-MM-DD HH24:MI:SS') AS start_time,
       TO_CHAR(end_time, 'YYYY-MM-DD HH24:MI:SS') AS end_time,
       output_bytes_display, compression_ratio
FROM v$rman_backup_job_details
WHERE start_time > SYSDATE - 1
ORDER BY start_time DESC;
EXIT;
EOF
```

<br/><br/>

## Resultado esperado

Al finalizar la práctica, deberás obtener salidas similares a las siguientes en tu entorno Oracle:

### **1. FRA correctamente configurada**

Comprobación desde SQL*Plus:

```sql
SQL> SHOW PARAMETER db_recovery_file_dest;

NAME                                 TYPE        VALUE
------------------------------------ ----------- ----------------------------------------
db_recovery_file_dest                string      /u01/app/oracle/fast_recovery_area
db_recovery_file_dest_size           big integer 20G
```

Visualización del directorio en Linux:

```bash
$ ls -ld /u01/app/oracle/fast_recovery_area
drwxr-xr-x. 3 oracle oinstall 4096 Nov 09 11:45 /u01/app/oracle/fast_recovery_area
```

<br/><br/>

### **2. Archivelog Mode activo**

Validación desde SQL*Plus:

```sql
SQL> SELECT name, log_mode FROM v$database;

NAME      LOG_MODE
--------- ----------------
ORCL      ARCHIVELOG
```

<br/><br/>

### **3. RMAN mostrando políticas y configuración actual**

Desde el prompt de RMAN:

```rman
RMAN> SHOW ALL;

CONFIGURE RETENTION POLICY TO RECOVERY WINDOW OF 7 DAYS;
CONFIGURE CONTROLFILE AUTOBACKUP ON;
CONFIGURE COMPRESSION ALGORITHM 'MEDIUM';
CONFIGURE DEVICE TYPE DISK PARALLELISM 2 BACKUP TYPE TO BACKUPSET;
CONFIGURE BACKUP OPTIMIZATION ON;
CONFIGURE ARCHIVELOG DELETION POLICY TO BACKED UP 1 TIMES TO DISK;

```

<br/><br/>

### **4. Evidencia del backup de controlfile**

Salida del comando de respaldo:

```rman
RMAN> BACKUP CURRENT CONTROLFILE FORMAT '/u01/app/oracle/fast_recovery_area/controlfile_test_%U.bkp';

Starting backup at 09-NOV-25
allocated channel: ORA_DISK_1
channel ORA_DISK_1: starting full backup of control file
channel ORA_DISK_1: backup complete, elapsed time: 00:00:03
Finished backup at 09-NOV-25
```

Verificación del resumen de backups:

```rman
RMAN> LIST BACKUP SUMMARY;

List of Backups
===============

Key     TY LV S Device Type Completion Time
------- -- -- - ----------- -------------------
1       B  F  A DISK        09-NOV-25
```

Comprobación del archivo generado:

```bash
$ ls -lh /u01/app/oracle/fast_recovery_area/controlfile_test_*.bkp
-rw-r-----. 1 oracle oinstall 14M Nov 09 11:58 controlfile_test_0m6c9v1t_1_1.bkp
```

