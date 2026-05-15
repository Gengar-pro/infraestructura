# 🖥️ Informe de Laboratorio 4.2 — Sección 3
## Servidor DNS Primario e Integración con Servicio Web

---

<div align="center">

**Universidad San Francisco Xavier de Chuquisaca**
**Facultad de Ciencias y Tecnología**
**Asignatura:** Infraestructura, Plataformas Tecnológicas y Redes — *SIS313*
**Docente:** Ing. Marcelo Quispe Ortega
**Semestre:** 1/2026

</div>

---

## 👤 Integrantes del Grupo

| Nombre completo | Rol en la práctica |
|---|---|
| **Luis Fernando Quispe Sulla** | Configuración DNS (VM1) / Documentación |
| Alex Calatayud Mamani | Configuración Web (VM2) / Pruebas |

**Dominio asignado al grupo:** `usmacapa.red`

---

> ### 📌 Nota sobre direccionamiento IP
> Las VMs se configuraron con adaptador en modo **Puente (Bridge)**, por lo que las IPs son asignadas dinámicamente por el DHCP de la red del aula. Para esta sesión se trabajó con:
>
> | Máquina Virtual | IP asignada |
> |---|---|
> | VM DNS (BIND9) | `192.168.1.7` |
> | VM Web (Nginx) | `192.168.1.8` |
>
> ⚠️ *Estas IPs pueden variar en cada sesión de laboratorio.*

---

## 🗺️ Arquitectura de la Red

```
┌─────────────────────────────────────────────┐
│               Red del Aula (Bridge)          │
│                192.168.1.0/24                │
│                                             │
│   ┌─────────────────┐   ┌─────────────────┐ │
│   │   VM1 — DNS     │   │   VM2 — Web     │ │
│   │   BIND9         │   │   Nginx         │ │
│   │  192.168.1.7    │◄──►  192.168.1.8   │ │
│   │  Puerto: 53     │   │  Puerto: 80     │ │
│   └────────┬────────┘   └─────────────────┘ │
│            │                                │
│   ┌────────▼────────┐                       │
│   │   PC Anfitriona │                       │
│   │   Windows       │                       │
│   │   DNS: 1.7      │                       │
│   └─────────────────┘                       │
└─────────────────────────────────────────────┘
```

---

## Parte 1 — Configuración de la VM DNS con BIND9

### 1.1 Verificación de la Interfaz de Red

Antes de comenzar cualquier configuración, se verificó la IP asignada por la red del aula usando el adaptador en modo Puente:

```bash
ip a
```

![Verificación de IP en VM DNS](https://github.com/user-attachments/assets/c6a698a2-007a-4e75-97ba-7d1a9eac8b48)
*Captura: La VM DNS recibió la IP `192.168.1.7` mediante DHCP del aula.*

---

### 1.2 Instalación del Servicio BIND9

Se actualizaron los repositorios e instaló BIND9 junto con sus utilidades de diagnóstico:

```bash
sudo apt update
sudo apt install bind9 bind9utils bind9-doc -y
```

> **¿Por qué BIND9?** Es el servidor DNS de código abierto más usado en Linux. `bind9utils` incluye herramientas como `named-checkzone` y `named-checkconf` que permiten validar la configuración antes de aplicarla.

---

### 1.3 Declaración de Zonas en `named.conf.local`

Se editó el archivo de configuración local para declarar las zonas DNS del dominio `usmacapa.red`:

```bash
sudo nano /etc/bind/named.conf.local
```

```dns
zone "usmacapa.red" {
    type master;
    file "/etc/bind/db.usmacapa.red";
};

zone "1.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.1";
};
```

![Configuración named.conf.local](https://github.com/user-attachments/assets/505cb720-5dbe-47d4-aa06-025a124909ef)
*Captura: Declaración de zona directa e inversa en `named.conf.local`.*

> **Zona directa:** resuelve nombres → IPs (ej. `www.usmacapa.red` → `192.168.1.8`)
> **Zona inversa:** resuelve IPs → nombres (ej. `192.168.1.8` → `www.usmacapa.red`)

---

### 1.4 Archivo de Zona Directa (`db.usmacapa.red`)

Se copió la plantilla base y se personalizó para el dominio del grupo:

```bash
sudo cp /etc/bind/db.local /etc/bind/db.usmacapa.red
sudo nano /etc/bind/db.usmacapa.red
```

```dns
$TTL    604800
@       IN      SOA     dns.usmacapa.red. admin.usmacapa.red. (
                              1         ; Serial      — versión del archivo
                         604800         ; Refresh     — cada cuánto refresca el secundario
                          86400         ; Retry       — reintento si falla el refresh
                        2419200         ; Expire      — cuánto tiempo es válido sin contacto
                         604800 )       ; Negative Cache TTL
;
; ─── Registro de servidor de nombres ────────────────────────────
@       IN      NS      dns.usmacapa.red.

; ─── Registros A (nombre → IP) ──────────────────────────────────
dns     IN      A       192.168.1.7        ; Servidor DNS
www     IN      A       192.168.1.8        ; Servidor Web

; ─── Registro CNAME (alias) ─────────────────────────────────────
web     IN      CNAME   www.usmacapa.red.  ; web.usmacapa.red → www.usmacapa.red
```

---

### 1.5 Archivo de Zona Inversa (`db.192.168.1`)

```bash
sudo cp /etc/bind/db.local /etc/bind/db.192.168.1
sudo nano /etc/bind/db.192.168.1
```

```dns
$TTL    604800
@       IN      SOA     dns.usmacapa.red. admin.usmacapa.red. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
; ─── Servidor de nombres ─────────────────────────────────────────
@       IN      NS      dns.usmacapa.red.

; ─── Registros PTR (IP → nombre) ────────────────────────────────
7       IN      PTR     dns.usmacapa.red.   ; .7 → servidor DNS
8       IN      PTR     www.usmacapa.red.   ; .8 → servidor Web
```

---

### 1.6 Validación y Reinicio del Servicio

Antes de reiniciar, se validó la sintaxis de la configuración y los archivos de zona:

```bash
# Validar configuración global
sudo named-checkconf

# Validar zona directa
sudo named-checkzone usmacapa.red /etc/bind/db.usmacapa.red

# Validar zona inversa
sudo named-checkzone 1.168.192.in-addr.arpa /etc/bind/db.192.168.1

# Aplicar cambios
sudo systemctl restart bind9
sudo systemctl status bind9
```

![Estado de BIND9 activo](https://github.com/user-attachments/assets/fb733b60-f86b-42d3-8cae-09091afeba41)
*Captura: BIND9 en estado `active (running)` tras la configuración.*

---

### 1.7 Reglas de Firewall (UFW)

Se habilitó el firewall permitiendo únicamente los puertos necesarios:

```bash
sudo apt install ufw -y
sudo ufw allow 53     # DNS (TCP y UDP)
sudo ufw allow 22     # SSH (acceso remoto)
sudo ufw enable
```

| Puerto | Protocolo | Servicio |
|--------|-----------|----------|
| 53 | TCP/UDP | DNS (BIND9) |
| 22 | TCP | SSH |

---

## Parte 2 — Configuración de la VM Web con Nginx

### 2.1 Verificación de la Interfaz de Red

```bash
ip a
```

![Verificación de IP en VM Web](https://github.com/user-attachments/assets/4e2f4e47-435a-42a2-ba74-8feeedf39593)
*Captura: La VM Web recibió la IP `192.168.1.8`.*

---

### 2.2 Instalación de Nginx

```bash
sudo apt update
sudo apt install nginx -y
sudo systemctl enable --now nginx
```

---

### 2.3 Creación del Contenido del Sitio Web

Se creó el directorio y página de inicio para el dominio:

```bash
sudo mkdir -p /var/www/usmacapa.red
sudo nano /var/www/usmacapa.red/index.html
```

```html
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>Usmacapa — SIS313</title>
</head>
<body>
    <h1>¡Bienvenidos a usmacapa.red!</h1>
    <p>Integrantes: Luis Fernando Quispe Sulla | Alex Calatayud Mamani</p>
    <p>Servidor Web — Laboratorio 4.2 · SIS313 · Semestre 1/2026</p>
</body>
</html>
```

---

### 2.4 Configuración del Virtual Host

```bash
sudo nano /etc/nginx/sites-available/usmacapa.red
```

```nginx
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

![Configuración del Virtual Host Nginx](https://github.com/user-attachments/assets/9fe146dc-a98e-441a-900f-5afb97e81b2f)
*Captura: Archivo de Virtual Host configurado para `usmacapa.red`.*

---

### 2.5 Activación del Sitio

```bash
# Crear enlace simbólico para activar el sitio
sudo ln -s /etc/nginx/sites-available/usmacapa.red /etc/nginx/sites-enabled/

# Probar la configuración
sudo nginx -t

# Reiniciar el servicio
sudo systemctl restart nginx
```

![Nginx test y reinicio](https://github.com/user-attachments/assets/7a858683-4140-4ce0-b873-acc1fcb0dff3)
*Captura: Nginx reporta `syntax is ok` y `test is successful`.*

---

### 2.6 Apuntar el DNS de la VM Web al Servidor BIND9

Para que la VM Web pueda resolver `usmacapa.red` usando nuestro servidor DNS propio:

```bash
sudo nano /etc/systemd/resolved.conf
```

```ini
[Resolve]
DNS=192.168.1.7
Domains=usmacapa.red
MulticastDNS=no
```

```bash
sudo systemctl restart systemd-resolved
```

---

### 2.7 Prueba de Conectividad desde la VM Web

```bash
curl http://www.usmacapa.red
```

![Resultado de curl](https://github.com/user-attachments/assets/69afee30-cea5-4521-8337-be6226f53825)
*Captura: La VM Web resuelve correctamente `www.usmacapa.red` y devuelve el HTML del sitio.*

---

## Parte 3 — Pruebas desde la PC Anfitriona (Windows)

### 3.1 Configuración del DNS en Windows

Para forzar que Windows use nuestro servidor DNS en lugar del del router:

1. `Windows + R` → escribir `ncpa.cpl` → Enter
2. Clic derecho en **Wi-Fi** → **Propiedades**
3. ❌ Desmarcar **"Protocolo de Internet versión 6 (TCP/IPv6)"** *(evita interferencia del router)*
4. Seleccionar **"Protocolo de Internet versión 4 (TCP/IPv4)"** → **Propiedades**
5. Configurar:
   - **DNS preferido:** `192.168.1.7` *(nuestro BIND9)*
   - **DNS alternativo:** `8.8.8.8` *(Google, por si falla)*

![Configuración DNS en Windows](https://github.com/user-attachments/assets/667c3c8f-b0e4-4925-a59a-4bcc7b9b5f92)
*Captura: Adaptador Wi-Fi apuntando al servidor DNS del laboratorio.*

---

### 3.2 Prueba de Resolución Directa (nombre → IP)

```cmd
nslookup www.usmacapa.red
```

**Resultado esperado:**
```
Servidor:  dns.usmacapa.red
Address:   192.168.1.7

Nombre:    www.usmacapa.red
Address:   192.168.1.8
```

✅ El servidor `dns.usmacapa.red` (192.168.1.7) resolvió correctamente `www` hacia `192.168.1.8`.

---

### 3.3 Prueba de Resolución Inversa (IP → nombre)

```cmd
nslookup 192.168.1.8
```

**Resultado esperado:**
```
Servidor:  dns.usmacapa.red
Address:   192.168.1.7

Nombre:    www.usmacapa.red
Address:   192.168.1.8
```

![nslookup directo e inverso](https://github.com/user-attachments/assets/9145b5fa-c6a9-40ff-8e54-9b81064aecaa)
*Captura: Resolución directa e inversa funcionando correctamente.*

---

### 3.4 Acceso al Sitio Web desde el Navegador

Se abrió el navegador y se accedió a:

```
http://www.usmacapa.red
```

![Sitio web en el navegador](https://github.com/user-attachments/assets/52603894-b1c9-4e42-86b0-2b321657f201)
*Captura: El navegador resuelve el dominio y muestra la página del grupo correctamente.*

---

## 4. Tabla Resumen de la Configuración

| Elemento | Detalle |
|---|---|
| **Dominio** | `usmacapa.red` |
| **Servidor DNS** | VM1 — BIND9 — `192.168.1.7` |
| **Servidor Web** | VM2 — Nginx — `192.168.1.8` |
| **Registro A (DNS)** | `dns.usmacapa.red` → `192.168.1.7` |
| **Registro A (Web)** | `www.usmacapa.red` → `192.168.1.8` |
| **Registro CNAME** | `web.usmacapa.red` → `www.usmacapa.red` |
| **Registro PTR (.7)** | `192.168.1.7` → `dns.usmacapa.red` |
| **Registro PTR (.8)** | `192.168.1.8` → `www.usmacapa.red` |
| **Puertos abiertos (DNS)** | 53 (TCP/UDP), 22 (SSH) |
| **Puerto Web** | 80 (HTTP) |
| **Modo de red** | Puente (Bridge) |

---

## 5. Conclusiones

**Luis Fernando Quispe Sulla:**

- La configuración de BIND9 requiere extrema precisión en la sintaxis, especialmente en los puntos finales (`.`) de los FQDNs dentro de los archivos de zona. Un punto mal puesto puede hacer que toda la zona falle silenciosamente.
- Las herramientas `named-checkconf` y `named-checkzone` son fundamentales para depurar antes de reiniciar el servicio, ahorrando tiempo de diagnóstico.
- El modo puente (Bridge) en las VMs fue clave para que las pruebas desde la PC anfitriona fueran posibles, simulando un entorno de red física real.
- Deshabilitar IPv6 en Windows fue un paso inesperado pero necesario: el router respondía antes que nuestro DNS para consultas IPv6, enmascarando errores de configuración.

**Conclusión general del grupo:**

- Se configuró exitosamente un servidor DNS primario con BIND9, incluyendo zona directa e inversa para el dominio `usmacapa.red`.
- La integración DNS + Nginx permitió acceder al sitio web por nombre de dominio en lugar de IP, demostrando el funcionamiento real de la resolución de nombres en una red local.
- Se verificaron tanto la resolución directa como la inversa desde la PC anfitriona Windows mediante `nslookup`.
- El laboratorio refuerza el entendimiento de cómo funciona el sistema de nombres de dominio (DNS) a nivel de infraestructura.

---

*Informe elaborado por **Luis Fernando Quispe Sulla** y Alex Calatayud Mamani — SIS313 · Semestre 1/2026*
