# Lab 5.1 — Hardening Integral y Seguridad TLS

**Universidad:** Universidad San Francisco Xavier de Chuquisaca (USFX)  
**Materia:** SIS313 — Seguridad en Redes  
**Grupo:** Usmacapa  
**Integrante 1:** Alex Calatayud Mamani — VM Web  
**Integrante 2:** Luis Quispe Sullca — VM DB  
**Fecha:** Mayo 2026  
**Link de GitHub** https://github.com/Gengar-pro/infraestructura/edit/main/files/Informe_lab5.1.md

---

## Índice

1. [Introducción](#1-introducción)
2. [Objetivos](#2-objetivos)
3. [Entorno de trabajo](#3-entorno-de-trabajo)
4. [Configuración VM Web](#4-configuración-vm-web)
5. [Configuración VM DB](#5-configuración-vm-db)
6. [Pruebas de verificación](#6-pruebas-de-verificación)
7. [Análisis de resultados](#7-análisis-de-resultados)
8. [Conclusiones](#8-conclusiones)
9. [Anexos](#9-anexos)

---

## 1. Introducción

El hardening (endurecimiento) es el proceso de reducir la superficie de ataque de un sistema. Un servidor recién instalado viene con muchas cosas habilitadas por defecto que no son necesarias, y cada una de ellas representa una puerta potencial para un atacante. El objetivo es deshabilitar todo lo innecesario y reforzar lo que sí se utiliza.

Este laboratorio aplica hardening en cinco capas simultáneas sobre dos servidores:

| Capa | Descripción |
|---|---|
| SSH Hardening | Puerto cambiado, autenticación por clave pública, sin acceso root |
| Firewall UFW | Denegar todo por defecto, abrir solo puertos necesarios |
| Kernel Hardening | Deshabilitar IP spoofing, volcados de memoria, SysRq |
| TLS/HTTPS | Cifrado de comunicaciones, TLS 1.2/1.3, certificado autofirmado |
| Cabeceras HTTP | HSTS, X-Frame-Options, X-Content-Type-Options, X-XSS-Protection |

---

## 2. Objetivos

- Aplicar hardening integral sobre un servidor web (Nginx) y un servidor de base de datos (MariaDB).
- Configurar TLS con certificado autofirmado y forzar HTTPS en todas las comunicaciones.
- Implementar un firewall con política de mínimo privilegio en ambos servidores.
- Fortalecer el servicio SSH mediante autenticación por clave pública y restricción de acceso.
- Aplicar parámetros de seguridad a nivel de kernel para prevenir ataques comunes.
- Configurar un servidor DNS local (BIND9) para resolver el dominio `usmacapa.local`.
- Realizar pruebas cruzadas entre ambas VMs para verificar la segmentación de red.

---

## 3. Entorno de trabajo

| VM | Rol | IP | Usuario |
|---|---|---|---|
| `web` | Nginx + TLS + DNS (BIND9) | `10.173.175.200` | alex |
| `db` | MariaDB | `10.173.175.24` | luisf |

Ambas VMs corren **Ubuntu Server 24.04** en VirtualBox con adaptador en modo **Bridge**, conectadas a la red del laboratorio.

---

## 4. Configuración VM Web

### 4.1 Red — IP estática

Se configuró una IP estática en la VM Web usando Netplan para que la dirección no cambie entre reinicios.

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: no
      addresses:
        - 10.173.175.200/24
      nameservers:
        addresses:
          - 8.8.8.8
      routes:
        - to: default
          via: 10.173.175.1
```

```bash
sudo netplan apply
ip addr show enp0s3
ping -c 3 8.8.8.8
```

---

### 4.2 SSH Hardening

El servicio SSH es el principal vector de ataque en servidores Linux. Se aplicaron las siguientes medidas de fortalecimiento.

#### Generación de par de claves (desde PC anfitriona Windows)

```powershell
ssh-keygen -t ed25519 -C "alex@lab51"
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh alex@10.173.175.200 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

<img width="852" height="237" alt="image1" src="https://github.com/user-attachments/assets/6106551f-a801-489c-82bd-59d466d1e3a4" />

*Clave pública registrada en el servidor*


#### Configuración de /etc/ssh/sshd_config

```bash
sudo nano /etc/ssh/sshd_config
sudo nano /etc/ssh/sshd_config.d/50-cloud-init.conf
```

<img width="953" height="816" alt="image3" src="https://github.com/user-attachments/assets/21ed3028-9181-486b-be85-523f70e9bf27" />


*Configuración SSH con hardening: PasswordAuthentication no*

```
Port 2222
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
AllowUsers alex
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket
sudo ss -tulnp | grep ssh
```

<img width="1032" height="95" alt="image4" src="https://github.com/user-attachments/assets/4c5fb2ff-5bda-4652-9087-4e7023c3de8e" />


*SSH escuchando correctamente en el puerto 2222*

| Parámetro | Razón |
|---|---|
| `Port 2222` | Reduce ataques de bots que escanean el puerto 22 |
| `PermitRootLogin no` | Evita ataques directos al usuario root |
| `PasswordAuthentication no` | Elimina ataques de fuerza bruta |
| `PubkeyAuthentication yes` | Fuerza autenticación criptográfica |
| `MaxAuthTries 3` | Limita intentos fallidos por conexión |
| `AllowUsers alex` | Lista blanca de usuarios permitidos |

---

### 4.3 Firewall UFW

Se aplicó la política de **mínimo privilegio**: denegar todo por defecto y abrir únicamente los puertos estrictamente necesarios.

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2222/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 53/tcp
sudo ufw allow 53/udp
sudo ufw enable

```

<img width="813" height="432" alt="image5" src="https://github.com/user-attachments/assets/e5f9d13b-2755-4df4-9f3b-d87b581aa232" />


*Proceso de configuración de reglas UFW*


```bash
sudo ufw status verbose
````
<img width="755" height="285" alt="image6" src="https://github.com/user-attachments/assets/359e8be6-8635-41e2-b142-570d711c6bd6" />



| Puerto | Razón |
|---|---|
| `2222/tcp` | SSH hardened — administración remota |
| `80/tcp` | HTTP — redirige automáticamente a HTTPS (301) |
| `443/tcp` | HTTPS — tráfico web cifrado con TLS |
| `53/tcp/udp` | DNS — respuestas del servidor BIND9 |

---

### 4.4 Kernel Hardening

```bash
sudo nano /etc/sysctl.conf
```

<img width="695" height="185" alt="image7" src="https://github.com/user-attachments/assets/404e5a28-3c50-42c0-b0b5-5381a12d1e1d" />


```
net.ipv4.ip_forward=1
net.ipv4.conf.all.rp_filter=1
kernel.sysrq=0
fs.suid_dumpable=0
```

```bash
sudo sysctl -p
```


**`ip_forward=1`** — La VM Web actúa como gateway, necesita reenviar paquetes de la DB hacia internet.  
**`rp_filter=1`** — Descarta paquetes con IPs de origen falsificadas (IP spoofing).  
**`sysrq=0`** — Deshabilita combinación de teclas que permite control del kernel desde acceso físico.  
**`suid_dumpable=0`** — Impide volcados de memoria de procesos privilegiados que podrían exponer contraseñas.

---

### 4.5 Instalación de Nginx

```bash
sudo apt update && sudo apt install nginx -y
sudo systemctl enable --now nginx
sudo mkdir -p /var/www/usmacapa.local
sudo nano /var/www/usmacapa.local/index.html
```

<img width="800" height="393" alt="image8" src="https://github.com/user-attachments/assets/202f7007-9a99-4aff-bcac-0fa530f0a2c5" />


*Contenido del sitio web del grupo Usmacapa*

```bash
sudo nano /etc/nginx/sites-available/usmacapa.local
```
<img width="545" height="216" alt="image9" src="https://github.com/user-attachments/assets/59447e59-6117-4238-8bee-3c5c2d7325db" />


```bash
sudo ln -s /etc/nginx/sites-available/usmacapa.local /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl restart nginx
```

---

### 4.6 Certificado SSL/TLS autofirmado

```bash
sudo mkdir -p /etc/nginx/ssl
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/nginx-selfsigned.key \
  -out /etc/nginx/ssl/nginx-selfsigned.crt \
  -subj "/C=BO/ST=Chuquisaca/L=Sucre/O=USFX/OU=SIS313/CN=usmacapa.local"
```

| Parámetro | Significado |
|---|---|
| `-x509` | Certificado autofirmado — nosotros somos la CA |
| `-nodes` | Clave privada sin contraseña — necesario para arranque automático de Nginx |
| `-days 365` | Válido por 1 año |
| `-newkey rsa:2048` | Clave RSA de 2048 bits |
| `-subj` | Datos del certificado: país, ciudad, organización, dominio |

```bash
sudo openssl x509 -in /etc/nginx/ssl/nginx-selfsigned.crt -text -noout | grep -E "Subject:|Issuer:|Not Before|Not After"
```

> Subject e Issuer son iguales — confirma que es autofirmado.

---

### 4.7 Nginx con HTTPS y cabeceras de seguridad

```bash
sudo nano /etc/nginx/sites-available/usmacapa.local
```

```nginx
server {
    listen 80;
    server_name usmacapa.local www.usmacapa.local;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl;
    server_name usmacapa.local www.usmacapa.local;
    root /var/www/usmacapa.local;
    index index.html;

    ssl_certificate /etc/nginx/ssl/nginx-selfsigned.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx-selfsigned.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH';
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;

    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    server_tokens off;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

```bash
sudo nginx -t && sudo systemctl restart nginx
```

| Cabecera | Ataque que previene |
|---|---|
| `Strict-Transport-Security` | Downgrade attack — fuerza HTTPS por 2 años |
| `X-Frame-Options: SAMEORIGIN` | Clickjacking — impide iframes ajenos |
| `X-Content-Type-Options: nosniff` | MIME sniffing — el navegador no adivina tipos de archivos |
| `X-XSS-Protection: 1; mode=block` | Cross-Site Scripting |

---

### 4.8 BIND9 — Servidor DNS

```bash
sudo apt install bind9 bind9utils -y
sudo nano /etc/bind/named.conf.local
```

```
zone "usmacapa.local" {
    type master;
    file "/etc/bind/db.usmacapa.local";
};
```

```bash
sudo cp /etc/bind/db.local /etc/bind/db.usmacapa.local
sudo nano /etc/bind/db.usmacapa.local
```

```
$TTL    604800
@   IN  SOA  ns1.usmacapa.local. admin.usmacapa.local. (
                  2 ; Serial
             604800 ; Refresh
              86400 ; Retry
            2419200 ; Expire
             604800 ) ; Negative Cache TTL
;
@   IN  NS   ns1.usmacapa.local.
ns1 IN  A    10.173.175.200
@   IN  A    10.173.175.200
www IN  A    10.173.175.200
```

```bash
sudo named-checkconf
sudo named-checkzone usmacapa.local /etc/bind/db.usmacapa.local
sudo systemctl restart bind9
dig @10.173.175.200 usmacapa.local
```

---

## 5. Configuración VM DB

### 5.1 Red — IP estática

La VM DB usa la VM Web como gateway para salir a internet y como servidor DNS.

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

!<img width="845" height="311" alt="Captura desde 2026-05-21 10-15-57" src="https://github.com/user-attachments/assets/0597b68c-89c0-4975-9ebd-3bc5efe11daf" />

*Netplan VM DB: IP 10.173.175.24, gateway y DNS apuntando a la VM Web*

```yaml
network:
  version: 2
  ethernets:
    enp0s8:
      dhcp4: no
      addresses:
        - 10.173.175.24/24
      nameservers:
        addresses:
          - 10.173.175.200
      routes:
        - to: default
          via: 10.173.175.200
```

```bash
sudo netplan apply
```

---

### 5.2 SSH Hardening

```bash
cat ~/.ssh/authorized_keys
```


<img width="916" height="86" alt="Captura desde 2026-05-21 10-21-06" src="https://github.com/user-attachments/assets/fcecd6b3-fbbc-4fa0-a01b-112139ccf93d" />

*Clave pública `web@usmacapa` registrada — la VM Web puede conectarse sin contraseña*

```bash
sudo nano /etc/ssh/sshd_config
```

<img width="762" height="745" alt="Captura desde 2026-05-21 10-17-57" src="https://github.com/user-attachments/assets/cc66dc3c-f50f-491d-899e-ec20b4fc4687" />


*sshd_config de la VM DB con hardening aplicado*

```
Port 2222
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
AllowUsers luisf
```

```bash
sudo systemctl daemon-reload && sudo systemctl restart ssh.socket
```

---

### 5.3 Firewall UFW

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 10.173.175.0/24 to any port 2222/tcp
sudo ufw allow from 10.173.175.200 to any port 3306/tcp
sudo ufw enable
sudo ufw status verbose
```

<img width="916" height="201" alt="Captura desde 2026-05-21 10-22-59" src="https://github.com/user-attachments/assets/699a0e28-c6bb-4d1f-8f6d-ba814280f40b" />


*UFW en VM DB: SSH desde la red del lab, MariaDB solo desde VM Web*

| Regla | Razón |
|---|---|
| `2222 from 10.173.175.0/24` | SSH desde cualquier PC del laboratorio |
| `3306 from 10.173.175.200` | MariaDB solo accesible desde el servidor Web |

---

### 5.4 Kernel Hardening

```bash
sudo nano /etc/sysctl.conf
```

```
net.ipv4.ip_forward=0
net.ipv4.conf.all.rp_filter=1
kernel.sysrq=0
fs.suid_dumpable=0
```

```bash
sudo sysctl -p
```

> `ip_forward=0` en la DB — no actúa como gateway y no debe reenviar paquetes.

---

### 5.5 Instalación de MariaDB

```bash
sudo apt update && sudo apt install mariadb-server -y
sudo systemctl enable --now mariadb
sudo mysql_secure_installation
```

| Pregunta | Respuesta |
|---|---|
| Remove anonymous users | Y |
| Disallow root login remotely | Y |
| Remove test database | Y |
| Reload privilege tables | Y |

```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

```
bind-address = 10.173.175.24
```

```bash
sudo systemctl restart mariadb
sudo systemctl status mariadb
```

<img width="916" height="476" alt="Captura desde 2026-05-21 10-25-24" src="https://github.com/user-attachments/assets/317d009d-cee0-40e9-a772-3d88450dba08" />


*MariaDB activo y escuchando en la IP interna*

```bash
sudo ss -tulnp | grep 3306
```

<img width="970" height="46" alt="Captura desde 2026-05-21 10-26-49" src="https://github.com/user-attachments/assets/c0046d55-bc18-4678-88de-9aeb06325557" />


*MariaDB escucha en 10.173.175.24:3306 — no expuesto en 0.0.0.0*

```bash
sudo mysql -u root -p
```

```sql
CREATE DATABASE lab51_db;
CREATE USER 'appuser'@'10.173.175.200' IDENTIFIED BY 'ContraseñaSegura123';
GRANT ALL PRIVILEGES ON lab51_db.* TO 'appuser'@'10.173.175.200';
FLUSH PRIVILEGES;
SHOW DATABASES;
EXIT;
```

<img width="970" height="472" alt="Captura desde 2026-05-21 10-29-03" src="https://github.com/user-attachments/assets/b5b2abe8-cbf1-4228-829a-ed7a6a26f1f8" />


*Base de datos lab51_db creada y visible en MariaDB*

---

## 6. Pruebas de verificación

### 6.1 Redirección HTTP → HTTPS

```bash
curl -I http://10.173.175.200
```

```
HTTP/1.1 301 Moved Permanently
Location: https://usmacapa.local/
```

### 6.2 Cabeceras de seguridad HTTPS

```bash
curl -k -I https://10.173.175.200
```

```
HTTP/1.1 200 OK
Server: nginx
Strict-Transport-Security: max-age=63072000; includeSubDomains
X-Frame-Options: SAMEORIGIN
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
```

### 6.3 Versiones TLS

```bash
openssl s_client -connect 10.173.175.200:443 -tls1_2 </dev/null
# Protocol: TLSv1.2 ✅

openssl s_client -connect 10.173.175.200:443 -tls1_1 </dev/null
# no protocols available ✅
```


### 6.4 Resolución DNS

```bash
dig @10.173.175.200 usmacapa.local +short
# 10.173.175.200 ✅
```

### 6.5 Puerto 3306 bloqueado desde cliente externo

```bash
nc -zv 10.173.175.24 3306
# Connection refused ✅
```

### 6.6 SSH en puerto 22 bloqueado

```bash
ssh luisf@10.173.175.24        # Connection refused ✅
ssh -p 2222 luisf@10.173.175.24 # Acceso concedido ✅
```

---

## 7. Análisis de resultados
- aclarar que el N/A da a conocer que dicha tarea no correspondia a la maquina

| Ítem verificado | VM Web | VM DB |
|---|---|---|
| IP estática configurada | ✅ 10.173.175.200 | ✅ 10.173.175.24 |
| SSH en puerto 2222 | ✅ | ✅ |
| Autenticación por clave pública | ✅ | ✅ |
| Root login deshabilitado | ✅ | ✅ |
| UFW activo con deny por defecto | ✅ | ✅ |
| Kernel hardening aplicado | ✅ | ✅ |
| Nginx activo con HTTPS | ✅ | N/A |
| Certificado TLS CN=usmacapa.local | ✅ | N/A |
| TLS 1.2 permitido / TLS 1.1 bloqueado | ✅ | N/A |
| Cabeceras de seguridad HTTP | ✅ | N/A |
| Redirección HTTP → HTTPS 301 | ✅ | N/A |
| BIND9 DNS activo y resolviendo | ✅ | N/A |
| MariaDB escucha solo en IP interna | N/A | ✅ |
| Puerto 3306 solo desde VM Web | N/A | ✅ |
| Base de datos lab51_db creada | N/A | ✅ |

---

## 8. Conclusiones

- El hardening no es una acción única sino una **estrategia en capas** — si una capa falla, las demás siguen protegiendo el sistema.

- La **autenticación por clave pública SSH** es fundamentalmente más segura que las contraseñas porque la clave privada nunca viaja por la red y es matemáticamente imposible de adivinar.

- El firewall con **política deny por defecto** obliga a pensar explícitamente qué se permite, reduciendo enormemente la superficie de ataque.

- Separar la base de datos en una VM diferente y **restringir el acceso al puerto 3306 por IP** implementa el principio de mínimo privilegio a nivel de red.

- El uso de **TLS 1.2/1.3** y la deshabilitación de versiones antiguas elimina vulnerabilidades conocidas como POODLE y BEAST.

- **BIND9** permite resolver el dominio `usmacapa.local` en toda la red sin modificar el archivo hosts de cada máquina, demostrando una solución DNS centralizada.

---

## 9. Anexos


### Glosario

| Término | Definición |
|---|---|
| Hardening | Proceso de reducir la superficie de ataque de un sistema |
| TLS | Transport Layer Security — protocolo de cifrado de comunicaciones |
| CA | Certificate Authority — entidad que firma certificados digitales |
| UFW | Uncomplicated Firewall — interfaz para iptables en Linux |
| HSTS | HTTP Strict Transport Security — fuerza el uso de HTTPS |
| IP Spoofing | Falsificación de la dirección IP de origen en un paquete |
| Core Dump | Volcado de memoria generado cuando un proceso falla |
| Cipher Suite | Combinación de algoritmos usados en una conexión TLS |
| Mínimo privilegio | Principio: solo dar los permisos estrictamente necesarios |
| Superficie de ataque | Puntos de entrada que un atacante puede explotar |
| BIND9 | Servidor DNS de código abierto para sistemas Linux |
| Ed25519 | Algoritmo criptográfico moderno para claves SSH |

