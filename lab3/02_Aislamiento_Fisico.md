# LABORATORIO 2: Aislamiento Físico Multi-vSwitch, Hardening L2 y Almacenamiento IP con MTU 9000
* **Nivel:** Intermedio (Host Client)
* **Objetivos de Aprendizaje:**
  1. Desplegar un segundo conmutador virtual estándar para aislar físicamente una zona DMZ.
  2. Implementar políticas de seguridad contra suplantación de identidad (*Spoofing*).
  3. Configurar un puerto **VMkernel** con *Jumbo Frames* (MTU 9000) para almacenamiento iSCSI/NFS.

---

###  Topología Propuesta y Esperada

```
                         [ HOST ESXi 8.0 STANDALONE: 10.160.10.10 ]
   +------------------------------------------------------------------------+
   |  [ vSwitch0 ] (MTU 9000)                  [ vSwitch-DMZ ](MTU 1500)    |
   |  Uplink: vmnic0                           Uplink: vmnic1               |
   |     |                                        |                         |
   |     +-- [ Management: vmk0 (10.160.10.10) ]  +-- [ PG_DMZ ]            |
   |     |     Gateway: 10.160.10.2                     (VLAN 50)           |
   |     |                                              (Hardened)          |
   |     +-- [ PG_Storage: vmk1 ] (MTU 9000)                |               |
   |           (VLAN 30 - 10.160.30.10)                     |               |
   +--------------------------------------------------------|---------------+
                 |                                          |
           [ vmnic0 ]                                 [ vmnic1 ]
                 |                                          |
    [ Switch Físico LAN / Storage ]               [ Switch Físico DMZ ]
```

---

###  Guía de Implementación Paso a Paso

#### Paso 2.1: Creación del `vSwitch-DMZ` con Hardening de Seguridad
1. Ve a **Redes** > **Conmutadores virtuales** > **Agregar conmutador virtual estándar**.
2. Parámetros:
   * **Nombre:** `vSwitch-DMZ` | **MTU:** `1500` | **Enlace ascendente 1:** `vmnic1`
3. Despliega la pestaña **Seguridad (Security)** y ajusta:
   * **Modo promiscuo:** `Rechazar (Reject)`
   * **Cambios en dirección MAC:** `Rechazar (Reject)`
   * **Transmisiones falsificadas:** `Rechazar (Reject)`
4. Haz clic en **Agregar**.

* **Resultado Esperado:** Se crea `vSwitch-DMZ` aislado en `vmnic1` con políticas de seguridad en estado *Reject*.

>  **Nota Conceptual - Políticas de Seguridad L2:**  
> * **Promiscuous Mode (Reject):** La VM sólo recibe paquetes dirigidos a su propia dirección MAC.
> * **MAC Address Changes (Reject):** El host descarta paquetes si el SO invitado altera su MAC respecto a la registrada en el archivo `.vmx`.
> * **Forged Transmits (Reject):** El host descarta tramas salientes cuyo encabezado de origen no coincida con la MAC asignada por el hipervisor (previene ARP Spoofing).

---

#### Paso 2.2: Creación del Port Group DMZ
1. Ve a **Grupos de puertos** > **Agregar grupo de puertos**.
2. Nombre: `PG_DMZ_External` | **VLAN ID:** `50` | **Conmutador:** `vSwitch-DMZ`.
3. Conecta una Micro-VM (`MicroVM-DMZ` con IP `10.160.50.50/24`) a este grupo para validar conectividad hacia la red DMZ.

---

#### Paso 2.3: Configuración de MTU 9000 y VMkernel de Almacenamiento
1. En **Conmutadores virtuales**, selecciona `vSwitch0` > **Editar configuración** > Cambia **MTU** a `9000` > **Guardar**.
2. Ve a **Adaptadores de red VMkernel** > **Agregar adaptador de red VMkernel**:
   * **Nuevo grupo de puertos:** `PG_Storage_NFS` | **VLAN ID:** `30` | **MTU:** `9000`
   * **Pila TCP/IP:** `Pila TCP/IP predeterminada`
   * **IPv4 Estática:** `10.160.30.10` | Máscara: `255.255.255.0`
3. Haz clic en **Crear**.

* **Resultado Esperado:** La interfaz `vmk1` queda lista con soporte de Jumbo Frames en `vSwitch0`.

###  Validación (CLI):
Accede por SSH al ESXi (`10.160.10.10`) y prueba la conectividad hacia la IP de la cabina de almacenamiento (`10.160.30.100`) forzando el flag *Do Not Fragment* (`-d`):
```bash
vmkping -I vmk1 -d -s 8972 10.160.30.100
```
* **Salida esperada:** `0% packet loss` con paquetes de 8972 bytes de carga útil (MTU 9000 total).

