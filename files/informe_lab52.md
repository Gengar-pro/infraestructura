# Laboratorio 5.2: Detección de Intrusiones y Respuesta a Incidentes

**Universidad San Francisco Xavier de Chuquisaca**  
**Asignatura:** Infraestructura, Plataformas Tecnológicas y Redes (SIS313)  
**Docente:** Ing. Marcelo Quispe Ortega  
**Estudiante:** Luis F.  
**Semestre:** 1/2026  
**Fecha:** 21 de Mayo de 2026  

---

## 🎯 Objetivo

Configurar un entorno de ataque/defensa con dos máquinas virtuales Ubuntu Server 24.04, implementando Fail2ban como sistema de prevención de intrusiones, simulando ataques de fuerza bruta y escaneos de puertos, analizando logs del sistema y aplicando el ciclo completo de respuesta a incidentes.

---

## 🛠️ Sección 1: Preparación del Entorno Virtual

### Arquitectura de Red

| VM | Hostname | Rol | IP Interna |
|---|---|---|---|
| Lab5.2-Server | `server` | Servidor Objetivo (Nginx, SSH, Fail2ban) | `192.168.30.2` |
| Lab5.2-Attacker | `attacker` | Máquina Atacante (Hydra, Nmap) | `192.168.30.3` |

- **Red interna:** `192.168.30.0/29`
- **Gateway:** La VM Server actúa como router/gateway para la VM Attacker

---

## 💻 Sección 2: Práctica Guiada — Ejercicios Individuales

---

### Ejercicio 1: Configuración de Red Estática

#### 1.1 Verificación de interfaces en la VM Server

Se ejecutó `ip a` para verificar las interfaces de red disponibles antes de configurar las IPs estáticas.

> 📌 **Pega la imagen 1 aquí** → `img01.png`

**Lo que muestra:** Las interfaces `enp0s3` (NAT) y `enp0s8` (Red Interna) disponibles en la VM Server, confirmando que VirtualBox asignó correctamente los dos adaptadores de red.

---

#### 1.2 Edición del archivo netplan en la VM Server

Se editó `/etc/netplan/50-cloud-init.yaml` para asignar la IP estática `192.168.30.2/29` a la interfaz `enp0s8`.

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: no
      optional: true
      addresses:
        - 192.168.30.2/29
      nameservers:
        addresses:
          - 8.8.8.8
```

> 📌 **Pega la imagen 2 aquí** → `img02.png`

**Lo que muestra:** El archivo de configuración de netplan abierto en nano con la IP estática `192.168.30.2/29` asignada a `enp0s8`.

---

#### 1.3 Aplicación de la configuración y verificación

Se aplicó la configuración con `sudo netplan apply` y se verificó con `ip a` que la IP quedó asignada correctamente.

```bash
sudo netplan apply
ip a show enp0s8
```

> 📌 **Pega la imagen 3 aquí** → `img03.png`

**Lo que muestra:** La interfaz `enp0s8` con la IP `192.168.30.2/29` activa, confirmando que netplan aplicó la configuración correctamente.

---

#### 1.4 Configuración de IP en la VM Attacker

Se configuró la IP estática `192.168.30.3/29` en la VM Attacker con la ruta por defecto apuntando al Server (`192.168.30.2`).

> 📌 **Pega la imagen 4 aquí** → `img04.png`

**Lo que muestra:** El archivo netplan de la VM Attacker con IP `192.168.30.3/29` y gateway `192.168.30.2` configurados.

---

#### 1.5 Habilitación de IP Forwarding y NAT en el Server

Se habilitó el reenvío de paquetes para que el Server actúe como router, permitiendo que la VM Attacker tenga salida a Internet.

```bash
sudo nano /etc/sysctl.conf   # net.ipv4.ip_forward=1
sudo sysctl -p
sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
sudo apt install iptables-persistent
sudo netfilter-persistent save
```

> 📌 **Pega la imagen 5 aquí** → `img05.png`

**Lo que muestra:** La confirmación de `sysctl -p` con `net.ipv4.ip_forward = 1` y la regla NAT de iptables activa.

---

#### 1.6 Verificación de conectividad entre VMs

Se ejecutó `ping 192.168.30.2` desde la VM Attacker para confirmar conectividad en la red interna.

> 📌 **Pega la imagen 6 aquí** → `img06.png`

**Lo que muestra:** Ping exitoso desde `192.168.30.3` hacia `192.168.30.2`, confirmando que la red interna `192.168.30.0/29` funciona correctamente.

---

### Ejercicio 2: Instalación y Configuración del Servidor Objetivo

#### 2.1 Instalación de Nginx y SSH

Se instalaron y habilitaron los servicios Nginx y OpenSSH Server en la VM Server.

```bash
sudo apt update
sudo apt install nginx openssh-server -y
sudo systemctl enable --now nginx
sudo systemctl enable --now sshd
```

> 📌 **Pega la imagen 7 aquí** → `img07.png`

**Lo que muestra:** La salida de `apt install` con Nginx y SSH instalados exitosamente.

---

#### 2.2 Creación del sitio web de prueba y verificación de servicios

Se creó una página web de prueba y se verificó que ambos servicios estén activos.

```bash
sudo bash -c 'echo "<h1>Servidor de Prueba - Lab 5.2</h1>" > /var/www/html/index.html'
sudo systemctl status nginx
sudo systemctl status sshd
sudo ss -tulnp | grep -E "nginx|sshd"
```

> 📌 **Pega la imagen 8 aquí** → `img08.png`

**Lo que muestra:** Estado `active (running)` de Nginx y SSH, con los puertos 22 y 80 escuchando en la interfaz de red.

---

### Ejercicio 3: Instalación y Configuración de Fail2ban

#### 3.1 Instalación de Fail2ban

Se instaló Fail2ban como sistema de prevención de intrusiones basado en host (HIPS).

```bash
sudo apt install fail2ban -y
sudo systemctl enable --now fail2ban
```

> 📌 **Pega la imagen 9 aquí** → `img09.png`

**Lo que muestra:** Instalación exitosa de Fail2ban y servicio habilitado.

---

#### 3.2 Configuración del archivo jail.local

Se creó el archivo de configuración local con las jails para SSH y Nginx.

```bash
sudo nano /etc/fail2ban/jail.local
```

```ini
[DEFAULT]
bantime  = 600
findtime = 600
maxretry = 3
backend  = systemd

[sshd]
enabled = true
port    = ssh
filter  = sshd
logpath = /var/log/auth.log
maxretry = 3

[nginx-http-auth]
enabled = true
port    = http,https
filter  = nginx-http-auth
logpath = /var/log/nginx/error.log
maxretry = 3
```

> 📌 **Pega la imagen 10 aquí** → `img10.png`

**Lo que muestra:** El archivo `jail.local` abierto en nano con las dos jails configuradas: `[sshd]` y `[nginx-http-auth]`.

---

#### 3.3 Verificación del estado de Fail2ban y sus jails

Se reinició Fail2ban y se verificó que ambas jails estén activas.

```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

> 📌 **Pega la imagen 11 aquí** → `img11.png`

**Lo que muestra:** La lista de jails activas (`sshd` y `nginx-http-auth`) con sus estadísticas: total de intentos fallidos, IPs baneadas y estado general del servicio.

---

### Ejercicio 4: Simulación de Ataque de Fuerza Bruta a SSH

#### 4.1 Instalación de Hydra y creación del diccionario de contraseñas

Desde la VM Attacker se instaló Hydra y se creó el archivo `passwords.txt` con contraseñas comunes.

```bash
sudo apt install hydra -y
nano passwords.txt
```

> 📌 **Pega la imagen 12 aquí** → `img12.png`

**Lo que muestra:** Hydra instalado y el archivo `passwords.txt` con las contraseñas de prueba listo para el ataque.

---

#### 4.2 Ejecución del ataque de fuerza bruta con Hydra

Se lanzó el ataque desde la VM Attacker contra el servicio SSH del Server.

```bash
hydra -l luisf -P passwords.txt ssh://192.168.30.2 -t 4 -V
```

> 📌 **Pega la imagen 13 aquí** → `img13.png`

**Lo que muestra:** La salida de Hydra con los intentos `[ATTEMPT]` fallidos para cada contraseña del diccionario, generando el tráfico malicioso que Fail2ban detectará.

---

#### 4.3 Verificación de intentos fallidos en auth.log

Desde el Server se analizaron los registros del sistema para confirmar que el ataque fue registrado.

```bash
sudo grep "Failed password" /var/log/auth.log
sudo grep "Failed password" /var/log/auth.log | wc -l
sudo grep "Failed password" /var/log/auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -nr
```

> 📌 **Pega la imagen 14 aquí** → `img14.png`

**Lo que muestra:** Los registros de intentos fallidos en `auth.log` con timestamps, IP origen (`192.168.30.3`) y puertos utilizados por Hydra en cada intento.

---

#### 4.4 Verificación del bloqueo automático por Fail2ban

Se verificó que Fail2ban detectó los intentos y baneó automáticamente la IP atacante.

```bash
sudo fail2ban-client status sshd
sudo iptables -L -n | grep f2b
```

> 📌 **Pega la imagen 15 aquí** → `img15.png`

**Lo que muestra:** La IP `192.168.30.3` en la "Banned IP list" de Fail2ban y la regla de firewall generada automáticamente en iptables, confirmando el bloqueo efectivo.

---

#### 4.5 Confirmación del bloqueo desde la VM Attacker

Se intentó conectar por SSH desde la VM Attacker después del baneo para confirmar que el bloqueo es efectivo.

```bash
ssh luisf@192.168.30.2
```

> 📌 **Pega la imagen 16 aquí** → `img16.png`

**Lo que muestra:** La conexión SSH desde `192.168.30.3` queda bloqueada/colgada, confirmando que la regla de iptables de Fail2ban está funcionando correctamente.

---

#### 4.6 Desbaneo de la IP para continuar las prácticas

Una vez confirmado el bloqueo, se desbaneó la IP para continuar con los siguientes ejercicios.

```bash
sudo fail2ban-client set sshd unbanip 192.168.30.3
```

> 📌 **Pega la imagen 17 aquí** → `img17.png`

**Lo que muestra:** Confirmación del desbaneo exitoso con respuesta `1` de Fail2ban, dejando la IP nuevamente activa para la siguiente fase del laboratorio.

---

### Ejercicio 5: Simulación de Escaneo y Reconocimiento Web

#### 5.1 Escaneo de puertos con Nmap

Desde la VM Attacker se realizó un escaneo de servicios contra el Server para identificar puertos abiertos y versiones.

```bash
sudo apt install nmap -y
nmap -sV 192.168.30.2
```

> 📌 **Pega la imagen 18 aquí** → `img18.png`

**Lo que muestra:** El resultado del escaneo Nmap con los puertos **22/tcp (SSH - OpenSSH)** y **80/tcp (HTTP - Nginx)** abiertos, con sus versiones detectadas.

---

#### 5.2 Reconocimiento web automatizado con curl

Se generaron 50 peticiones a rutas inexistentes para simular un escaneo de directorios web.

```bash
for i in $(seq 1 50); do
  curl -s -o /dev/null -w "%{http_code}" http://192.168.30.2/admin$i
  echo " -> /admin$i"
done
```

> 📌 **Pega la imagen 19 aquí** → `img19.png`

**Lo que muestra:** Las 50 peticiones generando código `404` para cada ruta `/admin1` a `/admin50`, simulando el comportamiento de una herramienta de escaneo de directorios web.

---

#### 5.3 Análisis del access.log de Nginx

Desde el Server se analizaron los logs de acceso de Nginx para detectar el patrón de escaneo.

```bash
sudo grep " 404 " /var/log/nginx/access.log | head -20
sudo grep " 404 " /var/log/nginx/access.log | awk '{print $1}' | sort | uniq -c | sort -nr
```

> 📌 **Pega la imagen 20 aquí** → `img20.png`

**Lo que muestra:** Las 54 peticiones 404 registradas desde `192.168.30.3` (4 del Nmap Scripting Engine + 50 del loop curl), evidenciando claramente el patrón de reconocimiento automatizado.

---

### Ejercicio 6: Ciclo de Respuesta a Incidentes

---

#### FASE 1 — IDENTIFICAR

Se usaron comandos de análisis forense para identificar el origen, tipo y alcance del ataque.

```bash
sudo lastb | head -20
sudo grep "Failed password" /var/log/auth.log | tail -20
sudo grep " 404 " /var/log/nginx/access.log | awk '{print $1}' | sort | uniq -c | sort -nr | head -10
```

> 📌 **Pega la imagen 21 aquí** → `img21.png`

**Lo que muestra:** Los tres comandos de identificación: `lastb` con los intentos de login fallidos registrados en el sistema, los últimos registros de contraseñas fallidas en `auth.log` con timestamps, y el conteo de 54 peticiones 404 desde la IP atacante.

**Conclusión de identificación:**
- **Servicios atacados:** SSH (`sshd`) y HTTP (`nginx`)
- **IP atacante:** `192.168.30.3`
- **Tipo de ataque:** Fuerza bruta SSH + reconocimiento web automatizado (404 scan)

---

#### FASE 2 — CONTENER

Fail2ban aplicó la contención automática. Se intentó aplicar bloqueo adicional con UFW.

```bash
sudo ufw deny from 192.168.30.3
```

> 📌 **Pega la imagen 22 aquí** → `img22.png`

**Lo que muestra:** El mensaje de error indicando que UFW no está instalado en el servidor. Esto no afecta la seguridad ya que Fail2ban aplicó las reglas directamente en iptables, que es el backend subyacente de UFW.

---

#### FASE 3 — ERRADICAR

Se verificó que no hubo ningún acceso exitoso desde la IP atacante.

```bash
sudo grep "Accepted" /var/log/auth.log
```

> 📌 **Pega la imagen 23 aquí** → `img23.png`

**Lo que muestra:** Los únicos accesos aceptados provienen de `10.0.2.2` (PC anfitriona vía NAT port forwarding), confirmando que **ningún acceso exitoso** ocurrió desde la IP atacante `192.168.30.3`. El sistema no fue comprometido.

---

#### FASE 4 — RECUPERAR

Se verificó que los tres servicios críticos siguen funcionando correctamente después del ataque.

```bash
sudo systemctl status sshd nginx fail2ban
```

> 📌 **Pega la imagen 24 aquí** → `img24.png`

**Lo que muestra:** Los tres servicios en estado `active (running)`: SSH activo desde las 15:57 UTC, Nginx desde las 16:13 UTC y Fail2ban desde las 16:23 UTC, confirmando que el servidor operó con normalidad durante y después del ataque.

---

#### FASE 5 — DOCUMENTAR

Se creó el archivo de incidente oficial en el servidor.

```bash
nano ~/incidente_001.md
```

```markdown
## Incidente #001 - Lab 5.2

- **Fecha:** 21 de Mayo de 2026
- **Hora de detección:** 16:28 UTC
- **IP Atacante:** 192.168.30.3
- **Servicio Afectado:** SSH (sshd) y HTTP (nginx)
- **Tipo de Ataque:** Fuerza bruta SSH + reconocimiento web (404 scan)
- **Intentos Fallidos SSH:** 8
- **Peticiones 404 Web:** 54
- **Accesos Exitosos desde IP Atacante:** Ninguno
- **Estado:** Resuelto
```

---

## 📊 Análisis de Resultados

### Resumen del Incidente #001

| Campo | Detalle |
|---|---|
| **Fecha** | 21 de Mayo de 2026 |
| **Hora de detección** | 16:28 UTC |
| **IP Atacante** | `192.168.30.3` |
| **Servicios afectados** | SSH (`sshd`) y HTTP (`nginx`) |
| **Tipo de ataque** | Fuerza bruta SSH + reconocimiento web (404 scan) |
| **Intentos fallidos SSH** | 8 registrados en `auth.log` |
| **Peticiones 404 web** | 54 (4 Nmap NSE + 50 curl) |
| **Accesos exitosos desde atacante** | **Ninguno** |
| **Respuesta automática** | Fail2ban baneó `192.168.30.3` tras 3 intentos fallidos |
| **Estado final** | ✅ Resuelto |

### Indicadores de Compromiso (IOC) Identificados

- Múltiples intentos de autenticación SSH fallidos en menos de 10 segundos desde una misma IP
- 4 peticiones del Nmap Scripting Engine a rutas características de vulnerabilidades conocidas (`/HNAP1`, `/sdk`, `/evox/about`)
- 50 peticiones 404 consecutivas con patrón incremental (`/admin1` a `/admin50`) en menos de 2 segundos

### Efectividad de Fail2ban

Fail2ban demostró ser altamente efectivo al detectar y bloquear automáticamente el ataque de fuerza bruta antes de que pudiera completarse. Con `maxretry = 3` y `findtime = 600`, la IP atacante fue baneada después del tercer intento fallido, en los primeros segundos del ataque.

---

## ✅ Conclusiones

1. **Fail2ban como HIPS:** La configuración de Fail2ban con las jails `sshd` y `nginx-http-auth` demostró ser efectiva para detectar y bloquear ataques automatizados de fuerza bruta en tiempo real.

2. **Importancia del análisis de logs:** Los archivos `auth.log` y `access.log` son fuentes de información crítica para detectar patrones de ataque. El análisis de logs permitió identificar la IP atacante, el tipo de ataque y el alcance del mismo.

3. **Ciclo de respuesta a incidentes:** Aplicar las fases de identificar, contener, erradicar, recuperar y documentar permite una respuesta estructurada y ordenada ante incidentes de seguridad, minimizando el tiempo de exposición y el impacto.

4. **Segregación de red:** La arquitectura de red interna aislada (`192.168.30.0/29`) permitió simular un entorno realista de ataque/defensa sin riesgo para redes externas.

5. **Integridad del servidor:** A pesar del ataque simulado, ninguna credencial fue comprometida y todos los servicios continuaron operando normalmente, validando la efectividad de las medidas de seguridad implementadas.

---

*Informe elaborado por Luis F. — SIS313 — Universidad San Francisco Xavier de Chuquisaca — 21/05/2026*
