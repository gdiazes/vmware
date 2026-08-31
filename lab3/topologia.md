
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

1. El servidor web es públicamente accesible sin exponer la IP interna.
2. Un compromiso en la DMZ no permite el movimiento lateral hacia la red de producción.
3. El tráfico de almacenamiento crítico permanece 100% aislado con tramas Jumbo (MTU 9000).
