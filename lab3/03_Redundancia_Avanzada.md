# LABORATORIO 3: Redundancia Avanzada (NIC Teaming), Failover Determinista y Pila TCP/IP de vMotion Aislada
* **Nivel:** Avanzado (Host Client + Standalone ESXi)
* **Objetivos de Aprendizaje:**
  1. Configurar enlaces redundantes (*NIC Teaming*) en un switch estándar.
  2. Implementar políticas de orden de conmutación por error (*Failover Order*) deterministas por Port Group.
  3. Desacoplar el enrutamiento L3 mediante la **Pila TCP/IP nativa de vMotion (vMotion Stack)**.

---

###  Topología Propuesta y Esperada

```
                         [ vSwitch-Produccion (MTU 1500) ]
                                        |
               +------------------------+------------------------+
               |                                                 |
      [ Uplink 1: vmnic0 ]                              [ Uplink 2: vmnic1 ]
               |                                                 |
  +------------v-----------------------+            +------------v-----------------------+
  |       PG_VM_VLAN100                |            |       PG_vMotion_VLAN40            |
  |  Active: vmnic0                    |            |  Active: vmnic1                    |
  |  Standby: vmnic1                   |            |  Standby: vmnic0                   |
  +------------+-----------------------+            +------------+-----------------------+
               |                                                 |
      [ MicroVM-Produccion ]                            [ vmk2: 10.160.40.10 ]
       (10.160.100.50/24)                             (Pila TCP/IP: vMotion Stack)
                                                       Gateway Dedicado: 10.160.40.2
```

---

###  Guía de Implementación Paso a Paso

#### Paso 3.1: Configuración de NIC Teaming en el vSwitch
1. En el Host Client, ve a **Redes** > pestaña **Conmutadores virtuales** > **Agregar conmutador virtual estándar**.
2. Parámetros:
   * **Nombre:** `vSwitch-Produccion`
   * **Enlace ascendente 1:** `vmnic0`
   * **Enlace ascendente 2:** Haz clic en **Agregar enlace ascendente** y selecciona `vmnic1`.
3. Haz clic en **Agregar**.

* **Resultado Esperado:** Un único switch virtual estándar respaldado por dos tarjetas de red físicas.

---

#### Paso 3.2: Configuración de la Pila TCP/IP de vMotion en ESXi
1. Ve a **Redes** > pestaña **Pilas TCP/IP (TCP/IP stacks)**.
2. Selecciona la **Pila de vMotion (vMotion stack)** y haz clic en **Editar configuración (Edit settings)**.
3. Parámetros de enrutamiento:
   * **Puerta de enlace IPv4 (Gateway):** `10.160.40.2`
   * **DNS primario:** `10.160.10.2`
4. Haz clic en **Guardar**.

* **Resultado Esperado:** La pila de vMotion cuenta con su propia tabla de enrutamiento y puerta de enlace independiente.

>  **Nota Conceptual - vMotion TCP/IP Stack en Host Standalone:**  
> Al habilitar la pila de vMotion, el hipervisor crea una tabla de enrutamiento independiente en el kernel. Esto permite que el tráfico de migración salga por su propio gateway L3 sin depender ni interferir con la ruta por defecto del puerto de administración (`vmk0` en `10.160.10.10` vía `10.160.10.2`).

---

#### Paso 3.3: Creación de Port Groups con Políticas de Failover Invertidas
1. Ve a **Grupos de puertos** > **Agregar grupo de puertos**:
   * **Nombre:** `PG_VM_VLAN100` | **VLAN ID:** `100` | **Conmutador:** `vSwitch-Produccion`.
   * Despliega **Formación de equipos de NIC (NIC teaming)** > Marca **Reemplazar conmutador virtual**.
   * **Adaptadores activos:** `vmnic0` | **Adaptadores en espera:** `vmnic1`.
2. Haz clic en **Agregar**.
3. Crea el segundo grupo para vMotion:
   * **Nombre:** `PG_vMotion_VLAN40` | **VLAN ID:** `40` | **Conmutador:** `vSwitch-Produccion`.
   * Despliega **Formación de equipos de NIC** > Marca **Reemplazar conmutador virtual**.
   * **Adaptadores activos:** `vmnic1` | **Adaptadores en espera:** `vmnic0`.
4. Haz clic en **Agregar**.

* **Resultado Esperado:** Las VMs de producción utilizan `vmnic0` por defecto y conmutan a `vmnic1` sólo en caso de fallo físico, mientras que el tráfico de vMotion utiliza `vmnic1` como enlace principal.

---

#### Paso 3.4: Despliegue de la Interfaz VMkernel en la Pila de vMotion
1. Ve a **Adaptadores de red VMkernel** > **Agregar adaptador de red VMkernel**.
2. Configura:
   * **Grupo de puertos:** `PG_vMotion_VLAN40`
   * **Pila TCP/IP:** Selecciona **Pila de vMotion (vMotion stack)**.
   * *El servicio vMotion queda activado por diseño de la pila.*
   * **IPv4 Estática:** `10.160.40.10` | Máscara: `255.255.255.0`.
3. Haz clic en **Crear**.

* **Resultado Esperado:** Se crea la interfaz `vmk2` vinculada a la pila de vMotion y gobernada por la política de failover de su Port Group.

---

###  Validación (CLI):
Verifica desde la shell SSH que la tabla de rutas de la pila de vMotion esté completamente desacoplada de la pila default:
```bash
esxcli network ip route ipv4 list -N vMotion
```
* **Salida esperada:**
```text
Network         Netmask        Gateway       Interface
--------------  -------------  ------------  ---------
default         0.0.0.0        10.160.40.2   vmk2
10.160.40.0     255.255.255.0  0.0.0.0       vmk2
```
