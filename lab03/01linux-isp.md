# GUÍA DE CONFIGURACIÓN Y VALIDACIÓN: Router Linux-ISP (`linux-isp`)

Esta guía detalla los pasos exactos para configurar la máquina virtual de Alpine Linux como nuestro proveedor de servicios de internet simulado (ISP) y pasarela de borde con **5 interfaces de red**, ajustada al entorno virtual y los segmentos de la topología.

---

## 1. PASO PREVIO: Mapeo de Adaptadores de Red en VMware
Asegúrate de agregar **5 adaptadores de red virtuales** a la máquina virtual de Alpine Linux en VMware Workstation, asignando redes personalizadas (*Custom: VMnetX*) de acuerdo con las etiquetas de interconexión de la topología:

### Tabla Resumen de Mapeo en VMware para `linux-isp`
| Adaptador VM | Interfaz en Linux | Segmento / Red | IP Asignada | Destino / Etiqueta Topología |
| :--- | :--- | :--- | :--- | :--- |
| **Adapter 1** | `eth0` | NAT  | `10.160.10.100` | Salida a Internet Real |
| **Adapter 2** | `eth1` | * WAN1* | `100.65.0.254` | Enlace **WAN1** hacia HQ-FGT (`Port1`) |
| **Adapter 3** | `eth2` | * WAN2* | `100.66.0.254` | Enlace **WAN2** hacia HQ-FGT (`Port2`) |
| **Adapter 4** | `eth3` | * WAN3* | `100.65.1.254` | Enlace **WAN3** hacia BR1-FGT (`Port3`) |
| **Adapter 5** | `eth4` | * WAN4* | `100.66.1.254` | Enlace **WAN4** hacia BR1-FGT (`Port2`) |

---

## 2. PASO 1: Configuración del Hostname y Zona Horaria (Lima)
Enciende la máquina virtual, inicia sesión como **`root`** (sin contraseña inicial) y ejecuta los siguientes comandos:

1. **Configurar el Hostname a `linux-isp`:**
   ```bash
   setup-hostname linux-isp
   ```
   *(Asegura la resolución local editando el archivo hosts)*:
   ```bash
   echo "127.0.0.1 linux-isp.localdomain linux-isp" >> /etc/hosts
   ```

2. **Configurar la Zona Horaria de Lima (Perú):**
   ```bash
   apk update && apk add tzdata
   setup-timezone -z America/Lima
   ```

3. **Reiniciar el equipo para aplicar el hostname:**
   ```bash
   reboot
   ```
   *(Vuelve a iniciar sesión como `root` tras el reinicio).*

---

## 3. PASO 2: Configuración del Archivo de Red
Edita el archivo de interfaces de red de Alpine para asignar las IPs estáticas a cada puerto:

```bash
vi /etc/network/interfaces
```

Reemplaza o ajusta el contenido para que coincida exactamente con esta estructura:

```text
auto lo
iface lo inet loopback

# eth0: Salida principal a Internet (NAT)
auto eth0
iface eth0 inet static
    address 10.160.10.100
    netmask 255.255.255.0
    gateway 10.160.10.2

# eth1: WAN1 hacia HQ-FGT (Port1)
auto eth1
iface eth1 inet static
    address 100.65.0.254
    netmask 255.255.255.0

# eth2: WAN2 hacia HQ-FGT (Port2)
auto eth2
iface eth2 inet static
    address 100.66.0.254
    netmask 255.255.255.0

# eth3: WAN3 hacia BR1-FGT (Port3)
auto eth3
iface eth3 inet static
    address 100.65.1.254
    netmask 255.255.255.0

# eth4: WAN4 hacia BR1-FGT (Port2)
auto eth4
iface eth4 inet static
    address 100.66.1.254
    netmask 255.255.255.0
```
*(Para guardar y salir en `vi`: presiona `Esc`, escribe `:wq` y presiona **Enter**).*

Reinicia el servicio de red local para aplicar los cambios:
```bash
rc-service networking restart
```

---

## 4. PASO 3: Habilitación de Enrutamiento y NAT (Iptables)
Para permitir que el tráfico de los firewalls atraviese el router `linux-isp` hacia el exterior, habilitamos el reenvío y el enmascaramiento:

```bash
# Instalar iptables
apk add iptables

# Habilitar el reenvío de paquetes IP de forma permanente
sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf

# Limpiar reglas previas y configurar NAT saliente por eth0
iptables -t nat -F
iptables -F FORWARD

iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT
iptables -A FORWARD -i eth2 -o eth0 -j ACCEPT
iptables -A FORWARD -i eth3 -o eth0 -j ACCEPT
iptables -A FORWARD -i eth4 -o eth0 -j ACCEPT
```

---

## 5. PASO 4: Validación Final del Router `linux-isp` (Check-list)

Verifica que cada uno de los siguientes puntos cumpla con el estado esperado:

- [ ] **1. Verificación de Hostname y Hora Local:**
  ```bash
  hostname
  date
  ```
  * *Estado esperado:* Debe retornar `linux-isp` y la fecha/hora correspondiente a la zona horaria de Lima.

- [ ] **2. Verificación de Interfaces de Red:**
  ```bash
  ip add
  ```
  * *Estado esperado:* Listado de `eth0` a `eth4` activas con la etiqueta `UP` y sus respectivas direcciones IP (`10.160.10.100`, `100.65.0.254`, `100.66.0.254`, `100.65.1.254`, `100.66.1.254`).

- [ ] **3. Prueba de Conectividad IP Internacional:**
  ```bash
  ping -c 3 8.8.8.8
  ```
  * *Estado esperado:* `0% packet loss` (IPs de Google respondiendo correctamente).

- [ ] **4. Prueba de Resolución DNS y Conectividad Local/Regional (`ln.pe`):**
  ```bash
  ping -c 3 ln.pe
  ```
  * *Estado esperado:* Resolución de nombre exitosa y `0% packet loss`, validando que el DNS y la salida global funcionan de extremo a extremo.

- [ ] **5. Verificación de Reglas de NAT:**
  ```bash
  iptables -t nat -L -v -n
  ```
  * *Estado esperado:* Presencia de la regla `MASQUERADE` enlazada a la interfaz `eth0`.

---
