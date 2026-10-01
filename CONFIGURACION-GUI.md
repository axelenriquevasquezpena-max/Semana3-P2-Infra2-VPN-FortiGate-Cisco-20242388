# Configuración documentada de FortiGate

Equipo: FortiGate VM64-KVM, FortiOS 7.0.9. GUI de administración: 192.168.6.137. No se utilizan comandos FortiGate en esta documentación.

## Interfaces y DHCP

En **Network → Interfaces**:

| Interfaz | Configuración |
|---|---|
| port1 | DHCP, dirección observada 192.168.6.137/24; administración |
| port2 | Puerto padre sin dirección IPv4 |
| VLAN10-USUARIOS | VLAN 10 sobre port2, rol LAN, 10.23.88.1/25, PING |
| WAN-ISP (port3) | Manual 198.51.100.2/30, rol WAN, PING |

En VLAN10-USUARIOS: DHCP Enabled; rango 10.23.88.2–10.23.88.126; máscara 255.255.255.128; gateway Same as Interface IP; DNS Same as System DNS; lease 604800 segundos.

## Rutas

En **Network → Static Routes**:

| Destino | Salida | Gateway | Distancia |
|---|---|---|---|
| 0.0.0.0/0 | WAN-ISP (port3) | 198.51.100.1 | 1 |
| 10.23.88.128/28 | VPN-CISCO-WEB | Interfaz del túnel | 10 |

La lista de rutas configuradas puede mostrar 203.0.113.2 como gateway del túnel. En la tabla activa se verificó 10.23.88.128/28 por VPN-CISCO-WEB. La distancia 1 del default WAN da preferencia a esa salida frente al default de administración por DHCP.

## VPN Custom: VPN-CISCO-WEB

En **VPN → IPsec Tunnels**:

- Peer estático: 203.0.113.2; interfaz WAN-ISP (port3).
- Pre-shared key: idéntica a Cisco, omitida del repositorio.
- IKEv1, Main (ID protection); Local ID vacío; XAUTH Disabled.
- NAT Traversal Enable; DPD On Demand.
- Fase 1: una propuesta DES/SHA1; solo DH5; lifetime 86400 segundos.
- Fase 2: local 10.23.88.0/25; remoto 10.23.88.128/28.
- Fase 2: DES/SHA1; PFS Enabled, solo DH5; replay detection Enabled.
- Puertos y protocolo All; Auto-negotiate y Autokey Keep Alive activados.
- Lifetime de fase 2: 3600 segundos.

Confirmar cada sección con su marca de verificación y guardar el formulario con OK. El error inicial de hash se corrigió guardando SHA1 en fase 1; posteriormente se igualó la PSK para resolver el fallo de autenticación.

## Objetos

| Objeto | Subred / servicio |
|---|---|
| RED-USUARIOS | 10.23.88.0/25 |
| SERVIDOR-WEB | 10.23.88.130/32 |
| PC-ADMIN | 192.168.6.1/32 |
| CISCO-WAN | 203.0.113.2/32 |
| HTTPS | TCP destino 443 |
| SSH-2222 | TCP destino 2222 |

## Políticas

En **Policy & Objects → Firewall Policy**:

| Campo | USUARIOS-A-WEB-VPN | ADMIN-SSH-WEB |
|---|---|---|
| Entrada | VLAN10-USUARIOS | port1 |
| Salida | VPN-CISCO-WEB | WAN-ISP (port3) |
| Origen | RED-USUARIOS | PC-ADMIN |
| Destino | SERVIDOR-WEB | CISCO-WAN |
| Servicio | HTTPS y PING | SSH-2222 |
| Schedule / Action | always / ACCEPT | always / ACCEPT |
| NAT | Disabled | Enabled, Use Outgoing Interface Address |
| Inspección | Flow-based, no-inspection | Flow-based, no-inspection |
| Perfiles de seguridad | Desactivados | Desactivados |
| Registro | All Sessions | All Sessions |

La regla de administración está limitada al origen Windows. No existe una regla alternativa de usuarios hacia el servidor por WAN. No se añadió una regla para conexiones nuevas del servidor hacia Kali.

## Prueba por GUI

Para desactivar el enlace: **Network → Interfaces → expandir port3 → editar VPN-CISCO-WEB → Status Disabled → OK**. Para restaurarlo: Enabled → OK y esperar Up en IPsec Tunnels. No confundir PING en Administrative Access con el permiso de tránsito de la política.

Para observar SNAT: **Log & Report → Forward Traffic**, filtrar origen 192.168.6.1 y destino 203.0.113.2, puerto 2222. En Details se verificó NAT Translation Source y NAT IP 198.51.100.2.
