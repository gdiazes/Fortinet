

# GUÍA TÉCNICA PASO A PASO: VPN IPsec Site-to-Site (Dual-ISP) con FortiOS 7.6 y Linux-ISP

## 1. OBJETIVOS DEL LABORATORIO
* Configurar un entorno de simulación avanzado utilizando **Alpine Linux** como router centralizador (Linux-ISP) con múltiples interfaces y NAT estático.
* Implementar un nodo FortiGate único en la Sede Principal (**HQ-FGT**) y un nodo FortiGate en la Sucursal (**BR1-FGT**).
* Establecer un túnel **VPN IPsec de Sitio a Sitio (Site-to-Site)** utilizando IKEv2 entre las sedes.
* Validar la comunicación de extremo a extremo entre los equipos finales de ambas redes LAN (`HQ-PC-1` y `BR1-PC-1`).

---

## 2. MAPA DE LA TOPOLOGÍA Y ESQUEMA DE DIRECCIONAMIENTO

```text
================================================================================================
                                        INTERNET EXTERNA
================================================================================================
                                               |
                                               | (eth0: IP Fija 10.160.10.100 / NAT)
+----------------------------------------------------------------------------------------------+
|                                    ROUTER LINUX - ISP                                        |
|                                                                                              |
|  - eth2 (Hacia HQ-WAN1)  : 100.65.0.254/24 <----+                +-----> eth4 (Br-WAN1) : 100.65.1.254/24  |
|  - eth3 (Hacia HQ-WAN2)  : 100.66.0.254/24 <--+ |                | +---> eth5 (Br-WAN2) : 100.66.1.254/24  |
+-----------------------------------------------|-|----------------|-|--------------------------+
                                                | |                | |
             +----------------------------------+ |                | +-------------------------+
             |                                    |                |                           |
             v                                    v                v                           v
+-------------------------------+                                      +-------------------------------+
|         SEDE HQ (HQ-FGT)      |                                      |      SUCURSAL (BR1-FGT)       |
|                               |                                      |                               |
| port2 (WAN1): 100.65.0.101/24 |                                      | port2 (WAN1): 100.65.1.111/24 |
| port3 (WAN2): 100.66.0.101/24 |                                      | port3 (WAN2): 100.66.1.111/24 |
| port4 (LAN) : 10.0.11.254/24  |                                      | port4 (LAN) : 172.20.1.254/24 |
+-------------------------------+                                      +-------------------------------+
             |                                                                        |
             v                                                                        v
+-------------------------------+                                      +-------------------------------+
|           HQ-PC-1             |                                      |           BR1-PC-1            |
|    IP: 10.0.11.50/24          | <========== [ TÚNEL IPSEC ] =========> |    IP: 172.20.1.60/24         |
+-------------------------------+            (IKEv2 / Cifrado)         +-------------------------------+
```

---

## FASE 1: Configuración del Router Central (Linux - ISP / Alpine Linux)

La máquina virtual Alpine Linux actuará como nuestro proveedor y pasarela de internet con **5 interfaces de red**.

1. **Configuración del archivo de red (`/etc/network/interfaces`):**
   ```text
   auto lo
   iface lo inet loopback

   auto eth0
   iface eth0 inet static
       address 10.160.10.100
       netmask 255.255.255.0
       gateway 10.160.10.2

   auto eth2
   iface eth2 inet static
       address 100.65.0.254
       netmask 255.255.255.0

   auto eth3
   iface eth3 inet static
       address 100.66.0.254
       netmask 255.255.255.0

   auto eth4
   iface eth4 inet static
       address 100.65.1.254
       netmask 255.255.255.0

   auto eth5
   iface eth5 inet static
       address 100.66.1.254
       netmask 255.255.255.0
   ```
   Aplica los cambios ejecutando: `rc-service networking restart`

2. **Habilitación de Enrutamiento y NAT en Linux-ISP:**
   ```bash
   sysctl -w net.ipv4.ip_forward=1
   echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf

   # Instalar iptables si no estuviese presente y configurar enmascaramiento
   apk update && apk add iptables
   iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
   iptables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
   iptables -A FORWARD -i eth2 -o eth0 -j ACCEPT
   iptables -A FORWARD -i eth3 -o eth0 -j ACCEPT
   iptables -A FORWARD -i eth4 -o eth0 -j ACCEPT
   iptables -A FORWARD -i eth5 -o eth0 -j ACCEPT
   ```

---

## FASE 2: Configuración Detallada de HQ-FGT (Sede Principal)

Abre la consola CLI de tu FortiGate en HQ e introduce el siguiente bloque de comandos:

```conf
# 1. Configuración de Interfaces Físicas
config system interface
    edit "port2"
        set mode static
        set ip 100.65.0.101 255.255.255.0
        set allowaccess ping https ssh
        set comment "WAN 1 - ISP Principal"
    next
    edit "port3"
        set mode static
        set ip 100.66.0.101 255.255.255.0
        set allowaccess ping https ssh
        set comment "WAN 2 - ISP Secundario"
    next
    edit "port4"
        set mode static
        set ip 10.0.11.254 255.255.255.0
        set allowaccess ping
        set comment "Red LAN HQ"
    next
end

# 2. Rutas Estáticas con prioridades (Dual-WAN)
config router static
    edit 1
        set dst 0.0.0.0 0.0.0.0
        set gateway 100.65.0.254
        set device "port2"
        set priority 10
    next
    edit 2
        set dst 0.0.0.0 0.0.0.0
        set gateway 100.66.0.254
        set device "port3"
        set priority 20
    next
end

# 3. Configuración VPN IPsec (Fase 1 y Fase 2 sobre port2 / WAN1)
config vpn ipsec phase1-interface
    edit "To_Branch"
        set interface "port2"
        set ike-version 2
        set peertype any
        set net-device enable
        set proposal aes256-sha256 aes128-sha1
        set remote-gw 100.65.1.111
        set psksecret FortiTopo2026*
    next
end

config vpn ipsec phase2-interface
    edit "To_Branch_p2"
        set phase1name "To_Branch"
        set proposal aes256-sha256 aes128-sha1
        set src-subnet 10.0.11.0 255.255.255.0
        set dst-subnet 172.20.1.0 255.255.255.0
    next
end

# 4. Políticas de Firewall (Tráfico bidireccional sin NAT)
config firewall policy
    edit 10
        set name "HQ_LAN_to_Branch_VPN"
        set srcintf "port4"
        set dstintf "To_Branch"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat disable
    next
    edit 11
        set name "Branch_VPN_to_HQ_LAN"
        set srcintf "To_Branch"
        set dstintf "port4"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat disable
    next
end

# 5. Ruta estática hacia la subred de la sucursal a través del túnel
config router static
    edit 10
        set dst 172.20.1.0 255.255.255.0
        set device "To_Branch"
    next
end
```

---

## FASE 3: Configuración Detallada de BR1-FGT (Sucursal)

Abre la consola CLI de tu FortiGate en la Sucursal e introduce el siguiente bloque de comandos:

```conf
# 1. Configuración de Interfaces Físicas
config system interface
    edit "port2"
        set mode static
        set ip 100.65.1.111 255.255.255.0
        set allowaccess ping https ssh
        set comment "WAN 1 - ISP Principal"
    next
    edit "port3"
        set mode static
        set ip 100.66.1.111 255.255.255.0
        set allowaccess ping https ssh
        set comment "WAN 2 - ISP Secundario"
    next
    edit "port4"
        set mode static
        set ip 172.20.1.254 255.255.255.0
        set allowaccess ping
        set comment "Red LAN Branch"
    next
end

# 2. Rutas Estáticas con prioridades (Dual-WAN)
config router static
    edit 1
        set dst 0.0.0.0 0.0.0.0
        set gateway 100.65.1.254
        set device "port2"
        set priority 10
    next
    edit 2
        set dst 0.0.0.0 0.0.0.0
        set gateway 100.66.1.254
        set device "port3"
        set priority 20
    next
end

# 3. Configuración VPN IPsec (Fase 1 y Fase 2 conectando a HQ WAN1)
config vpn ipsec phase1-interface
    edit "To_HQ"
        set interface "port2"
        set ike-version 2
        set peertype any
        set net-device enable
        set proposal aes256-sha256 aes128-sha1
        set remote-gw 100.65.0.101
        set psksecret FortiTopo2026*
    next
end

config vpn ipsec phase2-interface
    edit "To_HQ_p2"
        set phase1name "To_HQ"
        set proposal aes256-sha256 aes128-sha1
        set src-subnet 172.20.1.0 255.255.255.0
        set dst-subnet 10.0.11.0 255.255.255.0
    next
end

# 4. Políticas de Firewall (Tráfico bidireccional sin NAT)
config firewall policy
    edit 10
        set name "Branch_LAN_to_HQ_VPN"
        set srcintf "port4"
        set dstintf "To_HQ"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat disable
    next
    edit 11
        set name "HQ_VPN_to_Branch_LAN"
        set srcintf "To_HQ"
        set dstintf "port4"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat disable
    next
end

# 5. Ruta estática hacia la subred de HQ a través del túnel
config router static
    edit 10
        set dst 10.0.11.0 255.255.255.0
        set device "To_HQ"
    next
end
```

---

## FASE 4: Verificación y Pruebas de Funcionamiento

1. **Estado del Túnel IPsec:**
   Ejecuta el siguiente comando en cualquiera de los dos FortiGates para corroborar que la asociación de seguridad (SA) está arriba:
   ```bash
   get vpn ipsec tunnel summary
   ```

2. **Prueba de Conectividad End-to-End:**
   Desde el equipo **HQ-PC-1** (`10.0.11.50`) o directamente desde la consola CLI de **BR1-FGT** especificando su IP de origen local, realiza un ping hacia la red remota:
   ```bash
   execute ping-options source 172.20.1.254
   execute ping 10.0.11.50
   ```
   *El éxito de los paquetes devueltos confirmará que el túnel IKEv2 y las políticas de seguridad en FortiOS 7.6 operan de forma impecable.*
