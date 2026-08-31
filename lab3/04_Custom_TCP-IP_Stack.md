# LABORATORIO 4: Custom TCP/IP Stack (CLI), Traffic Shaping (QoS) y Diagnóstico con `pktcap-uw`
* **Nivel:** Experto / Alta Complejidad (CLI ESXi + Host Client)
* **Objetivos de Aprendizaje:**
  1. Instanciar una **Pila TCP/IP Personalizada (Custom Netstack)** vía CLI para aislamiento estricto de backup/DR.
  2. Implementar **Conformación de Tráfico (*Traffic Shaping / QoS*)** a nivel de Port Group estándar.
  3. Realizar captura, trazabilidad e inspección de paquetes a bajo nivel con `pktcap-uw` y `tcpdump-uw`.

---

###  Topología Propuesta y Esperada

```
                         [ HOST ESXi 8.0 STANDALONE: 10.160.10.10 ]
                                              |
                      +-----------------------+-----------------------+
                      |                                               |
  [ PG_Prod_Limitada (VLAN 100) ]                   [ PG_Backup_DR (VLAN 60) ]
  • Traffic Shaping: 10 Mbps Límite                 • Custom Netstack: "BackupStack"
  • Burst Size: 1024 KB                             • Gateway L3 Propio: 10.160.60.2
               |                                                      |
      [ MicroVM-Testing ]                                   [ vmk3: 10.160.60.10 ]
      (10.160.100.50/24)                                    (VLAN 60 - Backup)
```

---

###  Guía de Implementación Paso a Paso

#### Paso 4.1: Creación de una Pila TCP/IP Personalizada por CLI
1. Abre sesión SSH en el host ESXi (`10.160.10.10`) como `root`.
2. Crea la instancia del netstack en la memoria del kernel:
```bash
esxcli network ip netstack add -N "BackupStack"
```
3. Verifica la creación de la instancia:
```bash
esxcli network ip netstack list | grep BackupStack
```

* **Resultado Esperado:** La CLI retorna el nombre `BackupStack` en estado activo y listo para asignación de interfaces.

>  **Nota Conceptual - Custom TCP/IP Stacks en Standalone:**  
> Una pila TCP/IP personalizada es un espacio de nombres de red aislado (*Network Namespace*). Permite que agentes de backup locales o tareas de replicación hacia nubes secundarias tengan su propia tabla de enrutamiento, DNS y gateway sin alterar la tabla global del hipervisor.

---

#### Paso 4.2: Creación del Port Group y VMkernel Asociado al Custom Stack
1. En el Host Client, ve a **Grupos de puertos** > **Agregar grupo de puertos**:
   * **Nombre:** `PG_Backup_DR` | **VLAN ID:** `60` | **Conmutador:** `vSwitch-Produccion`.
2. En la terminal SSH, asocia una nueva interfaz VMkernel (`vmk3`) a este grupo y a la pila personalizada:
```bash
esxcli network ip interface add -p "PG_Backup_DR" -k vmk3 -N BackupStack -m 1500
```
3. Asigna la configuración IPv4 estática:
```bash
esxcli network ip interface ipv4 set -i vmk3 -t static -I 10.160.60.10 -N 255.255.255.0
```
4. Configura la ruta predeterminada exclusiva para esta pila apuntando a su gateway local:
```bash
esxcli network ip route ipv4 add -N BackupStack -n default -g 10.160.60.2
```

* **Resultado Esperado:** `vmk3` queda operativo bajo la instancia `BackupStack`.

---

#### Paso 4.3: Configuración de Traffic Shaping (QoS en vSwitch Estándar)
1. En el Host Client, ve a **Grupos de puertos** > **Agregar grupo de puertos**.
2. Configura:
   * **Nombre:** `PG_Prod_Limitada` | **VLAN ID:** `100` | **Conmutador:** `vSwitch-Produccion`.
3. Despliega la sección **Conformación del tráfico (Traffic shaping)** y selecciona **Activado (Enabled)**:
   * **Ancho de banda promedio (Average Bandwidth):** `10000` Kbit/s (10 Mbps).
   * **Ancho de banda máximo (Peak Bandwidth):** `15000` Kbit/s (15 Mbps).
   * **Tamaño de ráfaga (Burst Size):** `1024` KBytes.
4. Haz clic en **Agregar**.
5. Conecta la máquina `MicroVM-Testing` (`10.160.100.50/24`) a este grupo de puertos.

* **Resultado Esperado:** El hipervisor limitará el tráfico de salida de las VMs en este Port Group según los umbrales configurados.

>  **Nota Conceptual - Traffic Shaping en vSwitch Estándar:**  
> En conmutadores estándar (vSS), la conformación de tráfico sólo controla el **tráfico de salida (egress)** desde la máquina virtual hacia el switch. Utiliza un algoritmo de cubeta de fichas (*Token Bucket*) para suavizar picos de tráfico y evitar que una máquina sature el enlace físico compartido.

---

###  Prueba de Validación y Diagnóstico Forense de Paquetes
1. Inicia una captura en vivo en el puerto VMkernel personalizado (`vmk3`):
```bash
pktcap-uw --vmk vmk3 --dir 0 --stage 1 -o /tmp/backup_trace.pcap
```
2. En otra sesión SSH, envía paquetes de prueba hacia el appliance de backup (`10.160.60.100`) usando la pila aislada:
```bash
vmkping -S BackupStack 10.160.60.100
```
3. Detén la captura con `Ctrl + C` y analiza el volcado con `tcpdump-uw`:
```bash
tcpdump-uw -r /tmp/backup_trace.pcap -nn -v
```

* **Salida esperada:**
```text
reading from file /tmp/backup_trace.pcap, fail-type RAW
10:15:32.102341 IP (tos 0x0, ttl 64, id 1234, offset 0, flags [none], proto ICMP (1), length 84)
    10.160.60.10 > 10.160.60.100: ICMP echo request, id 512, seq 1, length 64
10:15:32.102890 IP (tos 0x0, ttl 64, id 4321, offset 0, flags [none], proto ICMP (1), length 84)
    10.160.60.100 > 10.160.60.10: ICMP echo reply, id 512, seq 1, length 64
```
* **Conclusión:** El paquete se procesa exclusivamente dentro de la instancia de memoria asignada a `BackupStack`, garantizando el aislamiento absoluto de la capa de transporte en un host ESXi independiente.

---
---

#  RÚBRICA DE EVALUACIÓN HOLÍSTICA (CRITERIOS DE DESEMPEÑO)

| Dimensión Técnica | Nivel Novato (0 - 59%) | Nivel Competente (60 - 84%) | Nivel Experto / Dominio (85 - 100%) |
| :--- | :--- | :--- | :--- |
| **Arquitectura vSS & VST (Lab 1)** | Desconoce el funcionamiento de los Port Groups o configura VLANs erróneas. | Configura Port Groups con VLANs funcionales pero no valida el aislamiento L2. | Implementa VST con aislamiento estricto y valida la segmentación mediante Micro-VMs. |
| **Hardening L2 & MTU 9000 (Lab 2)** | Deja políticas de seguridad en *Accept* y no comprende el impacto de MTU 9000. | Aplica políticas de seguridad pero falla en la configuración extremo a extremo del MTU. | Aplica *Hardening* total (Reject x3), configura MTU 9000 y valida sin fragmentación con `vmkping -d`. |
| **NIC Teaming & vMotion Stack (Lab 3)** | Configura uplinks sin redundancia o mezcla la pila default con tráfico de migración. | Crea el NIC Teaming pero no establece políticas de Failover deterministas por Port Group. | Configura Teaming activo/espera invertido y aísla la pila `vMotion Stack` con su propio Gateway L3. |
| **Custom Netstacks & QoS (Lab 4)** | Incapaz de utilizar la CLI `esxcli` para crear pilas personalizadas. | Instancia la pila TCP/IP personalizada pero no define rutas o falla al limitar ancho de banda. | Crea el *Netstack* por CLI, fija gateways independientes, aplica *Traffic Shaping* y audita paquetes con `pktcap-uw`. |

