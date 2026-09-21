# GUÍA DE LABORATORIO 6: ALMACENAMIENTO HÍBRIDO Y SEGREGACIÓN DE REDES (PRODUCCIÓN Y DESARROLLO)

La arquitectura de despliegue correspondiente a esta sesión práctica, enfocada en un único host autónomo, es ilustrada de forma gráfica en la **Figura 1** y detallada lógicamente en la **Topología de Texto** adjunta.

**Figura 1**
*Topología Operativa del Laboratorio 4: Host ESXi Individual con Almacenamiento Distribuido y Redes Aisladas*

<img width="1165" height="649" alt="image" src="https://github.com/user-attachments/assets/7173df97-0913-456a-b01a-2907d17669be" />

*Nota.* Elaboración propia. La imagen destaca la habilitación de cuatro volúmenes de almacenamiento conectados a un único hipervisor, la segmentación interna mediante nuevos conmutadores lógicos para Producción y Desarrollo, y el aprovisionamiento de máquinas virtuales (Alpine Linux) distribuidas equitativamente.

### Topología Lógica de Infraestructura (Formato Texto)
Para facilitar la comprensión de las relaciones lógicas que serán configuradas durante el laboratorio mediante comandos, el siguiente mapa estructural debe ser analizado:

```text
=======================================================================
TOPOLOGÍA LÓGICA DEL HIPERVISOR (MAPA DE CONFIGURACIÓN)
=======================================================================
[ HIPERVISOR ESXi ] -> Hostname: esxi-01-[apellido]

  1. CAPA DE REDES (vSwitches & Port Groups)
     ├─ vSwitch0 (Gestión Base) 
     │   ├─ Management Network  -> vmk0 (10.160.10.10) -> Uplink: vmnic0
     │   └─ Storage Network     -> vmk1 (10.160.10.11) (Para iSCSI/NFS)
     │
     ├─ vSwitch1 (Red Aislada: Producción)
     │   └─ PG: Red-Produccion  -> Sin Uplink físico (Aislamiento total L2)
     │
     └─ vSwitch2 (Red Aislada: Desarrollo)
         └─ PG: Red-Desarrollo  -> Sin Uplink físico (Aislamiento total L2)

  2. CAPA DE ALMACENAMIENTO (Datastores)
     ├─ DAS-SCSI-01      (Disco Local / Tecnología SCSI / VMFS-6)
     ├─ DAS-NVMe-01      (Disco Local / Tecnología NVMe / VMFS-6)
     ├─ SAN-iSCSI-01     (Red de Bloques / TrueNAS / VMFS-6)
     └─ NAS-NFS-Shared   (Red de Archivos / TrueNAS / NFSv3)
         ├─ /_ISOs       (Repositorio de imágenes de SO)
         └─ /_Software   (Repositorio de aplicativos)

  3. CAPA DE CÓMPUTO (Máquinas Virtuales - Alpine Linux)
     ├─ Alpine-SCSI-Prod -> Datastore: DAS-SCSI-01  -> Red: Red-Produccion
     ├─ Alpine-NVMe-Prod -> Datastore: DAS-NVMe-01  -> Red: Red-Produccion
     ├─ Alpine-iSCSI-Dev -> Datastore: SAN-iSCSI-01 -> Red: Red-Desarrollo
     └─ Alpine-NFS-Dev   -> Datastore: NAS-NFS-Shared-> Red: Red-Desarrollo
=======================================================================
```

## 1. Escenario de Negocio
Como Administrador de Infraestructura de la corporación *ACME Corp*, el hipervisor base autónomo ha sido desplegado exitosamente. Un nuevo mandato de seguridad de la arquitectura exige que los entornos de **Desarrollo** y **Producción** operen en conmutadores lógicos separados (aislamiento estricto de Capa 2). Paralelamente, se requiere la consolidación de cuatro tecnologías de almacenamiento (SCSI, NVMe, iSCSI y NFS) conectadas al mismo servidor físico. Su misión final será validar la viabilidad operativa desplegando una flota de micro-servidores virtuales (*Alpine Linux*), alojando una instancia en cada repositorio de datos y distribuyéndolas a través de las nuevas redes segregadas.

## 2. Objetivos de la Práctica
*   Estandarizar el *hostname* (nombre de host) del hipervisor basándose en la nomenclatura corporativa.
*   Diseñar y aprovisionar conmutadores virtuales (`vSwitch1` y `vSwitch2`) mediante línea de comandos (CLI) para segregar el tráfico de Producción y Desarrollo.
*   Aprovisionar discos locales (SCSI y NVMe) con el sistema de archivos VMFS-6.
*   Integrar un dispositivo *TrueNAS* para exportar bloques lógicos (iSCSI) y directorios de red (NFS), creando una estructura jerárquica de carpetas (`_ISOs`).
*   Validar la operatividad global mediante el despliegue concurrente de cuatro máquinas virtuales asociadas a diferentes *Datastores* y redes lógicas.

## 3. Requisitos Previos
*   Haber aprobado la **Guía de Laboratorio 03**.
*   **Identidad Corporativa:** El *hostname* del servidor ESXi deberá estar configurado con la primera letra del nombre y el apellido paterno del estudiante (Ej. `jperez`).
*   **Adición de Hardware Local:** La máquina virtual ESXi en *VMware Workstation* debe tener configurados dos discos duros virtuales adicionales de 20 GB cada uno (uno tipo **SCSI** y otro **NVMe**).
*   **Medios de Instalación:** La imagen ISO de **Alpine Linux** (`alpine-standard-x.x.x-x86_64.iso`) debe estar descargada en su PC físico.
*   **Dispositivo de Almacenamiento:** Una máquina virtual *TrueNAS* operativa en la subred NAT (`10.160.10.200`), con recursos disponibles para iSCSI y NFS.

---

## 4. Instrucciones Paso a Paso

### Fase 1: Identidad y Preparación de Red de Almacenamiento (VMkernel)
1. Ingrese a la Interfaz Web (*VMware Host Client*) a través de su navegador (`https://10.160.10.10`).
2. Navegue a **Networking > TCP/IP stacks > Default TCP/IP stack > Edit**. Modifique el *Host name* utilizando su inicial y apellido (Ej. `jperez`). Guarde los cambios.
3. En la pestaña **VMkernel NICs**, haga clic en **Add VMkernel NIC**.
4. Defina un nuevo *Port Group* llamado `Storage-Network` conectado al conmutador `vSwitch0`.
5. Asigne una IPv4 estática dedicada al almacenamiento: `10.160.10.11` (Máscara `255.255.255.0`). Haga clic en **Create**.

### Fase 2: Segregación de Redes (Producción y Desarrollo vía CLI)
*El aislamiento de los entornos debe ser configurado desde la consola.*
1. Abra su cliente **PuTTY** (SSH) y conéctese a la IP `10.160.10.10` como usuario `root`.
2. Aprovisione el conmutador virtual para **Producción** ejecutando:
   `esxcfg-vswitch -a vSwitch1`
3. Aprovisione el conmutador virtual para **Desarrollo** ejecutando:
   `esxcfg-vswitch -a vSwitch2`
4. Cree los Grupos de Puertos (*Port Groups*) correspondientes en cada conmutador recién creado:
   `esxcfg-vswitch -A "Red-Produccion" vSwitch1`
   `esxcfg-vswitch -A "Red-Desarrollo" vSwitch2`
5. Verifique que ambos conmutadores lógicos hayan sido creados correctamente listando la configuración:
   `esxcfg-vswitch -l`

### Fase 3: Creación de Datastores Locales (SCSI y NVMe)
1. Retorne al *VMware Host Client* web. En el panel izquierdo, seleccione **Storage** (Almacenamiento) > **Datastores** > **New datastore**.
2. Seleccione **Create new VMFS datastore** y presione *Next*.
3. Nombre el almacén como `DAS-SCSI-01`, seleccione el disco SCSI de 20 GB de la lista, elija **VMFS 6** y asigne todo el espacio disponible. Finalice el asistente.
4. Repita el proceso para el segundo disco. Nombre el almacén como `DAS-NVMe-01` y seleccione el disco con tecnología NVMe.

### Fase 4: Integración de Almacenamiento Externo (iSCSI y NFS)
*Los protocolos de red aprovisionados en TrueNAS serán montados en este paso.*
1. **Montaje NAS (NFS):**
   *   Vaya a **Storage > New datastore > Mount NFS datastore**.
   *   Nombre: `NAS-NFS-Shared`. Servidor NFS: `10.160.10.200`. Recurso compartido: `/mnt/Pool-VMware/NFS-Share`. Versión: **NFS 3**. Presione *Finish*.
2. **Montaje SAN (iSCSI):**
   *   En *Storage*, vaya a la pestaña **Adapters**. Haga clic en **Software iSCSI**.
   *   Habilite el servicio y en **Dynamic targets** agregue la IP de TrueNAS (`10.160.10.200`). Guarde la configuración.
   *   Vaya a **Datastores > New datastore > Create new VMFS datastore**.
   *   Nombre el almacén como `SAN-iSCSI-01`, seleccione el disco de TrueNAS descubierto en la red y formatéelo con **VMFS 6**.

### Fase 5: Estructuración de Directorios y Carga de Medios
1. En la pestaña *Datastores*, haga clic derecho sobre `NAS-NFS-Shared` y seleccione **Browse** (Explorar).
2. Utilizando la opción **Create directory**, genere las carpetas `_ISOs` y `_Software` para estandarizar el repositorio.
3. Ingrese a la carpeta `_ISOs` y haga clic en **Upload** (Cargar). Busque en su PC el archivo `alpine-standard.iso` y espere a que la carga finalice al 100%.

### Fase 6: Despliegue Distribuido y Aislado de VMs
*Las máquinas virtuales serán mapeadas siguiendo estrictamente la Topología Lógica.*
1. Diríjase a **Virtual Machines** y haga clic en **Create / Register VM**.
2. **Creación VM 1 (Producción / SCSI):** 
   *   Nombre: `Alpine-SCSI-Prod`. Guest OS: `Linux` / `Other 3.x Linux (64-bit)`.
   *   **Almacenamiento:** Seleccione `DAS-SCSI-01`.
   *   **Red (Network Adapter 1):** Cambie la red a **Red-Produccion**.
   *   Hardware: 1 vCPU, 512 MB RAM, 2 GB Disco (*Thin provisioned*).
   *   CD/DVD: *Datastore ISO file*. Apunte al archivo ISO de Alpine cargado en el NFS.
3. Repita el asistente para crear las tres máquinas restantes respetando las siguientes variables:
   *   **VM 2:** Nombre `Alpine-NVMe-Prod` | Datastore: `DAS-NVMe-01` | Red: **Red-Produccion**.
   *   **VM 3:** Nombre `Alpine-iSCSI-Dev` | Datastore: `SAN-iSCSI-01` | Red: **Red-Desarrollo**.
   *   **VM 4:** Nombre `Alpine-NFS-Dev` | Datastore: `NAS-NFS-Shared`| Red: **Red-Desarrollo**.
4. Encienda las cuatro máquinas virtuales.

---

## 5. Criterio de Validación y Evidencias
Para cumplir con el objetivo formativo, las siguientes capturas de pantalla deberán ser generadas y anexadas a su informe técnico:
1.  **Auditoría de Redes Aisladas (CLI):** Abra PuTTY (SSH), ejecute `hostname` y luego `esxcfg-vswitch -l` para evidenciar la existencia de los tres conmutadores (`vSwitch0`, `vSwitch1`, `vSwitch2`).
2.  **Auditoría de Almacenamiento (CLI):** Ejecute el comando `esxcli storage filesystem list` para demostrar que los cuatro volúmenes (SCSI, NVMe, iSCSI, NFS) se encuentran montados correctamente en el sistema de archivos del núcleo.
3.  **Auditoría de Orquestación (Web):** Capture la pantalla del *VMware Host Client* (Sección Virtual Machines), donde se visualicen las cuatro máquinas `Alpine-*` en estado **Encendido**. Se debe apreciar en las columnas correspondientes que están distribuidas rigurosamente en las redes de Producción y Desarrollo.

## 6. Reto de Expertos (Opcional)
En este laboratorio, los conmutadores lógicos de Producción y Desarrollo (`vSwitch1` y `vSwitch2`) fueron creados sin asignarles tarjetas de red físicas (Uplinks), logrando un aislamiento total (Host-Only).
*   **El Reto:** Asuma que su servidor físico dispone de una tarjeta de red adicional (`vmnic2`). Utilizando sus conocimientos de comandos CLI aprendidos en laboratorios previos, detalle (no ejecute, solo detalle en su informe) cuál sería la sintaxis exacta del comando `esxcfg-vswitch` requerida para enlazar el adaptador físico `vmnic2` al conmutador `vSwitch1` (Producción), permitiendo que esas máquinas tengan salida a Internet o al resto de la corporación.

---

## 7. Rúbrica de Evaluación: Laboratorio 4 (Escala Vigesimal)

**Puntaje Máximo:** 20 puntos. Se requiere un mínimo de 14/20 para aprobar la práctica.

| Criterio Evaluado | Excelente (4 pts) | Bueno (3 pts) | Regular (2 pts) | Deficiente (1 pt) |
| :--- | :--- | :--- | :--- | :--- |
| **1. Identidad y Red de Storage** | *Hostname* estandarizado exitosamente con el apellido. Puerto `vmk1` creado de manera estática (`10.160.10.11`). | Puerto `vmk1` creado, pero el *hostname* carece de la nomenclatura corporativa solicitada (`jperez`). | El puerto de red fue dejado en DHCP o asignado a un switch erróneo. | El *hostname* no fue editado y la red de almacenamiento no fue creada. |
| **2. Segregación de Redes (CLI)** | `vSwitch1` (Prod) y `vSwitch2` (Dev) creados por comando junto con sus respectivos *Port Groups* sin errores. | Los vSwitches fueron creados en consola, pero hubo errores tipográficos en los nombres de los *Port Groups*. | La segregación de redes fue evadida mediante consola, realizándose exclusivamente a través de la interfaz web. | Los conmutadores de red aislados no fueron aprovisionados en el sistema. |
| **3. Consolidación de Almacenamiento** | Los 4 Datastores (SCSI, NVMe, iSCSI, NFS) fueron descubiertos y montados. Carpetas jerárquicas creadas en el NAS. | Los 4 Datastores fueron montados, pero se omitió crear la estructura de carpetas (`_ISOs`) o cargar el archivo de Alpine. | Solo los discos locales fueron formateados; el almacenamiento en red (TrueNAS) falló en su montaje. | Incapacidad para inicializar el almacenamiento o integrar el dispositivo externo. |
| **4. Despliegue Distribuido de VMs** | Las 4 VMs fueron creadas, encendidas y mapeadas de forma exacta a sus respectivos Datastores y Redes (Prod/Dev). | Las 4 VMs fueron creadas, pero hubo un error de mapeo (ej. asignar una VM de Producción a la red de Desarrollo). | Las VMs fueron creadas pero no pudieron arrancar al no localizar la imagen ISO en el almacenamiento compartido. | El despliegue de las cargas de trabajo no fue ejecutado por el alumno. |
| **5. Documentación CLI y Reto** | Capturas finales (`hostname`, `vswitch -l`, `filesystem list`) generadas impecablemente. Reto Experto argumentado con la sintaxis correcta. | Capturas de CLI anexadas cubriendo los requisitos, pero el Reto Experto (Uplinks) fue omitido. | Evidencias adjuntadas desde la interfaz web, evadiendo la instrucción obligatoria de usar comandos CLI. | Ausencia total de evidencias gráficas y comandos en el reporte técnico final. |

**Puntaje Total Obtenido:** ___ / 20
