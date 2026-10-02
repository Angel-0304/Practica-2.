# Practica-2.
Infraestructura 1

# Infraestructura 1 — VPN Site-to-Site entre dos FortiGate

[Ver video demostrativo](https://itlaedudo-my.sharepoint.com/:f:/g/personal/20242356_itla_edu_do/IgDN_aKp_D9ATL6FPikZjtAoATCooH1UvsXBNc7rNldfzxk?e=jJUf2y)

## Objetivos

- Comunicar al usuario con el servidor a través del enlace VPN.
- Comprobar que la comunicación solo fluye si la VPN está activa.
- Configurar redes, NAT y VPN site-to-site en ambos FortiGate por GUI.
- Servidor web HTTPS en una red /28 y usuarios en una red /25 en la VLAN 10 con DHCP.

## Topología

![Topología lógica](diagramas/topologia-logica.png)

| Equipo | Plataforma | Función |
|---|---|---|
| ISP | Cisco IOL L2-ADVENTERPRISEK9 15.2 | IPs públicas; solo conoce sus redes conectadas |
| FG-USER | FortiGate VM64-KVM v6.4.0 | VLAN 10, DHCP, NAT y VPN |
| FG-Server | FortiGate VM64-KVM v6.4.0 | LAN /28, NAT y VPN |
| SW-Usuarios | Cisco IOL L2 | Trunk hacia el FortiGate y acceso en VLAN 10 |
| PC-Usuario | Docker pnetlab/ubuntu_sv | Pruebas de ping, traceroute y HTTPS |
| Web-Server | Ubuntu Server 20.04 | Apache con HTTPS (certificado autofirmado) |
| Cloud0 | Red de gestión de PNETLab | Acceso a la GUI de los FortiGate (port3) |

## Direccionamiento

| Puerto | FG-USER | FG-Server |
|---|---|---|
| port1 (WAN) | 20.24.23.2/30 | 20.24.56.2/30 |
| port2 (LAN) | VLAN10: 10.23.56.1/25 + DHCP | 10.23.56.129/28 |
| port3 (gestión) | 192.168.128.140 | 192.168.128.141 |

## VPN

Creada con **VPN → IPsec Wizard → Site to Site (FortiGate)** en ambos equipos, sin NAT entre sitios.

| Parámetro | FG-USER | FG-Server |
|---|---|---|
| Túnel | VPN-A-SERVER | VPN-A-USER |
| IP remota | 20.24.56.2 | 20.24.23.2 |
| Subred local | 10.23.56.0/25 | 10.23.56.128/28 |
| Subred remota | 10.23.56.128/28 | 10.23.56.0/25 |
| Propuesta negociada | des-md5 | des-md5 |

El asistente crea la interfaz del túnel, los grupos de direcciones, la ruta hacia la red remota, una ruta blackhole y dos políticas sin NAT.

## Verificación

**Con el túnel activo:**

```
root@PC-Usuario:/home# traceroute -n 10.23.56.130
 1  10.23.56.1
 2  20.24.56.2
 3  10.23.56.130
```

El ISP no aparece como salto porque el tráfico privado viaja encapsulado en ESP.

**Con el túnel deshabilitado** (Network → Interfaces → VPN-A-SERVER → Disabled):

```
ping 10.23.56.130          → Destination Net Unreachable
traceroute -n 10.23.56.130 → 1  10.23.56.1 !N
curl -k https://10.23.56.130 → Network is unreachable
```

La ruta blackhole descarta el tráfico en vez de dejarlo salir sin cifrar por la WAN.

## Problemas encontrados

| Problema | Causa | Solución |
|---|---|---|
| El FortiGate salía por la red de gestión | El DHCP de Cloud0 instaló una ruta por defecto con distancia 5 | IP fija en port3 del FG-USER y `set defaultgw disable` en el FG-Server |
| "VLAN ID used by another VLAN switch" | La VLAN 10 ya existía | Editar la interfaz existente |
| El PC no respondía ping al gateway | La VLAN10 no tenía PING en Administrative Access | Habilitar PING en la interfaz |
| Traceroute no instalado en el PC | El contenedor no lo trae | Instalarlo temporalmente por eth0 (red de Docker) |
| El túnel no subía después de la prueba | La interfaz seguía deshabilitada | Habilitarla (lo mostró `diagnose debug application ike -1`) |

## Archivos

- `running-configs/` — configuraciones de cada equipo
- `scripts/setup-webserver.sh` — instalación de Apache con HTTPS
- `documentacion/` — informe completo en PDF
