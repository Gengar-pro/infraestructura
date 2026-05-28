# 🚀 Guía de Implementación — Proyecto Final SIS313
## Marketplace Local · VirtualBox · Adaptador Puente
### Grupo 2 · Mayo 2026

---

## 📋 Distribución de VMs

Cada integrante tiene su(s) VM(s) en su propia PC. Todas conectadas en **Adaptador Puente (Bridge)** al mismo WiFi.

| Integrante | PC | VM(s) | Rol |
|---|---|---|---|
| Janet Chambi Condori | PC Janet | VM Proxy | Proxy + TLS + Fail2ban + Monitoreo |
| Calatayud Mamani Alex Josué | PC Alex | VM App1 | Aplicación 1 + Health Check |
| Quispe Anagua Jhon Christian | PC Jhon | VM App2 + VM Atacante | Aplicación 2 + Menú + Ataque |
| Quispe Sullca Luis Fernando | PC Luis | VM DB | Base de Datos + RAID 10 + Backups |

---

## ⚙️ CONFIGURACIÓN INICIAL (TODOS)

Cada integrante debe hacer esto en su VM antes de juntarse.

### Paso 1 — Configurar Adaptador Puente en VirtualBox

1. Apagar la VM
2. **Configuración → Red → Adaptador 1**
3. Conectar a: **Adaptador puente**
4. Nombre: Seleccionar tu **adaptador WiFi** (ej: Realtek, Intel Wireless)
5. Modo promiscuo: **Permitir todo**
6. Aceptar y encender la VM

### Paso 2 — Configurar red con DHCP

Dentro de la VM:

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
```

```bash
sudo chmod 600 /etc/netplan/50-cloud-init.yaml
sudo netplan apply
```

### Paso 3 — Ver tu IP asignada

```bash
ip addr show enp0s3
```

Anota tu IP (ej: `192.168.1.10`). La necesitarás para compartir con el grupo.

### Paso 4 — Verificar internet

```bash
ping -c 3 google.com
```

---

## 🔗 CUANDO SE JUNTEN — Compartir IPs

Todos se conectan al **mismo WiFi** y cada uno dice su IP. Llenen esta tabla:

```
VM PROXY    (Janet):       192.168.___.___
VM APP1     (Alex):        192.168.___.___
VM APP2     (Jhon):        192.168.___.___
VM DB       (Luis):        192.168.___.___
VM ATACANTE (Jhon):        192.168.___.___
```

> ⚠️ IMPORTANTE: Todos los comandos de esta guía usan variables como `IP_PROXY`, `IP_APP1`, etc. Cuando se junten, reemplacen con las IPs reales.

### Verificar que todas las VMs se ven

Desde cualquier VM:

```bash
ping -c 2 IP_PROXY
ping -c 2 IP_APP1
ping -c 2 IP_APP2
ping -c 2 IP_DB
```

---
---

# 🖥️ VM PROXY — Janet Chambi Condori

## Lo que instalas en tu PC (puedes hacer todo esto sola)

---

### PASO 1 — Instalar Nginx

```bash
sudo apt update
sudo apt install nginx -y
sudo systemctl enable --now nginx
sudo systemctl status nginx
```

---

### PASO 2 — Instalar Prometheus y Node Exporter

```bash
sudo apt install prometheus prometheus-node-exporter -y
sudo systemctl enable prometheus prometheus-node-exporter
sudo systemctl start prometheus prometheus-node-exporter
```

Verificar:

```bash
sudo systemctl status prometheus --no-pager
curl http://localhost:9100/metrics | head -5
```

---

### PASO 3 — Instalar Grafana

```bash
sudo apt-get install -y apt-transport-https software-properties-common wget
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
sudo apt update
sudo apt install grafana -y
sudo systemctl daemon-reload
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

Verificar:

```bash
sudo systemctl status grafana-server --no-pager
```

---

### PASO 4 — Generar certificado TLS autofirmado

```bash
sudo mkdir -p /etc/nginx/ssl

sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/nginx-selfsigned.key \
  -out /etc/nginx/ssl/nginx-selfsigned.crt \
  -subj "/C=BO/ST=Chuquisaca/L=Sucre/O=USFX/OU=SIS313/CN=marketplace.local"
```

Verificar:

```bash
sudo openssl x509 -in /etc/nginx/ssl/nginx-selfsigned.crt -text -noout | grep -E "Subject:|Not Before|Not After"
```

---

### PASO 5 — Instalar Fail2ban

```bash
sudo apt install fail2ban -y
sudo systemctl enable --now fail2ban
```

Crear configuración:

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

Reiniciar:

```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status
```

---

### PASO 6 — Configurar Nginx con HTTPS + Balanceo (CUANDO SE JUNTEN)

> ⚠️ Este paso se hace cuando ya tengan las IPs reales de App1 y App2

```bash
sudo nano /etc/nginx/sites-available/default
```

Reemplazar todo con (cambiar `IP_APP1` e `IP_APP2` por las IPs reales):

```nginx
# Redirección HTTP a HTTPS
server {
    listen 80;
    server_name _;
    return 301 https://$host$request_uri;
}

# Balanceo HTTPS
upstream loadbalancer {
    least_conn;
    server IP_APP1:3000;   # App 1 — Alex
    server IP_APP2:3000;   # App 2 — Jhon Christian
}

server {
    listen 443 ssl;
    server_name _;

    ssl_certificate /etc/nginx/ssl/nginx-selfsigned.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx-selfsigned.key;

    # Hardening TLS
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH';
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;

    # Cabeceras de seguridad
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # Ocultar versión
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

```bash
sudo nginx -t
sudo systemctl restart nginx
```

---

### PASO 7 — Configurar Prometheus targets (CUANDO SE JUNTEN)

```bash
sudo nano /etc/prometheus/prometheus.yml
```

Agregar en `scrape_configs` (cambiar IPs por las reales):

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

```bash
sudo systemctl restart prometheus
```

---

### PASO 8 — Configurar Grafana (CUANDO SE JUNTEN)

1. Abrir navegador: `http://IP_PROXY:3000`
2. Login: `admin` / `admin` → cambiar contraseña
3. **Connections → Data sources → Add → Prometheus**
4. URL: `http://localhost:9090`
5. Click **Save & test**
6. **Dashboards → New → Import → ID: 1860 → Load**
7. Configurar alerta CPU > 80%

---

### Verificación completa del Proxy

```bash
# TLS
curl -k https://localhost

# Redirección
curl -I http://localhost

# Cabeceras de seguridad
curl -k -I https://localhost

# Versión TLS
openssl s_client -connect localhost:443 -tls1_2 </dev/null 2>/dev/null | grep "Protocol"

# Fail2ban
sudo fail2ban-client status
sudo fail2ban-client status sshd

# Servicios
systemctl status nginx prometheus grafana-server fail2ban --no-pager

# Balanceo (cuando estén juntos)
for i in {1..6}; do
    curl -sk https://localhost/ 2>/dev/null | grep -o 'Hostname.*<' | head -1
    sleep 0.5
done
```

---

### Comandos útiles de Fail2ban

```bash
# Ver IPs baneadas
sudo fail2ban-client status sshd

# Ver reglas de firewall
sudo iptables -L -n | grep fail2ban

# Desbanear IP
sudo fail2ban-client set sshd unbanip IP_ATACANTE

# Ver logs
sudo tail -f /var/log/fail2ban.log

# Ver intentos fallidos SSH
sudo grep "Failed password" /var/log/auth.log | tail -20
sudo grep "Failed password" /var/log/auth.log | wc -l
```

---
---

# 🖥️ VM APP1 — Calatayud Mamani Alex Josué

## Lo que instalas en tu PC (puedes hacer todo esto solo)

---

### PASO 1 — Instalar Node.js con NVM

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
\. "$HOME/.nvm/nvm.sh"
nvm install 22
node -v
npm -v
```

---

### PASO 2 — Instalar PM2

```bash
npm install pm2@latest -g
pm2 --version
```

---

### PASO 3 — Clonar la aplicación

```bash
mkdir ~/apps && cd ~/apps
git clone https://github.com/marceloquispeortega/api-restful-crud-movies app1
cd ~/apps/app1 && npm install
```

---

### PASO 4 — Configurar variables de entorno (CUANDO SE JUNTEN)

```bash
cp ~/apps/app1/.env.example ~/apps/app1/.env
nano ~/apps/app1/.env
```

```env
PORT=3000
DB_HOST=IP_DB
DB_PORT=3306
DB_NAME=db_movies
DB_USER=usr_movies
DB_PASSWORD=secret
```

> Reemplazar `IP_DB` con la IP real de la VM de Luis Fernando

---

### PASO 5 — Lanzar con PM2 (CUANDO SE JUNTEN)

```bash
cd ~/apps/app1 && pm2 start app.js --name app1
pm2 status
```

Configurar auto-arranque:

```bash
pm2 startup
# Copiar y ejecutar el comando sudo env PATH=... que genera
pm2 save
```

---

### PASO 6 — Instalar Node Exporter

```bash
sudo apt update
sudo apt install prometheus-node-exporter -y
sudo systemctl enable --now prometheus-node-exporter
```

Verificar:

```bash
sudo systemctl status prometheus-node-exporter --no-pager
curl http://localhost:9100/metrics | head -5
```

---

### PASO 7 — Crear script de Health Check

```bash
mkdir -p ~/scripts
nano ~/scripts/health_check.sh
```

```bash
#!/bin/bash
# ============================================
# HEALTH CHECK — Proyecto Marketplace SIS313
# Autor: Calatayud Mamani Alex Josué
# ============================================

LOGFILE="/var/log/health_check.log"
sudo touch $LOGFILE 2>/dev/null
TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')

echo "[$TIMESTAMP] === Inicio Health Check ===" | sudo tee -a $LOGFILE

# 1. Verificar Nginx (Proxy)
IP_PROXY="${1:-localhost}"
if curl -sk -o /dev/null -w "%{http_code}" --max-time 5 https://$IP_PROXY 2>/dev/null | grep -qE "200|301|302"; then
    echo "[$TIMESTAMP] [OK] Nginx (Proxy) respondiendo." | sudo tee -a $LOGFILE
else
    echo "[$TIMESTAMP] [ALERTA] Nginx (Proxy) NO responde." | sudo tee -a $LOGFILE
fi

# 2. Verificar App1 local (PM2)
APP_NAME="app1"
if pm2 describe $APP_NAME 2>/dev/null | grep -q "online"; then
    echo "[$TIMESTAMP] [OK] $APP_NAME (PM2) está online." | sudo tee -a $LOGFILE
else
    echo "[$TIMESTAMP] [ALERTA] $APP_NAME está caída. Reiniciando..." | sudo tee -a $LOGFILE
    pm2 restart $APP_NAME 2>/dev/null
    sleep 3
    if pm2 describe $APP_NAME 2>/dev/null | grep -q "online"; then
        echo "[$TIMESTAMP] [OK] $APP_NAME reiniciada exitosamente." | sudo tee -a $LOGFILE
    else
        echo "[$TIMESTAMP] [CRITICO] $APP_NAME no pudo reiniciarse." | sudo tee -a $LOGFILE
    fi
fi

# 3. Verificar App2 remota
IP_APP2="${2:-localhost}"
if curl -s -o /dev/null -w "%{http_code}" --max-time 5 http://$IP_APP2:3000 2>/dev/null | grep -q "200"; then
    echo "[$TIMESTAMP] [OK] App2 ($IP_APP2) respondiendo." | sudo tee -a $LOGFILE
else
    echo "[$TIMESTAMP] [ALERTA] App2 ($IP_APP2) NO responde." | sudo tee -a $LOGFILE
fi

# 4. Verificar MariaDB
IP_DB="${3:-localhost}"
if nc -zw2 $IP_DB 3306 2>/dev/null; then
    echo "[$TIMESTAMP] [OK] MariaDB ($IP_DB:3306) accesible." | sudo tee -a $LOGFILE
else
    echo "[$TIMESTAMP] [ALERTA] MariaDB NO accesible." | sudo tee -a $LOGFILE
fi

# 5. Verificar uso de disco
DISK_USAGE=$(df -h / | awk 'NR==2 {print $5}' | sed 's/%//g')
if [ "$DISK_USAGE" -gt 85 ]; then
    echo "[$TIMESTAMP] [CRITICO] Disco al $DISK_USAGE%." | sudo tee -a $LOGFILE
else
    echo "[$TIMESTAMP] [OK] Disco al $DISK_USAGE%." | sudo tee -a $LOGFILE
fi

# 6. Verificar memoria
MEM_AVAILABLE=$(free -m | grep Mem | awk '{print $7}')
MEM_TOTAL=$(free -m | grep Mem | awk '{print $2}')
echo "[$TIMESTAMP] [INFO] Memoria: ${MEM_AVAILABLE}MB / ${MEM_TOTAL}MB disponible." | sudo tee -a $LOGFILE

# 7. Node Exporter
if curl -s -o /dev/null -w "%{http_code}" --max-time 3 http://localhost:9100/metrics 2>/dev/null | grep -q "200"; then
    echo "[$TIMESTAMP] [OK] Node Exporter activo." | sudo tee -a $LOGFILE
else
    echo "[$TIMESTAMP] [ALERTA] Node Exporter no responde." | sudo tee -a $LOGFILE
fi

echo "[$TIMESTAMP] === Fin Health Check ===" | sudo tee -a $LOGFILE
echo ""
echo "Últimas entradas del log:"
sudo tail -n 10 $LOGFILE
```

Dar permisos:

```bash
chmod +x ~/scripts/health_check.sh
```

Ejecución (cuando estén juntos):

```bash
# Pasar IPs como argumentos: IP_PROXY IP_APP2 IP_DB
~/scripts/health_check.sh 192.168.1.10 192.168.1.12 192.168.1.14
```

---

### PASO 8 — Programar cron (CUANDO SE JUNTEN)

```bash
crontab -e
```

Agregar:

```
*/5 * * * * /home/TU_USUARIO/scripts/health_check.sh IP_PROXY IP_APP2 IP_DB >> /var/log/health_check_cron.log 2>&1
```

---

### Verificación completa App1

```bash
pm2 status
pm2 logs --lines 10
curl http://localhost:3000/movies
~/scripts/health_check.sh
sudo systemctl status prometheus-node-exporter --no-pager
```

---
---

# 🖥️ VM APP2 + VM ATACANTE — Quispe Anagua Jhon Christian

## Lo que instalas en tu PC (puedes hacer todo esto solo)

---

## VM APP2

### PASO 1 — Instalar Node.js con NVM

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
\. "$HOME/.nvm/nvm.sh"
nvm install 22
node -v
npm -v
```

---

### PASO 2 — Instalar PM2

```bash
npm install pm2@latest -g
pm2 --version
```

---

### PASO 3 — Clonar la aplicación

```bash
mkdir ~/apps && cd ~/apps
git clone https://github.com/marceloquispeortega/api-restful-crud-movies app
cd ~/apps/app && npm install
```

---

### PASO 4 — Configurar variables de entorno (CUANDO SE JUNTEN)

```bash
cp ~/apps/app/.env.example ~/apps/app/.env
nano ~/apps/app/.env
```

```env
PORT=3000
DB_HOST=IP_DB
DB_PORT=3306
DB_NAME=db_movies
DB_USER=usr_movies
DB_PASSWORD=secret
```

> Reemplazar `IP_DB` con la IP real de la VM de Luis Fernando

---

### PASO 5 — Lanzar con PM2 (CUANDO SE JUNTEN)

```bash
cd ~/apps/app && pm2 start app.js --name app2
pm2 status
pm2 startup
pm2 save
```

---

### PASO 6 — Instalar Node Exporter

```bash
sudo apt update
sudo apt install prometheus-node-exporter -y
sudo systemctl enable --now prometheus-node-exporter
```

---

### PASO 7 — Crear menú interactivo

```bash
mkdir -p ~/scripts
nano ~/scripts/admin_menu.sh
```

```bash
#!/bin/bash
# ============================================
# MENÚ DE ADMINISTRACIÓN — Marketplace SIS313
# Autor: Quispe Anagua Jhon Christian
# ============================================

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
CYAN='\033[0;36m'
NC='\033[0m'

# IPs — CAMBIAR CUANDO SE JUNTEN
IP_PROXY="${IP_PROXY:-localhost}"
IP_APP1="${IP_APP1:-localhost}"
IP_APP2="${IP_APP2:-localhost}"
IP_DB="${IP_DB:-localhost}"

show_header() {
    clear
    echo -e "${CYAN}"
    echo "╔══════════════════════════════════════════════════╗"
    echo "║     MARKETPLACE SIS313 — PANEL DE CONTROL       ║"
    echo "║              Grupo 2 · 2026                     ║"
    echo "╚══════════════════════════════════════════════════╝"
    echo -e "${NC}"
}

while true; do
    show_header
    echo -e "${YELLOW}  1)${NC} Estado de todos los servicios"
    echo -e "${YELLOW}  2)${NC} Estado de PM2 (aplicaciones)"
    echo -e "${YELLOW}  3)${NC} Ver películas en la base de datos"
    echo -e "${YELLOW}  4)${NC} Ver comentarios recientes"
    echo -e "${YELLOW}  5)${NC} Verificar balanceo de carga"
    echo -e "${YELLOW}  6)${NC} Ver uso de disco y memoria"
    echo -e "${YELLOW}  7)${NC} Ver IPs baneadas por Fail2ban"
    echo -e "${YELLOW}  8)${NC} Reiniciar aplicación local"
    echo -e "${RED}  9)${NC} Simular ataque de fuerza bruta (DEMO)"
    echo -e "${RED}  0)${NC} Salir"
    echo ""
    read -p "  Seleccione [0-9]: " opcion

    case $opcion in
        1)
            show_header
            echo -e "${CYAN}=== Estado de Servicios ===${NC}"
            echo ""
            # Proxy
            if curl -sk -o /dev/null -w "%{http_code}" --max-time 3 https://$IP_PROXY 2>/dev/null | grep -qE "200|301"; then
                echo -e "  Nginx (Proxy):      ${GREEN}[ACTIVO]${NC}"
            else
                echo -e "  Nginx (Proxy):      ${RED}[CAÍDO]${NC}"
            fi
            # App1
            if curl -s -o /dev/null -w "%{http_code}" --max-time 3 http://$IP_APP1:3000 2>/dev/null | grep -q "200"; then
                echo -e "  App1:               ${GREEN}[ACTIVO]${NC}"
            else
                echo -e "  App1:               ${RED}[CAÍDO]${NC}"
            fi
            # App2 local
            if curl -s -o /dev/null -w "%{http_code}" --max-time 3 http://localhost:3000 2>/dev/null | grep -q "200"; then
                echo -e "  App2 (local):       ${GREEN}[ACTIVO]${NC}"
            else
                echo -e "  App2 (local):       ${RED}[CAÍDO]${NC}"
            fi
            # MariaDB
            if nc -zw2 $IP_DB 3306 2>/dev/null; then
                echo -e "  MariaDB:            ${GREEN}[ACTIVO]${NC}"
            else
                echo -e "  MariaDB:            ${RED}[CAÍDO]${NC}"
            fi
            # Prometheus
            if curl -s -o /dev/null -w "%{http_code}" --max-time 3 http://$IP_PROXY:9090 2>/dev/null | grep -q "200"; then
                echo -e "  Prometheus:         ${GREEN}[ACTIVO]${NC}"
            else
                echo -e "  Prometheus:         ${RED}[CAÍDO]${NC}"
            fi
            echo ""
            read -p "  Enter para continuar..."
            ;;
        2)
            show_header
            echo -e "${CYAN}=== Estado PM2 ===${NC}"
            pm2 status
            echo ""
            read -p "  Enter para continuar..."
            ;;
        3)
            show_header
            echo -e "${CYAN}=== Películas ===${NC}"
            curl -s http://localhost:3000/movies 2>/dev/null | python3 -m json.tool 2>/dev/null || curl -s http://localhost:3000/movies
            echo ""
            read -p "  Enter para continuar..."
            ;;
        4)
            show_header
            echo -e "${CYAN}=== Comentarios ===${NC}"
            curl -s http://localhost:3000/comments 2>/dev/null | python3 -m json.tool 2>/dev/null || curl -s http://localhost:3000/comments
            echo ""
            read -p "  Enter para continuar..."
            ;;
        5)
            show_header
            echo -e "${CYAN}=== Verificación de Balanceo ===${NC}"
            echo "  6 solicitudes al proxy..."
            echo ""
            for i in $(seq 1 6); do
                RESP=$(curl -sk https://$IP_PROXY/ 2>/dev/null | grep -oP 'Hostname.*?<' | sed 's/<//g' | head -1)
                if [ -z "$RESP" ]; then
                    RESP=$(curl -s http://$IP_PROXY/ 2>/dev/null | grep -oP 'Hostname.*?<' | sed 's/<//g' | head -1)
                fi
                echo -e "  Solicitud $i → ${GREEN}$RESP${NC}"
                sleep 0.5
            done
            echo ""
            read -p "  Enter para continuar..."
            ;;
        6)
            show_header
            echo -e "${CYAN}=== Disco y Memoria ===${NC}"
            echo ""
            echo "  --- Disco ---"
            df -h / | awk 'NR==2 {printf "  Usado: %s de %s (%s)\n", $3, $2, $5}'
            echo ""
            echo "  --- Memoria ---"
            free -h | grep Mem | awk '{printf "  Usada: %s de %s (Disponible: %s)\n", $3, $2, $7}'
            echo ""
            echo "  --- Uptime ---"
            echo "  $(uptime -p)"
            echo ""
            read -p "  Enter para continuar..."
            ;;
        7)
            show_header
            echo -e "${CYAN}=== IPs Baneadas (Fail2ban) ===${NC}"
            ssh -o ConnectTimeout=3 admin@$IP_PROXY "sudo fail2ban-client status sshd" 2>/dev/null || echo -e "  ${RED}No se pudo conectar al proxy${NC}"
            echo ""
            read -p "  Enter para continuar..."
            ;;
        8)
            show_header
            echo -e "${YELLOW}Reiniciando app local...${NC}"
            pm2 restart all
            sleep 2
            pm2 status
            echo ""
            read -p "  Enter para continuar..."
            ;;
        9)
            show_header
            echo -e "${RED}╔══════════════════════════════════════╗${NC}"
            echo -e "${RED}║   ⚠️  SIMULACIÓN DE ATAQUE ⚠️        ║${NC}"
            echo -e "${RED}║   Solo para demostración             ║${NC}"
            echo -e "${RED}╚══════════════════════════════════════╝${NC}"
            echo ""
            read -p "  ¿Ejecutar? (si/no): " CONFIRM
            if [ "$CONFIRM" = "si" ]; then
                if command -v hydra &>/dev/null; then
                    cat > /tmp/passwords.txt << 'EOF'
123456
password
admin
ubuntu
root
12345678
qwerty
letmein
welcome
princess
EOF
                    echo ""
                    echo "  Atacando SSH del proxy ($IP_PROXY)..."
                    hydra -l admin -P /tmp/passwords.txt ssh://$IP_PROXY -t 4 -V 2>/dev/null
                    rm -f /tmp/passwords.txt
                    echo ""
                    echo -e "${GREEN}  Verificando bloqueo...${NC}"
                    sleep 2
                    ssh -o ConnectTimeout=3 admin@$IP_PROXY "sudo fail2ban-client status sshd" 2>/dev/null || echo -e "  ${YELLOW}Posiblemente baneado${NC}"
                else
                    echo -e "  ${YELLOW}Hydra no instalado. Ejecutar: sudo apt install hydra${NC}"
                fi
            fi
            echo ""
            read -p "  Enter para continuar..."
            ;;
        0)
            clear
            echo -e "${GREEN}¡Hasta pronto! — Marketplace SIS313${NC}"
            exit 0
            ;;
        *)
            echo -e "${RED}  Opción inválida.${NC}"
            sleep 1
            ;;
    esac
done
```

Dar permisos:

```bash
chmod +x ~/scripts/admin_menu.sh
```

Ejecución (cuando estén juntos):

```bash
# Exportar IPs antes de ejecutar
export IP_PROXY=192.168.1.X
export IP_APP1=192.168.1.X
export IP_DB=192.168.1.X
~/scripts/admin_menu.sh
```

---

## VM ATACANTE (segunda VM en la PC de Jhon Christian)

### PASO 1 — Crear VM Atacante

1. Clonar la VM App2 o crear una nueva con Ubuntu Server
2. **Adaptador 1 → Puente (WiFi)**
3. Encender y configurar DHCP

### PASO 2 — Instalar herramientas

```bash
sudo apt update
sudo apt install hydra nmap -y
```

### PASO 3 — Ataque de fuerza bruta (CUANDO SE JUNTEN)

```bash
# Crear diccionario
cat > ~/passwords.txt << 'EOF'
123456
password
admin
ubuntu
root
12345678
qwerty
letmein
welcome
princess
secreto
marketplace
sis313
EOF

# Ataque a SSH del Proxy (cambiar IP y usuario)
hydra -l USUARIO_PROXY -P ~/passwords.txt ssh://IP_PROXY -t 4 -V
```

### PASO 4 — Escaneo de puertos

```bash
# Escaneo del Proxy
nmap -sV IP_PROXY

# Escaneo de toda la red
nmap -sV 192.168.1.0/24
```

### PASO 5 — Reconocimiento web

```bash
# 50 peticiones a rutas inexistentes
for i in $(seq 1 50); do
    curl -sk -o /dev/null -w "%{http_code}" https://IP_PROXY/admin$i
    echo " -> /admin$i"
done
```

---
---

# 🖥️ VM DB — Quispe Sullca Luis Fernando

## Lo que instalas en tu PC (puedes hacer todo esto solo)

---

### PASO 1 — Instalar MariaDB

```bash
sudo apt update
sudo apt install mariadb-server -y
sudo systemctl enable --now mariadb
```

---

### PASO 2 — Hardening MariaDB

```bash
sudo mysql_secure_installation
```

Responder:
```
Enter current password for root:        → Enter (vacío)
Switch to unix_socket authentication:   → Enter
Change root password?:                  → Enter → nueva contraseña
Remove anonymous users?:                → Enter
Disallow root login remotely?:          → Enter
Remove test database?:                  → Enter
Reload privilege tables?:               → Enter
```

---

### PASO 3 — Instalar mdadm para RAID

```bash
sudo apt install mdadm -y
```

---

### PASO 4 — Agregar discos en VirtualBox y crear RAID 10

1. **Apagar la VM**
2. **Configuración → Almacenamiento → Controlador SATA**
3. Agregar 4 discos nuevos de **2 GB** cada uno
4. Encender la VM

Verificar discos:

```bash
sudo fdisk -l
lsblk
```

Crear RAID 10 (adaptar nombres según lo que muestre lsblk):

```bash
sudo mdadm --create --verbose /dev/md10 --level=10 --raid-devices=4 /dev/sdb /dev/sdc /dev/sdd /dev/sde
```

Verificar:

```bash
cat /proc/mdstat
sudo mdadm --detail /dev/md10
```

---

### PASO 5 — Formatear y montar RAID

```bash
sudo mkfs.ext4 /dev/md10
sudo mkdir -p /mnt/raid10
sudo mount /dev/md10 /mnt/raid10
df -h /mnt/raid10
```

Montaje persistente:

```bash
sudo blkid /dev/md10
sudo nano /etc/fstab
```

Agregar (reemplazar UUID):

```
UUID=TU_UUID  /mnt/raid10  ext4  defaults,nofail  0  2
```

Guardar RAID:

```bash
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
sudo update-initramfs -u
sudo systemctl daemon-reload
sudo mount -a
```

---

### PASO 6 — Mover MariaDB al RAID

```bash
sudo systemctl stop mariadb
sudo rsync -av /var/lib/mysql/ /mnt/raid10/mysql/
sudo chown -R mysql:mysql /mnt/raid10/mysql/
```

Cambiar datadir:

```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

Cambiar:

```ini
datadir = /mnt/raid10/mysql
```

Si AppArmor bloquea:

```bash
echo "alias /var/lib/mysql/ -> /mnt/raid10/mysql/," | sudo tee -a /etc/apparmor.d/tunables/alias
sudo systemctl restart apparmor
```

Reiniciar:

```bash
sudo systemctl start mariadb
sudo systemctl status mariadb
```

---

### PASO 7 — Cambiar bind-address (CUANDO SE JUNTEN)

```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

Cambiar bind-address a tu IP WiFi:

```ini
bind-address = TU_IP_WIFI
```

```bash
sudo systemctl restart mariadb
```

---

### PASO 8 — Crear base de datos y usuarios (CUANDO SE JUNTEN)

```bash
sudo mysql -u root -p
```

```sql
CREATE DATABASE db_movies;

-- Crear usuario para App1 (cambiar IP por la real de Alex)
CREATE USER 'usr_movies'@'IP_APP1' IDENTIFIED BY 'secret';
GRANT ALL PRIVILEGES ON db_movies.* TO 'usr_movies'@'IP_APP1';

-- Crear usuario para App2 (cambiar IP por la real de Jhon)
CREATE USER 'usr_movies'@'IP_APP2' IDENTIFIED BY 'secret';
GRANT ALL PRIVILEGES ON db_movies.* TO 'usr_movies'@'IP_APP2';

FLUSH PRIVILEGES;

-- Crear tablas
USE db_movies;

CREATE TABLE movies (
    id serial PRIMARY KEY,
    title character varying(150) NOT NULL,
    year integer,
    UNIQUE(title)
);

INSERT INTO movies (title, year) VALUES
    ('Inception', 2010),
    ('The Matrix', 1999),
    ('Pulp Fiction', 1994),
    ('The Dark Knight', 2008),
    ('Eternal Sunshine of the Spotless Mind', 2004),
    ('Forrest Gump', 1994),
    ('Fight Club', 1999),
    ('The Godfather', 1972),
    ('Interstellar', 2014),
    ('Parasite', 2019);

CREATE TABLE comments (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    career VARCHAR(100),
    message TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE visit_counter (
    id INT PRIMARY KEY,
    total BIGINT DEFAULT 0,
    last_visit DATETIME
);

quit
```

---

### PASO 9 — Configurar UFW

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

---

### PASO 10 — Instalar Node Exporter

```bash
sudo apt install prometheus-node-exporter -y
sudo systemctl enable --now prometheus-node-exporter
```

---

### PASO 11 — Script de backup

```bash
sudo mkdir -p /mnt/raid10/backups
sudo nano /opt/backup_db.sh
```

```bash
#!/bin/bash
# ============================================
# BACKUP AUTOMÁTICO — MariaDB sobre RAID 10
# Autor: Quispe Sullca Luis Fernando
# ============================================

BACKUP_DIR="/mnt/raid10/backups"
DB_NAME="db_movies"
TIMESTAMP=$(date '+%Y%m%d_%H%M%S')
BACKUP_FILE="$BACKUP_DIR/${DB_NAME}_${TIMESTAMP}.sql"
DAYS_TO_KEEP=7

# Crear backup
sudo mysqldump -u root $DB_NAME > $BACKUP_FILE

if [ $? -eq 0 ]; then
    gzip $BACKUP_FILE
    echo "$(date '+%Y-%m-%d %H:%M:%S') [OK] Backup: ${BACKUP_FILE}.gz" | sudo tee -a /var/log/backup_db.log
else
    echo "$(date '+%Y-%m-%d %H:%M:%S') [ERROR] Falló backup" | sudo tee -a /var/log/backup_db.log
fi

# Rotación
find $BACKUP_DIR -name "*.sql.gz" -mtime +$DAYS_TO_KEEP -delete
echo "$(date '+%Y-%m-%d %H:%M:%S') [INFO] Rotación completada" | sudo tee -a /var/log/backup_db.log
```

```bash
sudo chmod +x /opt/backup_db.sh
sudo /opt/backup_db.sh
ls -lh /mnt/raid10/backups/
```

---

### PASO 12 — Script de restauración

```bash
sudo nano /opt/restore_db.sh
```

```bash
#!/bin/bash
# ============================================
# RESTAURACIÓN DE BASE DE DATOS
# Autor: Quispe Sullca Luis Fernando
# ============================================

BACKUP_DIR="/mnt/raid10/backups"
DB_NAME="db_movies"

echo "=== Backups Disponibles ==="
ls -lh $BACKUP_DIR/*.sql.gz 2>/dev/null

if [ $? -ne 0 ]; then
    echo "[ERROR] No hay backups."
    exit 1
fi

echo ""
read -p "Archivo a restaurar: " BACKUP_FILE

if [ ! -f "$BACKUP_DIR/$BACKUP_FILE" ]; then
    echo "[ERROR] No encontrado: $BACKUP_DIR/$BACKUP_FILE"
    exit 1
fi

echo "⚠️  Esto sobreescribirá $DB_NAME"
read -p "¿Continuar? (si/no): " CONFIRM

if [ "$CONFIRM" = "si" ]; then
    gunzip -k "$BACKUP_DIR/$BACKUP_FILE"
    SQL_FILE="${BACKUP_DIR}/${BACKUP_FILE%.gz}"
    sudo mysql -u root $DB_NAME < "$SQL_FILE"
    rm -f "$SQL_FILE"
    echo "[OK] Restauración completada desde $BACKUP_FILE"
else
    echo "Cancelado."
fi
```

```bash
sudo chmod +x /opt/restore_db.sh
```

---

### PASO 13 — Programar cron

```bash
sudo crontab -e
```

Agregar:

```
# Backup cada hora
0 * * * * /opt/backup_db.sh >> /var/log/backup_db.log 2>&1
```

---

### PASO 14 — Simulación de fallo de disco RAID

```bash
# Marcar disco como fallido
sudo mdadm --manage /dev/md10 --fail /dev/sdb

# Ver estado degradado
sudo mdadm --detail /dev/md10

# Verificar datos siguen accesibles
sudo mysql -u root -p -e "USE db_movies; SELECT * FROM movies;"

# Remover disco fallido
sudo mdadm --manage /dev/md10 --remove /dev/sdb

# Agregar disco nuevo (previamente añadido en VirtualBox)
sudo mdadm --manage /dev/md10 --add /dev/sdf

# Monitorear reconstrucción
watch cat /proc/mdstat
```

---

### Verificación completa DB

```bash
# RAID
sudo mdadm --detail /dev/md10

# MariaDB
sudo systemctl status mariadb --no-pager

# Datos
sudo mysql -u root -p -e "USE db_movies; SELECT * FROM movies; SELECT COUNT(*) FROM comments;"

# Backups
ls -lh /mnt/raid10/backups/
cat /var/log/backup_db.log

# UFW
sudo ufw status numbered

# Node Exporter
sudo systemctl status prometheus-node-exporter --no-pager
```

---
---

# 📋 CHECKLIST PARA EL DÍA DE LA DEFENSA

## Antes de llegar

Cada uno debe tener listo en su PC:
- [ ] VM encendida y funcionando
- [ ] Software instalado (Nginx/Node.js/MariaDB/Hydra)
- [ ] Scripts creados y con permisos
- [ ] RAID creado (Luis Fernando)
- [ ] Certificado TLS generado (Janet)
- [ ] Fail2ban configurado (Janet)

## Al llegar — Todos al mismo WiFi

1. [ ] Conectar todas las PCs al mismo WiFi
2. [ ] Verificar IPs de cada VM: `ip addr`
3. [ ] Hacer ping entre todas las VMs
4. [ ] Luis Fernando: cambiar bind-address + crear usuarios con IPs reales
5. [ ] Alex y Jhon: actualizar `.env` con IP real de la DB
6. [ ] Janet: actualizar `upstream` de Nginx con IPs reales de Apps
7. [ ] Janet: actualizar `prometheus.yml` con IPs reales
8. [ ] Lanzar Apps: `pm2 start` en App1 y App2
9. [ ] Verificar todo funciona

## Secuencia de demo

1. **Janet:** Mostrar página web → `https://IP_PROXY` (navegador)
2. **Janet:** Demostrar balanceo → recargar, hostname cambia
3. **Janet:** Mostrar TLS → `curl -k -I https://IP_PROXY` (cabeceras)
4. **Alex:** Demostrar failover → `pm2 stop app1` → página sigue funcionando
5. **Alex:** Ejecutar health check → `~/scripts/health_check.sh`
6. **Janet:** Mostrar Grafana → dashboards de las 4 VMs
7. **Jhon Christian:** Abrir menú interactivo → mostrar opciones
8. **Jhon Christian:** Simular ataque → Hydra desde VM atacante
9. **Janet:** Mostrar Fail2ban → IP del atacante baneada
10. **Luis Fernando:** Mostrar RAID → `sudo mdadm --detail /dev/md10`
11. **Luis Fernando:** Simular fallo de disco → datos siguen accesibles
12. **Luis Fernando:** Mostrar backup → `ls -lh /mnt/raid10/backups/`
13. **Luis Fernando:** Restaurar DB desde backup

---

*Guía del Proyecto Final · Grupo 2 · SIS313 · USFX · 2026*
