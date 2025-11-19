
# Capítulo 5. Gestión de la Seguridad en Bases de Datos Multitenant

<br/><br/>

## **Práctica 5.1 Gestión de Usuarios Comunes, Lockdown Profiles y Auditoría en PDBs**

En esta práctica realizarás un recorrido completo por los principales mecanismos de seguridad en un entorno Oracle Multitenant. Iniciarás creando un **usuario común** con alcance en todo el contenedor y una **PDB dedicada** con su respectivo usuario local para operaciones internas. Posteriormente implementarás un **PDB Lockdown Profile** para restringir comandos administrativos críticos dentro de la PDB y validarás su funcionamiento mediante pruebas directas. Finalmente, configurarás una política de **auditoría unificada** para registrar operaciones sensibles (como DELETE) y verificarás la captura de dichos eventos desde el repositorio central del CDB.


[Práctica 5.1](Practica_5-1.MD)

<br/><br/>

## **Práctica 5.2 Configuración de Auditoría Unificada en una PDB**

En esta práctica profundizarás en la **auditoría unificada a nivel de PDB**, aprendiendo a crear políticas específicas basadas en acciones DML o DDL ejecutadas dentro del contenedor pluggable. Configurarás una política de auditoría centrada en eventos DELETE, la habilitarás dentro de una PDB y realizarás operaciones controladas para generar registros auditables. Finalmente, consultarás los resultados tanto desde la vista local de la PDB como desde el contenedor raíz para validar la consolidación de auditorías. 

[Práctica 5.2](Practica_5-2.MD)






