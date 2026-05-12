# Informe de Laboratorio 4.2 — Sección 3: Práctica Grupal
## Servidor DNS Primario e Integración con Servicio Web

**Universidad San Francisco Xavier de Chuquisaca**  
**Asignatura:** Infraestructura, Plataformas Tecnológicas y Redes (SIS313)  
**Docente:** Ing. Marcelo Quispe Ortega  
**Semestre:** 1/2026  

**Integrantes:**
- Luis Quispe Sullca
- Alex Calatayud Mamani

**Dominio del grupo:** `usmacapa.red`  

> **Nota:** Las VMs se configuraron con adaptador en modo **Puente (Bridge)**, por lo que las IPs son asignadas dinámicamente por la red del aula. En este informe se utilizaron las IPs:
> - VM DNS: `192.168.1.7`
> - VM Web: `192.168.1.8`
> Estas IPs pueden variar en cada sesión.

---

## 1. Configuración de la VM DNS (BIND9)

### 1.1 Configuración de Red

Se configuró el adaptador en modo Puente para obtener una IP en la red del aula. Se verificó la IP asignada con:

```bash
ip a
```
<img width="947" height="313" alt="Captura de pantalla 2026-05-11 213358" src="https://github.com/user-attachments/assets/c6a698a2-007a-4e75-97ba-7d1a9eac8b48" />

---

### 1.2 Instalación de BIND9

```bash
sudo apt update
sudo apt install bind9 bind9utils bind9-doc -y
```

---

### 1.3 Configuración de la Zona en `named.conf.local`

```bash
sudo nano /etc/bind/named.conf.local
```

Se agregaron las zonas directa e inversa:

```
zone "usmacapa.red" {
    type master;
    file "/etc/bind/db.usmacapa.red";
};

zone "1.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.1";
};
```
<img width="962" height="212" alt="Captura de pantalla 2026-05-11 213824" src="https://github.com/user-attachments/assets/505cb720-5dbe-47d4-aa06-025a124909ef" />

---

### 1.4 Creación del Archivo de Zona Directa

```bash
sudo cp /etc/bind/db.local /etc/bind/db.usmacapa.red
sudo nano /etc/bind/db.usmacapa.red
```

Contenido del archivo:

```
$TTL    604800
@       IN      SOA     dns.usmacapa.red. admin.usmacapa.red. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      dns.usmacapa.red.
dns     IN      A       192.168.1.7
www     IN      A       192.168.1.8
web     IN      CNAME   www.usmacapa.red.
```

---

### 1.5 Creación del Archivo de Zona Inversa

```bash
sudo cp /etc/bind/db.local /etc/bind/db.192.168.1
sudo nano /etc/bind/db.192.168.1
```

Contenido del archivo:

```
$TTL    604800
@       IN      SOA     dns.usmacapa.red. admin.usmacapa.red. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      dns.usmacapa.red.
7       IN      PTR     dns.usmacapa.red.
8       IN      PTR     www.usmacapa.red.
```

---

### 1.6 Verificación y Reinicio de BIND9

```bash
sudo named-checkconf
sudo named-checkzone usmacapa.red /etc/bind/db.usmacapa.red
sudo named-checkzone 1.168.192.in-addr.arpa /etc/bind/db.192.168.1
sudo systemctl restart bind9
sudo systemctl status bind9
```
<img width="797" height="523" alt="image" src="https://github.com/user-attachments/assets/fb733b60-f86b-42d3-8cae-09091afeba41" />

---

### 1.7 Apertura del Puerto 53 en el Firewall

```bash
sudo apt install ufw -y
sudo ufw allow 53
sudo ufw allow 22
sudo ufw enable
```

---

## 2. Configuración de la VM Web (Nginx)

### 2.1 Configuración de Red

Se configuró el adaptador en modo Puente. Se verificó la IP asignada con:

```bash

```

<img width="947" height="432" alt="Captura de pantalla 2026-05-11 214942" src="https://github.com/user-attachments/assets/4e2f4e47-435a-42a2-ba74-8feeedf39593" />

---

### 2.2 Instalación de Nginx

```bash
sudo apt update
sudo apt install nginx -y
sudo systemctl enable --now nginx
```

---

### 2.3 Creación del Sitio Web

```bash
sudo mkdir -p /var/www/usmacapa.red
sudo nano /var/www/usmacapa.red/index.html
```

Contenido del archivo:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Usmacapa - SIS313</title>
</head>
<body>
    <h1>Bienvenidos a usmacapa.red</h1>
    <p>Integrantes: Luis Quispe Sullca, Alex Calatayud Mamani</p>
    <p>Servidor Web - Laboratorio 4.2</p>
</body>
</html>
```

---

### 2.4 Configuración del Virtual Host

```bash
sudo nano /etc/nginx/sites-available/usmacapa.red
```

Contenido:

```
server {
    listen 80;
    server_name www.usmacapa.red usmacapa.red;
    root /var/www/usmacapa.red;
    index index.html;
    location / {
        try_files $uri $uri/ =404;
    }
}
```
<img width="946" height="151" alt="Captura de pantalla 2026-05-11 215121" src="https://github.com/user-attachments/assets/9fe146dc-a98e-441a-900f-5afb97e81b2f" />

---

### 2.5 Activación del Sitio

```bash
sudo ln -s /etc/nginx/sites-available/usmacapa.red /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

<img width="946" height="348" alt="Captura de pantalla 2026-05-11 215346" src="https://github.com/user-attachments/assets/7a858683-4140-4ce0-b873-acc1fcb0dff3" />

---

### 2.6 Configuración del DNS en la VM Web

Para que la VM Web resuelva el dominio `usmacapa.red` usando el servidor BIND9:

```bash
sudo nano /etc/systemd/resolved.conf
```

Se agregó:

```
[Resolve]
DNS=192.168.1.7
Domains=usmacapa.red
MulticastDNS=no
```

```bash
sudo systemctl restart systemd-resolved
```

---

### 2.7 Prueba desde la VM Web

```bash
curl http://www.usmacapa.red
```

<img width="886" height="217" alt="Captura de pantalla 2026-05-11 215508" src="https://github.com/user-attachments/assets/69afee30-cea5-4521-8337-be6226f53825" />

---

## 3. Verificación desde la PC Anfitriona (Windows)

### 3.1 Configuración del DNS en Windows

Se deshabilitó IPv6 en el adaptador Wi-Fi para evitar que el router interfiriera con el DNS:

1. `Windows + R` → `ncpa.cpl` → Enter
2. Clic derecho en **Wi-Fi** → **Propiedades**
3. Se desmarcó **"Protocolo de Internet versión 6 (TCP/IPv6)"**
4. En **Protocolo de Internet versión 4 (TCP/IPv4)** → **Propiedades**
5. Se configuró DNS preferido: `192.168.1.7` y DNS alternativo: `8.8.8.8`

<img width="487" height="240" alt="Captura de pantalla 2026-05-11 215603" src="https://github.com/user-attachments/assets/667c3c8f-b0e4-4925-a59a-4bcc7b9b5f92" />


---

### 3.2 Prueba de Resolución Directa

```cmd
nslookup www.usmacapa.red
```

Resultado esperado:
```
Servidor:  dns.usmacapa.red
Address:   192.168.1.7

Nombre:    www.usmacapa.red
Address:   192.168.1.8
```


---

### 3.3 Prueba de Resolución Inversa

```cmd
nslookup 192.168.1.8
```

Resultado esperado:
```
Servidor:  dns.usmacapa.red
Address:   192.168.1.7

Nombre:    www.usmacapa.red
Address:   192.168.1.8
```

<img width="1067" height="273" alt="image" src="https://github.com/user-attachments/assets/9145b5fa-c6a9-40ff-8e54-9b81064aecaa" />


---

### 3.4 Acceso Web desde el Navegador

Se accedió desde el navegador a:

```
http://www.usmacapa.red
```

<img width="952" height="196" alt="Captura de pantalla 2026-05-11 215737" src="https://github.com/user-attachments/assets/52603894-b1c9-4e42-86b0-2b321657f201" />


---

## 4. Conclusiones

- Se configuró exitosamente un servidor DNS primario con BIND9, incluyendo zona directa e inversa para el dominio `usmacapa.red`.
- Se integró el servidor DNS con un servidor web Nginx, permitiendo el acceso al sitio por nombre de dominio en lugar de dirección IP.
- Se verificó la resolución de nombres desde la PC anfitriona Windows, tanto de forma directa como inversa.
- El uso del adaptador puente (Bridge) permitió que ambas VMs fueran visibles en la red física, simulando un entorno de red real.
