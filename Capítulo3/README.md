# Capítulo 3. Respaldo y Duplicación

<br/><br/>

## **Práctica 3.1 Explorar configuración RMAN y registros de backup en CDB**

Explorarás la configuración de **RMAN** en un entorno multitenant (CDB), diferenciando su comportamiento frente a bases tradicionales. Configurarás políticas de retención, **FRA**, modo **Archivelog** y revisarás vistas del diccionario con metadata de respaldos.
[Práctica 3.1](Practica_3-1.MD)

<br/><br/>

## **Práctica 3.2 Backup completo de CDB y restauración de una PDB específica**

Implementarás una estrategia completa de **backup y recuperación** con RMAN en un CDB. Realizarás un respaldo total, simularás pérdida de datafiles en una PDB y restaurarás solo la afectada, preservando el resto del entorno.
[Práctica 3.2](Practica_3-2.MD)

<br/><br/>

## **Práctica 3.3 Active Duplicate de una PDB hacia la misma CDB**

Aprenderás a **clonar una PDB** dentro de la misma CDB usando **RMAN Active Duplicate**, creando una copia funcional sin respaldo previo. Configurarás la red, ejecutarás la clonación y validarás la independencia entre ambas PDBs.
[Práctica 3.3](Practica_3-3.MD)

<br/><br/>

## **Práctica 3.4 RMAN Duplicate de CDB hacia entorno de pruebas (opcional)**

Realizarás la **duplicación completa de un CDB** hacia un nuevo entorno con **RMAN Duplicate (Active Database)**. Crearás la instancia auxiliar, ejecutarás la copia y ajustarás la configuración para independizar el ambiente clonado.
[Práctica 3.4](Practica_3-4.MD)


