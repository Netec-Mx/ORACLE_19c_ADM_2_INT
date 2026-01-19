# Práctica 3.3 Duplicando una PDB hacia la misma CDB



<br/><br/>

## Objetivos

* Configurar la infraestructura de conectividad Oracle Net (listener, tnsnames) necesaria para operaciones.
* Ejecutar una clonación activa de PDB usando la linea de comandos de SQL sin requerir backups previos.
* Validar la independencia completa entre la PDB origen y la PDB clonada mediante pruebas de modificación de datos.
* Documentar el consumo de recursos (red, I/O, memoria) durante el proceso y comparar con métodos alternativos de clonación.


<br/><br/>

## Tiempo estimado

- 60 minutos

<br/><br/>

## Tabla de ayuda


| **Comando / Vista / Parámetro**                   | **¿Para qué sirve?**                                                                      |
| ------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| `lsnrctl status`                                  | Verifica que el listener de Oracle esté en ejecución y escuche el puerto configurado.     |
| `tnsping <servicio>`                              | Comprueba la conectividad de red y la resolución del alias definido en `tnsnames.ora`.    |
| `sqlplus / as sysdba`                             | Conecta a la base de datos como usuario administrativo SYSDBA.                            |
| `ARCHIVE LOG LIST`                                | Muestra si la base de datos está en modo ARCHIVELOG o NOARCHIVELOG.                       |
| `SHOW PARAMETER file_name_convert`                | Verifica si existe conversión automática de rutas entre archivos de diferentes PDBs.      |
| `CDB_DATA_FILES`                                  | Lista los datafiles de todas las PDBs abiertas en la CDB.                                 |
| `V$CONTAINERS`                                    | Muestra los nombres (`NAME`) y el identificador (`CON_ID`) de todos los contenedores.     |
| `V$DATAFILE`                                      | Lista los datafiles sin requerir que las PDBs estén abiertas (visible en modo MOUNTED).   |
| `V$SESSION_LONGOPS`                               | Muestra el progreso de operaciones de larga duración, como duplicaciones RMAN.            |
| `V$RMAN_STATUS`                                   | Contiene el historial de operaciones RMAN (BACKUP, RESTORE, DUPLICATE, etc.).             |
| `V$PDB_IOSTAT_DETAIL`                             | Reporta estadísticas de I/O (lecturas y escrituras físicas) por PDB.                      |
| `CREATE DATABASE PDB_COPY FROM PDB_ORIGINAL ...`           | Ejecuta una clonación por comando de una PDB sin requerir respaldos previos.                   |
| `RMAN DUPLICATE PLUGGABLE DATABASE ...`           | Ejecuta una clonación activa de una PDB sin requerir respaldos previos.                   |
| `ALTER PLUGGABLE DATABASE ... OPEN`               | Abre una PDB en modo `READ WRITE` o `READ ONLY`.                                          |
| `ALTER PLUGGABLE DATABASE ... RENAME GLOBAL_NAME` | Permite renombrar una PDB existente dentro de la misma CDB.                               |
| `DROP PLUGGABLE DATABASE ... INCLUDING DATAFILES` | Elimina una PDB y sus datafiles físicos asociados.                                        |
| `CONTAINERS(view)`                                | Consulta consolidada que permite acceder a una vista en todas las PDBs (cross-container). |
| `V$SYSSTAT`                                       | Muestra estadísticas globales del sistema, incluyendo uso de memoria.                     |
| `df -h`                                           | Verifica espacio disponible en el sistema de archivos a nivel de servidor.                |
| `mkdir -p <ruta>` / `chmod` / `chown`             | Crea y asigna permisos a directorios para almacenamiento de datafiles de la PDB clonada.  |



<br/><br/>

## Objetivo visual

![diagrama1](../images/img4-1.png)

<br/><br/>

## **Prerequisitos**

**Labs previos requeridos:**

* Configuración de RMAN y backups de CDB completado.
* Estrategias de backup de PDBs completado.
* Haber creado el usuario TESTUSER y otorgado privilegios del ejercicio anterior.

<br/><br/>

## **Verificación Inicial del Ambiente**

Ejecuta los siguientes comandos para validar el entorno antes de iniciar:

```bash
# Verificar que el listener está en ejecución
lsnrctl status

# Verificar variables de ambiente
echo $ORACLE_HOME
echo $ORACLE_SID

# Verificar espacio en disco disponible
df -h $ORACLE_BASE/oradata
```

```sql
-- Conectar como SYSDBA
sqlplus / as sysdba

-- Verificar estado de la CDB
SELECT name, open_mode, cdb FROM v$database;

-- Verificar PDBs existentes
SELECT pdb_name, status, open_mode FROM cdb_pdbs ORDER BY pdb_name;

-- Verificar tamaño de la PDB origen (ejemplo: PDB1)
SELECT c.name AS pdb_name,
       ROUND(SUM(df.bytes)/1024/1024/1024, 2) AS data_gb
FROM   v$datafile df
JOIN   v$containers c
  ON   df.con_id = c.con_id
WHERE  c.name = 'PDB1'
GROUP  BY c.name
ORDER  BY c.name;
```

<br/><br/>

## Instrucciones

### Tarea 1. Preparar la PDB Origen con Datos de Prueba

```sql
ALTER SESSION SET CONTAINER = PDB1;

-- Solo en caso de no haber creado el usuario `testuser` ejecutar los siguientes comandos SQL.
CREATE USER testuser IDENTIFIED BY Oracle123
DEFAULT TABLESPACE users
QUOTA UNLIMITED ON users;

GRANT CREATE SESSION, CREATE TABLE TO testuser;

-- Conexión a testuser
CONNECT testuser/Oracle123@localhost:1521/PDB1

CREATE TABLE sales_data (
    id NUMBER PRIMARY KEY,
    customer_name VARCHAR2(100),
    product_name VARCHAR2(100),
    sale_amount NUMBER(10,2),
    sale_date DATE,
    notes VARCHAR2(4000)
);

BEGIN
    FOR i IN 1..100000 LOOP
        INSERT INTO sales_data VALUES (
            i,
            'Customer_' || i,
            'Product_' || MOD(i, 100),
            ROUND(DBMS_RANDOM.VALUE(10, 10000), 2),
            SYSDATE - DBMS_RANDOM.VALUE(1, 365),
            RPAD('Sample data for testing ', 4000, 'X')
        );
        IF MOD(i, 10000) = 0 THEN
            COMMIT;
        END IF;
    END LOOP;
    COMMIT;
END;
/

SELECT COUNT(*) FROM sales_data;

SELECT
    tablespace_name,
    ROUND(SUM(bytes)/1024/1024, 2) AS size_mb
FROM user_segments
GROUP BY tablespace_name;
EXIT
```

> **Nota:** Debe haber alrededor de 100,000 registros y un tamaño de varios MB.

<br/><br/>

### Tarea 2. Configurar Servicios de Red para la PDB Origen

```bash
lsnrctl services
vi $ORACLE_HOME/network/admin/tnsnames.ora
```

Agregar al final del archivo:

```
PDB1 =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = localhost)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = PDB1)
    )
  )

PDB1_CLONE =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = localhost)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = pdb1_clone)
    )
  )
```

```bash
tnsping PDB1
tnsping ORCL
tnsping PDB1_CLONE
```
> **Nota:** Verifica que los pings y conexiones respondan correctamente.

<br/><br/>

### Tarea 3. Crear la Pluggable Database PDB1_CLONE

```sql
sqlplus / as sysdba
CREATE PLUGGABLE DATABASE pdb1_clone FROM pdb1
FILE_NAME_CONVERT = ('/u01/app/oracle/oradata/ORCL/pdb1/', '/u01/app/oracle/oradata/ORCL/pdb1_clone/');

show pdbs;

```

> **Nota:** El directorio de la nueva pluggable database se crea automaticamente`.

<br/><br/>

### Tarea 4. Abrir la nueva pluggable database
```sql
ALTER PLUGGABLE DATABASE pdb1_clone OPEN;
ALTER SESSION SET CONTAINER=pdb1_clone;

SELECT name, restricted FROM v$pdbs WHERE name = 'PDB1_CLONE';

SELECT name, pdb, network_name
FROM cdb_services
WHERE pdb = 'PDB1_CLONE'
ORDER BY name;
```

> **Nota:** La nueva PDB esta lista para ser usada.
<br/><br/>

### Tarea 5. Validar Independencia de Datos entre PDB Origen y Clonada

```sql
SELECT COUNT(*) FROM testuser.sales_data;

UPDATE testuser.sales_data
SET customer_name = 'CLONED_' || customer_name
WHERE ROWNUM <= 1000;
COMMIT;

SELECT customer_name FROM testuser.sales_data WHERE ROWNUM <= 5;

ALTER SESSION SET CONTAINER = PDB1;
SELECT customer_name FROM testuser.sales_data WHERE ROWNUM <= 5;
```

> **Nota:** Los datos modificados solo deben reflejarse en la PDB clonada.

<br/><br/>

### Tarea 6. Renombrar la PDB Clonada (Opcional)

```sql
CONNECT / AS SYSDBA
ALTER PLUGGABLE DATABASE PDB1_CLONE CLOSE IMMEDIATE;
ALTER PLUGGABLE DATABASE PDB1_COPY OPEN RESTRICTED;
ALTER SESSION SET CONTAINER = PDB1_CLONE;
ALTER PLUGGABLE DATABASE PDB1_CLONE RENAME GLOBAL_NAME TO PDB1_COPY;
ALTER PLUGGABLE DATABASE PDB1_COPY CLOSE IMMEDIATE;
ALTER PLUGGABLE DATABASE PDB1_COPY OPEN READ WRITE;

SELECT pdb_name, status FROM cdb_pdbs WHERE pdb_name = 'PDB1_COPY';
SELECT name, open_mode FROM v$pdbs WHERE name = 'PDB1_COPY';
```

> **Nota:** Debe aparecer con el nuevo nombre `PDB1_COPY`.

<br/><br/>

### Tarea 12. Documentar Consumo de Recursos Durante la Clonación

```sql
CONNECT / AS SYSDBA

SELECT
    operation, status, start_time, end_time,
    ROUND((end_time - start_time) * 24 * 60, 2) AS duration_minutes,
    ROUND(mbytes_processed, 2) AS mb_processed,
    ROUND(mbytes_processed / NULLIF((end_time - start_time) * 24 * 60, 0), 2) AS mb_per_minute
FROM v$rman_status
WHERE operation = 'DUPLICATE'
    AND start_time > SYSDATE - 1
ORDER BY start_time DESC
FETCH FIRST 5 ROWS ONLY;

SELECT name, value, ROUND(value/1024/1024, 2) AS value_mb
FROM v$sysstat
WHERE name LIKE '%memory%' AND value > 0
ORDER BY value DESC
FETCH FIRST 10 ROWS ONLY;
```

> **Nota:** Compara el consumo de recursos entre la PDB original y la clonada.

<br/><br/>

## **Limpieza del Ambiente (Opcional)**

```sql
CONNECT / AS SYSDBA
ALTER PLUGGABLE DATABASE PDB1_COPY CLOSE IMMEDIATE;
DROP PLUGGABLE DATABASE PDB1_COPY INCLUDING DATAFILES;

SELECT pdb_name, status FROM cdb_pdbs WHERE pdb_name = 'PDB1_COPY';
```

```bash
ls -l $ORACLE_BASE/oradata/ORCL/pdb1_clone/
rm -rf $ORACLE_BASE/oradata/ORCL/pdb1_clone/
```

> **Nota:** Si se desea conservar la PDB, solo cerrarla:

```sql
ALTER PLUGGABLE DATABASE PDB1_COPY CLOSE IMMEDIATE;
SELECT name, open_mode FROM v$pdbs WHERE name = 'PDB1_COPY';
```

<br/><br/>

> **Nota:**
> La salida mostrada en esta práctica, incluyendo los valores de **identificadores (`CON_ID`, `PDB_ID`)**, los nombres de los **canales RMAN**, los tiempos de ejecución y los **tamaños de los datafiles**, puede variar ligeramente según el entorno del participante.
> Estas diferencias dependen del **hardware**, la **configuración de la CDB/PDB**, la **capacidad de almacenamiento**, y los **recursos asignados** en la máquina utilizada durante el curso.
> Por lo tanto, los resultados observados deben considerarse **referenciales**, no exactos, siempre que las operaciones se completen sin errores y se valide la creación correcta de la PDB clonada.
