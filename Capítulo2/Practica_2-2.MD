# Práctica 2.2 Crear y probar servicios locales y comunes para una PDB



<br/><br/>

## Objetivos

1. Crear servicios locales en PDB con `DBMS_SERVICE` y HA (failover/CLB).
2. Publicar un servicio **común** desde `CDB$ROOT`.
3. Validar registro dinámico en listener con `lsnrctl services` y logs.
4. Probar conectividad por **TNS** y **EZCONNECT**.
5. Diagnosticar problemas con **tnsping**, listener.log y `v$session`.

<br/><br/>

## Tiempo estimado
- 60 minutos

<br/><br/>

## Prerrequisitos (breve)

* CDB 19c con ≥1 PDB abierta; listener activo; usuario `oracle`; privilegios `SYSDBA`.
* `ORACLE_HOME`/`ORACLE_SID` configurados; SQL*Plus, lsnrctl, tnsping disponibles.

<br/><br/>

## Tabla de ayuda

| ¿Para qué sirve?      | Ejemplo rápido                                                                                                                                                            |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Ver estado CDB/PDBs   | `SELECT name,cdb,open_mode FROM v$database; SELECT con_id,name,open_mode FROM v$pdbs;`                                                                                    |
| Servicios registrados | `SELECT con_id,name,network_name FROM v$services ORDER BY 1,2;`                                                                                                           |
| Servicios activos     | `SELECT name,pdb FROM v$active_services ORDER BY 1;`                                                                                                                      |
| Crear servicio        | `BEGIN DBMS_SERVICE.CREATE_SERVICE('sales_app','sales_app.example.com'); END; /`                                                                                          |
| Modificar (HA)        | `BEGIN DBMS_SERVICE.MODIFY_SERVICE('sales_app',failover_method=>'BASIC',failover_type=>'SELECT',clb_goal=>DBMS_SERVICE.CLB_GOAL_SHORT,aq_ha_notifications=>TRUE); END; /` |
| Iniciar/Detener       | `BEGIN DBMS_SERVICE.START_SERVICE('sales_app'); END; /` / `...STOP_SERVICE...`                                                                                            |
| Listener servicios    | `lsnrctl services LISTENER`                                                                                                                                               |
| EZCONNECT             | `sqlplus usr/pwd@host:1521/sales_app.example.com`                                                                                                                         |
| TNS (tnsnames.ora)    | Alias con `SERVICE_NAME=sales_app.example.com`                                                                                                                            |
| Sesiones por servicio | `SELECT username,service_name FROM v$session WHERE username='SALES_USER';`                                                                                                |

<br/><br/>

## Instrucciones

### Tarea 1. Preparación.- Desde la cuenta oracle verifica las variables de medio ambiente.

```bash
env|grep ORA
sqlplus -v
lsnrctl status LISTENER
-- Si el listener no esta activo ejecutar el comando
lsnrctl start LISTENER
```

```sql
-- Verificar las CDB y PDBs
sqlplus / as sysdba
SELECT name,open_mode,cdb FROM v$database;
SELECT con_id,name,open_mode FROM v$pdbs ORDER BY con_id;
```

<br/><br/>

### Tarea 2. Servicios **locales** en PDB1

```sql
-- Ir a PDB1
ALTER SESSION SET CONTAINER=pdb1;
SHOW CON_NAME;

-- Servicios existentes
SELECT name,network_name,pdb FROM dba_services ORDER BY name;

-- 1) Crear SALES_APP
BEGIN
  DBMS_SERVICE.CREATE_SERVICE(
    service_name => 'sales_app',
    network_name => 'sales_app.example.com'
  );
  DBMS_SERVICE.MODIFY_SERVICE(
    service_name          => 'sales_app',
    aq_ha_notifications   => TRUE,
    failover_method       => 'BASIC',
    failover_type         => 'SELECT',
    failover_retries      => 180,
    failover_delay        => 5,
    clb_goal              => DBMS_SERVICE.CLB_GOAL_SHORT,
    dtp                   => TRUE
  );
  DBMS_SERVICE.START_SERVICE('sales_app');
END;
/

-- 2) Crear REPORTS_APP (consultas largas)
BEGIN
  DBMS_SERVICE.CREATE_SERVICE(
    service_name => 'reports_app',
    network_name => 'reports_app.example.com'
  );
  DBMS_SERVICE.MODIFY_SERVICE(
    service_name          => 'reports_app',
    aq_ha_notifications   => TRUE,
    failover_method       => 'BASIC',
    failover_type         => 'SELECT',
    clb_goal              => DBMS_SERVICE.CLB_GOAL_LONG
  );
  DBMS_SERVICE.START_SERVICE('reports_app');
END;
/

-- Ver servicios en PDB1
SELECT name,network_name,clb_goal FROM v$services
WHERE pdb='PDB1' AND name IN ('sales_app','reports_app');
```

```bash
# Listener registra dinámicamente (espera breve)
sleep 10
lsnrctl services LISTENER | grep -E "sales_app|reports_app" -A 4
```

<br/><br/>

### Tarea 3. Conectividad TNS y EZCONNECT

**tnsnames.ora**

```
SALES_APP =
  (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=localhost)(PORT=1521))
    (CONNECT_DATA=(SERVER=DEDICATED)(SERVICE_NAME=sales_app.example.com)))

REPORTS_APP =
  (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=localhost)(PORT=1521))
    (CONNECT_DATA=(SERVER=DEDICATED)(SERVICE_NAME=reports_app.example.com)))

SALES_APP_FO =
  (DESCRIPTION=
    (ADDRESS_LIST=(ADDRESS=(PROTOCOL=TCP)(HOST=localhost)(PORT=1521)))
    (CONNECT_DATA=(SERVER=DEDICATED)(SERVICE_NAME=sales_app.example.com)
      (FAILOVER_MODE=(TYPE=SELECT)(METHOD=BASIC)(RETRIES=180)(DELAY=5))))
```

```bash
tnsping SALES_APP ; tnsping REPORTS_APP ; tnsping SALES_APP_FO
```

```sql
-- Usuario de prueba en PDB1 (si no existe)
ALTER SESSION SET CONTAINER=pdb1;
CREATE USER sales_user IDENTIFIED BY "oracle_4U";
GRANT CONNECT, RESOURCE, UNLIMITED TABLESPACE TO sales_user;
```

```bash
# Con TNS
echo "SELECT sys_context('USERENV','SERVICE_NAME') sn, sys_context('USERENV','CON_NAME') cn FROM dual;" \
| sqlplus -S sales_user/oracle_4U@SALES_APP

# Con EZCONNECT
echo "SELECT 'OK' FROM dual;" \
| sqlplus -S sales_user/oracle_4U@localhost:1521/reports_app.example.com
```

<br/><br/>

### Tarea 4. Servicio **común** desde `CDB$ROOT`

```sql
ALTER SESSION SET CONTAINER=CDB$ROOT; SHOW CON_NAME;

BEGIN
  DBMS_SERVICE.CREATE_SERVICE(
    service_name => 'common_monitoring',
    network_name => 'common_monitoring.example.com'
  );
  DBMS_SERVICE.MODIFY_SERVICE(
    service_name => 'common_monitoring',
    aq_ha_notifications => TRUE,
    clb_goal => DBMS_SERVICE.CLB_GOAL_SHORT
  );
  DBMS_SERVICE.START_SERVICE('common_monitoring');
END;
/

-- Ver en CDB
SELECT con_id,name,network_name,pdb FROM cdb_services
WHERE name='COMMON_MONITORING';
```

**tnsnames.ora (agregar)**

```
COMMON_MONITORING =
  (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=localhost)(PORT=1521))
    (CONNECT_DATA=(SERVER=DEDICATED)(SERVICE_NAME=common_monitoring.example.com)))
```

```bash
tnsping COMMON_MONITORING
```

<br/>

### Tarea 5. Monitoreo y diagnóstico

```sql
-- Consolidado
SELECT con_id,name,network_name,pdb,creation_date
FROM cdb_services
WHERE name IN ('SALES_APP','REPORTS_APP','COMMON_MONITORING')
ORDER BY con_id,name;

-- Sesiones por servicio
SELECT s.username, s.service_name, s.machine, s.program, s.status
FROM v$session s
WHERE s.username IN ('SALES_USER') ORDER BY s.service_name;
```

```bash
# Logs del listener
cd $ORACLE_BASE/diag/tnslsnr/$(hostname)/listener/trace
grep -i "service_register\|sales_app\|reports_app\|common_monitoring" listener.log | tail -20
```

<br/>

### Tarea 6. Ciclo de vida y arranque automático

```sql
-- Detener / Iniciar
ALTER SESSION SET CONTAINER=pdb1;
BEGIN DBMS_SERVICE.STOP_SERVICE('reports_app'); END; /
BEGIN DBMS_SERVICE.START_SERVICE('reports_app'); END; /
SELECT name,enabled FROM v$services WHERE name='REPORTS_APP';

-- Trigger de arranque automático en PDB1
CREATE OR REPLACE TRIGGER start_pdb_services
  AFTER STARTUP ON pdb1.DATABASE
BEGIN
  DBMS_SERVICE.START_SERVICE('sales_app');
  DBMS_SERVICE.START_SERVICE('reports_app');
END;
/

-- Probar: cerrar/abrir PDB1
ALTER SESSION SET CONTAINER=CDB$ROOT;
ALTER PLUGGABLE DATABASE pdb1 CLOSE IMMEDIATE;
ALTER PLUGGABLE DATABASE pdb1 OPEN;

ALTER SESSION SET CONTAINER=pdb1;
SELECT name,enabled FROM v$services WHERE name IN ('SALES_APP','REPORTS_APP');
```

<br/><br/>

## Resultado esperado

* Servicios **SALES_APP** (CLB_SHORT, HA) y **REPORTS_APP** (CLB_LONG) **activos** en **PDB1**.
* Servicio **COMMON_MONITORING** iniciado desde **ROOT** y resolviendo por TNS.
* `tnsping` y conexiones **TNS/EZCONNECT** exitosas.
* Listener registra los servicios en `lsnrctl services` y en `listener.log`.
* Trigger de arranque automático **habilitado** y funcional tras reabrir PDB1.

<br/><br/>

## Verificación final (checklist)

```bash
# 1) Listener publica servicios (>=3)
lsnrctl services LISTENER | grep -c -E "sales_app|reports_app|common_monitoring"

# 2) TNS/EZCONNECT OK
echo "SELECT 'OK' FROM dual;" | sqlplus -S sales_user/oracle_4U@SALES_APP
echo "SELECT 'OK' FROM dual;" | sqlplus -S sales_user/oracle_4U@localhost:1521/reports_app.example.com
tnsping COMMON_MONITORING
```

```sql
-- 3) Estado de servicios
SELECT name,network_name,pdb,enabled FROM v$services
WHERE name IN ('SALES_APP','REPORTS_APP') ORDER BY 1;

-- 4) Servicio común visible en CDB
SELECT con_id,name,network_name,pdb FROM cdb_services
WHERE name='COMMON_MONITORING';

-- 5) Sesiones de prueba (si conectaste)
SELECT username,service_name FROM v$session
WHERE username='SALES_USER' ORDER BY 2;
```

<br/><br/>

## Referencias oficiales

* **DBMS_SERVICE** (creación/atributos/HA) — *PL/SQL Packages & Types*
* **Vistas** `V$SERVICES`, `V$ACTIVE_SERVICES`, `CDB_SERVICES`, `DBA_SERVICES` — *Reference 19c*
* **Oracle Net Services** — *Administering Listener and Naming Methods*
