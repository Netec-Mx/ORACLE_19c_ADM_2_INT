 
# Práctica 3.4 RMAN DUPLICATE de CDB hacia entorno de pruebas (Opcional)
 
 <br/><br/>
 
## Objetivos
 
 * Crear y configurar una instancia Oracle auxiliar completa, incluyendo *pfile*, *password file* y configuración de *listener* para operaciones **RMAN DUPLICATE**.
 * Ejecutar un proceso completo de **RMAN DUPLICATE TARGET DATABASE** utilizando transferencia activa (**FROM ACTIVE DATABASE**) con conversión automática de rutas de archivos.
 * Validar y ajustar la configuración post-duplicación de la **CDB** y todas sus **PDBs**, incluyendo parámetros de instancia, servicios y componentes dependientes del entorno.
 * Documentar y automatizar el proceso de *refresh* periódico de ambientes de desarrollo mediante scripts reutilizables y *runbooks* operativos.
 * Resolver problemas comunes durante operaciones de duplicación relacionados con conectividad, espacio en disco y configuración de red.
 
 <br/><br/>
 
## Tiempo estimado
 
 - 60 minutos 
 
 <br/><br/>
 
## Objetivo visual
 
 ![diagrama1](../images/img4-1.png)
 
 <br/><br/>
 
## Tabla de ayuda
 
 
 | **Comando**                                                                  | **Descripción**                                                                               |
 | ---------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
 | `orapwd file=orapwCDBTEST password=Oracle123 entries=10 format=12`           | Crea el archivo de contraseñas necesario para autenticar al usuario SYS en la instancia auxiliar.  |
 | `cat > $ORACLE_HOME/dbs/initCDBTEST.ora`                                     | Genera el archivo **PFILE** con los parámetros mínimos para iniciar la instancia auxiliar.         |
 | `lsnrctl reload`                                                             | Recarga el listener tras modificar su configuración (`listener.ora`).                              |
 | `rman target sys/Oracle123@ORCL auxiliary sys/Oracle123@CDBTEST`             | Inicia sesión en RMAN conectando simultáneamente la base origen y la auxiliar.                     |
 | `DUPLICATE TARGET DATABASE TO CDBTEST FROM ACTIVE DATABASE NOFILENAMECHECK;` | Realiza la duplicación en línea de la base de datos origen a la auxiliar sin usar backups previos. |
 | `CREATE SPFILE='/u02/oradata/CDBTEST/spfileCDBTEST.ora' FROM MEMORY;`        | Crea el **SPFILE** definitivo tomando la configuración actual en memoria.                          |
 | `DBMS_SCHEDULER.DISABLE(name => job_name, force => TRUE);`                   | Deshabilita tareas automáticas en el ambiente duplicado para evitar ejecuciones en pruebas.        |
 | `ALTER SYSTEM REGISTER;`                                                     | Registra los servicios de la nueva CDB/PDB en el listener de red.                                  |
 | `find /u02/oradata/CDBTEST -type f -name "*.dbf"`                            | Verifica la existencia física de los datafiles generados por la duplicación.                       |
 
 
 
 <br/><br/>
 
## **Configuración inicial**
 
 Antes de comenzar, valida el estado del ambiente:
 
 ```bash
 # Validar que la CDB origen existe y está operativa
 export ORACLE_SID=ORCL
 export ORACLE_HOME=/u01/app/oracle/product/19c/dbhome_1
 export PATH=$ORACLE_HOME/bin:$PATH
 ```
 
 ```sql
 sqlplus / as sysdba <<EOF
 SELECT name, open_mode, cdb FROM v$database;
 SELECT name, open_mode FROM v$pdbs;
 EXIT;
 EOF
 ```
 
 Validar espacio disponible en disco:
 
 ```bash
 df -h /u01 /u02
 ```
 
 > **Requerimiento mínimo:** 60 GB libres en alguno de los filesystems `/u01` o `/u02`.
 
 <br/><br/>
 
## Instrucciones
 
### Tarea 1. Preparación del Entorno Auxiliar
 
#### **Paso 1.** Crear estructura de directorios para la CDB auxiliar
 
 ```bash
 # Crear directorios base para la nueva instancia CDBTEST
 mkdir -p /u02/oradata/CDBTEST
 mkdir -p /u02/oradata/CDBTEST/pdbseed
 mkdir -p /u02/oradata/CDBTEST/pdb1
 mkdir -p /u02/oradata/CDBTEST/pdb2
 mkdir -p /u02/fast_recovery_area/CDBTEST
 mkdir -p /u02/admin/CDBTEST/adump
 ```
 
 ```bash
 # Verificar permisos
 ls -ld /u02/oradata/CDBTEST
 chown -R oracle:oinstall /u02/oradata/CDBTEST /u02/fast_recovery_area/CDBTEST /u02/admin/CDBTEST
 ```
 
 
#### **Paso 2.** Crear el password file para la instancia auxiliar
 
 ```bash
 cd $ORACLE_HOME/dbs
 
 # Crear password file (usar la misma contraseña que la CDB origen)
 orapwd file=orapwCDBTEST password=Oracle123 entries=10 format=12
 
 # Verificar creación
 ls -lh orapwCDBTEST
 ```
 
 
#### **Paso 3.** Crear el archivo PFILE inicial
 
 ```bash
 cat > $ORACLE_HOME/dbs/initCDBTEST.ora <<'EOF'
 # Parámetros mínimos para RMAN DUPLICATE
 db_name=CDBTEST
 db_unique_name=CDBTEST
 db_block_size=8192
 sga_target=4G
 pga_aggregate_target=1G
 processes=300
 db_create_file_dest=/u02/oradata/CDBTEST
 db_recovery_file_dest=/u02/fast_recovery_area/CDBTEST
 db_recovery_file_dest_size=20G
 audit_file_dest=/u02/admin/CDBTEST/adump
 control_files=('/u02/oradata/CDBTEST/control01.ctl','/u02/fast_recovery_area/CDBTEST/control02.ctl')
 undo_tablespace=UNDOTBS1
 enable_pluggable_database=TRUE
 compatible=19.0.0
 diagnostic_dest=/u02/admin
 EOF
 ```
 
 ```bash
 # Verificar contenido
 cat $ORACLE_HOME/dbs/initCDBTEST.ora
 ```
 
 
#### **Paso 4.** Configurar el listener para la instancia auxiliar
 
 ```bash
 # Agregar entrada al listener.ora para CDBTEST
 cat >> $ORACLE_HOME/network/admin/listener.ora <<'EOF'
 
 SID_LIST_LISTENER =
   (SID_LIST =
     (SID_DESC =
       (GLOBAL_DBNAME = CDBTEST)
       (ORACLE_HOME = /u01/app/oracle/product/19c/dbhome_1)
       (SID_NAME = CDBTEST)
     )
   )
 EOF
 ```
 
 ```bash
 # Recargar y validar listener
 lsnrctl reload
 lsnrctl status | grep -A 2 CDBTEST
 ```
 
 
#### **Paso 5. Configurar el archivo tnsnames.ora**
 
 ```bash
 cat >> $ORACLE_HOME/network/admin/tnsnames.ora <<'EOF'
 CDBTEST =
   (DESCRIPTION =
     (ADDRESS = (PROTOCOL = TCP)(HOST = localhost)(PORT = 1521))
     (CONNECT_DATA =
       (SERVER = DEDICATED)
       (SERVICE_NAME = CDBTEST)
     )
   )
 
 ORCL =
   (DESCRIPTION =
     (ADDRESS = (PROTOCOL = TCP)(HOST = localhost)(PORT = 1521))
     (CONNECT_DATA =
       (SERVER = DEDICATED)
       (SERVICE_NAME = ORCL)
     )
   )
 EOF
 ```
 
 ```bash
 # Verificar configuración
 cat $ORACLE_HOME/network/admin/tnsnames.ora | grep -A 7 "CDBTEST ="
 ```
 
 
#### **Paso 6.** Iniciar la instancia auxiliar en modo NOMOUNT
 
 ```bash
 export ORACLE_SID=CDBTEST
 
 sqlplus / as sysdba <<EOF
 STARTUP NOMOUNT PFILE='$ORACLE_HOME/dbs/initCDBTEST.ora';
 SHOW PARAMETER db_name;
 SHOW PARAMETER control_files;
 EXIT;
 EOF
 ```
 
 <br/><br/>
 
### Tarea 2. Ejecución de RMAN DUPLICATE
 
#### **Paso 1.** Validar conectividad RMAN a ambas instancias
 
 ```bash
 export ORACLE_SID=ORCL
 
 rman <<EOF
 CONNECT TARGET sys/Oracle123@ORCL
 CONNECT AUXILIARY sys/Oracle123@CDBTEST
 SHOW ALL;
 EXIT;
 EOF
 ```
 
 
#### **Paso 2.** Crear el script de duplicación
 
 ```bash
 cat > /home/oracle/duplicate_cdb.rman <<'EOF'
 # Script RMAN para duplicar ORCL a CDBTEST
 CONNECT TARGET sys/Oracle123@ORCL
 CONNECT AUXILIARY sys/Oracle123@CDBTEST
 
 RUN {
   ALLOCATE CHANNEL c1 DEVICE TYPE DISK;
   ALLOCATE CHANNEL c2 DEVICE TYPE DISK;
   ALLOCATE AUXILIARY CHANNEL aux1 DEVICE TYPE DISK;
   ALLOCATE AUXILIARY CHANNEL aux2 DEVICE TYPE DISK;
 
   DUPLICATE TARGET DATABASE TO CDBTEST
     FROM ACTIVE DATABASE
     SPFILE
       SET db_unique_name='CDBTEST'
       SET db_name='CDBTEST'
       SET db_create_file_dest='/u02/oradata/CDBTEST'
       SET db_recovery_file_dest='/u02/fast_recovery_area/CDBTEST'
       SET db_recovery_file_dest_size='20G'
       SET control_files='/u02/oradata/CDBTEST/control01.ctl','/u02/fast_recovery_area/CDBTEST/control02.ctl'
       SET audit_file_dest='/u02/admin/CDBTEST/adump'
       SET diagnostic_dest='/u02/admin'
       SET log_archive_dest_1='LOCATION=/u02/fast_recovery_area/CDBTEST/arch'
     NOFILENAMECHECK;
 
   RELEASE CHANNEL c1;
   RELEASE CHANNEL c2;
   RELEASE CHANNEL aux1;
   RELEASE CHANNEL aux2;
 }
 EOF
 ```
 
 ```bash
 # Verificar script
 cat /home/oracle/duplicate_cdb.rman
 ```
 
 
#### **Paso 3.** Ejecutar duplicación con logging
 
 ```bash
 # Ejecutar el proceso de duplicación
 rman @/home/oracle/duplicate_cdb.rman LOG /home/oracle/duplicate_cdb.log
 
 # Monitorear progreso
 tail -f /home/oracle/duplicate_cdb.log
 ```
 
 
#### **Paso 4.** Verificar finalización exitosa
 
 ```bash
 tail -50 /home/oracle/duplicate_cdb.log | grep -i "finished duplicate"
 
 export ORACLE_SID=CDBTEST
 sqlplus / as sysdba <<EOF
 SELECT name, open_mode, cdb, created FROM v$database;
 SELECT instance_name, status FROM v$instance;
 EXIT;
 EOF
 ```
 
 <br/><br/>
 

### Tarea 3. Validación y Configuración Post-Duplicación
 
#### **Paso 1.** Abrir todas las PDBs
 
 ```sql
 sqlplus / as sysdba <<EOF
 SELECT name, open_mode, restricted FROM v$pdbs;
 ALTER PLUGGABLE DATABASE ALL OPEN;
 SELECT name, open_mode, restricted FROM v$pdbs;
 EXIT;
 EOF
 ```
 
 
 
#### **Paso 2.** Configurar apertura automática
 
 ```sql
 sqlplus / as sysdba <<EOF
 ALTER PLUGGABLE DATABASE ALL SAVE STATE;
 SELECT con_name, state FROM cdb_pdb_saved_states;
 EXIT;
 EOF
 ```
 
 
 
#### **Paso 3.** Validar integridad de datos
 
 ```sql
 sqlplus sys/Oracle123@localhost:1521/PDB1 as sysdba <<EOF
 SELECT owner, object_type, COUNT(*)
 FROM dba_objects
 WHERE owner NOT IN ('SYS','SYSTEM','OUTLN','ORACLE_OCM','XDB','APEX_050000','MDSYS','CTXSYS')
 GROUP BY owner, object_type
 ORDER BY owner, object_type;
 
 SELECT tablespace_name, status, contents FROM dba_tablespaces;
 EXIT;
 EOF
 ```
 
 
#### **Paso 4.** Ajustar configuraciones del entorno duplicado
 
 ```sql
 sqlplus / as sysdba <<EOF
 BEGIN
   FOR job IN (SELECT owner, job_name FROM cdb_scheduler_jobs WHERE enabled='TRUE') LOOP
     EXECUTE IMMEDIATE 'ALTER SESSION SET CONTAINER=' || job.owner;
     DBMS_SCHEDULER.DISABLE(name => job.job_name, force => TRUE);
   END LOOP;
 END;
 /
 
 ALTER SYSTEM SET job_queue_processes=0 SCOPE=BOTH;
 ALTER SYSTEM SET aq_tm_processes=0 SCOPE=BOTH;
 EXIT;
 EOF
 ```
 
 
#### **Paso 5.** Registrar servicios en el listener
 
 ```sql
 sqlplus / as sysdba <<EOF
 ALTER SYSTEM REGISTER;
 SHOW PARAMETER service_names;
 EXIT;
 EOF
 ```
 
 ```bash
 lsnrctl services | grep -i cdbtest
 ```
 
 
 
#### **Paso 6.** Crear SPFILE y validar parámetros
 
 ```sql
 sqlplus / as sysdba <<EOF
 CREATE SPFILE='/u02/oradata/CDBTEST/spfileCDBTEST.ora' FROM MEMORY;
 
 SELECT name, value FROM v$parameter
 WHERE name IN ('db_name','db_unique_name','db_create_file_dest','db_recovery_file_dest','control_files')
 ORDER BY name;
 EXIT;
 EOF
 ```
 
 
 
#### **Paso 7.** Validar archivos físicos del CDB duplicado
 
 ```sql
 sqlplus / as sysdba <<EOF
 SET LINESIZE 200
 COLUMN name FORMAT A70
 SELECT name, bytes/1024/1024 AS size_mb, con_id FROM v$datafile ORDER BY con_id, name;
 EXIT;
 EOF
 ```
 
 ```bash
 find /u02/oradata/CDBTEST -type f -name "*.dbf" | sort
 ```
 
 <br/><br/>
 
 
 
## Resultado esperado
 
 * Todos los comandos SQL, RMAN y Unix son sintácticamente válidos para Oracle 19c.
 * Se usa `FROM ACTIVE DATABASE` con `NOFILENAMECHECK`, por lo que no requiere backups previos.
 * La duplicación creará automáticamente la CDB y sus PDBs, configurando parámetros independientes.
 
 <br/><br/>
 
 ### Verificación del ambiente origen
 
 ```sql
 SQL> SELECT name, open_mode, cdb FROM v$database;
 
 NAME      OPEN_MODE            CDB
 --------- -------------------- ---
 CDB1      READ WRITE           YES
 
 SQL> SELECT name, open_mode FROM v$pdbs;
 
 NAME            OPEN_MODE
 --------------- --------------------
 PDB$SEED        READ ONLY
 PDB1            READ WRITE
 PDB1_CLONE      READ WRITE
 ```
 
 <br/><br/>
 
 ### Validación de espacio en disco
 
 ```bash
 $ df -h /u01 /u02
 Filesystem      Size  Used Avail Use% Mounted on
 /dev/sda1        80G   12G   68G  15% /u01
 /dev/sdb1       100G   35G   65G  35% /u02
 ```
 
 <br/><br/>
 
 ### Creación del password file y PFILE
 
 ```bash
 $ ls -lh $ORACLE_HOME/dbs/orapwCDBTEST
 -rw-r----- 1 oracle oinstall 1.5K Nov 09 18:35 orapwCDBTEST
 
 $ head -5 $ORACLE_HOME/dbs/initCDBTEST.ora
 # Parámetros mínimos para RMAN DUPLICATE
 db_name=CDBTEST
 db_unique_name=CDBTEST
 db_block_size=8192
 sga_target=4G
 ```
 
 <br/><br/>
 
 ### Validación del listener y TNS
 
 ```bash
 $ lsnrctl reload
 Listener reload completed successfully.
 
 $ lsnrctl status | grep -A 2 CDBTEST
 Service "CDBTEST" has 1 instance(s).
   Instance "CDBTEST", status UNKNOWN, has 1 handler(s).
 ```
 
 <br/><br/>
 
 ### Conectividad RMAN
 
 ```bash
 $ rman
 RMAN> CONNECT TARGET sys/Oracle123@CDB1
 connected to target database: CDB1 (DBID=123456789)
 
 RMAN> CONNECT AUXILIARY sys/Oracle123@CDBTEST
 connected to auxiliary database: CDBTEST (not mounted)
 ```
 
 <br/><br/>
 
 ### Ejecución del RMAN DUPLICATE
 
 ```text
 RMAN> @/home/oracle/duplicate_cdb.rman
 ...
 Starting Duplicate Db at 09-NOV-25
 ...
 channel aux1: starting datafile copy
 input datafile file number=1 name=/u01/oradata/CDB1/system01.dbf
 output file name=/u02/oradata/CDBTEST/system01.dbf
 ...
 Finished Duplicate Db at 09-NOV-25
 ```
 
 ### Verificación posterior a la duplicación
 
 ```sql
 SQL> SELECT name, open_mode, cdb FROM v$database;
 
 NAME      OPEN_MODE            CDB
 --------- -------------------- ---
 CDBTEST   READ WRITE           YES
 
 SQL> SELECT name, open_mode FROM v$pdbs;
 
 NAME            OPEN_MODE
 --------------- --------------------
 PDB$SEED        READ ONLY
 PDB1            MOUNTED
 PDB1_CLONE      MOUNTED
 ```
 
 <br/><br/>
 
 ### Apertura y configuración automática de PDBs
 
 ```sql
 SQL> ALTER PLUGGABLE DATABASE ALL OPEN;
 
 SQL> SELECT name, open_mode FROM v$pdbs;
 
 NAME            OPEN_MODE
 --------------- --------------------
 PDB$SEED        READ ONLY
 PDB1            READ WRITE
 PDB1_CLONE      READ WRITE
 
 SQL> ALTER PLUGGABLE DATABASE ALL SAVE STATE;
 
 SQL> SELECT con_name, state FROM cdb_pdb_saved_states;
 
 CON_NAME   STATE
 ---------- ----------
 PDB1       OPEN
 PDB1_CLONE OPEN
 ```
 
 <br/><br/>
 
 ### Validación de objetos y tablespaces
 
 ```sql
 SQL> SELECT owner, object_type, COUNT(*)
   2  FROM dba_objects
   3  WHERE owner NOT IN ('SYS','SYSTEM','XDB','MDSYS','CTXSYS')
   4  GROUP BY owner, object_type
   5  ORDER BY owner, object_type;
 
 OWNER      OBJECT_TYPE     COUNT(*)
 ----------- --------------- --------
 HR          TABLE           7
 HR          INDEX           8
 HR          VIEW            2
 
 SQL> SELECT tablespace_name, status, contents FROM dba_tablespaces;
 
 TABLESPACE_NAME                STATUS    CONTENTS
 ------------------------------ --------- ---------
 SYSTEM                         ONLINE    PERMANENT
 SYSAUX                         ONLINE    PERMANENT
 UNDOTBS1                       ONLINE    UNDO
 USERS                          ONLINE    PERMANENT
 ```
 
 <br/><br/>
 
 ### Validación final de parámetros y archivos
 
 ```sql
 SQL> SELECT name, value
   2  FROM v$parameter
   3  WHERE name IN ('db_name','db_unique_name','control_files');
 
 NAME               VALUE
 ------------------ --------------------------------------------
 control_files      /u02/oradata/CDBTEST/control01.ctl
 db_name            CDBTEST
 db_unique_name     CDBTEST
 ```
 
 ```bash
 $ find /u02/oradata/CDBTEST -type f -name "*.dbf" | sort
 /u02/oradata/CDBTEST/system01.dbf
 /u02/oradata/CDBTEST/sysaux01.dbf
 /u02/oradata/CDBTEST/undotbs01.dbf
 /u02/oradata/CDBTEST/users01.dbf
 /u02/oradata/CDBTEST/pdbseed/system01.dbf
 /u02/oradata/CDBTEST/pdb1/system01.dbf
 /u02/oradata/CDBTEST/pdb1_clone/system01.dbf
 ```
 
 > **Nota importante:**
 Los nombres, IDs, tamaños y fechas de creación pueden variar según la máquina usada en curso.
 
 Lo relevante es confirmar que:
 
 * El **CDBTEST** esté en modo **READ WRITE**
 * Las **PDBs** estén abiertas y guardadas con **SAVE STATE**
 * Los **datafiles** físicos existan en `/u02/oradata/CDBTEST/`
 * El listener muestre los servicios **CDBTEST** y sus PDBs registrados.
 
 