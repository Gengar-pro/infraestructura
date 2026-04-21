# 🌐 Proxy Inverso con Balanceador de Carga Avanzado y Servidores Web NGINX

<div align="center">

![NGINX](https://img.shields.io/badge/NGINX-009639?style=for-the-badge&logo=nginx&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu_Server_24.04-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![Alpine](https://img.shields.io/badge/Alpine_Linux_3.22-0D597F?style=for-the-badge&logo=alpinelinux&logoColor=white)
![PHP](https://img.shields.io/badge/PHP--FPM-777BB4?style=for-the-badge&logo=php&logoColor=white)
![Node.js](https://img.shields.io/badge/Node.js-339933?style=for-the-badge&logo=nodedotjs&logoColor=white)
![Python](https://img.shields.io/badge/Python_3-3776AB?style=for-the-badge&logo=python&logoColor=white)

**Universidad San Francisco Xavier de Chuquisaca**
**Asignatura:** Infraestructura, Plataformas Tecnológicas y Redes — `SIS313`
**Docente:** Ing. Marcelo Quispe Ortega | **Semestre:** 1/2026

</div>

---

## 👑 Lead Developer — Variante 4

<div align="center">

| | Nombre | Carrera | Rol en esta variante |
|:---:|:---|:---|:---:|
| 👤 | Chambi Condori Janet | Ingeniería de Sistemas | Colaboradora |
| 👤 | Calatayud Mamani Alex Josue | Ingeniería de Sistemas | Colaborador |
| 👤 | Jhon Christian Quispe Anagua | Ingeniería de Sistemas | Colaborador |
| ⭐ | **Quispe Sullca Luis Fernando** | **Ingeniería en Ciencias de la Computación** | **`LEAD DEVELOPER`** |

</div>

> 💡 **Nota:** Todos los integrantes contribuyeron por igual al desarrollo del laboratorio. El rol de *Lead Developer* en esta variante es asignado a **Luis Fernando Quispe Sullca** como representante del flujo de trabajo colaborativo en Git.

---

## 📋 Tabla de Contenidos

- [Introducción](#introducción)
- [Tecnologías](#tecnologías)
- [Arquitectura del Sistema](#arquitectura-del-sistema)
- [Metodología](#metodología)
- [Análisis de Resultados](#análisis-de-resultados)
- [Conclusiones](#conclusiones)
- [Evidencia](#evidencia)
- [Anexos](#anexos)

---

## 📖 Introducción

### 🔀 Proxy Inverso

Un **proxy inverso** es un servidor que se sitúa delante de uno o varios servidores backend y reenvía las solicitudes de los clientes hacia ellos. A diferencia de un proxy tradicional (que actúa en nombre del cliente), el proxy inverso actúa **en nombre del servidor**. El cliente nunca se comunica directamente con el servidor backend — solo ve la dirección del proxy.

Sus principales ventajas son:

- 🔒 Oculta la arquitectura interna del sistema
- 🛡️ Centraliza el control de acceso y seguridad
- ⚖️ Permite distribuir la carga entre múltiples servidores
- 🚀 Facilita la terminación SSL y el caché de contenido

### ⚖️ Balanceo de Carga

El **balanceo de carga** es el proceso de distribuir el tráfico de red entrante entre múltiples servidores backend. Los tres algoritmos principales implementados son:

| Algoritmo | Descripción | Persistencia |
|:---|:---|:---:|
| 🔄 **Round Robin** | Distribuye las peticiones de forma secuencial y cíclica entre todos los servidores disponibles. Algoritmo por defecto en NGINX. | ❌ |
| 📊 **Least Connection** | Envía cada nueva petición al servidor con el menor número de conexiones activas. Ideal para sesiones largas. | ❌ |
| 🔑 **IP Hash** | Calcula un hash de la IP del cliente, mapeándolo siempre al mismo backend. Garantiza persistencia de sesión. | ✅ |

---

## 🛠️ Tecnologías

<div align="center">

| Capa | Tecnología | Versión | Rol |
|:---:|:---:|:---:|:---|
| 🔀 Proxy / Balanceador | NGINX | Latest | Proxy inverso + balanceo de carga |
| 🖥️ Proxy OS | Ubuntu Server | 24.04 LTS | Sistema operativo del proxy |
| 🐧 Backends OS | Alpine Linux | 3.22 | Sistema operativo de los backends |
| 🐘 Backend 1 & 3 | PHP-FPM | 8.3 | Servidor web dinámico (FastCGI) |
| 🟢 Backend 2 & 4 | Node.js | LTS | Servidor HTTP con JS |
| 🐍 Backend 5 & 6 | Python 3 | stdlib | Servidor HTTP con http.server |
| 🔧 Virtualización | VirtualBox | Latest | Gestión de VMs |
| 📡 Red | Hotspot Móvil | — | Conectividad entre equipos |

</div>

---

## 🏗️ Arquitectura del Sistema

### 📊 Tabla de Infraestructura

| Máquina | Rol | Host Físico | IP | SO | Tecnología |
|:---|:---|:---|:---:|:---:|:---:|
| `Lab3-Proxy` | Proxy + Balanceador + NAT | PC Principal (Ubuntu Desktop) | `192.168.237.100` | Ubuntu Server 24.04 | NGINX |
| `Lab3-Web1` | Servidor Web Backend | Laptop 1 | `192.168.237.101` | Alpine Linux 3.22 | PHP-FPM |
| `Lab3-Web2` | Servidor Web Backend | Laptop 1 | `192.168.237.102` | Alpine Linux 3.22 | Node.js |
| `Lab3-Web3` | Servidor Web Backend | Laptop 2 | `192.168.237.103` | Alpine Linux 3.22 | PHP-FPM |
| `Lab3-Web4` | Servidor Web Backend | Laptop 2 | `192.168.237.104` | Alpine Linux 3.22 | Node.js |
| `Lab3-Web5` | Servidor Web Backend | Laptop 3 | `192.168.237.105` | Alpine Linux 3.22 | Python 3 |
| `Lab3-Web6` | Servidor Web Backend | Laptop 3 | `192.168.237.106` | Alpine Linux 3.22 | Python 3 |

### 🗺️ Diagrama de Red

```
                        INTERNET
                            |
                    [Hotspot Celular]
                    192.168.237.149/24 (gateway)
                            |
              +-------------+-------------+
              |                           |
    [PC Principal]                  [Laptops 1,2,3]
    Ubuntu Desktop                  Windows
    wlp3s0: DHCP                    WiFi: DHCP
              |
        [VM: Proxy]
        enp0s3: NAT (Warp SSH)
        enp0s8: 192.168.237.100
        NGINX Balanceador
              |
    +---------+---------+---------+---------+---------+---------+
    |         |         |         |         |         |         |
 [Web1]    [Web2]    [Web3]    [Web4]    [Web5]    [Web6]
  .101       .102      .103      .104      .105      .106
  PHP        Node.js   PHP       Node.js   Python    Python
```

### 🔌 Configuración de Adaptadores de Red

| Adaptador | Tipo | Propósito |
|:---|:---:|:---|
| `eth0` (Adaptador 1) | NAT | Acceso SSH desde Warp |
| `eth1` (Adaptador 2) | Adaptador Puente (Bridged) | Comunicación con el balanceador |

| VM | Puerto SSH Host | Puerto HTTP Host |
|:---:|:---:|:---:|
| Web1 | `2222` | `8080` |
| Web2 | `2223` | `8081` |
| Web3 | `2224` | `8082` |
| Web4 | `2225` | `8083` |
| Web5 | `2226` | `8084` |
| Web6 | `2227` | `8085` |

---

## ⚙️ Metodología

### 🔧 Bloque 1 — Configuración del Proxy (Ubuntu Server 24.04)

#### IP estática con Netplan

```yaml
# /etc/netplan/00-installer-config.yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: yes
    enp0s8:
      dhcp4: no
      addresses:
        - 192.168.237.100/24
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```

```bash
sudo netplan apply
sudo chmod 600 /etc/netplan/00-installer-config.yaml
```

#### Instalación de NGINX

```bash
sudo apt update && sudo apt install -y nginx
sudo systemctl enable nginx
sudo systemctl start nginx
```

#### IP Forwarding y NAT

```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
sudo apt install -y iptables-persistent
sudo netfilter-persistent save
```

---

### 🐧 Bloque 2 — Backends Alpine Linux

#### 🐘 Backends PHP (Web1, Web3)

```bash
apk add nginx php php-fpm curl nano
rc-update add php-fpm83 default
rc-update add nginx default
rc-service php-fpm83 start
```

```nginx
server {
    listen 80;
    server_name _;
    root /var/www/localhost/htdocs;

    location / {
        index index.php index.html;
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
```

#### 🟢 Backends Node.js (Web2, Web4)

```javascript
const http = require('http');
const os = require('os');
const { execSync } = require('child_process');

const hostname = os.hostname();
const hostip = execSync('hostname -I').toString().trim().split(' ')[0];

const server = http.createServer((req, res) => {
    res.statusCode = 200;
    res.setHeader('Content-Type', 'text/html; charset=utf-8');
    res.end(`
    <!DOCTYPE html>
    <html lang="es">
    <head><meta charset="UTF-8"><title>Servidor ${hostname}</title></head>
    <body>
        <h1>🟢 Node.js | Servidor: ${hostname} | IP: ${hostip}</h1>
        <p>Tecnología: Node.js ${process.version}</p>
    </body>
    </html>
    `);
});

server.listen(3000, '127.0.0.1');
```

#### 🐍 Backends Python (Web5, Web6)

```python
from http.server import HTTPServer, BaseHTTPRequestHandler
import socket, subprocess

hostname = socket.gethostname()
hostip = subprocess.check_output(['hostname', '-I']).decode().strip().split()[0]

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        html = f"""<!DOCTYPE html>
<html lang="es"><head><meta charset="UTF-8"></head>
<body>
    <h1>🐍 Python | Servidor: {hostname} | IP: {hostip}</h1>
    <p>Tecnología: Python3 (http.server)</p>
</body></html>"""
        self.send_response(200)
        self.send_header('Content-Type', 'text/html; charset=utf-8')
        self.end_headers()
        self.wfile.write(html.encode())
    def log_message(self, format, *args): pass

if __name__ == '__main__':
    HTTPServer(('127.0.0.1', 5000), Handler).serve_forever()
```

---

### ⚖️ Bloque 3 — Configuración del Balanceador de Carga

```nginx
upstream backend {
    least_conn;

    server 192.168.237.101:80;
    server 192.168.237.102:80;
    server 192.168.237.103:80;
    server 192.168.237.104:80;
    server 192.168.237.105:80;
    server 192.168.237.106:80;
}

server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;

    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```bash
sudo nginx -t && sudo systemctl restart nginx
```

---

## 📊 Análisis de Resultados

### Comparación de Algoritmos

| Algoritmo | Distribución | Persistencia | Caso de Uso Ideal |
|:---|:---|:---:|:---|
| 🔄 Round Robin | Secuencial y cíclica | ❌ | Servidores homogéneos, sesiones cortas |
| 📊 Least Connection | Al servidor con menos conexiones activas | ❌ | Tiempos de respuesta variables |
| 🔑 IP Hash | Hash de la IP del cliente | ✅ | Aplicaciones con estado de sesión |

### Comportamiento bajo Falla

- ✅ NGINX detecta automáticamente backends caídos y redirige el tráfico sin intervención manual.
- ⚠️ Con todos los backends caídos, el proxy devuelve `502 Bad Gateway`.
- 🔁 Al restaurar los backends, el balanceador retoma la distribución automáticamente.

### Diferencias entre Tecnologías Backend

| Aspecto | PHP-FPM | Node.js | Python |
|:---|:---:|:---:|:---:|
| Comunicación con NGINX | FastCGI (`:9000`) | HTTP proxy_pass (`:3000`) | HTTP proxy_pass (`:5000`) |
| Arranque automático | `rc-update` (nativo) | Script `/etc/local.d` | Script `/etc/local.d` |
| Dependencias | `php`, `php-fpm` | `nodejs`, `npm` | `python3` (stdlib) |
| Complejidad config | Media | Baja | Baja |

> 💡 Desde el punto de vista del balanceador, las tres tecnologías son **completamente transparentes** — el proxy solo ve `IP:80`.

---

## 💡 Conclusiones

**Janet Chambi Condori (Ing. de Sistemas):**
> *[Conclusión personal sobre lo aprendido]*

**Alex Josue Calatayud Mamani (Ing. de Sistemas):**
> *[Conclusión personal sobre lo aprendido]*

**Jhon Christian Quispe Anagua (Ing. de Sistemas):**
> *[Conclusión personal sobre lo aprendido]*

**Luis Fernando Quispe Sullca (Ing. en Ciencias de la Computación):**
> *[Conclusión personal sobre lo aprendido]*

---

## 📸 Evidencia

> 📦 **Se adjuntan capturas del archivo `.zip` comprimido con 4 variantes de contenido idéntico como evidencia del flujo de trabajo Git colaborativo.**

| # | Sección | Descripción |
|:---:|:---|:---|
| 4.1 | Red del Proxy | `ip addr show enp0s8`, `ip route`, `ping -c 4 8.8.8.8` |
| 4.2 | NGINX activo | `sudo systemctl status nginx` |
| 4.3 | IP Forwarding y NAT | `cat /proc/sys/net/ipv4/ip_forward` + `iptables -t nat -L` |
| 4.4 | Backends individuales | `curl` a cada uno de los 6 backends |
| 4.5 | Round Robin (terminal) | 8 peticiones con alternancia secuencial |
| 4.6 | Round Robin (navegador) | Mínimo 5 recargas mostrando alternancia |
| 4.7 | Least Connection (AB) | `ab -n 100 -c 10 http://localhost/` |
| 4.8 | Caída de servidor | `curl` funcionando con un servidor caído |
| 4.9 | IP Hash (normal) | Navegador mostrando siempre el mismo servidor |
| 4.10 | IP Hash (incógnito) | Ventana incógnito con el mismo comportamiento |
| 4.11 | Falla total | Error `502 Bad Gateway` con todos los backends caídos |
| 4.12 | Restauración | Sistema distribuyendo tráfico tras restaurar backends |

---

## 📎 Anexos

<details>
<summary>📄 Anexo A — Configuración completa NGINX (Proxy)</summary>

```nginx
upstream backend {
    least_conn;
    server 192.168.237.101:80;
    server 192.168.237.102:80;
    server 192.168.237.103:80;
    server 192.168.237.104:80;
    server 192.168.237.105:80;
    server 192.168.237.106:80;
}

server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;
    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

</details>

<details>
<summary>📄 Anexo B — Netplan completo del Proxy</summary>

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: yes
    enp0s8:
      dhcp4: no
      addresses:
        - 192.168.237.100/24
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```

</details>

<details>
<summary>📄 Anexo C — Interfaces de red Alpine (ejemplo Web1)</summary>

```
auto eth0
iface eth0 inet dhcp

auto eth1
iface eth1 inet static
    address 192.168.237.101
    netmask 255.255.255.0
    gateway 192.168.237.149
    dns-nameservers 8.8.8.8
```

</details>

<details>
<summary>📄 Anexo D — Scripts de arranque automático</summary>

```bash
# /etc/local.d/nodejs.start
#!/bin/sh
cd /var/www/nodejs
node index.js > /tmp/nodejs.log 2>&1 &
```

```bash
# /etc/local.d/python.start
#!/bin/sh
cd /var/www/python
python3 app.py > /tmp/python.log 2>&1 &
```

</details>

---

<div align="center">

**Universidad San Francisco Xavier de Chuquisaca — SIS313 — Semestre 1/2026**
*Laboratorio 3.1 — Proxy Inverso con Balanceador de Carga Avanzado*

![Made with](https://img.shields.io/badge/Made%20with-❤️%20y%20NGINX-009639?style=flat-square)

</div>
