# LABORATORIO 2: ARQUITECTURA DMZ CON SERVIDOR WEB PÚBLICO, HARDENING L2 Y STORAGE MTU 9000

**Nivel:** Intermedio / Profesional  
**Entorno:** VMware ESXi 8.0 Standalone (Nested en VMware Workstation)  
**Puerta de Enlace Workstation:** `10.160.10.2`  
**IP de Gestión ESXi (`vmk0`):** `10.160.10.10`

---

##  Topología Arquitectónica Integral (Lab 2)

```
                       [ TU PC FÍSICA (WINDOWS) / NAVEGADOR WEB ]
                                           |
                                  Acceso HTTP: http://10.160.10.254:80
                                           |
                    [ VMware Workstation NAT Gateway: 10.160.10.2 ]
                                           |
 ==========================================|========================================================================
   [ SERVIDOR ESXi 8.0 STANDALONE: 10.160.10.10 ]                                                                  |
                                           |                                                                        |
               +---------------------------v---------------------------+                                            |
               |                  VM ROUTER / FIREWALL                 |                                            |
               |                 (Alpine Linux / 128MB)                |                                            |
               +-----+---------------------+---------------------+-----+                                            |
                     |                     |                     |                                                  |
           [ eth0 (WAN) ]           [ eth1 (LAN) ]        [ eth2 (DMZ) ]                                            |
          10.160.10.254/24         10.160.10.254/24      10.160.50.254/24                                           |
          GW: 10.160.10.2                 |                     |                                                   |
                 |                        |                     |                                                   |
 ----------------|------------------------|---------------------|---------------------------------------------------
                 |                        |                     |                                                   |
       +---------v------------------------v----+           +----v--------------------+                              |
       |               vSwitch0                |           |       vSwitch-DMZ       |                              |
       |      (Red Interna y Storage)          |           |   (Red Pública / DMZ)   |                              |
       |             (MTU 9000)                |           |        (MTU 1500)       |                              |
       |           Uplink: vmnic0              |           |      Uplink: vmnic1     |                              |
       +---------+------------------+----------+           +------------+------------+                              |
                 |                  |                                   |                                           |
                 |                  |                                   |                                           |
    +------------v-----+     +------v-----------+              +--------v-----------+                               |
    | Management (vmk0)|     | PG_LAN_Interna   |              | PG_DMZ_External    |                               |
    | VLAN 0 / MTU 1500|     | VLAN 10          |              | VLAN 50 (Hardened) |                               |
    | 10.160.10.10/24  |     +--------+---------+              +--------+-----------+                               |
    | GW: 10.160.10.2  |              |                                 |                                           |
    +------------------+              |                                 |                                           |
                             +--------v---------+              +--------v---------+                                 |
    +------------------+     | MicroVM-Interna  |              | MicroVM-DMZ      |                                 |
    | PG_Storage (vmk1)|     | (Base de Datos)  |              | (Servidor Web)   |                                 |
    | VLAN 30 / MTU 9000     | 10.160.10.60/24  |              | 10.160.50.50/24  |                                 |
    | 10.160.30.10/24  |     | GW: 10.160.10.254|              | Puerto 80 Activo |                                 |
    | (Sin Gateway)    |     +------------------+              | GW: 10.160.50.254|                                 |
    +------------------+                                       +------------------+                                 |
 ====================================================================================================================
                 |                                                              |
           [ vmnic0 ]                                                     [ vmnic1 ]
                 |                                                              |
    [ Switch LAN / Storage Interno ]                              [ Switch DMZ / Enlace Perimetral ]
```

---

##  Matriz de Configuración e Interfaces

| Elemento | Conectado a | vSwitch | VLAN | MTU | IP / Máscara | Gateway | Función Principal |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **ESXi (`vmk0`)** | *Management Network* | `vSwitch0` | 0 | 1500 | `10.160.10.10/24` | `10.160.10.2` | Acceso Web y SSH al hipervisor. |
| **ESXi (`vmk1`)** | `PG_Storage_NFS` | `vSwitch0` | 30 | **9000** | `10.160.30.10/24` | *Ninguno* | Tráfico de almacenamiento de alta velocidad. |
| **VM-Router (`eth0`)**| `VM Network` | `vSwitch0` | 0 | 1500 | `10.160.10.254/24`| `10.160.10.2` | **WAN:** Recibe peticiones web y da salida NAT. |
| **VM-Router (`eth1`)**| `PG_LAN_Interna` | `vSwitch0` | 10 | 1500 | `10.160.10.254/24`| *Es Gateway* | Puerta de enlace de la red interna protegida. |
| **VM-Router (`eth2`)**| `PG_DMZ_External` | `vSwitch-DMZ`| 50 | 1500 | `10.160.50.254/24`| *Es Gateway* | Puerta de enlace de la zona DMZ. |
| **`MicroVM-Interna`** | `PG_LAN_Interna` | `vSwitch0` | 10 | 1500 | `10.160.10.60/24` | `10.160.10.254` | Simula un servidor de Base de Datos interno. |
| **`MicroVM-DMZ`** | `PG_DMZ_External` | `vSwitch-DMZ`| 50 | 1500 | `10.160.50.50/24` | `10.160.50.254` | **Servidor Web HTTP (Puerto 80 activo).** |

---

##  GUÍA DE IMPLEMENTACIÓN PASO A PASO

---

### FASE 1: Configurar MTU 9000 y VMkernel de Almacenamiento en `vSwitch0`

#### Paso 1.1: Habilitar Jumbo Frames en el Switch Principal
1. Accede al **ESXi Host Client** (`https://10.160.10.10/ui`).
2. Ve a **Redes (Networking)** > pestaña **Conmutadores virtuales (Virtual switches)**.
3. Selecciona `vSwitch0` y haz clic en **Editar configuración (Edit settings)**.
4. Cambia el campo **MTU** de `1500` a `9000`.
5. Haz clic en **Guardar (Save)**.

* **Resultado Esperado:** `vSwitch0` queda preparado para procesar tramas gigantes (*Jumbo Frames*) sin descartar paquetes.

>  **Nota Conceptual - Jumbo Frames (MTU 9000):**  
> El tráfico de almacenamiento (NFS/iSCSI) transfiere grandes bloques de datos. Al elevar el MTU a 9000 bytes, reducimos la cantidad de paquetes necesarios en un factor de 6, disminuyendo drásticamente el uso de CPU y la sobrecarga de cabeceras en el hipervisor.

---

#### Paso 1.2: Crear el VMkernel de Almacenamiento Aislado
1. En **Redes**, ve a la pestaña **Adaptadores de red VMkernel (VMkernel NICs)** > Haz clic en **Agregar adaptador de red VMkernel**.
2. Parámetros:
   * **Grupo de puertos:** *Nuevo grupo de puertos* $\rightarrow$ Nombre: `PG_Storage_NFS`
   * **Conmutador virtual:** `vSwitch0`
   * **ID de VLAN:** `30`
   * **MTU:** `9000`
   * **Pila TCP/IP:** `Pila TCP/IP predeterminada (Default TCP/IP stack)`
   * **Configuración IPv4:** Estática
     * **Dirección IP:** `10.160.30.10` | **Máscara:** `255.255.255.0`
3. Haz clic en **Crear**.

* **Resultado Esperado:** Se crea la interfaz `vmk1` con MTU 9000. Observa que **no se le asigna puerta de enlace**, garantizando que el almacenamiento nunca pueda enrutarse hacia Internet.

---

### FASE 2: Desplegar `vSwitch-DMZ` con Hardening L2 Estricto

#### Paso 2.1: Creación del vSwitch Físicamente Aislado
1. Ve a **Conmutadores virtuales** > **Agregar conmutador virtual estándar**.
2. Parámetros:
   * **Nombre:** `vSwitch-DMZ`
   * **MTU:** `1500`
   * **Enlace ascendente 1 (Uplink 1):** `vmnic1`
3. Despliega la sección **Seguridad (Security)** y fuerza todas las directivas a **Rechazar (Reject)**:
   * **Modo promiscuo:** `Rechazar (Reject)`
   * **Cambios en dirección MAC:** `Rechazar (Reject)`
   * **Transmisiones falsificadas:** `Rechazar (Reject)`
4. Haz clic en **Agregar**.

* **Resultado Esperado:** Un conmutador independiente asociado a la tarjeta física `vmnic1`, protegido a nivel de hardware contra ataques de suplantación.

>  **Nota Conceptual - Hardening de Seguridad L2:**  
> * **Promiscuous Mode (Reject):** Impide que una máquina virtual comprometida en la DMZ ponga su tarjeta en modo monitor para capturar el tráfico de otras VMs.
> * **MAC Address Changes (Reject):** El hipervisor desconecta el puerto si el sistema operativo intenta cambiar la dirección MAC asignada por VMware.
> * **Forged Transmits (Reject):** Previene ataques de envenenamiento ARP (*ARP Poisoning*) al bloquear paquetes salientes cuya MAC origen no coincida con la registrada.

---

#### Paso 2.2: Crear los Port Groups de LAN Interna y DMZ
1. Ve a **Grupos de puertos (Port groups)** > **Agregar grupo de puertos**:
   * **Nombre:** `PG_LAN_Interna` | **VLAN ID:** `10` | **Conmutador:** `vSwitch0`.
   * Clic en **Agregar**.
2. Haz clic nuevamente en **Agregar grupo de puertos**:
   * **Nombre:** `PG_DMZ_External` | **VLAN ID:** `50` | **Conmutador:** `vSwitch-DMZ`.
   * Clic en **Agregar**.

* **Resultado Esperado:** Dos dominios de difusión independientes en switches virtuales físicamente separados.

---

### FASE 3: Despliegue y Configuración de la `VM-Router` (Alpine Linux)

#### Paso 3.1: Asignar 3 Tarjetas de Red a la VM Router
1. Clona o crea una Micro-VM Alpine llamada `VM-Router`.
2. Edita su configuración de hardware y asegúrate de tener **3 adaptadores de red**:
   * **Adaptador de red 1:** Conectar a `VM Network` (en `vSwitch0`).
   * **Adaptador de red 2:** Conectar a `PG_LAN_Interna` (en `vSwitch0`).
   * **Adaptador de red 3:** Conectar a `PG_DMZ_External` (en `vSwitch-DMZ`).
3. Guarda los cambios y enciende la VM.

---

#### Paso 3.2: Configurar Enrutamiento, NAT y Port Forwarding (DNAT)
Abre la consola web de `VM-Router`, inicia sesión como `root` y ejecuta:

```ash
# 1. Habilitar el enrutamiento IP en el Kernel
sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf

# 2. Configurar IPs estáticas en cada pata de red
ip addr add 10.160.10.254/24 dev eth0 && ip link set eth0 up  # WAN
ip route add default via 10.160.10.2                          # Salida a Workstation

ip addr add 10.160.10.254/24 dev eth1 && ip link set eth1 up  # LAN Interna
ip addr add 10.160.50.254/24 dev eth2 && ip link set eth2 up  # DMZ

# 3. Instalar iptables para NAT y Firewall
apk add iptables

# 4. Regla de Salida (SNAT / Masquerade) para que LAN y DMZ naveguen
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# 5. Regla de Publicación Web (DNAT): Redirigir puerto 80 entrante hacia la DMZ
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 -j DNAT --to-destination 10.160.50.50:80
iptables -A FORWARD -p tcp -d 10.160.50.50 --dport 80 -j ACCEPT

# 6. Regla de Seguridad (Firewall): Bloquear que la DMZ inicie tráfico hacia la LAN
iptables -A FORWARD -i eth2 -o eth1 -m state --state NEW -j DROP
```

* **Resultado Esperado:** `VM-Router` actúa como router perimetral, publica el puerto web y aísla la red interna.

>  **Nota Conceptual - DNAT (Destination NAT / Port Forwarding):**  
> Cuando un usuario externo solicita `http://10.160.10.254:80`, el router reescribe la cabecera IP de destino cambiándola por `10.160.50.50:80` y conmuta el paquete hacia `vSwitch-DMZ`. Para el usuario externo, la IP interna real del servidor web permanece totalmente oculta.

---

### FASE 4: Despliegue de Servidores (`MicroVM-Interna` y Servidor Web `MicroVM-DMZ`)

#### Paso 4.1: Configurar `MicroVM-Interna` (Simulación de Base de Datos)
1. Conecta `MicroVM-Interna` al Port Group `PG_LAN_Interna`.
2. Enciende la VM y configura su red en consola:
```ash
ip addr add 10.160.10.60/24 dev eth0 && ip link set eth0 up
ip route add default via 10.160.10.254
```

---

#### Paso 4.2: Configurar y Levantar el Servidor Web en `MicroVM-DMZ`
1. Conecta `MicroVM-DMZ` al Port Group `PG_DMZ_External`.
2. Enciende la VM y configura su direccionamiento:
```ash
ip addr add 10.160.50.50/24 dev eth0 && ip link set eth0 up
ip route add default via 10.160.50.254
```
3. Crea una página HTML personalizada e inicia el demonio HTTP nativo de Alpine:
```ash
# Crear la estructura web
mkdir -p /var/www/localhost/htdocs
cat <<EOF > /var/www/localhost/htdocs/index.html
<!DOCTYPE html>
<html>
<head><title>DMZ ESXi 8.0</title></head>
<body style="font-family: Arial; text-align: center; background-color: #f0f4f8; padding: 50px;">
  <h1 style="color: #0056b3;">¡Servidor Web en DMZ Operativo!</h1>
  <p><strong>Hipervisor:</strong> VMware ESXi 8.0 Standalone</p>
  <p><strong>Ubicación:</strong> vSwitch-DMZ (vmnic1) | <strong>IP Interna:</strong> 10.160.50.50</p>
  <p style="color: green;">✔ Aislamiento L2 Hardened Activo</p>
</body>
</html>
EOF

# Iniciar el servidor web en segundo plano en el puerto 80
httpd -p 80 -h /var/www/localhost/htdocs/
```

* **Resultado Esperado:** El servidor web queda escuchando peticiones en `10.160.50.50:80`.

---

##  FASE 5: PRUEBAS DE VALIDACIÓN Y AUDITORÍA DE SEGURIDAD

Ejecuta las siguientes 4 pruebas para validar la solución de extremo a extremo:

---

###  Prueba 1: Consumo del Servidor Web desde tu PC Física (Windows)
Abre tu navegador (Chrome / Edge / Firefox) en tu máquina Windows y entra a:
```text
http://10.160.10.254
```
* **Resultado Esperado:** Se visualiza la página web servida por `MicroVM-DMZ`. El Port Forwarding (DNAT) funciona perfectamente a través de `vSwitch0` $\rightarrow$ `VM-Router` $\rightarrow$ `vSwitch-DMZ`.

---

###  Prueba 2: Salida a Internet desde la DMZ
Desde la consola de `MicroVM-DMZ`, haz ping a la puerta de enlace de VMware Workstation:
```ash
ping -c 3 10.160.10.2
```
* **Resultado Esperado:** `0% packet loss`. La DMZ puede actualizar paquetes o descargar dependencias saliendo por NAT.

---

###  Prueba 3: Auditoría de Seguridad (Bloqueo DMZ hacia LAN Interna)
Desde la consola de `MicroVM-DMZ` (`10.160.50.50`), intenta atacar o hacer ping al servidor interno `MicroVM-Interna` (`10.160.10.60`):
```ash
ping -c 3 10.160.10.60
```
* **Resultado Esperado:** `100% packet loss` (*Bloqueado exitosamente por la regla de firewall en la VM-Router*). La red interna está a salvo aunque el servidor web sea vulnerado.

---

###  Prueba 4: Validación de Storage Aislado a 9000 MTU
Desde la consola SSH del ESXi (`10.160.10.10`), valida que el canal de almacenamiento opera sin fragmentar:
```bash
vmkping -I vmk1 -d -s 8972 10.160.30.100
```
* **Resultado Esperado:** `0% packet loss`. El tráfico de discos viaja por `vSwitch0` a velocidad de bus y no pasa por el router.

---

##  Conclusión del Laboratorio 2
Has construido una **Zona Desmilitarizada (DMZ) de grado corporativo**:
1. El servidor web es públicamente accesible sin exponer la IP interna.
2. Un compromiso en la DMZ no permite el movimiento lateral hacia la red de producción.
3. El tráfico de almacenamiento crítico permanece 100% aislado con tramas Jumbo (MTU 9000).
