# GUÍA DE LABORATORIO: VPN IPSEC SITE-TO-SITE CON FORTIGATE (FortiOS 7.6)

---

### I. Capacidades
* Implementar y poner en marcha un túnel VPN IPsec Site-to-Site utilizando el asistente (*IPsec Wizard*) y validar parámetros sobre **FortiOS 7.6**.
* Analizar y auditar la creación automática de políticas de firewall y tablas de enrutamiento estático (incluyendo rutas de descarte o *Blackhole*).
* Diagnosticar y resolver problemas de negociación y conectividad entre redes LAN privadas mediante herramientas CLI y clientes finales.

---

### II. Seguridad
* En este laboratorio está prohibida la manipulación del hardware, conexiones eléctricas o de red. Así como la ingesta de alimentos y bebidas.
* Ubicar maletines y/o mochilas en el lugar destinado para tal fin.
* Dejar la mesa de trabajo y la silla utilizada limpias y ordenadas.
* Mantener la confidencialidad y robustez en la configuración de credenciales de acceso administrativo y claves precompartidas (PSK).

---

### III. Fundamento Teórico
Una red privada virtual (**VPN**) sobre **IPsec** permite interconectar sedes remotas a través de una red no confiable (Internet), garantizando:
1. **Confidencialidad:** Cifrado simétrico de datos (AES-CBC o AES-GCM).
2. **Integridad y Autenticación de Origen:** Verificación de paquetes mediante HMAC (SHA-256) o algoritmos AEAD.
3. **Antirrepetición:** Mediante numeración secuencial de paquetes.

En **FortiOS 7.6**, las negociaciones se basan primordialmente en **IKEv2** (RFC 7296), optimizando el intercambio de mensajes:
* **Fase 1 (IKE SA):** Negocia un canal seguro entre los dos FortiGate autenticándose mediante una clave precompartida (*Pre-Shared Key* o PSK).
* **Fase 2 (Child SA / IPsec SA):** Negocia los túneles unidireccionales de datos sobre los que transita el tráfico de las redes LAN locales y remotas (*Proxy-IDs* / Selectores de fase 2).

Adicionalmente, el sistema implementa rutas **Blackhole** con una distancia administrativa mayor a la ruta del túnel. Si el túnel cae, el tráfico con destino a la red corporativa remota se descarta silenciosamente en el firewall, evitando que se encamine sin cifrar a través de la ruta por defecto de Internet (*routing leak*).

---

### IV. Normas Empleadas

#### Orientación de Seguridad
*Tener especial cuidado al momento de hacer la instalación física de los equipos, así como el reconocimiento de partes. Evite cualquier contacto directo con el circuito de la fuente de energía y sus cables cuando esta se encuentra energizada, verifique que las conexiones de tierra y los aislamientos estén en buen estado.*

* **Estándar Criptográfico:** NIST SP 800-77 (Directrices para VPN IPsec).
* **Protocolo de Negociación:** IKEv2 (RFC 7296) y encapsulamiento ESP (RFC 4303).

---

### V. Recursos
* PC/Workstation con procesador con soporte de virtualización (Intel VT-x / AMD-V) y mínimo 8 GB de RAM.
* Software de virtualización: **VMware Workstation Pro 16/17** o VMware Player.
* 2x Appliances virtuales **FortiGate-VM** con imagen **FortiOS 7.6**.
* 2x Máquinas virtuales cliente (Windows 10/11 o Linux Alpine/TinyCore) para validación de extremos LAN.
* Conmutadores virtuales:
  * **VMnet8 (Modo NAT):** Simula la nube de Internet (WAN).
  * **VMnet2 (Host-only):** LAN Sede 1.
  * **VMnet3 (Host-only):** LAN Sede 2.
* Navegador Web (Chrome/Firefox) y cliente SSH/Terminal.

---

### VI. Metodología para el Desarrollo de la Tarea
* El desarrollo del laboratorio es individual.
* Cada estudiante configurará la topología respetando la nomenclatura y direccionamiento asignado.
* Se registrarán evidencias de cada etapa (capturas legibles que incluyan fecha/hora y nombre del dispositivo).

---

### VII. Procedimiento

```
                    [ INTERNET / WAN (VMnet8) ]
                     Red: 192.168.81.0/24
                                |
               +----------------+----------------+
               |                                 |
   port1: 192.168.81.10/24           port1: 192.168.81.20/24
        +--------------+                  +--------------+
        |  FGT-Site1   |                  |  FGT-Site2   |
        +--------------+                  +--------------+
   port2: 172.16.1.1/24              port2: 172.18.1.1/24
               |                                 |
        [ LAN Sede 1 ]                    [ LAN Sede 2 ]
       (VMnet2 - Host-Only)              (VMnet3 - Host-Only)
               |                                 |
       Cliente PC1:                      Cliente PC2:
       IP: 172.16.1.10/24                IP: 172.18.1.10/24
       GW: 172.16.1.1                    GW: 172.18.1.1
```

#### Tabla de Direccionamiento
| Dispositivo | Interfaz | Dirección IP / Máscara | Gateway | Rol |
| :--- | :--- | :--- | :--- | :--- |
| **FGT-Site1** | `port1` | `192.168.81.10/24` | `192.168.81.2` | WAN (Simulada Internet) |
| **FGT-Site1** | `port2` | `172.16.1.1/24` | N/A | LAN Local Sede 1 |
| **FGT-Site2** | `port1` | `192.168.81.20/24` | `192.168.81.2` | WAN (Simulada Internet) |
| **FGT-Site2** | `port2` | `172.18.1.1/24` | N/A | LAN Local Sede 2 |
| **PC1 (Sede 1)**| `eth0` | `172.16.1.10/24` | `172.16.1.1` | Host de prueba LAN 1 |
| **PC2 (Sede 2)**| `eth0` | `172.18.1.10/24` | `172.18.1.1` | Host de prueba LAN 2 |

---

#### Paso 1: Configuración Inicial de Red y Enrutamiento Base

Realice estas configuraciones iniciales en ambos FortiGate:

1. **En FGT-Site1:**
   * Cambie el Hostname a `FGT-Site1` en **System > Settings**.
   * Configure interfaces en **Network > Interfaces**:
     * `port1` (WAN): Alias `WAN`, IP `192.168.81.10/24`, Ping habilitado.
     * `port2` (LAN): Alias `LAN`, IP `172.16.1.1/24`, Ping habilitado.
   * Cree la ruta por defecto en **Network > Static Routes > Create New**:
     * **Destination:** `0.0.0.0/0.0.0.0`
     * **Gateway IP:** `192.168.81.2`
     * **Interface:** `port1` (WAN)

2. **En FGT-Site2:**
   * Cambie el Hostname a `FGT-Site2` en **System > Settings**.
   * Configure interfaces en **Network > Interfaces**:
     * `port1` (WAN): Alias `WAN`, IP `192.168.81.20/24`, Ping habilitado.
     * `port2` (LAN): Alias `LAN`, IP `172.18.1.1/24`, Ping habilitado.
   * Cree la ruta por defecto en **Network > Static Routes > Create New**:
     * **Destination:** `0.0.0.0/0.0.0.0`
     * **Gateway IP:** `192.168.81.2`
     * **Interface:** `port1` (WAN)

---

#### Paso 2: Creación del Túnel IPsec en FGT-Site1 (IPsec Wizard)

1. En **FGT-Site1**, diríjase a **VPN > IPsec Wizard**.
2. **Paso 1 - VPN Setup:**
   * **Name:** `To_Site2`
   * **Template type:** `Site to Site`
   * **NAT configuration:** `No NAT between sites`
   * **Remote device type:** `FortiGate`
   * Haga clic en **Next**.

3. **Paso 2 - Authentication:**
   * **Remote device:** Seleccione `IP Address`.
   * **Remote IP address:** `192.168.81.20` *(IP WAN de Site 2)*.
   * **Outgoing Interface:** `port1` *(WAN)*.
   * **Authentication method:** `Pre-shared Key`.
   * **Pre-shared Key:** `Tecsup2024*` *(Importante: Debe ser exactamente la misma en ambos extremos)*.
   * Haga clic en **Next**.

4. **Paso 3 - Policy & Routing:**
   * **Local interface:** `port2` *(LAN)*.
   * **Local subnets:** `172.16.1.0/24`.
   * **Remote subnets:** `172.18.1.0/24`.
   * **Internet Access:** `None`.
   * Haga clic en **Create**.

---

#### Paso 3: Creación del Túnel IPsec en FGT-Site2 (IPsec Wizard)

1. En **FGT-Site2**, diríjase a **VPN > IPsec Wizard**.
2. **Paso 1 - VPN Setup:**
   * **Name:** `To_Site1`
   * **Template type:** `Site to Site`
   * **NAT configuration:** `No NAT between sites`
   * **Remote device type:** `FortiGate`
   * Haga clic en **Next**.

3. **Paso 2 - Authentication:**
   * **Remote device:** Seleccione `IP Address`.
   * **Remote IP address:** `192.168.81.10` *(IP WAN de Site 1)*.
   * **Outgoing Interface:** `port1` *(WAN)*.
   * **Authentication method:** `Pre-shared Key`.
   * **Pre-shared Key:** `Tecsup2024*` *(Misma clave usada en Site 1)*.
   * Haga clic en **Next**.

4. **Paso 3 - Policy & Routing:**
   * **Local interface:** `port2` *(LAN)*.
   * **Local subnets:** `172.18.1.0/24`.
   * **Remote subnets:** `172.16.1.0/24`.
   * **Internet Access:** `None`.
   * Haga clic en **Create**.

---

#### Paso 4: Verificación de Objetos Creados Automáticamente

Compruebe en ambos equipos que el asistente configuró los siguientes elementos:

1. **Políticas de Firewall (**`Policy & Objects > Firewall Policy`**):**
   * Regla de salida: Tráfico desde `port2` (LAN local) hacia la interfaz virtual IPsec (`To_SiteX`) sin NAT.
   * Regla de entrada: Tráfico desde la interfaz virtual IPsec (`To_SiteX`) hacia `port2` (LAN local) sin NAT.
2. **Rutas Estáticas (**`Network > Static Routes`**):**
   * Una ruta estática hacia la red remota usando como interfaz de salida el túnel IPsec (`Distance: 10`).
   * Una ruta **Blackhole** hacia la red remota con mayor distancia (`Distance: 254`) para evitar fugas de tráfico no cifrado.

---

#### Paso 5: Levantamiento y Monitoreo del Túnel en FortiOS 7.6

1. Ingrese a **VPN > IPsec Tunnels** en **FGT-Site1**.
2. Inicialmente el estado puede figurar en reposo (flecha roja o inactivo).
3. Seleccione el túnel `To_Site2`, haga clic derecho y elija **Bring Up > All Phase 2 Selectors** (o haga doble clic sobre el estado para forzar la activación).
4. El indicador de **Status** pasará a color verde, confirmando que la Fase 1 (IKE SA) y la Fase 2 (IPsec SA) se negociaron con éxito.

---

#### Paso 6: Validación de Conectividad y Pruebas ICMP

1. **Prueba desde CLI de FortiGate (Demostración de Origen de Paquetes):**
   * Abra la consola CLI de **FGT-Site2** y ejecute:
     ```bash
     execute ping 172.16.1.1
     ```
     *Resultado esperado:* **Falla (100% packet loss)**.
     *Motivo:* FortiOS utiliza por defecto la IP de la WAN (`192.168.81.20`) como origen del ping, y dicha IP no está autorizada en los selectores de fase 2 del túnel.

2. **Prueba corrigiendo la IP Origen:**
   * En la misma consola CLI de **FGT-Site2**, establezca la IP LAN como origen:
     ```bash
     execute ping-options source 172.18.1.1
     execute ping 172.16.1.1
     ```
     *Resultado esperado:* **Éxito (0% packet loss)**. Los paquetes son encapsulados y cifrados por el túnel IPsec.

3. **Prueba Extremo a Extremo entre Clientes LAN:**
   * Desde **PC1** (`172.16.1.10`), ejecute un ping continuo hacia **PC2** (`172.18.1.10`):
     ```bash
     ping 172.18.1.10
     ```
   * La respuesta debe ser exitosa, validando que el tráfico entre ambas LANs atraviesa el túnel de manera transparente.

---

#### Paso 7: Comandos de Diagnóstico y Troubleshooting en FortiOS 7.6

Ejecute los siguientes comandos en la consola CLI de cualquiera de los FortiGate para auditar el túnel:

1. **Resumen rápido de túneles activos:**
   ```bash
   get vpn ipsec tunnel summary
   ```
2. **Listado detallado de SA y tráfico cursado (bytes in/out):**
   ```bash
   diagnose vpn tunnel list name To_Site1
   ```
3. **Depuración en tiempo real del protocolo IKE (útil ante fallas de PSK o Fase 1):**
   ```bash
   diagnose vpn ike log filter dst-addr4 192.168.81.10
   diagnose debug application ike -1
   diagnose debug enable
   ```
   *(Para deshabilitar la depuración ejecute: `diagnose debug disable`)*.

---

### Entregables del Laboratorio
Adjuntar en el informe final:
1. Captura de pantalla de **Network > Interfaces** de ambos FortiGate.
2. Captura de pantalla de **VPN > IPsec Tunnels** mostrando el túnel en estado UP (verde) en FortiOS 7.6.
3. Captura del comando `execute ping-options source` y el ping exitoso desde la consola CLI.
4. Captura del ping bidireccional exitoso entre las máquinas virtuales cliente (**PC1** y **PC2**).
