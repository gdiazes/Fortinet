Para deshabilitar las políticas de complejidad de contraseñas (que exigen mayúsculas, símbolos o longitudes mayores) y fijar la contraseña del usuario `admin` a **`tecsup00`**, se utilizan dos bloques en FortiOS:

1. **`config system password-policy`**: Desactiva las restricciones de complejidad y expiración.
2. **`config system admin`**: Asigna la contraseña `tecsup00` directamente en texto claro al usuario administrador.

---

### Bloque de Configuración Actualizado para `BR1-FGT`

Copia y pega este bloque completo en la consola de **`BR1-FGT`**:

```fortios
# ==============================================================
# 1. DESHABILITAR POLÍTICA DE COMPLEJIDAD Y ASIGNAR PASSWORD
# ==============================================================
config system password-policy
    set status disable
end

config system admin
    edit "admin"
        set password "tecsup00"
    next
end

# ==============================================================
# 2. PARÁMETROS GLOBALES Y ZONA HORARIA (LIMA)
# ==============================================================
config system global
    set hostname "BR1-FGT"
    set timezone 59
    set admintimeout 60
end

# ==============================================================
# 3. CONFIGURACIÓN DE INTERFACES (port1, port2 y port3)
# ==============================================================
config system interface
    edit "port1"
        set alias "WAN3"
        set mode static
        set ip 100.65.1.111 255.255.255.0
        set allowaccess ping
        set role wan
        set status up
    next
    edit "port2"
        set alias "WAN4"
        set mode static
        set ip 100.66.1.111 255.255.255.0
        set allowaccess ping
        set role wan
        set status up
    next
    edit "port3"
        set alias "LANBR1"
        set mode static
        set ip 172.20.1.254 255.255.255.0
        set allowaccess ping https ssh http
        set role lan
        set status up
    next
end

# ==============================================================
# 4. ENRUTAMIENTO ESTÁTICO HACIA LINUX-ISP
# WAN3 Primario (Prioridad 10) / WAN4 Respaldo (Prioridad 20)
# ==============================================================
config router static
    edit 1
        set dst 0.0.0.0 0.0.0.0
        set gateway 100.65.1.254
        set device "port1"
        set distance 10
        set priority 10
        set comment "Default-GW-WAN3"
    next
    edit 2
        set dst 0.0.0.0 0.0.0.0
        set gateway 100.66.1.254
        set device "port2"
        set distance 10
        set priority 20
        set comment "Default-GW-WAN4-Backup"
    next
end

# ==============================================================
# 5. CONFIGURACIÓN DE DNS
# ==============================================================
config system dns
    set primary 8.8.8.8
    set secondary 1.1.1.1
end

# ==============================================================
# 6. OBJETO DE RED PARA LA LAN DE BRANCH
# ==============================================================
config firewall address
    edit "NET_LAN_BR1"
        set subnet 172.20.1.0 255.255.255.0
        set comment "Segmento LAN Sucursal Branch"
    next
end

# ==============================================================
# 7. POLÍTICAS DE FIREWALL (Salida a Internet con NAT)
# ==============================================================
config firewall policy
    edit 1
        set name "LANBR1_to_WAN3"
        set srcintf "port3"
        set dstintf "port1"
        set action accept
        set srcaddr "NET_LAN_BR1"
        set dstaddr "all"
        set schedule "always"
        set service "ALL"
        set nat enable
        set comments "Salida a Internet via WAN3"
    next
    edit 2
        set name "LANBR1_to_WAN4"
        set srcintf "port3"
        set dstintf "port2"
        set action accept
        set srcaddr "NET_LAN_BR1"
        set dstaddr "all"
        set schedule "always"
        set service "ALL"
        set nat enable
        set comments "Salida de respaldo via WAN4"
    next
end
```

---

### Verificación del cambio de credenciales

1. Abre una nueva pestaña o sesión SSH hacia la IP de gestión de `port3` (`172.20.1.254`) o desde la consola:
   * **Usuario:** `admin`
   * **Contraseña:** `tecsup00`
2. El acceso será inmediato y sin advertencias de complejidad ni peticiones de cambio forzado de clave.
