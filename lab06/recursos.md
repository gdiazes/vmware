# GUÍA DE LABORATORIO 6: ARQUITECTURA EMPRESARIAL MULTI-SITIO Y ALTA DISPONIBILIDAD (vSAN, vDS y CROSS-vMOTION)

El diseño arquitectónico de este laboratorio colaborativo es ilustrado en la **Figura 1**, representando una topología distribuida geográficamente entre dos centros de datos.

**Figura 1**
*Arquitectura Empresarial VMware vSphere 8.0: Data Centers Lima y Arequipa*

<img width="1408" height="768" alt="image" src="https://github.com/user-attachments/assets/643aad91-fd75-4b09-8a1f-57ad84a22827" />


*Nota.* Elaboración propia. La imagen ilustra la topología objetivo del caso práctico, donde dos sitios físicos independientes son unificados bajo una misma capa de orquestación y red distribuida.

## 1. Escenario de Negocio (CBL - Aprendizaje Basado en Casos)
**Contexto Organizacional:**
Una de las instituciones financieras más grandes del país ha iniciado el "Proyecto Bicentenario". El objetivo es garantizar la continuidad del negocio ante desastres naturales. Se ha construido un Centro de Datos Primario en Lima (Site A) y un Centro de Datos Secundario en Arequipa (Site B).

**El Problema:**
Actualmente, ambos centros de datos operan como islas aisladas. Si un servidor en Lima falla críticamente, la recuperación en Arequipa requiere intervención manual, causando interrupciones inaceptables en los servicios financieros.

**El Reto Técnico (Misión del Equipo):**
Ustedes han sido contratados como un escuadrón de Arquitectos de Infraestructura (*Senior Architects*). Su misión es unificar ambos sitios. El Sitio de Lima será configurado con una arquitectura hiperconvergente mediante **vSAN** (utilizando los discos locales de los hosts). El Sitio de Arequipa utilizará almacenamiento tradicional externo (NAS/NFS). Finalmente, una red distribuida (**vDS**) deberá extenderse entre ambas ciudades para permitir que las máquinas virtuales sean migradas en caliente de Lima a Arequipa (*Cross-Site vMotion*) sin pérdida de conectividad ni cambio de direcciones IP.

## 2. Objetivos de la Práctica
*   Planificar y coordinar el despliegue de una topología unificada utilizando recursos de hardware distribuidos (Dos PCs físicas conectadas en LAN).
*   Desplegar un entorno hiperconvergente habilitando un **Clúster vSAN** de 3 nodos.
*   Configurar un **vSphere Distributed Switch (vDS)** que abarque hosts ubicados en diferentes clústeres lógicos.
*   Administrar un vCenter Server unificado para gobernar múltiples Centros de Datos virtuales (Datacenters lógicos).
*   Ejecutar una migración en caliente (*vMotion*) cruzando fronteras de almacenamiento y clúster (*Cross-Cluster/Cross-Datastore vMotion*).

## 3. Asignación de Roles y Requisitos de Hardware
Dado que la topología global supera los límites de un computador personal, la carga de virtualización anidada (*Nested Virtualization*) será balanceada entre dos estudiantes. Ambos computadores (PC-1 y PC-2) deben estar conectados al mismo conmutador físico (LAN del laboratorio) y contar con 32 GB de RAM cada uno.

**Estudiante 1 (Arquitecto Site A - Lima):**
*   **Recursos Asignados (PC-1):** 3 Máquinas Virtuales ESXi (`ESXi-L01`, `ESXi-L02`, `ESXi-L03`). Cada host configurado con 6 GB de RAM y dos discos virtuales (uno para sistema, otro para vSAN). Total de RAM consumida: ~18 GB.
*   **Responsabilidad:** Creación del Clúster de Lima y aprovisionamiento del *Datastore* hiperconvergente vSAN.

**Estudiante 2 (Arquitecto Site B - Arequipa y Gestión Global):**
*   **Recursos Asignados (PC-2):** 2 Máquinas Virtuales ESXi (`ESXi-A01`, `ESXi-A02`) con 6 GB de RAM cada una; 1 Máquina *TrueNAS* (NAS Appliance) con 4 GB de RAM; y 1 *vCenter Server Appliance* (VCSA - Despliegue *Tiny*) con 12 GB de RAM. Total de RAM consumida: ~28 GB.
*   **Responsabilidad:** Aprovisionamiento del almacenamiento externo, despliegue del orquestador central (vCenter) y creación del conmutador distribuido (vDS).

---

## 4. Instrucciones Paso a Paso (Ejecución Colaborativa)

### Fase 1: Despliegue de Nodos y Conectividad Base (Ambos Estudiantes)
*La base de cómputo debe ser establecida y las direcciones IP deben ser alcanzables entre ambos computadores físicos.*
1.  **(Estudiante 1):** En *VMware Workstation*, despliegue los tres hosts del Sitio A. Asigne direcciones IP estáticas secuenciales (Ej. `10.160.10.11`, `.12`, `.13`). Asegúrese de agregar un disco adicional de 100 GB a cada host, el cual será reclamado posteriormente por vSAN.
2.  **(Estudiante 2):** Despliegue los dos hosts del Sitio B (`10.160.10.21`, `.22`) y la máquina TrueNAS (`10.160.10.200`).
3.  **(Colaboración):** Desde la consola de comandos de un host en el Sitio A, un paquete ICMP (Ping) debe ser enviado hacia un host del Sitio B. Si la respuesta es exitosa, la conectividad inter-sitio (Backbone) está garantizada.

### Fase 2: Orquestación Centralizada (Estudiante 2 con apoyo de Estudiante 1)
*Un único punto de control será establecido para gobernar ambas ciudades.*
1.  **(Estudiante 2):** Monte la imagen ISO de vCenter Server Appliance (VCSA). Ejecute el instalador UI y despliegue la *appliance* con tamaño "Tiny" apuntando al host `ESXi-A01`. Asigne la IP `10.160.10.5` al vCenter.
2.  **(Estudiante 2):** Una vez que vCenter esté operativo, ingrese a la interfaz web (vSphere Client). Cree dos objetos de tipo "Datacenter": uno nombrado `Datacenter-Lima` y otro `Datacenter-Arequipa`.
3.  **(Estudiante 1):** Proporcione las credenciales `root` de sus tres hosts al Estudiante 2.
4.  **(Estudiante 2):** Añada los hosts `ESXi-L01`, `L02` y `L03` al `Datacenter-Lima`. Seguidamente, añada los hosts `ESXi-A01` y `A02` al `Datacenter-Arequipa`.

### Fase 3: Configuración de Almacenamiento Dispar (Roles Divididos)
*Cada centro de datos utilizará una arquitectura de persistencia de datos diferente.*
1.  **(Estudiante 1 - Lima):** En vCenter, cree un Clúster lógico dentro del `Datacenter-Lima` e introduzca sus tres hosts en él. En la configuración del clúster, habilite el servicio **vSAN**. Complete el asistente reclamando los discos vacíos de 100 GB de cada host para conformar un único *Datastore* hiperconvergente distribuido (`vsanDatastore`).
2.  **(Estudiante 2 - Arequipa):** Acceda a TrueNAS y exporte un recurso compartido NFS. En vCenter, cree un Clúster en el `Datacenter-Arequipa` e integre sus dos hosts. Vaya a la pestaña de almacenamiento de este clúster y monte el volumen NFS remoto (nombrado `NAS-Arequipa-NFS`).

### Fase 4: Despliegue de Red Distribuida vDS (Colaborativo)
*La Capa 2 de red será extendida a través del backbone para unificar ambas ciudades lógicamente.*
1.  **(Estudiante 2):** En vCenter, diríjase a la vista de Redes (*Networking*). A nivel global, cree un nuevo **vSphere Distributed Switch (vDS)** denominado `vDS-Inter-DC`.
2.  Cree un Grupo de Puertos Distribuido (*Distributed Port Group*) denominado `Red-Produccion-Global`.
3.  Haga clic derecho en el `vDS-Inter-DC` y seleccione **Add and Manage Hosts** (Agregar y administrar hosts).
4.  **(Colaboración):** Seleccione los 5 hosts totales (3 de Lima y 2 de Arequipa). Asigne al menos un enlace físico (*vmnic*) de cada host como Uplink del vDS. Finalice el asistente. La red ha sido extendida exitosamente entre ambas sedes.

### Fase 5: Validación Operativa y Migración (Cross-Site vMotion)
*El escenario de negocio será resuelto moviendo una carga crítica entre ciudades sin apagarla.*
1.  **(Estudiante 1):** Cree una máquina virtual de prueba (`VM-APP-Finanzas`) en el Clúster de Lima. Alójela en el `vsanDatastore` y conéctela al grupo de puertos distribuido `Red-Produccion-Global`.
2.  Encienda la máquina e inicie un comando `ping` continuo desde su PC física hacia la IP de esta VM.
3.  **(Colaboración):** Suponga que el Sitio A (Lima) experimentará un corte eléctrico inminente programado. Haga clic derecho sobre `VM-APP-Finanzas` y seleccione **Migrate** (Migrar).
4.  Seleccione **Change both compute resource and storage** (Cambiar tanto recurso de cómputo como almacenamiento).
5.  Como destino de cómputo, seleccione un host del Clúster de Arequipa (Ej. `ESXi-A01`). Como destino de almacenamiento, seleccione el `NAS-Arequipa-NFS`.
6.  Finalice el asistente. Observe la ventana de *Recent Tasks* mientras la máquina es transferida a través de la red física del laboratorio hacia la PC del Estudiante 2, manteniendo el `ping` ininterrumpido.

---

## 5. Criterio de Validación y Evidencias
Para cumplir con el objetivo formativo de esta sesión integradora, las siguientes evidencias deberán ser generadas y presentadas conjuntamente por el equipo:
1.  **Auditoría de Inventario (vCenter):** Una captura del cliente vSphere mostrando el inventario completo, donde se evidencien los dos Datacenters (Lima y Arequipa) poblados con sus respectivos clústeres y hosts.
2.  **Auditoría de Almacenamiento (vSAN y NFS):** Una captura de la vista de almacenamiento evidenciando que el `vsanDatastore` cuenta con una capacidad consolidada (sumatoria de los 3 discos de los hosts) y que el `NAS-Arequipa-NFS` se encuentra montado.
3.  **Topología de Red (vDS):** Una captura de la vista *Topology* del conmutador distribuido, demostrando que hosts de ambos sitios (`ESXi-L01` y `ESXi-A01`) están enlazados al mismo vDS.
4.  **Éxito del Escenario:** Una captura final del panel de Tareas Recientes (*Recent Tasks*) mostrando la tarea de "Relocate virtual machine" (Migración cruzada) con el estado de *Completed* al 100%.

---

## 6. Rúbrica de Evaluación: Laboratorio 6 (Escala Vigesimal - Evaluación Grupal)

**Puntaje Máximo:** 20 puntos. Se requiere un mínimo de 14/20 para aprobar la práctica. La calificación será asignada al equipo en conjunto.

| Criterio Evaluado | Excelente (4 pts) | Bueno (3 pts) | Regular (2 pts) | Deficiente (1 pt) |
| :--- | :--- | :--- | :--- | :--- |
| **1. Arquitectura Base e Interconexión** | Los 5 hosts y TrueNAS son desplegados; el *ping* inter-PCs es exitoso. Los recursos físicos fueron balanceados impecablemente entre los estudiantes. | Los nodos fueron desplegados, pero se experimentaron demoras en la conectividad física LAN entre las dos PCs de los estudiantes. | Al menos un nodo ESXi no pudo ser integrado a la red, requiriendo que la topología se reduzca para continuar. | La comunicación entre la PC-1 y la PC-2 falló; el laboratorio no pudo ser integrado. |
| **2. Orquestación Central (vCenter)** | vCenter es desplegado correctamente; la taxonomía lógica (2 Datacenters, 2 Clústeres) es estructurada y poblada sin errores. | vCenter es operativo, pero los hosts fueron agregados sin la estructura lógica solicitada (Datacenters/Clusters). | El despliegue de vCenter requirió múltiples reintentos debido a fallas de DNS o recursos de RAM mal calculados. | vCenter no logró arrancar o sus servicios colapsaron permanentemente. |
| **3. Almacenamiento Heterogéneo** | vSAN es habilitado consolidando los discos de Lima, y NFS es montado en Arequipa de forma estable. | Ambas tecnologías fueron configuradas, pero surgieron alertas de salud (*Health Checks*) menores en el clúster vSAN. | Solo una de las tecnologías de almacenamiento (vSAN o NFS) logró ser aprovisionada de manera funcional. | Ninguno de los almacenamientos compartidos pudo ser inicializado. |
| **4. Redes Distribuidas (vDS)** | El vDS es creado y los 5 hosts de diferentes sitios lógicos son adheridos a él exitosamente mediante sus Uplinks. | El vDS es creado, pero solo los hosts de un sitio fueron adheridos al mismo. | El vDS presenta advertencias de desajuste de MTU (Maximum Transmission Unit) o falta de Uplinks operativos. | La topología de red se mantuvo en vSwitches estándar; el vDS fue evadido. |
| **5. Resolución del Caso (Migración)** | El *Cross-vMotion* fue ejecutado exitosamente, trasladando cómputo y almacenamiento entre ciudades (PCs) sin corte de ping. | La migración concluyó, pero se experimentó una caída temporal de la red del *Guest OS* durante el proceso. | La migración falló a medio camino por diferencias de compatibilidad de procesador (EVC no habilitado). | El escenario de negocio no fue superado; la VM no logró ser migrada de sitio. |

**Puntaje Total Obtenido por el Equipo:** ___ / 20

---

### Referencias

Broadcom. (2024e). *VMware vSAN 8.0 Architecture and Planning Guide*. VMware Docs. Recuperado de https://docs.vmware.com

Broadcom. (2024f). *vSphere Networking: Distributed Switches*. VMware Technical Library.

IBM. (2023). *Cross-Site Virtual Machine Mobility and Disaster Recovery*. IBM IT Resiliency Guides.

Intel Corporation. (2023). *Intel® 64 and IA-32 Architectures Software Developer’s Manual*.

VMware. (2023). *vCenter Server and Host Management: Advanced vMotion*. Broadcom Knowledge Base.
