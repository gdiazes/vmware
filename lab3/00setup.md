###  Parámetros Globales de Red (VMware Workstation NAT Network)
* **Subred de Gestión (Management Subnet):** `10.160.10.0/24`
* **Puerta de Enlace Predeterminada (Workstation NAT Gateway):** `10.160.10.2`
* **Servidor DNS (Workstation NAT DNS):** `10.160.10.2`
* **IP del Servidor ESXi (`vmk0`):** `10.160.10.10`
* **Máscara de Subred:** `255.255.255.0` (`/24`)

```
  +-----------------------------------------------------------------------------------------+
  |                           VMWARE WORKSTATION HOST (Hypervisor L1)                       |
  |   +---------------------------------------------------------------------------------+   |
  |   | Virtual Network Editor (NAT VMnet8): Subred 10.160.10.0/24 | Gateway 10.160.10.2|   |
  |   +---------------------------------------+-----------------------------------------+   |
  |                                           |                                             |
  |             +-----------------------------v-----------------------------+               |
  |             |             VM ANIDADA: ESXi 8.0 (Hypervisor L2)          |               |
  |             |             IP Gestión: 10.160.10.10                      |               |
  |             |             Adaptadores de Red: vmnic0, vmnic1            |               |
  |             +-----------------------------------------------------------+               |
  +-----------------------------------------------------------------------------------------+
```

---

##  FASE PREVIA: Despliegue de la Máquina Virtual Ultraligera (Alpine Micro-VM)

Para validar la conectividad en cada laboratorio sin agotar la memoria RAM de tu máquina física anfitriona, utilizaremos **Alpine Linux (Virtual Edition)**.

### Especificaciones Técnicas:
* **vCPU:** 1 vCPU | **RAM:** 128 MB | **Disco:** 1 GB (*Thin Provisioning* — uso real: ~50 MB)
* **ISO:** `alpine-virt-3.x.x-x86_64.iso` (~60 MB)

---

### Paso a Paso: Creación de la Micro-VM

#### 1. Carga de la ISO al Datastore
1. Descarga la ISO oficial: [Alpine Linux Virtual ISO](https://alpinelinux.org/downloads/).
2. Accede al **ESXi Host Client**: `https://10.160.10.10/ui`.
3. Ve a **Almacenamiento (Storage)** > Selecciona el datastore > **Explorador de almacenes de datos (Datastore browser)**.
4. Crea la carpeta `ISOs` y haz clic en **Cargar (Upload)** para subir el archivo `.iso`.

* **Resultado Esperado:** Archivo disponible en `[datastore1] ISOs/alpine-virt-*.iso`.

#### 2. Creación y Registro de la VM
1. Ve a **Máquinas virtuales (Virtual Machines)** > **Crear / Registrar máquina virtual**.
2. Parámetros del Asistente:
   * **Nombre:** `Template-Alpine-MicroVM`
   * **Compatibilidad:** `ESXi 8.0 virtual machine`
   * **Familia de SO:** `Linux` | **Versión:** `Other Linux (64-bit)`
3. En **Personalizar configuración**:
   * **CPU:** `1` | **Memoria:** `128 MB`
   * **Disco duro 1:** `1 GB` > Marcar **Aprovisionamiento fino (Thin provisioned)**.
   * **Adaptador de red 1:** Red `VM Network` > Tipo: **VMXNET3**.
   * **Unidad de CD/DVD 1:** Seleccionar **Archivo ISO de almacén de datos** y apuntar a la ISO de Alpine.
4. Finaliza el asistente.

* **Resultado Esperado:** VM registrada consumiendo únicamente 128 MB de RAM.

#### 3. Configuración de Red en Alpine (Ejecución en Memoria)
1. Enciende la VM y abre su **Consola Web**. Inicia sesión con el usuario `root` (sin contraseña).
2. Comando para fijar direccionamiento estático apuntando al gateway de VMware Workstation:
```ash
ip addr add 10.160.10.50/24 dev eth0
ip link set eth0 up
ip route add default via 10.160.10.2
```

>  **Nota Conceptual - El Adaptador VMXNET3:**  
> El adaptador **VMXNET3** es un controlador paravirtualizado optimizado para vSphere que ofrece soporte nativo para *Jumbo Frames* (MTU 9000), multicores de recepción (RSS) y mínima sobrecarga de CPU en comparación con controladores emulados como *E1000e*.


2. **Aislamiento de Seguridad L2 y L3:**  
   * A nivel **L2**, las políticas de seguridad (*Promiscuous Mode*, *MAC Changes*, *Forged Transmits*) en estado **Reject** previenen ataques de *man-in-the-middle* y suplantación dentro del hipervisor.
   * A nivel **L3**, las **Pilas TCP/IP Dedicadas (*vMotion* y *Custom Stacks*)** eliminan el riesgo de enrutamiento asimétrico, permitiendo que cada servicio del hipervisor tenga su propia tabla de rutas y puerta de enlace independiente de la red de administración (`vmk0` en `10.160.10.10`).

3. **Diagnóstico Forense:**  
   Herramientas nativas de ESXi 8 como `esxcli network ip netstack`, `vmkping` y `pktcap-uw` son esenciales para validar la correcta encapsulación de paquetes, el soporte de *Jumbo Frames* y la ausencia de fugas de tráfico entre pilas de red.
