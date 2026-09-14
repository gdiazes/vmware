Instalación de Alpine Linux + Servidor DNS y NTP

---

## FASE 1: Preparación del Medio

1. **Descarga:** Obtén la ISO en [alpinelinux.org/downloads](https://alpinelinux.org/downloads/) (elige la versión **Standard** para hardware real o **Virtual** para máquinas virtuales).
2. **Creación del USB:** Quema la imagen en una memoria USB utilizando **Rufus** (modo DD), **Ventoy** o mediante terminal con:
   ```bash
   dd if=alpine-standard-*.iso of=/dev/sdX bs=4M status=progress
   ```
3. **Arranque:** Inicia el equipo desde el medio USB configurándolo en la BIOS/UEFI.

---

## FASE 2: Inicio del Entorno en Vivo

1. En el indicador de inicio, verás:
   ```text
   localhost login:
   ```
2. Escribe **`root`** y pulsa `Enter` (no solicita clave inicialmente).
3. Inicia el script de instalación automática:
   ```bash
   setup-alpine
   ```

---

## FASE 3: Asistente de Instalación Base (`setup-alpine`)

Responde a las preguntas del asistente paso a paso:

1. **Keyboard Layout (Teclado):** Escribe `es` (España) o `latam` (Latinoamérica).
2. **Hostname:** Asígnale un nombre descriptivo a tu servidor, por ejemplo: `servidor-red`.
3. **Network (Red):**
   * Selecciona la interfaz primaria (ej. `eth0`).
   * **Importante:** Al tratarse de un servidor DNS y NTP, se recomienda asignar una **IP Estática**:
     * En lugar de `dhcp`, escribe `manual` o configura la IP cuando te pregunte (ej. `192.168.1.10`), máscara de subred (`255.255.255.0` o `24`) y puerta de enlace (ej. `192.168.1.1`).
4. **Root Password:** Establece una clave segura para el superusuario.
5. **Timezone:** Escribe tu región y ciudad (ej. `America/Mexico_City`, `Europe/Madrid`).
6. **Proxy:** Presiona `Enter` para `none`.
7. **NTP Client:** Selecciona **`chrony`** (fundamental para el paso posterior).
8. **Mirror (Repositorios):** Escribe **`f`** para que el sistema evalúe y seleccione automáticamente el servidor de descarga más veloz.
9. **Setup a user:** Puedes omitirlo por ahora pulsando `no` o crear un usuario estándar.
10. **SSH Server:** Selecciona **`openssh`**.
11. **Almacenamiento (Disco):**
    * Selecciona el disco de destino (ej. `sda`, `vda` o `nvme0n1`).
    * Método de uso: Escribe obligatoriamente **`sys`** (esto realiza una instalación tradicional en disco persistente).
    * Confirma el formateo escribiendo **`y`**.

---

## FASE 4: Reinicio y Post-Instalación

1. Finalizada la instalación, retira el USB y reinicia:
   ```bash
   reboot
   ```
2. Inicia sesión como **`root`** con la clave que creaste.
3. Habilita los repositorios comunitarios editando el archivo con el editor `vi`:
   ```bash
   vi /etc/apk/repositories
   ```
   *(Descomenta la línea que termina en `/community` quitando el `#`, presiona `ESC`, escribe `:wq` y pulsa `Enter`)*.
4. Actualiza los índices y el sistema:
   ```bash
   apk update && apk upgrade
   ```

---

## FASE 5: Configuración del Servidor NTP (Chrony)

Por defecto, Alpine instaló `chrony` solo como cliente. Ahora lo configuraremos para que actúe como **servidor horario** para los demás equipos de tu red local.

1. **Editar la configuración de Chrony:**
   ```bash
   vi /etc/chrony/chrony.conf
   ```
2. **Permitir peticiones de red:** Busca la directiva `allow` y especifica el rango de red que podrá sincronizarse con este servidor (ajusta según tu segmento local):
   ```text
   # Permitir sincronización a la red local
   allow 192.168.1.0/24
   ```
3. *(Opcional)* Si este servidor pierde conexión a Internet, puedes indicarle que siga sirviendo su hora local como confiable:
   ```text
   local stratum 10
   ```
4. **Habilitar e iniciar el servicio en OpenRC:**
   ```bash
   rc-update add chronyd default
   rc-service chronyd restart
   ```
5. **Verificar el funcionamiento:**
   ```bash
   chronyc tracking
   ```
   *(El puerto UDP 123 quedará abierto a la escucha).*

---

## FASE 6: Configuración del Servidor DNS (Dnsmasq)

Utilizaremos **`dnsmasq`**, el estándar predilecto en Alpine por su bajísimo consumo de recursos (menos de 5 MB de RAM), capacidad de caché veloz y resolución de nombres locales.

1. **Instalar Dnsmasq:**
   ```bash
   apk add dnsmasq
   ```
2. **Respaldar el archivo original y crear uno limpio:**
   ```bash
   mv /etc/dnsmasq.conf /etc/dnsmasq.conf.bak
   vi /etc/dnsmasq.conf
   ```
3. **Pegar la siguiente configuración básica:**
   ```text
   # Escuchar en la interfaz loopback y en la IP de la red local
   listen-address=127.0.0.1,192.168.1.10
   port=53

   # Servidores DNS upstream (hacia donde reenviar si no está en caché)
   server=1.1.1.1
   server=8.8.8.8

   # No reenviar nombres simples (sin punto)
   domain-needed
   bogus-priv

   # Tamaño de la caché de consultas DNS
   cache-size=1000

   # Definir un dominio local
   domain=mired.local
   local=/mired.local/

   # Asignar nombres fijos locales directamente (nombre -> IP)
   address=/servidor.mired.local/192.168.1.10
   address=/nas.mired.local/192.168.1.20
   ```
   *(Asegúrate de cambiar `192.168.1.10` por la IP fija real de tu servidor Alpine).*

4. **Habilitar e iniciar el servicio:**
   ```bash
   rc-update add dnsmasq default
   rc-service dnsmasq start
   ```

---

## FASE 7: Pruebas de Funcionamiento

Para verificar que tus servicios responden correctamente, instala la herramienta `bind-tools` (incluye `dig` y `nslookup`):
```bash
apk add bind-tools
```

1. **Probar el Servidor DNS localmente:**
   ```bash
   dig @127.0.0.1 servidor.mired.local +short
   # Debe devolver: 192.168.1.10

   dig @127.0.0.1 google.com +short
   # Debe devolver la IP de Google (resolución hacia internet funcional)
   ```

2. **Probar el Servidor NTP:**
   ```bash
   chronyc sources -v
   ```

