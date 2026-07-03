# 🔐 FortiGate VPN Client-to-Site — IPSec Dial-up

<div align="center">

![FortiGate](https://img.shields.io/badge/FortiGate-FortiOS-red?style=for-the-badge&logo=fortinet)
![Linux](https://img.shields.io/badge/Cliente-Linux-yellow?style=for-the-badge&logo=linux)
![IPSec](https://img.shields.io/badge/IPSec-IKEv1--Aggressive-green?style=for-the-badge)
![XAUTH](https://img.shields.io/badge/Auth-PSK%20%2B%20XAUTH-purple?style=for-the-badge)
![Platform](https://img.shields.io/badge/Platform-PNETLab-orange?style=for-the-badge)
![License](https://img.shields.io/badge/Uso-Educativo-gray?style=for-the-badge)

**Sael Germán García** | Matrícula: `2025-0725`  
Asignatura: Seguridad de Redes | Profesor: Jonathan Rondón  
Instituto Tecnológico de las Américas — ITLA | 2026

</div>

---

## 📋 Descripción

Configuración y verificación de una **VPN Client-to-Site usando FortiGate como servidor VPN y un cliente Linux con StrongSwan**. La práctica utiliza **IPSec Dial-up** para que un usuario remoto pueda autenticarse, recibir una dirección IP del pool VPN y alcanzar la red LAN interna ubicada detrás del FortiGate.

El cliente se autentica con **PSK + XAUTH** (usuario/contraseña), el FortiGate asigna la IP `10.7.25.193` vía **Mode Config** y el cliente accede a la LAN interna `10.7.25.64/27`. La conectividad se valida desde ambos extremos.

> 💡 **Claves técnicas:**
> - **IKEv1 Aggressive Mode** — el cliente no tiene IP fija en el servidor; el túnel se inicia dinámicamente (`type = dynamic` / `Remote Gateway = Dialup User`).
> - **XAUTH** — agrega una capa de autenticación usuario/contraseña sobre el PSK, usando el grupo `IPSEC_USERS`.
> - **Mode Config** — el FortiGate asigna automáticamente una IP del pool `10.7.25.192/28` al cliente (`leftsourceip=%config` en StrongSwan).
> - **PFS desactivado en Phase 2** — requerido para compatibilidad con el modo aggressive de IKEv1.

> ⚠️ **Nota sobre traceroute:** Pueden aparecer asteriscos en el primer salto o el mensaje "Destination port unreachable" al llegar al destino. Esto no indica falla en la VPN. El ping exitoso y la llegada al destino final confirman conectividad válida.

---

## 🗺️ Topología de Red

Un cliente Linux remoto se conecta al FortiGate FGT1 a través de un ISP simulado. El FortiGate asigna una IP del pool VPN al cliente y le da acceso a la LAN interna donde está la VPC-LAN.

### 📊 Direccionamiento IP

| Dispositivo | Interfaz | IP / Máscara | Función |
|:-----------:|:--------:|:-------------:|---------|
| Cliente-Linux | ens4 | 10.7.25.5/30 | Cliente remoto hacia ISP |
| ISP | Ethernet0/0 | 10.7.25.6/30 | Enlace hacia Cliente-Linux |
| ISP | Ethernet0/1 | 10.7.25.2/30 | Enlace hacia FGT1 port3 |
| FGT1 | port3 | 10.7.25.1/30 | WAN hacia ISP |
| FGT1 | port2 | 10.7.25.65/27 | Gateway LAN interna |
| FGT1 | port1 | 192.168.226.210/24 | Gestión HTTP |
| VPC-LAN | eth0 | 10.7.25.66/27 | Host LAN interna |
| **Cliente VPN** | **virtual** | **10.7.25.193/32** | **IP asignada por Mode Config** |
| **Pool VPN** | IPSEC_CLIENT_POOL | **10.7.25.193 – 10.7.25.206** | **Direcciones para clientes VPN** |

---

## ⚙️ Parámetros de la VPN

| Parámetro | Valor |
|:---------:|-------|
| Tipo de VPN | IPSec Dial-up Client-to-Site |
| Servidor VPN | FortiGate FGT1 |
| Cliente VPN | Linux con StrongSwan |
| Interfaz WAN FortiGate | port3 — 10.7.25.1/30 |
| IKE | IKEv1 modo aggressive |
| Autenticación | PSK + XAUTH |
| Clave precompartida | VPN12345 |
| Usuario XAUTH | vpnuser / vpnpass |
| Grupo de usuarios | IPSEC_USERS |
| Phase 1 | DES-MD5 / DES-SHA1, DH Group 5 |
| Phase 2 | DES-SHA1, PFS desactivado |
| Nombre del túnel | IPSEC_CL |
| Mode Config | Habilitado (asignación automática de IP) |
| Red local protegida | 10.7.25.64/27 |
| Pool VPN | 10.7.25.192/28 |
| NAT | Desactivado en políticas VPN |

---

## 🔍 Funcionamiento

El proceso de conexión sigue estas etapas:

**1. Conectividad inicial** — Cliente-Linux configura `ens4` con `10.7.25.5/30` y una ruta host hacia el FortiGate `10.7.25.1` vía ISP `10.7.25.6`.

**2. Negociación IKEv1 Aggressive** — StrongSwan inicia la negociación IKEv1 en modo aggressive usando PSK `VPN12345`. El FortiGate acepta cualquier peer dinámico (`type = dynamic`).

**3. Autenticación XAUTH** — El FortiGate solicita usuario y contraseña al cliente. StrongSwan envía `vpnuser/vpnpass`, que el FortiGate valida contra el grupo `IPSEC_USERS`.

**4. Mode Config / Asignación de IP** — El FortiGate asigna al cliente la IP `10.7.25.193` del pool `IPSEC_CLIENT_POOL`. El cliente aplica esta IP como fuente virtual (`leftsourceip=%config`).

**5. Phase 2 / Túnel IPSec** — Se establece la SA de IPSec con selectores `LAN (10.7.25.64/27) ↔ Pool (10.7.25.192/28)`. El cliente puede alcanzar la VPC-LAN y la VPC-LAN puede alcanzar al cliente.

---

## 🚀 Scripts de Configuración

### ISP
```cisco
hostname ISP
ip cef
interface Ethernet0/0
 description HACIA_CLIENTE_LINUX
 ip address 10.7.25.6 255.255.255.252
 no shutdown
interface Ethernet0/1
 description HACIA_FGT1_PORT3
 ip address 10.7.25.2 255.255.255.252
 no shutdown
```

### FortiGate FGT1 — Interfaces y Rutas (FortiOS CLI)
```fortios
config system hostname
    set hostname FGT1
end

config system interface
    edit "port1"
        set alias "MGMT_NET"
        set mode static
        set ip 192.168.226.210 255.255.255.0
        set allowaccess ping http https ssh
        set role wan
    next
    edit "port2"
        set alias "LAN_INTERNA"
        set mode static
        set ip 10.7.25.65 255.255.255.224
        set allowaccess ping http https ssh
        set role lan
    next
    edit "port3"
        set alias "WAN_HACIA_ISP"
        set mode static
        set ip 10.7.25.1 255.255.255.252
        set allowaccess ping
        set role wan
    next
end

config router static
    edit 1
        set dst 0.0.0.0 0.0.0.0
        set gateway 10.7.25.2
        set device "port3"
        set distance 5
    next
    edit 2
        set dst 10.7.25.5 255.255.255.255
        set gateway 10.7.25.2
        set device "port3"
    next
end
```

### FortiGate FGT1 — Usuario, Grupo y VPN (FortiOS CLI)
```fortios
config firewall address
    edit "LAN_10.7.25.64_27"
        set subnet 10.7.25.64 255.255.255.224
    next
    edit "IPSEC_CLIENT_POOL"
        set type iprange
        set start-ip 10.7.25.193
        set end-ip 10.7.25.206
    next
end

config user local
    edit "vpnuser"
        set type password
        set passwd vpnpass
    next
end

config user group
    edit "IPSEC_USERS"
        set member "vpnuser"
    next
end

config vpn ipsec phase1-interface
    edit "IPSEC_CL"
        set type dynamic
        set interface "port3"
        set ike-version 1
        set mode aggressive
        set peertype any
        set net-device disable
        set mode-cfg enable
        set dpd on-idle
        set dhgrp 5
        set xauthtype auto
        set authusrgrp "IPSEC_USERS"
        set ipv4-start-ip 10.7.25.193
        set ipv4-end-ip 10.7.25.206
        set ipv4-netmask 255.255.255.240
        set psksecret VPN12345
    next
end

config vpn ipsec phase2-interface
    edit "IPSEC_CL_P2"
        set phase1name "IPSEC_CL"
        set proposal des-sha1
        set pfs disable
        set src-subnet 10.7.25.64 255.255.255.224
        set dst-subnet 10.7.25.192 255.255.255.240
    next
end

config router static
    edit 30
        set dst 10.7.25.192 255.255.255.240
        set device "IPSEC_CL"
    next
end

config firewall policy
    edit 30
        set name "IPSEC_CLIENT_to_LAN"
        set srcintf "IPSEC_CL"
        set dstintf "port2"
        set srcaddr "IPSEC_CLIENT_POOL"
        set dstaddr "LAN_10.7.25.64_27"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat disable
    next
    edit 40
        set name "LAN_to_IPSEC_CLIENT"
        set srcintf "port2"
        set dstintf "IPSEC_CL"
        set srcaddr "LAN_10.7.25.64_27"
        set dstaddr "IPSEC_CLIENT_POOL"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat disable
    next
end
```

### Cliente Linux — Red y StrongSwan
```bash
# Configuración de red
sudo ip link set ens4 up
sudo ip addr flush dev ens4
sudo ip addr add 10.7.25.5/30 dev ens4
sudo ip route replace 10.7.25.1/32 via 10.7.25.6 dev ens4

# Instalación
sudo apt update
sudo apt install strongswan traceroute -y
```

### `/etc/ipsec.conf`
```
config setup
    uniqueids=no

conn FGT-IPSEC-CLIENT
    keyexchange=ikev1
    aggressive=yes
    authby=secret
    left=10.7.25.5
    leftsourceip=%config
    leftauth=psk
    leftauth2=xauth
    xauth_identity=vpnuser
    right=10.7.25.1
    rightauth=psk
    rightsubnet=10.7.25.64/27
    ike=des-sha1-modp1536!
    esp=des-sha1!
    auto=add
```

### `/etc/ipsec.secrets`
```
: PSK "VPN12345"
vpnuser : XAUTH "vpnpass"
```

### Comandos para levantar la VPN
```bash
sudo ipsec restart
sudo ipsec up FGT-IPSEC-CLIENT
ip route
ping -c 4 10.7.25.65
ping -c 4 10.7.25.66
traceroute 10.7.25.66
sudo ipsec statusall
```

### Verificación en FortiGate
```fortios
get vpn ipsec tunnel summary
diagnose vpn ike gateway list
diagnose vpn tunnel list name IPSEC_CL
```

### VPC-LAN — Prueba de retorno
```bash
ping 10.7.25.193
trace 10.7.25.193
```

---

## ✅ Verificación

| Acción / Comando | Estado esperado |
|:----------------:|-----------------|
| ISP interfaces | Ethernet0/0 y 0/1 up/up |
| GUI → Network → Interfaces (FGT1) | port2, port3 activas |
| GUI → Network → Static Routes | Default, host cliente y ruta pool VPN |
| GUI → User & Authentication → Users | `vpnuser` habilitado |
| GUI → User & Authentication → Groups | `IPSEC_USERS` con `vpnuser` |
| GUI → VPN → IPSec Tunnels | `IPSEC_CL` configurado (Dialup) |
| GUI → Monitor → IPSec Monitor | Túnel `IPSEC_CL_0` activo con tráfico |
| `sudo ipsec up FGT-IPSEC-CLIENT` | Conexión establecida |
| `ip route` en Linux | Ruta hacia `10.7.25.64/27` por interfaz VPN |
| `ping 10.7.25.65` / `ping 10.7.25.66` | ✅ Exitoso |
| `sudo ipsec statusall` | Túnel INSTALLED, `10.7.25.64/27` instalado |
| `diagnose vpn ike gateway list` | `xauth-user: vpnuser`, IP `10.7.25.193` asignada |
| `diagnose vpn tunnel list name IPSEC_CL` | Tráfico recibido y enviado |
| `ping 10.7.25.193` desde VPC-LAN | ✅ Exitoso (retorno LAN→cliente) |

---

## 📊 Resultados

| Prueba | Resultado |
|:------:|:---------:|
| Interfaces ISP | ✅ up/up |
| IP VPC-LAN (`10.7.25.66/27`) | ✅ Configurada |
| Interfaces FGT1 (port2, port3) | ✅ Activas en GUI |
| Rutas estáticas en FGT1 | ✅ Default + host cliente + pool VPN |
| Usuario `vpnuser` y grupo `IPSEC_USERS` | ✅ Creados y habilitados |
| Túnel IPSec `IPSEC_CL` (Dialup, IKEv1 aggressive) | ✅ Configurado |
| Políticas firewall (cliente↔LAN, NAT off) | ✅ Configuradas |
| Monitor IPSec FGT1 | ✅ `IPSEC_CL_0` activo con tráfico |
| StrongSwan conectado | ✅ IP `10.7.25.193` asignada |
| Diagnóstico IKE gateway FGT1 | ✅ `vpnuser` autenticado, IP asignada |
| Diagnóstico túnel FGT1 | ✅ Tráfico recibido y enviado |
| Ping Linux → LAN interna | ✅ Exitoso |
| Traceroute Linux → VPC-LAN | ✅ Llega al destino |
| Ping VPC-LAN → cliente VPN (`10.7.25.193`) | ✅ Exitoso (bidireccional) |

---

## 📁 Archivos del Repositorio

| Archivo | Descripción |
|:-------:|-------------|
| [`SaelGerman_2025-0725_Script_FortiGate_Client_to_Site_IPSec_Dialup.txt`](SaelGerman_2025-0725_Script_FortiGate_Client_to_Site_IPSec_Dialup.txt) | Scripts de configuración FortiOS CLI, ISP y Linux |
| [`SaelGerman_2025-0725_FortiGate-Client-to-Site_P2.pdf`](SaelGerman_2025-0725_FortiGate-Client-to-Site_P2.pdf) | Documentación técnica completa |

---

## 🖼️ Capturas de Pantalla

- 📸 [Figura 1 — Topología FortiGate Client-to-Site IPSec Dial-up](SaelGerman_2025-0725_Capturas_FortiGate_Client_to_Site_IPSec_Dialup_GitHub/01_topologia_fortigate_client_to_site_ipsec_dialup.png)
- 📸 [Figura 2 — Interfaces del ISP (up/up)](SaelGerman_2025-0725_Capturas_FortiGate_Client_to_Site_IPSec_Dialup_GitHub/02_isp_show_ip_interface_brief.png)
- 📸 [Figura 3 — IP de la VPC-LAN](SaelGerman_2025-0725_Capturas_FortiGate_Client_to_Site_IPSec_Dialup_GitHub/03_vpc_lan_show_ip.png)
- 📸 [Figura 4 — Interfaces de FGT1 desde la GUI](SaelGerman_2025-0725_Capturas_FortiGate_Client_to_Site_IPSec_Dialup_GitHub/04_fgt1_interfaces_gui.png)
- 📸 [Figura 5 — Rutas estáticas del FortiGate](SaelGerman_2025-0725_Capturas_FortiGate_Client_to_Site_IPSec_Dialup_GitHub/05_fgt1_static_routes_gui.png)
- 📸 [Figura 6 — Usuario local vpnuser en FortiGate](SaelGerman_2025-0725_Capturas_FortiGate_Client_to_Site_IPSec_Dialup_GitHub/06_fgt1_user_definition_vpnuser.png)
- 📸 [Figura 7 — Grupo IPSEC_USERS en FortiGate](SaelGerman_2025-0725_Capturas_FortiGate_Client_to_Site_IPSec_Dialup_GitHub/07_fgt1_user_groups_ipsec_users.png)
- 📸 [Figura 8 — Configuración del túnel IPSec IPSEC_CL](SaelGerman_2025-0725_Capturas_FortiGate_Client_to_Site_IPSec_Dialup_GitHub/08_fgt1_ipsec_tunnel_ipsec_cl_config.png)
- 📸 [Figura 9 — Políticas firewall para tráfico VPN](SaelGerman_2025-0725_Capturas_FortiGate_Client_to_Site_IPSec_Dialup_GitHub/09_fgt1_firewall_policies.png)
- 📸 [Figura 10 — Monitor IPSec con túnel activo (IPSEC_CL_0)](SaelGerman_2025-0725_Capturas_FortiGate_Client_to_Site_IPSec_Dialup_GitHub/10_fgt1_ipsec_monitor_tunnel_up.png)
- 📸 [Figura 11 — Pruebas de ping y traceroute desde Linux hacia LAN interna](SaelGerman_2025-0725_Capturas_FortiGate_Client_to_Site_IPSec_Dialup_GitHub/11_linux_ping_traceroute_lan_interna.png)
- 📸 [Figura 12 — Estado de StrongSwan (sudo ipsec statusall)](SaelGerman_2025-0725_Capturas_FortiGate_Client_to_Site_IPSec_Dialup_GitHub/12_linux_sudo_ipsec_statusall.png)
- 📸 [Figura 13 — Diagnóstico IKE gateway en FortiGate (vpnuser, IP asignada)](SaelGerman_2025-0725_Capturas_FortiGate_Client_to_Site_IPSec_Dialup_GitHub/13_fgt1_diagnose_vpn_ike_gateway_list.png)
- 📸 [Figura 14 — Diagnóstico del túnel IPSEC_CL (tráfico activo)](SaelGerman_2025-0725_Capturas_FortiGate_Client_to_Site_IPSec_Dialup_GitHub/14_fgt1_diagnose_vpn_tunnel_list_ipsec_cl.png)
- 📸 [Figura 15 — Prueba de retorno VPC-LAN → cliente VPN (10.7.25.193)](SaelGerman_2025-0725_Capturas_FortiGate_Client_to_Site_IPSec_Dialup_GitHub/15_vpc_ping_trace_cliente_vpn_10_7_25_193.png)

---

## 📎 Recursos

📄 **Documentación Técnica:** [Ver Informe PDF](SaelGerman_2025-0725_FortiGate-Client-to-Site_P2.pdf)  
▶️ **Video Demostración:** [Ver en YouTube](https://youtu.be/5XwdIhZGxPQ)

---

## 📚 Referencias

1. Fortinet. *FortiGate IPSec Dial-up VPN Administration Guide*. Documentación oficial FortiOS.
2. StrongSwan Project. *IKEv1 with XAUTH and Mode Config*. Documentación oficial StrongSwan.
3. Reconocimiento especial: Troubleshooting, base del script y documentación apoyado en Inteligencia Artificial.

---

<div align="center">

*Este laboratorio fue desarrollado exclusivamente con fines académicos y educativos.*

</div>
