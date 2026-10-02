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

## Running-configs

<details>
<summary><b>ISP</b></summary>

```
ISP#show running-config
Building configuration...

Current configuration : 1378 bytes
!
version 15.4
service timestamps debug datetime msec
service timestamps log datetime msec
no service password-encryption
!
hostname ISP
!
boot-start-marker
boot-end-marker
!
no aaa new-model
mmi polling-interval 60
no mmi auto-configure
no mmi pvc
mmi snmp-timeout 180
!
no ip domain lookup
ip cef
no ipv6 cef
!
multilink bundle-name authenticated
!
redundancy
!
interface Loopback0
 description Simula Internet
 ip address 8.8.8.8 255.255.255.255
!
interface Ethernet0/0
 description Enlace hacia FG-USER port1
 ip address 20.24.23.1 255.255.255.252
!
interface Ethernet0/1
 description Enlace hacia FG-Server port1
 ip address 20.24.56.1 255.255.255.252
!
interface Ethernet0/2
 no ip address
 shutdown
!
interface Ethernet0/3
 no ip address
 shutdown
!
interface Ethernet1/0
 no ip address
 shutdown
!
interface Ethernet1/1
 no ip address
 shutdown
!
interface Ethernet1/2
 no ip address
 shutdown
!
interface Ethernet1/3
 no ip address
 shutdown
!
ip forward-protocol nd
!
no ip http server
no ip http secure-server
!
control-plane
!
banner motd ^CISP - Laboratorio VPN Site-to-Site - Matricula 2024-2356^C
!
line con 0
 exec-timeout 0 0
 logging synchronous
line aux 0
line vty 0 4
 login
 transport input none
!
end
```

</details>

<details>
<summary><b>SW-Usuarios</b></summary>

```
SW-Usuarios#show running-config
Building configuration...

Current configuration : 1149 bytes
!
version 15.2
service timestamps debug datetime msec
service timestamps log datetime msec
no service password-encryption
service compress-config
!
hostname SW-Usuarios
!
boot-start-marker
boot-end-marker
!
no aaa new-model
!
no ip domain-lookup
ip cef
no ipv6 cef
!
spanning-tree mode rapid-pvst
spanning-tree extend system-id
!
vlan internal allocation policy ascending
!
interface Ethernet0/0
 description Trunk hacia FG-USER port2
 switchport trunk allowed vlan 10
 switchport trunk encapsulation dot1q
 switchport mode trunk
!
interface Ethernet0/1
 description Access hacia PC-Usuario
 switchport access vlan 10
 switchport mode access
 spanning-tree portfast edge
!
interface Ethernet0/2
 description No usado
 shutdown
!
interface Ethernet0/3
 description No usado
 shutdown
!
ip forward-protocol nd
!
no ip http server
no ip http secure-server
!
control-plane
!
banner motd ^CSW-Usuarios - Laboratorio VPN - Matricula 2024-2356^C
!
line con 0
 exec-timeout 0 0
 logging synchronous
line aux 0
line vty 0 4
 login
!
end
```

</details>

<details>
<summary><b>FG-USER</b></summary>

```
PEGAR AQUÍ LA SALIDA DE "show" DEL FG-USER
```

</details>

<details>
<summary><b>FG-Server</b></summary>

```
PEGAR AQUÍ LA SALIDA DE "show" DEL FG-SERVER
```

</details>

- `documentacion/` — informe completo en PDF
