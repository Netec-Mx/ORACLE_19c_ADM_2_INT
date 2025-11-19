
# Capítulo 6. Recuperación y Flashback

<br/><br/>

## **Práctica 6.1 Recuperación de una PDB usando Flashback Database**

En esta práctica aplicarás las capacidades de **Flashback Database** para recuperar una **Pluggable Database (PDB)** después de un error lógico. Habilitarás Flashback en la CDB, crearás un punto de restauración mediante el SCN, simularás la pérdida de datos y finalmente revertirás la PDB al estado anterior utilizando FLASHBACK PLUGGABLE DATABASE.  

[Práctica 6.1](Practica_6-1.MD)

<br/><br/>

## **Práctica 6.2 Restauración de la CDB Root ante la pérdida de un Datafile utilizando RMAN**

En esta práctica simularás un **fallo catastrófico** en la base de datos multitenant ocasionado por la pérdida de un **datafile crítico (SYSTEM) de la CDB Root**, y utilizarás **RMAN** para ejecutar una **restauración y recuperación completa**. Validarás los errores de arranque, iniciarás la CDB en modo *MOUNT*, restaurarás los datafiles perdidos desde un respaldo existente y aplicarás los *archived redo logs* necesarios hasta dejar la base de datos completamente funcional.

[Práctica 6.2](Practica_6-2.MD)
