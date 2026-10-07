# 1er Parcial – Seguridad de Redes

**Estudiante:** Sebastián  
**Matrícula:** 2168  
**Asignatura:** Seguridad de Redes  
**Video de la demostración:** [Ver video](PEGAR_AQUI_EL_ENLACE_DEL_VIDEO)

---

## 1. Propósito

Implementar una red segmentada de dos sitios (Sede Usuarios y Sede Servidores) protegida por dos FortiGate, aplicando:

- Microsegmentación del Web Server hacia el DB Server y hacia endpoints de actualización (FQDN).
- Acceso remoto seguro (solo SSH por VTY 0 4).
- VLANs con trunk 802.1Q y DHCP para usuarios.
- VPN IPsec site-to-site entre ambos FortiGate.
- Políticas de acceso HTTP de usuarios hacia el Web Server, con registro de denegados.
- Salida a Internet con NAT.

---

## 2. Topología

![Topología](imagenes/topologia.png)

### Diagrama lógico

```mermaid
flowchart LR
    subgraph S1["Sitio 1 - Sede Usuarios"]
        PC["PC-USUARIOS<br/>10.21.68.10/26<br/>(DHCP, VLAN 10)"]
        SW["SW-SITIO01<br/>Vlan10: 10.21.68.2/26"]
        FG1["FG1-USUARIOS<br/>port1: 21.68.1.2/24<br/>port2.10: 10.21.68.1/26<br/>port2.20: 10.21.68.65/26"]
        PC --- SW
        SW ---|"Trunk 802.1Q<br/>VLAN 10 / 20"| FG1
    end

    ISP(("ISP<br/>21.68.1.1 / 21.68.2.1"))

    subgraph S2["Sitio 2 - Sede Servidores (DMZ /28)"]
        FG2["FG2-SERVER<br/>port1: 21.68.2.2/24<br/>port2: 10.21.68.129/28"]
        WEB["Web Server<br/>10.21.68.130/28<br/>Apache :80"]
        DB["DB Server<br/>10.21.68.131/28<br/>MariaDB :3306"]
        FG2 --- WEB
        FG2 --- DB
    end

    FG1 --- ISP
    ISP --- FG2
    FG1 <-.->|"VPN IPsec"| FG2
    WEB -->|"TCP 3306"| DB
```

---

## 3. Direccionamiento (matrícula 2168)

### Tabla de subredes

| Segmento | Red | Máscara | Rango útil | Gateway | Propósito |
|---|---|---|---|---|---|
| VLAN 10 (Usuarios) | 10.21.68.0/26 | 255.255.255.192 | 10.21.68.1 – 10.21.68.62 | 10.21.68.1 | Clientes con DHCP |
| VLAN 20 (Administrativos) | 10.21.68.64/26 | 255.255.255.192 | 10.21.68.65 – 10.21.68.126 | 10.21.68.65 | Usuarios administrativos |
| DMZ / Servidores (/28) | 10.21.68.128/28 | 255.255.255.240 | 10.21.68.129 – 10.21.68.142 | 10.21.68.129 | Servidores Web y Base de Datos |
| WAN Sitio 1 (FG1 ↔ ISP) | 21.68.1.0/24 | 255.255.255.0 | 21.68.1.1 – 21.68.1.254 | 21.68.1.1 | Enlace WAN sede usuarios |
| WAN Sitio 2 (FG2 ↔ ISP) | 21.68.2.0/24 | 255.255.255.0 | 21.68.2.1 – 21.68.2.254 | 21.68.2.1 | Enlace WAN sede servidores |

### Sitio 1 (Sede Usuarios)

| Dispositivo | Interfaz | Dirección IP | Máscara | Gateway | Rol / Notas |
|---|---|---|---|---|---|
| FG1-USUARIOS | port1 (WAN) | 21.68.1.2 | /24 | 21.68.1.1 | Conexión al ISP / NAT |
| FG1-USUARIOS | port2.10 (VLAN 10) | 10.21.68.1 | /26 | N/A | Gateway usuarios (DHCP Server) |
| FG1-USUARIOS | port2.20 (VLAN 20) | 10.21.68.65 | /26 | N/A | Gateway administrativos |
| SW-SITIO01 | Vlan10 (SVI) | 10.21.68.2 | /26 | 10.21.68.1 | Gestión remota (SSH exclusivo) |
| PC-USUARIOS | eth0 | 10.21.68.10 | /26 | 10.21.68.1 | Asignada por DHCP |

### Sitio 2 (Sede Servidores / Data Center)

| Dispositivo | Interfaz | Dirección IP | Máscara | Gateway | Rol / Notas |
|---|---|---|---|---|---|
| FG2-SERVER | port1 (WAN) | 21.68.2.2 | /24 | 21.68.2.1 | Conexión al ISP / peer VPN |
| FG2-SERVER | port2 (Servidores) | 10.21.68.129 | /28 | N/A | Gateway de la red /28 |
| Web Server | eth0 | 10.21.68.130 | /28 | 10.21.68.129 | Servidor Apache (puerto 80) |
| DB Server | eth0 | 10.21.68.131 | /28 | 10.21.68.129 | Servidor MariaDB (puerto 3306) |

---

## 4. Estructura del repositorio

```
1ER-PARCIAL-SEGURIDAD-DE-REDES/
├── README.md
├── configs/
│   ├── FG1-USUARIOS.txt
│   ├── FG2-SERVER.txt
│   └── SW-SITIO01.txt
└── imagenes/
```

---

## 5. Implementación y evidencias

### Requisito 1 – Repositorio de GitHub

Este repositorio contiene el README con la documentación, los archivos de configuración de cada equipo en `configs/` y las evidencias en `imagenes/`.

---

### Requisito 2 – Microsegmentación del Web Server

El Web Server (10.21.68.130) solo puede iniciar conexiones hacia el DB Server (10.21.68.131) por el puerto **3306** y hacia los endpoints de actualización (objetos FQDN) por HTTP/HTTPS. No hay Internet abierto ni SSH ni ICMP hacia otros destinos.

**Política de microsegmentación:**

![Microsegmentación](imagenes/micropigmentacion%20requisito%206.png)

**Conexión permitida vía puerto 3306:**

![Conexión por 3306](imagenes/conecct%20via%20puerto%203306.png)

---

### Requisito 3 – Acceso remoto por VTY

Se configuraron únicamente las líneas **VTY 0 4**, con acceso solo por SSH y autenticación con usuario local.

![Requisito 3](imagenes/Requisito%203.png)

**Acceso remoto por VTY:**

![Acceso remoto por VTY](imagenes/requisito%20%233%20acceso%20remoto%20por%20vty.png)

**Solo VTY 0 4 con SSH únicamente:**

![VTY 0 4 solo SSH](imagenes/acceso%20remoto%20por%20vty%20vty%2004%20only%20ssh.png)

**SSH completamente funcional:**

![SSH funcional](imagenes/ssh%20completamente%20funcional.png)

---

### Requisito 4 – Direccionamiento y VLANs

Direccionamiento basado en la matrícula **2168** (ver sección 3):

- VLAN 10 (usuarios, con DHCP): `10.21.68.0/26`
- VLAN 20 (administrativos): `10.21.68.64/26`
- Servidores en red `/28`: `10.21.68.128/28`
- Trunk 802.1Q entre SW-SITIO01 y FG1-USUARIOS
- Hostname configurado en todos los equipos

Las configuraciones completas están en la carpeta [`configs/`](configs/).

---

### Requisito 5 – VPN entre dos FortiGate

VPN IPsec entre FG1-USUARIOS (21.68.1.2) y FG2-SERVER (21.68.2.2). El PC de usuarios llega al Web Server a través de la VPN y la comunicación fluye solo si la VPN está activa.

**Túnel VPN activo:**

![VPN activa](imagenes/acceso%20vpn%20.png)

**Traceroute al servidor:**

![Traceroute](imagenes/tracereoute.png)

**Pérdida de comunicación al bajar la VPN:**

![VPN caída](imagenes/requisito%20%235%20vpn%20caida.png)

![VPN caída no funciona](imagenes/vpn%20caido%20no%20funciona.png)

---

### Requisito 6 – Acceso Usuarios → Web por HTTP

Política que permite a la VLAN 10 acceder al Web Server por HTTP (80). Cualquier otro servicio hacia el Web desde los usuarios queda denegado con registro (log en *Forward Traffic*).

**Acceso permitido:**

![Acceso HTTP permitido](imagenes/%23requisisto%20%236.png)

**Intento bloqueado de otro servicio:**

![Otro servicio denegado](imagenes/requisito%20%236%20denegando%20cualquier%20otro%20servicio.png)

---

### Requisito 7 – Servidores

Web Server con **Apache (HTTP)** operativo y DB Server con **MariaDB** escuchando en el puerto **3306**.

![MariaDB escuchando en 3306](imagenes/requisito%207%20escuchando%20por%203306%20.png)

---

### Requisito 8 – Salida a Internet

Ruta por defecto y NAT en el FortiGate. El PC de usuarios tiene salida hacia el ISP (`ping` y `traceroute` funcionando).

![Salida a Internet](imagenes/requisito%20%238%20salida%20a%20internet.png)

---

## 6. Archivos de configuración

| Equipo | Archivo |
|---|---|
| FG1-USUARIOS | [configs/FG1-USUARIOS.txt](configs/FG1-USUARIOS.txt) |
| FG2-SERVER | [configs/FG2-SERVER.txt](configs/FG2-SERVER.txt) |
| SW-SITIO01 | [configs/SW-SITIO01.txt](configs/SW-SITIO01.txt) |

---

## 7. Video

[Enlace al video de la demostración](PEGAR_AQUI_EL_ENLACE_DEL_VIDEO)
