# Pruebas de aceptación

## Direccionamiento y conectividad local

Kali:

```bash
ip -br -4 addr
ip route
ping -c 4 10.23.88.1
```

Observado: 10.23.88.21/25, default por DHCP hacia 10.23.88.1 y ping 4/4.

WEB-P2:

```bash
ip -br -4 addr
ip route
```

Observado: 10.23.88.130/28, default 10.23.88.129.

## VPN activa

Mostrar Up en FortiGate mediante GUI. En Cisco:

```text
show crypto isakmp sa
show crypto ipsec sa
```

Se observó QM_IDLE, asociaciones ESP ACTIVE y tráfico cifrado/descifrado. Los contadores son acumulativos; comparar antes y después de generar tráfico.

En Kali:

```bash
ping -c 4 10.23.88.130
curl -k -I --connect-timeout 10 https://10.23.88.130/vpn.html
sudo traceroute -I -n -N 1 -q 1 -w 3 -m 8 10.23.88.130
```

Resultados: ping 4/4, HTTP 200 OK y traceroute que alcanza 10.23.88.130. La prueba con una sonda a la vez mostró 10.23.88.1, un salto sin respuesta y el destino. El número de salto varió en otras ejecuciones; los asteriscos no demuestran por sí solos pérdida de conectividad al destino.

## VPN desactivada y restaurada

Desactivar la interfaz VPN en la GUI del FortiGate, guardar y repetir ping/curl. Se observó 100 % de pérdida y error curl 28 por timeout. Reactivar, guardar y esperar Up. Se recuperó ping 4/4 y HTTP 200 OK. La evidencia final incluye ambos estados en la misma terminal.

## NAT en ambos peers

Flujo de administración independiente del usuario de VLAN 10:

```text
Windows 192.168.6.1 → FortiGate SNAT 198.51.100.2
→ Cisco 203.0.113.2:2222 → WEB-P2 10.23.88.130:22
```

La ruta de Windows usada en PowerShell con privilegios de administrador fue:

```powershell
route add 203.0.113.2 mask 255.255.255.255 192.168.6.137
ssh -p 2222 axel@203.0.113.2
```

Con la sesión abierta, ejecutar en Cisco:

```text
show ip nat translations
```

Se observó la traducción TCP 203.0.113.2:2222 a 10.23.88.130:22, con origen externo 198.51.100.2. Cerrar SSH con `exit` y revisar el registro reciente ADMIN-SSH-WEB en la GUI del FortiGate: IP original 192.168.6.1, NAT IP 198.51.100.2, tráfico en ambos sentidos. TCP reset from client describe el cierre de esa sesión, no una denegación de política.

## Incidencia durante la validación

Hubo una interrupción posterior a las pruebas del túnel: IKE/ESP activos pero contadores sin aumento y timeouts desde Kali. Se comprobaron rutas, objetos, servicios, gateway local y capturas; renovar ESP no recuperó el tráfico. Reiniciar FortiGate por GUI lo recuperó. Posteriormente se repitió satisfactoriamente la desactivación y reactivación sin otro reinicio. No se determinó la causa raíz; no se atribuye a una versión o defecto específico.
