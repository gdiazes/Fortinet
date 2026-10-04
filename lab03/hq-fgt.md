# GUÍA DEFINITIVA: Setup Inicial y Validación de HQ-NGFW-1 (FortiGate-VM)

Esta guía detalla los pasos exactos para configurar la máquina virtual **`HQ-NGFW-1`** en VMware Workstation como el firewall de borde de la sede principal (HQ). Contempla la fase inicial de licenciamiento temporal vía DHCP/NAT, el mapeo ordenado de sus 3 adaptadores virtuales, la desactivación de la política de contraseñas para fijar `tecsup00`, la configuración de interfaces según la topología, el enrutamiento redundante, las políticas con NAT, el sistema de alias y el guardado en la tabla de revisiones.

---

### 1. PASO PREVIO: Mapeo de Adaptadores de Red en VMware Workstation

En los ajustes de la máquina virtual `HQ-NGFW-1` en VMware Workstation, configura exactamente 3 adaptadores de red:

#### Tabla Resumen de Mapeo en VMware para HQ-NGFW-1

| Adaptador VM | Interfaz FortiOS | Red / Segmento VMware | IP a Asignar | Gateway / Destino en Topología |
| :--- | :--- | :--- | :--- | :--- |
| **Network Adapter 1** | **`port1`** | **Modo NAT** *(Fase 1: Licencia)*<br>$\rightarrow$ Luego: **LAN Segment: WAN1** | `100.65.0.101/24` | Enlace a `linux-isp` (`eth1: 100.65.0.254`) |
| **Network Adapter 2** | **`port2`** | **LAN Segment: WAN2** | `100.66.0.101/24` | Enlace a `linux-isp` (`eth2: 100.66.0.254`) |
| **Network Adapter 3** | **`port3`** | **LAN Segment: LANHQ** | `10.0.11.254/24` | Red local HQ (`HQ-PC-01: 10.0.11.50`) |

---

### 2. PASO 1: Conexión Temporal por NAT y Activación de Licencia

1. **Configurar en VMware:** Coloca el **Network Adapter 1** temporalmente en modo **NAT** y asegúrate de que **Connected** esté marcado.
2. **Encender la VM e iniciar sesión:** Usuario `admin` (sin contraseña inicial).
3. **Configurar `port1` en DHCP desde la consola:**
   ```fortios
   config system interface
       edit "port1"
           set mode dhcp
           set allowaccess ping https ssh http
           set status up
       next
   end
   ```
4. **Verificar salida real a Internet:**
   ```fortios
   execute ping 8.8.8.8
   execute ping directregistration.fortinet.com
   ```
5. **Activar la licencia:**
   * Abre un navegador en tu máquina física ingresando a la IP que `port1` obtuvo por DHCP (`https://<IP_DHCP>`), autentícate con tus credenciales de FortiCloud o carga tu archivo de evaluación `.lic`.
6. **Confirmar que la licencia esté activa:**
   ```fortios
   get system status | grep License
   ```
   *(El estado de `License Status` debe ser `Valid`).*
7. **Regresar el adaptador a la topología:**
   * En los ajustes de la VM en VMware, cambia el **Network Adapter 1** de modo **NAT** al segmento definitivo: **LAN Segment: WAN1**.

---

### 3. PASO 2: Contraseña `tecsup00`, Hostname y Zona Horaria (Lima)

Copia y pega este bloque en el CLI para remover las restricciones de contraseña, asignar `tecsup00` y fijar la zona horaria de Lima (código `59`):

```fortios
# 1. Desactivar complejidad y asignar contraseña
config system password-policy
    set status disable
end

config system admin
    edit "admin"
        set password "tecsup00"
    next
end

# 2. Hostname, Zona Horaria (Lima) y guardado automático al cerrar sesión
config system global
    set hostname "HQ-NGFW-1"
    set timezone 59
    set admintimeout 60
    set revision-backup-on-logout enable
end
```

---

### 4. PASO 3: Configuración de Interfaces de Red y Servidores DNS

Configura las direcciones IP estáticas definitivas según el esquema de red:

```fortios
# 1. Configuración de interfaces
config system interface
    edit "port1"
        set alias "WAN1"
        set mode static
        set ip 100.65.0.101 255.255.255.0
        set allowaccess ping ssh
        set role wan
        set status up
    next
    edit "port2"
        set alias "WAN2"
        set mode static
        set ip 100.66.0.101 255.255.255.0
        set allowaccess ping ssh
        set role wan
        set status up
    next
    edit "port3"
        set alias "LANHQ"
        set mode static
        set ip 10.0.11.254 255.255.255.0
        set allowaccess ping https ssh http
        set role lan
        set status up
    next
end

# 2. Servidores DNS públicos
config system dns
    set primary 8.8.8.8
    set secondary 1.1.1.1
end
```

---

### 5. PASO 4: Enrutamiento Estático y Políticas de Salida con NAT

Configura el esquema de failover activo/respaldo hacia `linux-isp` (WAN1 con prioridad 10 y WAN2 con prioridad 20), junto con las políticas de seguridad para permitir tráfico desde la LAN:

```fortios
# 1. Rutas por defecto hacia el ISP
config router static
    edit 1
        set dst 0.0.0.0 0.0.0.0
        set gateway 100.65.0.254
        set device "port1"
        set distance 10
        set priority 10
        set comment "Default-GW-WAN1-Principal"
    next
    edit 2
        set dst 0.0.0.0 0.0.0.0
        set gateway 100.66.0.254
        set device "port2"
        set distance 10
        set priority 20
        set comment "Default-GW-WAN2-Backup"
    next
end

# 2. Objeto de red para la LAN de HQ
config firewall address
    edit "NET_LAN_HQ"
        set subnet 10.0.11.0 255.255.255.0
        set comment "Subred LAN HQ"
    next
end

# 3. Políticas de navegación con NAT
config firewall policy
    edit 1
        set name "LAN_to_WAN1"
        set srcintf "port3"
        set dstintf "port1"
        set action accept
        set srcaddr "NET_LAN_HQ"
        set dstaddr "all"
        set schedule "always"
        set service "ALL"
        set nat enable
        set comments "Salida principal a Internet via WAN1"
    next
    edit 2
        set name "LAN_to_WAN2"
        set srcintf "port3"
        set dstintf "port2"
        set action accept
        set srcaddr "NET_LAN_HQ"
        set dstaddr "all"
        set schedule "always"
        set service "ALL"
        set nat enable
        set comments "Salida de respaldo via WAN2"
    next
end
```

---

### 6. PASO 5: Sistema de Alias y Guardado en Tabla de Revisiones

Configura los atajos para saltar por SSH a los demás equipos de la topología y facilitar los comandos de diagnóstico, cerrando con el guardado del punto de restauración maestro:

```fortios
# 1. Atajos de conectividad SSH y diagnóstico
config system alias
    edit "branch"
        set command "execute ssh admin@100.65.1.111"
    next
    edit "branch2"
        set command "execute ssh admin@100.66.1.111"
    next
    edit "isp"
        set command "execute ssh root@100.65.0.254"
    next
    edit "pc"
        set command "execute ssh alumno@10.0.11.50"
    next
    edit "rt"
        set command "get router info routing-table all"
    next
    edit "arp"
        set command "get system arp"
    next
    edit "snif"
        set command "diagnose sniffer packet any 'icmp' 4 0 l"
    next
    edit "sessions"
        set command "diagnose sys session list"
    next
end
```

#### Guardar la Revisión Maestra en la Flash interna:
```fortios
execute backup config flash "Setup Inicial HQ-NGFW-1 Completo y Validado"
```

---

### 7. PASO 6: Validación Final de HQ-NGFW-1 (Check-list)

Verifica que cada uno de los siguientes puntos cumpla con el estado esperado:

#### 1. Verificación de Hora Local y Hostname:
```fortios
execute time
```
* **Estado esperado:** Prompt `HQ-NGFW-1 #` con fecha y hora correspondiente a Lima (GMT-5).

#### 2. Verificación de Interfaces y Capa 2:
```fortios
alias arp
```
* **Estado esperado:** MACs aprendidas para `100.65.0.254` (port1), `100.66.0.254` (port2) y `10.0.11.50` (port3).

#### 3. Verificación de Enrutamiento:
```fortios
alias rt
```
* **Estado esperado:**
  * Dos rutas por defecto `S* 0.0.0.0/0` vía `100.65.0.254` (prioridad 10) y `100.66.0.254` (prioridad 20).
  * Tres redes directamente conectadas (`C`): `100.65.0.0/24` (port1), `100.66.0.0/24` (port2) y `10.0.11.0/24` (port3).

#### 4. Pruebas de Conectividad hacia el ISP y Salida a Internet:
```fortios
execute ping 100.65.0.254
execute ping 100.66.0.254
execute ping 8.8.8.8
execute ping google.com
```
* **Estado esperado:** `0% packet loss` en todas las pruebas.

#### 5. Prueba de Salto SSH hacia la Sucursal Branch:
```fortios
alias branch
```
* **Estado esperado:** Conexión exitosa por SSH hacia `BR1-FGT` solicitando la contraseña `tecsup00`.

#### 6. Verificación de la Revisión Guardada:
```fortios
execute revision list config
```
* **Estado esperado:** Debe listar la revisión guardada con su respectivo `ID` y comentario `"Setup Inicial HQ-NGFW-1 Completo y Validado"`.
