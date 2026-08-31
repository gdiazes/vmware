
# LABORATORIO 1: Fundamentos de Redes Estándar, Segmentación L2 (VST) y Port Groups
* **Nivel:** Básico (Host Client)
* **Objetivo de Aprendizaje:** Comprender la abstracción del hardware de red hacia la capa lógica, desplegando un conmutador virtual estándar (*vSS*) con segmentación por VLAN mediante *Virtual Switch Tagging* (VST).

---

###  Topología Propuesta y Esperada

```
[ CAPA FÍSICA / VMWARE WORKSTATION ]          [ vmnic0 (Uplink Físico - Conectado a VMnet8) ]
                                                                     |
[ CAPA HIPERVISOR ESXi 8.0 ]                                    +----+----+
                                                                | vSwitch0| (MTU 1500)
                                                                +----+----+
                                                                     |
                        +--------------------------------------------+--------------------------------------------+
                        |                                            |                                            |
[ GRUPOS DE PUERTOS ]   | PG_RRHH                                    | PG_Finanzas                                | Management Network
                        | (VLAN ID: 10)                              | (VLAN ID: 20)                              | (VLAN ID: 0 / Untagged)
                        +--------+-----------------------------------+--------+-----------------------------------+--------+
                                 |                                            |                                            |
[ CAPA DE ACCESO ]          [ MicroVM-RRHH ]                            [ MicroVM-FIN ]                                  [ vmk0 ]
                         (10.160.10.50/24)                            (10.160.20.50/24)                           (10.160.10.10/24)
                         Gateway: 10.160.10.2                         Gateway: 10.160.20.2                        Gateway: 10.160.10.2
```

---

###  Guía de Implementación Paso a Paso

#### Paso 1.1: Inspección de `vSwitch0`
1. En el Host Client (`https://10.160.10.10/ui`), ve a **Redes (Networking)** > pestaña **Conmutadores virtuales (Virtual switches)**.
2. Haz clic sobre `vSwitch0` para entrar a su panel visual.

* **Resultado Esperado:** Diagrama interactivo que muestra `vSwitch0` vinculado a `vmnic0` y a los grupos predeterminados *Management Network* (con la IP `10.160.10.10`) y *VM Network*.

>  **Nota Conceptual - El Conmutador Virtual Estándar (vSS):**  
> Un vSwitch de ESXi no realiza aprendizaje dinámico de direcciones MAC desde el exterior (*uplink*). El hipervisor conoce de antemano la MAC de cada adaptador virtual (vNIC) registrado. Por esta razón, el vSS **no sufre bucles de red L2 y no requiere Spanning Tree Protocol (STP)**.

---

#### Paso 1.2: Creación de Port Groups Segmentados (VLAN 10 y VLAN 20)
1. Ve a **Redes** > pestaña **Grupos de puertos (Port groups)** > Haz clic en **Agregar grupo de puertos (Add port group)**.
2. Configura:
   * **Nombre:** `PG_RRHH` | **ID de VLAN:** `10` | **Conmutador virtual:** `vSwitch0`
3. Clic en **Agregar**.
4. Repite el proceso para Finanzas:
   * **Nombre:** `PG_Finanzas` | **ID de VLAN:** `20` | **Conmutador virtual:** `vSwitch0`
5. Clic en **Agregar**.

* **Resultado Esperado:** Dos nuevos Port Groups listados con sus respectivos IDs de VLAN asociados a `vSwitch0`.

>  **Nota Conceptual - Virtual Switch Tagging (VST):**  
> Al asignar un ID entre 1 y 4094, el vSwitch opera en modo **VST**. El vSwitch recibe la trama, **le retira la etiqueta VLAN (untagging)** y entrega el paquete limpio a la máquina virtual, liberando de esta tarea a la CPU del sistema operativo invitado.

---

#### Paso 1.3: Conexión y Validación con Micro-VMs
1. Despliega dos clones de la Micro-VM: `MicroVM-RRHH` y `MicroVM-FIN`.
2. Asigna la tarjeta de red de `MicroVM-RRHH` al grupo `PG_RRHH` y la de `MicroVM-FIN` al grupo `PG_Finanzas`.
3. Enciende ambas VMs y asigna sus IPs en consola:
   * `MicroVM-RRHH`: `10.160.10.50/24` (Gateway: `10.160.10.2`)
   * `MicroVM-FIN`: `10.160.20.50/24`

###  Validación (Checkpoint):
Desde la consola de `MicroVM-RRHH`, haz ping a `MicroVM-FIN` (`10.160.20.50`):
```ash
ping -c 3 10.160.20.50
```
* **Salida esperada:** `100% packet loss` (Aislamiento L2 exitoso).

Desde `MicroVM-RRHH`, haz ping a la puerta de enlace de VMware Workstation (`10.160.10.2`):
```ash
ping -c 3 10.160.10.2
```
* **Salida esperada:** Respuesta exitosa (`0% packet loss`).

