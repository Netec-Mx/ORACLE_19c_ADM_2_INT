
# Práctica 3.2 Backup Completo de CDB y Restauración de una PDB Específica

<br/><br/>

## Objetivos

* Ejecutar un backup completo de un CDB incluyendo todas las PDBs utilizando RMAN.
* Simular una pérdida de datafiles en una PDB específica y ejecutar su recuperación selectiva.
* Validar la integridad post-recuperación y calcular métricas RTO/RPO del proceso.

<br/><br/>

## Tiempo estimado

- 90 minutos

<br/><br/>

## Tabla de ayuda

| **Comando**                                     | **Descripción**                                                                                                     |
| ----------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| `BACKUP DATABASE PLUS ARCHIVELOG DELETE INPUT;` | Realiza un respaldo completo del CDB incluyendo todas las PDBs y elimina los archivos de archivelog ya respaldados. |
| `BACKUP PLUGGABLE DATABASE pdb_name;`           | Crea un respaldo físico únicamente de la PDB especificada.                                                          |
| `RESTORE PLUGGABLE DATABASE pdb_name;`          | Restaura los datafiles de una PDB específica a partir del respaldo disponible.                                      |
| `RECOVER PLUGGABLE DATABASE pdb_name;`          | Aplica los archivelogs para recuperar la PDB hasta el punto de fallo.                                               |
| `VALIDATE PLUGGABLE DATABASE pdb_name;`         | Verifica la integridad física de los respaldos de la PDB.                                                           |
| `LIST BACKUP OF PLUGGABLE DATABASE pdb_name;`   | Muestra la lista de respaldos disponibles para la PDB especificada.                                                 |


<br/><br/>

## Objetivo visual

![RMAN Flujo](../images/img4-1.png)

> El flujo muestra cómo RMAN permite configurar, respaldar, restaurar y duplicar bases multitenant, asegurando disponibilidad, recuperación rápida y continuidad operativa del entorno Oracle.

<br/><br/>


## Instrucciones si no completaste la Práctica 3.1

#### **Paso 1.** Validar que el CDB está en modo ARCHIVELOG

```sql
sqlplus / as sysdba

-- Verificar modo de archivado
SELECT log_mode, force_logging FROM v$database;

-- Si el resultado es NOARCHIVELOG, ejecutar:
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE ARCHIVELOG;
ALTER DATABASE OPEN;
```

#### **Paso 2.** Verificar configuración de la Fast Recovery Area (FRA)

```sql
-- Consultar configuración de FRA
SHOW PARAMETER db_recovery_file_dest;
```

#### **Paso 3.** Listar PDB existentes y su estado

```sql
SET LINESIZE 180
COL NAME FORMAT A15
SELECT con_id, name, open_mode, round(total_size/1024/1024/1024) AS size_gb
FROM v$pdbs
ORDER BY con_id;
```

#### **Paso 4.** Crear datos de prueba en PDB1 (PDB objetivo)

```sql
-- Conectar a PDB1
ALTER SESSION SET CONTAINER=PDB1;

-- Crear usuario de prueba
CREATE USER testuser IDENTIFIED BY Oracle123
  DEFAULT TABLESPACE users
  QUOTA UNLIMITED ON users;

GRANT CONNECT, RESOURCE TO testuser;

-- Conectar como testuser
CONNECT testuser/Oracle123@localhost:1521/PDB1

-- Crear tabla con datos significativos
CREATE TABLE customer_backup_test (
  customer_id       NUMBER PRIMARY KEY,
  customer_name     VARCHAR2(100),
  email             VARCHAR2(100),
  registration_date DATE,
  account_balance   NUMBER(10,2),
  status            VARCHAR2(20)
);

-- Insertar datos de prueba (100,000 registros)
BEGIN
  FOR i IN 1..100000 LOOP
    INSERT INTO customer_backup_test VALUES (
      i,
      'Customer_' || i,
      'customer' || i || '@example.com',
      SYSDATE - DBMS_RANDOM.VALUE(1, 3650),
      DBMS_RANDOM.VALUE(100, 50000),
      CASE WHEN MOD(i, 3) = 0 THEN 'ACTIVE' ELSE 'INACTIVE' END
    );
    IF MOD(i, 10000) = 0 THEN COMMIT; END IF;
  END LOOP;
  COMMIT;
END;
/

-- Verificar datos insertados
SELECT COUNT(*) AS total_records FROM customer_backup_test;

-- Registrar checksum para validación posterior
SELECT SUM(customer_id) AS checksum_id,
       SUM(account_balance) AS checksum_balance
FROM customer_backup_test;
```

> Anota estos valores para compararlos en la validación post-recuperación.

#### **Paso 5.** Forzar un switch de logfile para asegurar el archivado

```sql
CONNECT / AS SYSDBA
ALTER SYSTEM SWITCH LOGFILE;
ALTER SYSTEM CHECKPOINT;
```

<br/><br/>

## Instrucciones

### Tarea 1. Ejecución de backup completo del CDB

#### **Paso 1.** Conectar a RMAN y verificar configuración

```bash
rman target /
```

```rman
-- Mostrar configuración actual de RMAN
SHOW ALL;
```

#### **Paso 2.** Registrar timestamp de inicio del backup

```bash
# En otra terminal, registrar hora de inicio
date '+%Y-%m-%d %H:%M:%S' > /tmp/backup_start_time.txt
cat /tmp/backup_start_time.txt
```

#### **Paso 3.** Ejecutar backup completo del CDB

```rman
-- Conectate con RMAN a la TARGET actual

RMAN TARGET /

-- Backup completo: database + archivelog + delete input

BACKUP AS COMPRESSED BACKUPSET
  DATABASE PLUS ARCHIVELOG
  DELETE INPUT
  TAG 'CDB_FULL_BACKUP_LAB';
exit
```

#### **Paso 4.** Registrar timestamp de fin y calcular duración

```bash
date '+%Y-%m-%d %H:%M:%S' > /tmp/backup_end_time.txt
echo "Backup completado en:"
echo "Inicio: $(cat /tmp/backup_start_time.txt)"
echo "Fin:    $(cat /tmp/backup_end_time.txt)"
```

#### **Paso 5.** Verificar contenido del backup

```rman
-- Resumen de backups
RMAN TARGET /
LIST BACKUP SUMMARY;

-- Backups específicos de PDB1
LIST BACKUP OF PLUGGABLE DATABASE PDB1;
```

#### **Paso 6.** Validar integridad del backup

```rman
-- Validar backup completo sin restaurar
LIST BACKUP SUMMARY
-- Anota los numeros de backup que correspondan a la etiqueta 'CDB_FULL_BACKUP_LAB'
-- Valida con los KEYs de los respaldos
VALIDATE BACKUPSET n1, n2, etc;
EXIT
```

#### **Paso 7.** Consultar métricas detalladas del backup

```sql
sqlplus / AS SYSDBA

SET LINESIZE 200
COL status              FORMAT A10
COL input_type          FORMAT A15
COL output_device_type  FORMAT A10
COL time_taken_display  FORMAT A15
COL input_bytes_display FORMAT A15
COL output_bytes_display FORMAT A15

SELECT session_key, status, input_type, output_device_type,
       time_taken_display, input_bytes_display, output_bytes_display,
       compression_ratio
FROM   v$rman_backup_job_details
WHERE  start_time > SYSDATE - 1
ORDER  BY start_time DESC;
```

<br/><br/>

### Tarea 2. Simulación de pérdida de datos en PDB1

#### **Paso 1.** Identificar ubicación de datafiles de PDB1

```sql
-- Conectar al CDB como SYSDBA
CONNECT / AS SYSDBA

-- Cambiar al contenedor PDB1
ALTER SESSION SET CONTAINER=PDB1;

-- Listar datafiles de PDB1
SET LINESIZE 200
COL name FORMAT A80
SELECT file#, name, bytes/1024/1024 AS size_mb, status
FROM   v$datafile
ORDER  BY file#;
```

> Anota el `file#` del datafile **USERS** (típicamente el último).

#### **Paso 2.** Generar actividad adicional antes de la simulación de fallo

```sql
-- Conectar como testuser para generar más transacciones
CONNECT testuser/Oracle123@localhost:1521/PDB1

-- Insertar registros adicionales
INSERT INTO customer_backup_test
SELECT customer_id + 100000,
       'PostBackup_' || customer_name,
       'postbackup_' || email,
       SYSDATE,
       account_balance * 1.1,
       'ACTIVE'
FROM   customer_backup_test
WHERE  ROWNUM <= 10000;

COMMIT;

-- Registrar nuevo checksum
SELECT COUNT(*) AS total_records_before_failure,
       SUM(customer_id) AS checksum_before_failure
FROM   customer_backup_test;
```

> Anota estos valores como **“Pre-Failure State”**.

#### **Paso 3.** Forzar CHECKPOINT y SWITCH LOGFILE

```sql
CONNECT / AS SYSDBA
ALTER SYSTEM SWITCH LOGFILE;
ALTER SYSTEM CHECKPOINT;
```

#### **Paso 4.** Cerrar PDB1 para simular la pérdida

```sql
-- Cerrar la PDB afectada
ALTER PLUGGABLE DATABASE PDB1 CLOSE IMMEDIATE;

-- Verificar estado
SELECT name, open_mode FROM v$pdbs WHERE name = 'PDB1';
EXIT;
```

#### **Paso 5.** Eliminar el datafile de USERS para simular corrupción/pérdida
```bash
# ADVERTENCIA: Operación destructiva; asegúrate de tener backup
# Verificar existencia del datafile
ls -lh /u01/app/oracle/oradata/ORCL/pdb1/users01.dbf

# Registrar tamaño y timestamp antes de eliminar
stat /u01/app/oracle/oradata/ORCL/pdb1/users01.dbf > /tmp/deleted_datafile_info.txt

# Eliminar el datafile (SIMULACIÓN DE PÉRDIDA)
rm -f /u01/app/oracle/oradata/ORCL/pdb1/users01.dbf

# Verificar eliminación
ls -lh /u01/app/oracle/oradata/ORCL/pdb1/
```

#### **Paso 6.** Intentar abrir PDB1 para confirmar el fallo

```sql
sqlplus / AS SYSDBA
ALTER PLUGGABLE DATABASE PDB1 OPEN;
```

#### **Paso 7.** Registrar timestamp del incidente

```bash
date '+%Y-%m-%d %H:%M:%S' > /tmp/failure_detection_time.txt
echo "Fallo detectado en: $(cat /tmp/failure_detection_time.txt)"
```

<br/><br/>

### Tarea 3. Restauración y recuperación de PDB1

#### **Paso 1.** Conectar a RMAN y verificar disponibilidad de backups

```bash
rman target /
```

```rman
-- Verificar que RMAN puede acceder a los backups
LIST BACKUP OF PLUGGABLE DATABASE PDB1 SUMMARY;
```

#### **Paso 2.** Restaurar datafiles de PDB1

```rman
-- Registrar hora de inicio de restauración (desde RMAN con HOST)
HOST echo "Restore iniciado: $(date '+%Y-%m-%d %H:%M:%S')" > /tmp/restore_start_time.txt;

-- Ejecutar RESTORE de PDB1
RUN {
  SET NEWNAME FOR DATAFILE 12 TO '/u01/app/oracle/oradata/ORCL/PDB1/users01.dbf';
  RESTORE PLUGGABLE DATABASE PDB1;
}
```

#### **Paso 3.** Recuperar PDB1 aplicando archivelogs

```rman
-- Registrar hora de inicio de recuperación
HOST echo "Recovery iniciado: $(date '+%Y-%m-%d %H:%M:%S')" > /tmp/recover_start_time.txt;

-- Ejecutar RECOVER de PDB1
RECOVER PLUGGABLE DATABASE PDB1;
```

#### **Paso 4.** Registrar timestamp de finalización de la recuperación

```rman
HOST echo "Recovery finalizado: $(date '+%Y-%m-%d %H:%M:%S')" > /tmp/recover_end_time.txt;
EXIT;
```

#### **Paso 5.** Abrir PDB1 y verificar estado

```sql
sqlplus / AS SYSDBA

-- Abrir PDB1 en modo READ WRITE
ALTER PLUGGABLE DATABASE PDB1 OPEN;

-- Verificar estado de apertura
SELECT name, open_mode, restricted, recovery_status
FROM   v$pdbs
WHERE  name = 'PDB1';
```

#### **Paso 6.** Validar integridad de datos post-recuperación

```sql
-- Cambiar al contenedor PDB1
ALTER SESSION SET CONTAINER=PDB1;

-- Verificar que todos los datafiles están online
SELECT file#, name, status, enabled
FROM   v$datafile
ORDER  BY file#;

-- Conectar como testuser y verificar datos
CONNECT testuser/Oracle123@localhost:1521/PDB1

-- Verificar count y checksum
SELECT COUNT(*) AS total_records_after_recovery,
       SUM(customer_id) AS checksum_after_recovery
FROM   customer_backup_test;

-- Verificar datos específicos insertados post-backup
SELECT COUNT(*) AS postbackup_records
FROM   customer_backup_test
WHERE  customer_name LIKE 'PostBackup_%';
```

> Si este valor es **10000**, la recuperación aplicó correctamente los archivelogs.

#### **Paso 7.** Validar integridad física con RMAN

```sql
EXIT
```

```bash
rman target /
```

```rman
-- Validar integridad física de PDB1
VALIDATE PLUGGABLE DATABASE PDB1;
```

> Verifica que `STATUS = OK` y `MARKED CORRUPT = 0` para todos los datafiles.

#### **Paso 8.** Verificar que otras PDB no fueron afectadas

```sql
sqlplus / AS SYSDBA

-- Verificar estado de todas las PDB
SELECT con_id, name, open_mode, restricted, recovery_status
FROM   v$pdbs
ORDER  BY con_id;

-- Cambiar a PDB2 y verificar operatividad
ALTER SESSION SET CONTAINER=PDB2;

-- Consulta simple para confirmar acceso
SELECT tablespace_name, status FROM dba_tablespaces;
```

> Si la consulta devuelve resultados sin errores, **PDB2** no fue impactada.

<br/><br/>

### Tarea 4. Análisis de RTO/RPO y documentación

#### **Paso 1.** Calcular RTO observado

```bash
echo "=== ANÁLISIS DE RTO ==="
echo "Detección de fallo: $(cat /tmp/failure_detection_time.txt)"
echo "Inicio de restore:  $(cat /tmp/restore_start_time.txt)"
echo "Inicio de recover:  $(cat /tmp/recover_start_time.txt)"
echo "Fin de recover:     $(cat /tmp/recover_end_time.txt)"

# RTO = Tiempo desde detección hasta PDB operativa
# Ejemplo:
# 14:52:30 → 15:01:23  =>  8 minutos 53 segundos
```

#### **Paso 2.** Calcular RPO

```sql
sqlplus / AS SYSDBA

ALTER SESSION SET CONTAINER=PDB1;

SELECT 'Backup SCN'  AS checkpoint_type,
       TO_CHAR(checkpoint_change#) AS scn,
       TO_CHAR(checkpoint_time, 'YYYY-MM-DD HH24:MI:SS') AS checkpoint_time
FROM   v$datafile_header
WHERE  file# = 12
UNION ALL
SELECT 'Current SCN' AS checkpoint_type,
       TO_CHAR(current_scn) AS scn,
       TO_CHAR(SYSDATE, 'YYYY-MM-DD HH24:MI:SS') AS checkpoint_time
FROM   v$database;
```

> RPO observado = 0 segundos (recuperación hasta el punto de fallo).
> En producción dependerá de la frecuencia de backups y disponibilidad de archivelogs.

#### **Paso 3.** Consultar métricas detalladas de la recuperación

```sql
SET LINESIZE 200
COL item      FORMAT A30
COL units     FORMAT A10
COL sofar     FORMAT 999999999
COL total     FORMAT 999999999
COL timestamp FORMAT A20

SELECT item, units, sofar, total,
       TO_CHAR(timestamp, 'YYYY-MM-DD HH24:MI:SS') AS timestamp
FROM   v$recovery_progress
WHERE  start_time > SYSDATE - 1
ORDER  BY start_time DESC;
```

#### **Paso 4.** Generar reporte técnico desde RMAN

```sql
EXIT;
```

```bash
rman target /
```

```rman
LIST RESTORE OF PLUGGABLE DATABASE PDB1;
LIST RECOVER OF PLUGGABLE DATABASE PDB1;
```

#### **Paso 5.** Crear documento de resumen del incidente

```bash
cat > /tmp/recovery_summary_report.txt << 'EOF'
=======================================================
REPORTE DE RECUPERACIÓN - LAB 03-02-01
=======================================================

INFORMACIÓN DEL INCIDENTE:
--------------------------
Fecha del incidente: 2023-12-15
PDB afectada: PDB1
Tipo de fallo: Pérdida de datafile USERS (file# 12)
Causa: Eliminación accidental del archivo físico

BACKUP UTILIZADO:
-----------------
Tag: CDB_FULL_BACKUP_LAB
Tipo: FULL BACKUP + ARCHIVELOG
Tamaño comprimido: 8.32 GB
Tiempo de backup: 12 minutos 34 segundos
SCN del backup: 2847563

PROCESO DE RECUPERACIÓN:
------------------------
1. Detección del fallo: 14:52:30
2. Cierre de PDB: 14:53:00
3. Inicio de RESTORE: 14:55:00
4. Fin de RESTORE: 14:59:35 (4 min 35 seg)
5. Inicio de RECOVER: 14:59:37
6. Fin de RECOVER: 15:01:23 (1 min 46 seg)
7. Apertura de PDB: 15:01:30

MÉTRICAS OBSERVADAS:
--------------------
RTO (Recovery Time Objective): 8 minutos 53 segundos
RPO (Recovery Point Objective): 0 segundos (pérdida de datos: NINGUNA)
Archivelogs aplicados: 2 (sequences 46-47)
Tamaño de datos recuperados: 2.15 GB
Tasa promedio de aplicación: 14.892 KB/sec

VALIDACIÓN POST-RECUPERACIÓN:
------------------------------
- Integridad física: OK (RMAN VALIDATE exitoso)
- Integridad lógica: OK (checksums coinciden)
- Registros recuperados: 110,000
- Registros post-backup recuperados: 10,000
- Otras PDB afectadas: NINGUNA

RECOMENDACIONES:
----------------
1. Implementar backups incrementales para reducir ventanas de recuperación
2. Configurar paralelización de RMAN para mejorar tiempos de restore
3. Considerar implementar Data Guard para alta disponibilidad
4. Establecer alertas automáticas ante pérdida de datafiles

Generado por: [Nombre del estudiante]
Fecha: $(date '+%Y-%m-%d %H:%M:%S')
=======================================================
EOF


# Visualizar el reporte
cat /tmp/recovery_summary_report.txt
```

<br/><br/>

## Resultado esperado

* El backup completo del CDB se ejecuta correctamente.
* **PDB1** se restaura y abre en modo `READ WRITE` con todos sus registros intactos.
* Se calculan y documentan los tiempos **RTO** y **RPO** observados, y se genera el reporte en `/tmp/recovery_summary_report.txt`.

<br/><br/>


### Tabla de ayuda adicional: Archivos generados en `/tmp/`

| **Archivo**                        | **Descripción / Propósito**                                                                          |
| ---------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `/tmp/backup_start_time.txt`       | Registra la hora de inicio del respaldo completo del CDB.                                            |
| `/tmp/backup_end_time.txt`         | Registra la hora de finalización del respaldo completo del CDB.                                      |
| `/tmp/deleted_datafile_info.txt`   | Guarda la información del datafile eliminado (nombre, tamaño y timestamp antes de la eliminación).   |
| `/tmp/failure_detection_time.txt`  | Almacena la fecha y hora en que se detectó el fallo en la PDB.                                       |
| `/tmp/restore_start_time.txt`      | Guarda la hora en la que inicia la restauración de la PDB.                                           |
| `/tmp/recover_start_time.txt`      | Registra la hora en que se inicia la fase de recuperación aplicando archivelogs.                     |
| `/tmp/recover_end_time.txt`        | Registra la hora en que finaliza la recuperación de la PDB.                                          |
| `/tmp/recovery_summary_report.txt` | Contiene el reporte final del laboratorio, con métricas RTO/RPO, resumen del proceso y validaciones. |

