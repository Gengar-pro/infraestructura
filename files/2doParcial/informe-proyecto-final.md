# Proyecto Final — Marketplace Local
## Infraestructura de Alta Disponibilidad, RAID 10, Fail2ban y Monitoreo
### SIS313 · Infraestructura, Plataformas Tecnológicas y Redes · USFX · 2026

---

<div align="center">

| | |
|---|---|
| **Asignatura** | Infraestructura, Plataformas Tecnológicas y Redes (SIS313) |
| **Docente** | Ing. Marcelo Quispe Ortega |
| **Semestre** | 1/2026 |
| **Proyecto** | N° 16 — Marketplace Local con Monitoreo y Escalabilidad |

</div>

---

## Integrantes del Grupo 2

| # | Nombre Completo | Carrera | Rol en el Proyecto |
|---|---|---|---|
| 1 | **Calatayud Mamani Alex Josué** | Ingeniería de Sistemas | Aplicación 1 + Script de Health Check |
| 2 | **Chambi Condori Janet** | Ingeniería de Sistemas | Proxy + TLS + Monitoreo + Fail2ban |
| 3 | **Quispe Sullca Luis Fernando** | Ing. en Ciencias de la Computación | Base de Datos + RAID 10 + Backups |
| 4 | **Quispe Anagua Jhon Christian** | Ingeniería de Sistemas | Aplicación 2 + Menú Interactivo + VM Atacante |

---

## Diagrama de Arquitectura

![Diagrama de arquitectura del Marketplace](img/diagrama-arquitectura.png)

---

## Temas Avanzados Integrados

| Código | Tema | Responsable |
|---|---|---|
| T2 | Almacenamiento RAID 10 y tolerancia a fallos | Luis Fernando |
| T4 | Proxy inverso y balanceo de carga (Nginx) | Janet |
| T8 | Despliegue de aplicaciones (Node.js + PM2) | Alex + Jhon Christian |
| T9 | Bases de datos centralizadas (MariaDB) | Luis Fernando |
| T10 | Monitoreo integral (Prometheus + Grafana) | Janet |
| T12 | Seguridad TLS/SSL (certificados, HSTS, cifrados) | Janet |
| T13 | Detección de intrusiones (Fail2ban) | Janet + Jhon Christian |
| T14 | Automatización con Bash (health check, menú) | Alex + Jhon Christian |
| T15 | Backups automatizados y recuperación | Luis Fernando |

---

## Arquitectura del Sistema

Todas las VMs están en VirtualBox con **Adaptador Puente (Bridge)** conectadas al mismo WiFi. Las IPs son asignadas por DHCP.

| VM | Responsable | IP | Servicios |
|---|---|---|---|
| Proxy | Janet | 192.168.1.15 | Nginx, TLS, Fail2ban, Prometheus, Grafana |
| App1 | Alex Josué | 192.168.1.X | Node.js, PM2, Health Check, Node Exporter |
| App2 | Jhon Christian | 192.168.1.X | Node.js, PM2, Menú Interactivo, Node Exporter |
| DB | Luis Fernando | 192.168.1.X | MariaDB, RAID 10, Backups, UFW, Node Exporter |
| Atacante | Jhon Christian | 192.168.1.X | Hydra, Nmap |

### Flujo de Tráfico

```
Usuario (navegador)
       ↓ HTTPS (puerto 443)
[VM Proxy — Nginx + TLS + Fail2ban]
       ↓ Round Robin
[VM App1 — Node.js:3000]    [VM App2 — Node.js:3000]
       ↓                          ↓
       └──────────┬───────────────┘
                  ↓
[VM DB — MariaDB:3306 sobre RAID 10]

[Prometheus:9090] ← Node Exporter:9100 (todas las VMs)
       ↓
[Grafana:3000 — Dashboards + Alertas]

[VM Atacante — Hydra + Nmap] → ataca → [VM Proxy] → Fail2ban bloquea
```

---
---

# VM PROXY — Janet Chambi Condori

**Servicios:** Nginx + TLS/HTTPS + Fail2ban + Prometheus + Grafana + Node Exporter

---

## 1. Configuración de Red

Se configuró el adaptador de red en modo Puente con DHCP para obtener una IP del router WiFi.

**Netplan:**

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
```

![Configuración del netplan en VM Proxy](img/proxy/01-netplan-config.png)

**Aplicar configuración y verificar:**

```bash
sudo chmod 600 /etc/netplan/50-cloud-init.yaml
sudo netplan apply
```

![Comandos de netplan apply](img/proxy/02-netplan-apply.png)

**Verificar IP asignada:**

```bash
ip addr show enp0s3
```

![IP asignada por DHCP al Proxy](img/proxy/03-ip-addr.png)

**Verificar conectividad a internet:**

```bash
ping -c 3 google.com
```

![Ping exitoso a google.com](img/proxy/04-ping-google.png)

---

## 2. Instalación de Nginx

```bash
sudo apt update
sudo apt install nginx -y
```

![Instalación de Nginx con apt](img/proxy/05-nginx-install.png)

**Verificar estado:**

```bash
sudo systemctl status nginx
```

![Nginx activo y corriendo](img/proxy/06-nginx-status.png)

---

## 3. Instalación de Prometheus y Node Exporter

```bash
sudo apt install prometheus prometheus-node-exporter -y
sudo systemctl start prometheus prometheus-node-exporter
sudo systemctl status prometheus --no-pager
curl http://localhost:9100/metrics | head -5
```

![Prometheus activo y métricas de Node Exporter](img/proxy/07-prometheus-status.png)

---

## 4. Instalación de Grafana

```bash
sudo apt-get install -y apt-transport-https software-properties-common wget
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
sudo apt update && sudo apt install grafana -y
sudo systemctl daemon-reload
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
sudo systemctl status grafana-server --no-pager
```

![Grafana instalado y activo](img/proxy/08-grafana-status.png)

---

## 5. Generación del Certificado TLS Autofirmado

```bash
sudo mkdir -p /etc/nginx/ssl
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/nginx-selfsigned.key \
  -out /etc/nginx/ssl/nginx-selfsigned.crt \
  -subj "/C=BO/ST=Chuquisaca/L=Sucre/O=USFX/OU=SIS313/CN=marketplace.local"
```

![Generación del certificado TLS](img/proxy/09-tls-generacion.png)

**Verificar certificado:**

```bash
sudo openssl x509 -in /etc/nginx/ssl/nginx-selfsigned.crt -text -noout | grep -E "Subject:|Not Before|Not After"
```

![Verificación del certificado — Subject, fechas de validez](img/proxy/10-tls-verificacion.png)

---

## 6. Instalación y Configuración de Fail2ban

```bash
sudo apt install fail2ban -y
```

![Instalación de Fail2ban](img/proxy/11-fail2ban-install.png)

**Configuración de jails:**

```bash
sudo nano /etc/fail2ban/jail.local
```

```ini
[DEFAULT]
bantime = 600
findtime = 600
maxretry = 3
backend = systemd

[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3

[nginx-http-auth]
enabled = true
port = http,https
filter = nginx-http-auth
logpath = /var/log/nginx/error.log
maxretry = 3

[nginx-botsearch]
enabled = true
port = http,https
filter = nginx-botsearch
logpath = /var/log/nginx/access.log
maxretry = 5
```

![Configuración de jail.local con 3 jails](img/proxy/12-fail2ban-config.png)

**Reiniciar y verificar:**

```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status
```

![Fail2ban activo con 3 jails](img/proxy/13-fail2ban-status.png)

---

## 7. Configuración de Nginx con HTTPS + Balanceo de Carga

```bash
sudo nano /etc/nginx/sites-available/default
```

```nginx
# Redirección HTTP a HTTPS
server {
    listen 80;
    server_name _;
    return 301 https://$host$request_uri;
}

# Balanceo HTTPS
upstream loadbalancer {
    server IP_APP1:3000;
    server IP_APP2:3000;
}

server {
    listen 443 ssl;
    server_name _;

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
        proxy_pass http://loadbalancer;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

![Configuración de Nginx con TLS + balanceo + cabeceras de seguridad](img/proxy/14-nginx-config-tls.png)

**Verificar sintaxis:**

```bash
sudo nginx -t
```

![nginx -t — syntax is ok](img/proxy/15-nginx-test.png)

---

## 8. Configuración de Prometheus (Targets)

```bash
sudo nano /etc/prometheus/prometheus.yml
```

```yaml
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  - job_name: 'node-proxy'
    static_configs:
      - targets: ['localhost:9100']
  - job_name: 'node-app1'
    static_configs:
      - targets: ['IP_APP1:9100']
  - job_name: 'node-app2'
    static_configs:
      - targets: ['IP_APP2:9100']
  - job_name: 'node-db'
    static_configs:
      - targets: ['IP_DB:9100']
```

![Configuración de prometheus.yml con 5 targets](img/proxy/16-prometheus-targets.png)

---

## 9. Verificaciones del Proxy

**Verificar HTTPS:**

```bash
curl -k https://localhost
```

**Verificar redirección HTTP → HTTPS:**

```bash
curl -I http://localhost
```

**Verificar cabeceras de seguridad:**

```bash
curl -k -I https://localhost
```

**Verificar protocolo TLS:**

```bash
openssl s_client -connect localhost:443 -tls1_2 </dev/null 2>/dev/null | grep "Protocol"
```

![Verificaciones de TLS, redirección y cabeceras](img/proxy/17-verificaciones-tls.png)

**Verificar Fail2ban:**

```bash
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

![Estado de Fail2ban — 3 jails activos](img/proxy/18-fail2ban-verificacion.png)

![Estado del jail sshd](img/proxy/19-fail2ban-sshd.png)

---

## 10. Verificación del Balanceo de Carga

```bash
for i in {1..6}; do
    curl -sk https://localhost/ | grep -o '"hostname":"[^"]*"'
    sleep 0.5
done
```

![Balanceo alternando entre App1 y App2](img/proxy/20-balanceo-verificacion.png)

---

## 11. Dashboard de Grafana

Se configuró Grafana con el datasource de Prometheus y se importó el dashboard Node Exporter Full (ID: 1860) para monitorear las 4 VMs en tiempo real.

Acceso: `http://IP_PROXY:3000`

![Dashboard de Grafana mostrando métricas del Proxy — CPU, memoria, disco, red](img/proxy/21-grafana-dashboard.png)

---
---

# VM APP1 — Calatayud Mamani Alex Josué

**Servicios:** Node.js + PM2 + Health Check Script + Cron + Node Exporter

---

## 1. Instalación de Node.js con NVM

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
\. "$HOME/.nvm/nvm.sh"
nvm install 22
node -v
npm -v
```

![Instalación de NVM y Node.js](img/app1/01-nodejs-install.png)

---

## 2. Instalación de PM2

```bash
npm install pm2@latest -g
pm2 --version
```

---

## 3. Clonar y Configurar la Aplicación

```bash
mkdir ~/marketplace && cd ~/marketplace
git clone https://github.com/marceloquispeortega/api-restful-crud-movies .
npm install
```

**Configurar variables de entorno:**

```bash
nano .env
```

```env
PORT=3000
DB_HOST=IP_DB
DB_PORT=3306
DB_NAME=db_movies
DB_USER=alex
DB_PASSWORD=secret
```

---

## 4. Lanzar con PM2

```bash
pm2 start app.js --name app1
pm2 status
curl http://localhost:3000
```

![PM2 start y curl mostrando respuesta del marketplace](img/app1/02-pm2-start.png)

**Configurar auto-arranque:**

```bash
pm2 startup
pm2 save
```

---

## 5. Script de Health Check

```bash
mkdir -p ~/scripts
nano ~/scripts/health_check.sh
```

El script verifica el estado de Nginx (Proxy), App1 local (PM2), App2 remota, MariaDB, disco y memoria. Si algún servicio está caído, intenta reiniciarlo automáticamente.

![Código del script health_check.sh](img/app1/03-healthcheck-script.png)

**Ejecución del health check:**

```bash
chmod +x ~/scripts/health_check.sh
~/scripts/health_check.sh
```

![Health check ejecutado — todos los servicios OK](img/app1/04-healthcheck-ejecucion.png)

---

## 6. Programar Cron (cada 5 minutos)

```bash
crontab -e
# Agregar:
*/5 * * * * /home/alex/scripts/health_check.sh
```

```bash
crontab -l
```

![Crontab configurado para health check cada 5 minutos](img/app1/05-crontab.png)

---

## 7. Instalar Node Exporter

```bash
sudo apt install prometheus-node-exporter -y
sudo systemctl enable --now prometheus-node-exporter
```

---

## 8. Verificación Completa

```bash
pm2 status
pm2 logs --lines 10
curl http://localhost:3000/movies
```

![Verificación completa de App1 — PM2 online, logs, curl](img/app1/06-verificacion.png)

---
---

# VM APP2 — Quispe Anagua Jhon Christian

**Servicios:** Node.js + PM2 + Menú Interactivo + Node Exporter

---

## 1. Instalación de Node.js con NVM

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
\. "$HOME/.nvm/nvm.sh"
nvm install 22
```

![Instalación de NVM y Node.js en App2](img/app2/01-nodejs-install.png)

---

## 2. Instalación de PM2

```bash
npm install pm2@latest -g
```

![PM2 instalado en App2](img/app2/02-pm2-install.png)

---

## 3. Clonar la Aplicación

```bash
mkdir ~/apps && cd ~/apps
git clone https://github.com/marceloquispeortega/api-restful-crud-movies app1
cd ~/apps/app1 && npm install
```

![Git clone y npm install](img/app2/03-git-clone.png)

---

## 4. Configurar Variables de Entorno

```bash
nano ~/apps/app1/.env
```

```env
PORT=3000
DB_HOST=IP_DB
DB_PORT=3306
DB_NAME=db_movies
DB_USER=jhon
DB_PASSWORD=secret
```

![Archivo .env configurado](img/app2/04-env-config.png)

---

## 5. Lanzar con PM2

```bash
cd ~/apps/app1 && pm2 start app.js --name app2
pm2 status
```

![PM2 status mostrando app2 online](img/app2/05-pm2-start.png)

**Configurar auto-arranque:**

```bash
pm2 startup
pm2 save
```

![PM2 startup configurado](img/app2/06-pm2-startup.png)

---

## 6. Instalar Node Exporter

```bash
sudo apt install prometheus-node-exporter -y
sudo systemctl enable --now prometheus-node-exporter
```

![Node Exporter activo en App2](img/app2/07-node-exporter.png)

---

## 7. Menú Interactivo de Administración

Se creó un menú bash con 10 opciones que permite monitorear el estado de todos los servicios, verificar el balanceo de carga, ver datos de la base de datos y simular ataques controlados.

```bash
nano ~/scripts/admin_menu.sh
chmod +x ~/scripts/admin_menu.sh
```

![Menú interactivo con colores mostrando las 10 opciones](img/app2/08-menu-interactivo.png)

---
---

# VM DB — Quispe Sullca Luis Fernando

**Servicios:** MariaDB (sobre RAID 10) + Backups automáticos + UFW + Node Exporter

---

## 1. Instalación de MariaDB

```bash
sudo apt update
sudo apt install mariadb-server -y
sudo systemctl enable --now mariadb
```

![Instalación de MariaDB Server](img/db/01-mariadb-install.png)

---

## 2. Hardening de MariaDB

```bash
sudo mysql_secure_installation
```

Se configuró: contraseña root, eliminación de usuarios anónimos, deshabilitar root remoto, eliminar base de datos de prueba.

![mysql_secure_installation completado](img/db/02-mysql-secure.png)

---

## 3. Instalación de mdadm

```bash
sudo apt install mdadm -y
```

![Instalación de mdadm para RAID](img/db/03-mdadm-install.png)

---

## 4. Creación del RAID 10

Se agregaron 4 discos virtuales de 2 GB en VirtualBox y se creó el arreglo RAID 10:

```bash
sudo mdadm --create --verbose /dev/md10 --level=10 --raid-devices=4 /dev/sdb /dev/sdc /dev/sdd /dev/sde
```

![Creación del RAID 10 con mdadm](img/db/04-raid10-create.png)

**Verificación del RAID:**

```bash
cat /proc/mdstat
sudo mdadm --detail /dev/md10
```

![Estado del RAID 10 — 4 discos activos, sync completado](img/db/05-raid10-verify.png)

---

## 5. Formateo, Montaje y Persistencia

```bash
sudo mkfs.ext4 /dev/md10
sudo mkdir -p /mnt/raid10
sudo mount /dev/md10 /mnt/raid10
```

![Formateo ext4 y montaje del RAID](img/db/06-raid10-mount.png)

**Configuración de fstab y guardado del RAID:**

```bash
sudo blkid /dev/md10
sudo nano /etc/fstab
# UUID=TU_UUID  /mnt/raid10  ext4  defaults,nofail  0  2

sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
sudo update-initramfs -u
sudo systemctl daemon-reload
sudo mount -a
```

![Configuración de fstab y mdadm.conf persistente](img/db/07-fstab-config.png)

---

## 6. Mover MariaDB al RAID 10

```bash
sudo systemctl stop mariadb
sudo rsync -av /var/lib/mysql/ /mnt/raid10/mysql/
sudo chown -R mysql:mysql /mnt/raid10/mysql/
```

**Cambiar datadir en la configuración:**

```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
# datadir = /mnt/raid10/mysql
```

![Cambio de datadir a /mnt/raid10/mysql](img/db/08-datadir-change.png)

**Configurar AppArmor y reiniciar:**

```bash
echo "alias /var/lib/mysql/ -> /mnt/raid10/mysql/," | sudo tee -a /etc/apparmor.d/tunables/alias
sudo systemctl restart apparmor
sudo systemctl start mariadb
sudo systemctl status mariadb
```

![AppArmor configurado y MariaDB corriendo sobre RAID](img/db/09-apparmor-mariadb.png)

---

## 7. Cambiar bind-address

```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
# bind-address = IP_WIFI_DB
sudo systemctl restart mariadb
```

![bind-address configurado y MariaDB activo](img/db/10-bind-address.png)

![Estado de MariaDB tras cambio de bind-address](img/db/11-mariadb-status.png)

---

## 8. Verificación de persistencia tras reinicio

```bash
sudo reboot
# Después del reinicio:
cat /proc/mdstat
df -h /mnt/raid10
sudo systemctl status mariadb
```

![RAID 10 y MariaDB persisten después del reinicio](img/db/12-persistencia-reboot.png)

---

## 9. Configuración de UFW

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from IP_APP1 to any port 3306
sudo ufw allow from IP_APP2 to any port 3306
sudo ufw allow 22
sudo ufw allow 9100
sudo ufw enable
sudo ufw status numbered
```

![Reglas UFW — solo App1 y App2 pueden acceder al puerto 3306](img/db/13-ufw-config.png)

---

## 10. Node Exporter + Script de Backup

```bash
sudo apt install prometheus-node-exporter -y
sudo systemctl enable --now prometheus-node-exporter
```

**Script de backup automático:**

```bash
sudo nano /opt/backup_db.sh
```

El script ejecuta `mysqldump` para respaldar la base de datos, comprime el archivo con `gzip`, y realiza rotación eliminando backups de más de 7 días.

![Node Exporter activo y ejecución del script de backup](img/db/14-node-exporter-backup.png)

---

## 11. Script de Backup (`backup_db.sh`)

```bash
#!/bin/bash
BACKUP_DIR="/mnt/raid10/backups"
DB_NAME="db_movies"
TIMESTAMP=$(date '+%Y%m%d_%H%M%S')
BACKUP_FILE="$BACKUP_DIR/${DB_NAME}_${TIMESTAMP}.sql"
DAYS_TO_KEEP=7

sudo mysqldump -u root $DB_NAME > $BACKUP_FILE
if [ $? -eq 0 ]; then
    gzip $BACKUP_FILE
    echo "$(date) [OK] Backup: ${BACKUP_FILE}.gz" | sudo tee -a /var/log/backup_db.log
else
    echo "$(date) [ERROR] Falló backup" | sudo tee -a /var/log/backup_db.log
fi

find $BACKUP_DIR -name "*.sql.gz" -mtime +$DAYS_TO_KEEP -delete
```

![Contenido del script backup_db.sh](img/db/15-backup-script.png)

---

## 12. Script de Restauración (`restore_db.sh`)

El script lista los backups disponibles, solicita confirmación y restaura la base de datos desde el archivo seleccionado.

![Script de restauración y archivos en /opt/](img/db/16-restore-script.png)

---

## 13. Configuración de Cron

```bash
sudo crontab -e
# 0 * * * * /opt/backup_db.sh >> /var/log/backup_db.log 2>&1
```

![Crontab configurado para backup cada hora](img/db/17-crontab-config.png)

![Tarea cron programada](img/db/18-cron-programado.png)

---

## 14. Verificación Final — Datos y Backups

```bash
sudo mysql -u root -p -e "USE db_movies; SELECT * FROM products;"
ls -lh /mnt/raid10/backups/
sudo ufw status numbered
```

![Verificación de datos, backups y UFW](img/db/19-verificacion-datos.png)

![Tabla products con 10 productos del marketplace](img/db/20-products-tabla.png)

---
---

# VM ATACANTE — Simulación de Ataques Controlados

**Responsable:** Quispe Anagua Jhon Christian  
**Herramientas:** Hydra (fuerza bruta) + Nmap (escaneo de puertos)

---

## 1. Ataque de Fuerza Bruta con Hydra

Se simuló un ataque de fuerza bruta contra el servicio SSH del Proxy utilizando un diccionario de 10 contraseñas:

```bash
hydra -l janet -P passwords.txt ssh://172.24.199.154 -t 4 -V
```

El ataque intentó 10 contraseñas y no encontró ninguna válida. Fail2ban detectó los intentos fallidos y bloqueó automáticamente la IP del atacante.

---

## 2. Escaneo de Puertos con Nmap

```bash
nmap -sV 172.24.199.154
```

El escaneo identificó los puertos abiertos del Proxy: SSH (22), HTTP (80), HTTPS (443), Grafana (3000), Prometheus (9090) y Node Exporter (9100).

---

## 3. Reconocimiento Web

```bash
for i in $(seq 1 50); do
    curl -sk -o /dev/null -w "%{http_code}" https://172.24.199.154/admin$i
    echo " -> /admin$i"
done
```

Se generaron 50 peticiones a rutas inexistentes para simular un escaneo web automatizado, generando códigos 404 que quedan registrados en los logs de Nginx.

![Ataque Hydra + escaneo Nmap + reconocimiento web](img/atacante/01-hydra-nmap-recon.png)

---

## 4. Escaneo Completo de la Red

```bash
nmap -sV 172.24.199.0/24
```

El escaneo de toda la subred identificó las 5 VMs del proyecto con sus servicios activos.

![Escaneo Nmap de toda la red — 5 hosts detectados](img/atacante/02-nmap-red-completa.png)

---

## 5. Verificación del Bloqueo (desde el Proxy)

Desde la VM Proxy, Janet verificó que Fail2ban bloqueó la IP del atacante:

```bash
sudo fail2ban-client status sshd
sudo iptables -L -n | grep fail2ban
```

La IP del atacante aparece en la lista de IPs baneadas, confirmando que el sistema de detección de intrusiones funciona correctamente.

---
---

# Resultados y Análisis

## Resumen de Servicios Implementados

| Servicio | VM | Puerto | Estado |
|---|---|---|---|
| Nginx (Proxy + TLS) | Proxy | 80, 443 | ✅ Activo |
| Fail2ban (3 jails) | Proxy | — | ✅ Activo |
| Prometheus | Proxy | 9090 | ✅ Activo |
| Grafana | Proxy | 3000 | ✅ Activo |
| Node.js App1 (PM2) | App1 | 3000 | ✅ Activo |
| Health Check (cron) | App1 | — | ✅ Cada 5 min |
| Node.js App2 (PM2) | App2 | 3000 | ✅ Activo |
| MariaDB (sobre RAID 10) | DB | 3306 | ✅ Activo |
| RAID 10 (4 discos) | DB | — | ✅ Activo |
| Backups automáticos | DB | — | ✅ Cada hora |
| UFW | DB | — | ✅ Activo |
| Node Exporter | Todas | 9100 | ✅ Activo (x4) |

## Failover Demostrado

Al detener App1 con `pm2 stop app1`, Nginx redirigió automáticamente el 100% del tráfico a App2 sin interrupción del servicio. Al restaurar App1 con `pm2 restart app1`, el balanceo volvió a funcionar con normalidad.

## RAID 10 Demostrado

Al marcar un disco como fallido con `mdadm --manage /dev/md10 --fail /dev/sdb`, el arreglo continuó operando en modo degradado y los datos permanecieron accesibles. Esto demuestra la tolerancia a fallos del RAID 10.

## Fail2ban Demostrado

Al ejecutar un ataque de fuerza bruta con Hydra desde la VM Atacante, Fail2ban detectó los intentos fallidos de autenticación y bloqueó automáticamente la IP del atacante mediante reglas de iptables, demostrando la protección activa contra intrusiones.

---

# Conclusiones

El proyecto demostró exitosamente la implementación de una infraestructura de alta disponibilidad para un marketplace local, integrando 9 temas avanzados de la asignatura. La arquitectura de múltiples capas (proxy, aplicación, datos, monitoreo y seguridad) refleja las mejores prácticas utilizadas en entornos de producción reales.

La combinación de RAID 10 para tolerancia a fallos, Fail2ban para detección de intrusiones, TLS para cifrado de tráfico, backups automáticos con cron y monitoreo integral con Prometheus y Grafana proporciona una infraestructura robusta, segura y observable que podría soportar un marketplace real de comerciantes locales.

---

*Proyecto Final elaborado por el Grupo 2 — SIS313 · USFX · Mayo 2026*
