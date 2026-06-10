# 🚀 Guía Completa — Proyecto Final Marketplace
## Centro de Datos · Feria · SIS313 · 2026

---

## 📋 Distribución de VMs

| VM | IP | Responsable | Rol |
|---|---|---|---|
| server-174 | 192.168.100.174 | Janet | Proxy + TLS + BIND9 + Prometheus + Grafana |
| server-175 | 192.168.100.175 | Alex Josué | App 1 + Health Check |
| server-176 | 192.168.100.176 | Jhon Christian | App 2 + Menú Interactivo |
| server-177 | 192.168.100.177 | Luis Fernando | DB (MariaDB) + Backups |

## 🔑 Conexión

```bash
ssh adming9@192.168.100.174   # Proxy (Janet)
ssh adming9@192.168.100.175   # App1 (Alex)
ssh adming9@192.168.100.176   # App2 (Jhon)
ssh adming9@192.168.100.177   # DB (Luis Fernando)
# Contraseña: 4dm1ng9
```

## 🎯 Temas Avanzados (8 temas)

| Código | Tema | Responsable |
|---|---|---|
| T4 | Proxy inverso y balanceo de carga (Nginx) | Janet |
| T7 | DNS primario con BIND9 | Janet |
| T8 | Despliegue de aplicaciones (Node.js + PM2) | Alex + Jhon |
| T9 | Bases de datos centralizadas (MariaDB) | Luis Fernando |
| T10 | Monitoreo integral (Prometheus + Grafana) | Janet |
| T12 | Seguridad TLS/SSL | Janet |
| T14 | Automatización con Bash (health check, menú) | Alex + Jhon |
| T15 | Backups automatizados y recuperación | Luis Fernando |

---

## ⚡ ORDEN DE IMPLEMENTACIÓN

```
PASO 1 → Luis Fernando configura la DB (server-177)
PASO 2 → Alex y Jhon configuran las Apps (server-175 y server-176)
PASO 3 → Janet configura el Proxy con todo (server-174)
```

> La DB va primero porque las Apps necesitan conectarse a ella.

---
---

# 🖥️ PASO 1 — VM DB (server-177) · Luis Fernando

```bash
ssh adming9@192.168.100.177
```

---

## 1.1 Verificar red

```bash
ip addr
ping -c 3 google.com
ping -c 3 192.168.100.174
```

---

## 1.2 Instalar MariaDB

```bash
sudo apt update
sudo apt install mariadb-server -y
sudo systemctl enable --now mariadb
sudo systemctl status mariadb
```

---

## 1.3 Hardening de MariaDB

```bash
sudo mysql_secure_installation
```

Responder:
- Enter current password: `Enter` (vacío)
- Switch to unix_socket: `Enter`
- Change root password: `Enter` → nueva contraseña
- Remove anonymous users: `Enter`
- Disallow root login remotely: `Enter`
- Remove test database: `Enter`
- Reload privilege tables: `Enter`

---

## 1.4 Cambiar bind-address

```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

Buscar y cambiar:

```ini
bind-address = 192.168.100.177
```

```bash
sudo systemctl restart mariadb
sudo systemctl status mariadb
```

---

## 1.5 Crear base de datos, usuarios y tablas

```bash
sudo mysql -u root -p
```

```sql
-- Crear base de datos
CREATE DATABASE db_marketplace;

-- Crear usuario para App1 (192.168.100.175)
CREATE USER 'usr_market'@'192.168.100.175' IDENTIFIED BY 'market2026';
GRANT ALL PRIVILEGES ON db_marketplace.* TO 'usr_market'@'192.168.100.175';

-- Crear usuario para App2 (192.168.100.176)
CREATE USER 'usr_market'@'192.168.100.176' IDENTIFIED BY 'market2026';
GRANT ALL PRIVILEGES ON db_marketplace.* TO 'usr_market'@'192.168.100.176';

FLUSH PRIVILEGES;

-- Crear tablas
USE db_marketplace;

CREATE TABLE products (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(150) NOT NULL,
    description TEXT,
    price DECIMAL(10,2) NOT NULL,
    category VARCHAR(50),
    image_url VARCHAR(255),
    seller VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

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

-- Insertar productos de ejemplo
INSERT INTO products (name, description, price, category, image_url, seller) VALUES
    ('Laptop HP Pavilion', 'Laptop HP 15.6" Intel Core i5, 8GB RAM, 256GB SSD', 4500.00, 'Tecnología', 'https://images.unsplash.com/photo-1496181133206-80ce9b88a853?w=400', 'TechStore Sucre'),
    ('Smartphone Samsung A54', 'Samsung Galaxy A54 128GB, pantalla AMOLED 6.4"', 2200.00, 'Tecnología', 'https://images.unsplash.com/photo-1598327105666-5b89351aff97?w=400', 'MóvilCenter'),
    ('Mochila North Face', 'Mochila resistente 40L para senderismo', 350.00, 'Deportes', 'https://images.unsplash.com/photo-1553062407-98eeb64c6a62?w=400', 'Aventura Bolivia'),
    ('Auriculares Sony WH-1000', 'Auriculares inalámbricos con cancelación de ruido', 1800.00, 'Tecnología', 'https://images.unsplash.com/photo-1505740420928-5e560c06d30e?w=400', 'AudioPro'),
    ('Zapatillas Nike Air Max', 'Nike Air Max 90, talla disponible 38-44', 890.00, 'Moda', 'https://images.unsplash.com/photo-1542291026-7eec264c27ff?w=400', 'SneakerHub'),
    ('Cámara Canon EOS', 'Canon EOS M50 Mark II, 24.1MP, 4K Video', 5200.00, 'Tecnología', 'https://images.unsplash.com/photo-1516035069371-29a1b244cc32?w=400', 'FotoMundo'),
    ('Bicicleta Mountain Bike', 'Bicicleta MTB 29" aluminio, 21 velocidades', 2800.00, 'Deportes', 'https://images.unsplash.com/photo-1485965120184-e220f721d03e?w=400', 'BiciSucre'),
    ('Reloj Casio G-Shock', 'Reloj deportivo resistente al agua 200m', 650.00, 'Accesorios', 'https://images.unsplash.com/photo-1524592094714-0f0654e20314?w=400', 'TimeZone'),
    ('Teclado Mecánico RGB', 'Teclado gaming mecánico switches blue, RGB', 420.00, 'Tecnología', 'https://images.unsplash.com/photo-1587829741301-dc798b83add3?w=400', 'GamerStore'),
    ('Libro Clean Code', 'Robert C. Martin - Clean Code, edición en español', 180.00, 'Libros', 'https://images.unsplash.com/photo-1544716278-ca5e3f4abd8c?w=400', 'LibroSucre');

-- Verificar
SELECT * FROM products;
SHOW TABLES;

quit
```

---

## 1.6 Instalar Node Exporter

```bash
sudo apt install prometheus-node-exporter -y
sudo systemctl enable --now prometheus-node-exporter
sudo systemctl status prometheus-node-exporter
```

---

## 1.7 Configurar UFW

```bash
sudo apt install ufw -y
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 192.168.100.175 to any port 3306
sudo ufw allow from 192.168.100.176 to any port 3306
sudo ufw allow from 192.168.100.174 to any port 9100
sudo ufw allow 22
sudo ufw enable
sudo ufw status numbered
```

---

## 1.8 Script de Backup

```bash
sudo mkdir -p /var/backups/marketplace
sudo nano /opt/backup_db.sh
```

```bash
#!/bin/bash
# Backup automático — MariaDB Marketplace
# Autor: Quispe Sullca Luis Fernando

BACKUP_DIR="/var/backups/marketplace"
DB_NAME="db_marketplace"
TIMESTAMP=$(date '+%Y%m%d_%H%M%S')
BACKUP_FILE="$BACKUP_DIR/${DB_NAME}_${TIMESTAMP}.sql"
DAYS_TO_KEEP=7

sudo mysqldump -u root $DB_NAME > $BACKUP_FILE

if [ $? -eq 0 ]; then
    gzip $BACKUP_FILE
    echo "$(date '+%Y-%m-%d %H:%M:%S') [OK] Backup: ${BACKUP_FILE}.gz" | sudo tee -a /var/log/backup_db.log
else
    echo "$(date '+%Y-%m-%d %H:%M:%S') [ERROR] Falló backup" | sudo tee -a /var/log/backup_db.log
fi

find $BACKUP_DIR -name "*.sql.gz" -mtime +$DAYS_TO_KEEP -delete
echo "$(date '+%Y-%m-%d %H:%M:%S') [INFO] Rotación completada" | sudo tee -a /var/log/backup_db.log
```

```bash
sudo chmod +x /opt/backup_db.sh
sudo /opt/backup_db.sh
ls -lh /var/backups/marketplace/
```

---

## 1.9 Script de Restauración

```bash
sudo nano /opt/restore_db.sh
```

```bash
#!/bin/bash
# Restauración de BD — Marketplace
# Autor: Quispe Sullca Luis Fernando

BACKUP_DIR="/var/backups/marketplace"
DB_NAME="db_marketplace"

echo "=== Backups Disponibles ==="
ls -lh $BACKUP_DIR/*.sql.gz 2>/dev/null

if [ $? -ne 0 ]; then
    echo "[ERROR] No hay backups."
    exit 1
fi

echo ""
read -p "Archivo a restaurar: " BACKUP_FILE

if [ ! -f "$BACKUP_DIR/$BACKUP_FILE" ]; then
    echo "[ERROR] No encontrado."
    exit 1
fi

echo "⚠️  Esto sobreescribirá $DB_NAME"
read -p "¿Continuar? (si/no): " CONFIRM

if [ "$CONFIRM" = "si" ]; then
    gunzip -k "$BACKUP_DIR/$BACKUP_FILE"
    SQL_FILE="${BACKUP_DIR}/${BACKUP_FILE%.gz}"
    sudo mysql -u root $DB_NAME < "$SQL_FILE"
    rm -f "$SQL_FILE"
    echo "[OK] Restauración completada"
else
    echo "Cancelado."
fi
```

```bash
sudo chmod +x /opt/restore_db.sh
```

---

## 1.10 Programar cron (cada hora)

```bash
sudo crontab -e
```

Agregar:

```
0 * * * * /opt/backup_db.sh >> /var/log/backup_db.log 2>&1
```

---

## 1.11 Verificación completa DB

```bash
sudo systemctl status mariadb --no-pager
sudo mysql -u root -p -e "USE db_marketplace; SELECT COUNT(*) FROM products;"
sudo ufw status numbered
ls -lh /var/backups/marketplace/
sudo systemctl status prometheus-node-exporter --no-pager
```

---
---

# 🖥️ PASO 2A — VM App1 (server-175) · Alex Josué

```bash
ssh adming9@192.168.100.175
```

---

## 2A.1 Instalar Node.js con NVM

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
\. "$HOME/.nvm/nvm.sh"
nvm install 22
node -v
npm -v
```

---

## 2A.2 Instalar PM2

```bash
npm install pm2@latest -g
pm2 --version
```

---

## 2A.3 Clonar la aplicación

```bash
mkdir ~/apps && cd ~/apps
git clone https://github.com/marceloquispeortega/api-restful-crud-movies marketplace
cd ~/apps/marketplace && npm install
```

---

## 2A.4 Configurar .env

```bash
cp .env.example .env
nano .env
```

```env
PORT=3000
DB_HOST=192.168.100.177
DB_PORT=3306
DB_NAME=db_marketplace
DB_USER=usr_market
DB_PASSWORD=market2026
```

---

## 2A.5 Actualizar app.js con la ruta del Marketplace

> Descargar el archivo `marketplace-route.js` que Janet tiene y reemplazar el `app.get('/', ...)` en `app.js` con ese contenido.

```bash
nano ~/apps/marketplace/app.js
```

> Buscar `app.get('/', ...` y reemplazar todo el bloque hasta `});` con el contenido del marketplace-route.js

---

## 2A.6 Lanzar con PM2

```bash
cd ~/apps/marketplace && pm2 start app.js --name app1
pm2 status
pm2 logs --lines 5
```

Debe mostrar: `Servidor ejecutándose en el puerto 3000` y `Conexión a MariaDB exitosa`

Auto-arranque:

```bash
pm2 startup
# Copiar y ejecutar el comando sudo env PATH=... que genera
pm2 save
```

---

## 2A.7 Instalar Node Exporter

```bash
sudo apt update
sudo apt install prometheus-node-exporter -y
sudo systemctl enable --now prometheus-node-exporter
```

---

## 2A.8 Script de Health Check

```bash
mkdir -p ~/scripts
nano ~/scripts/health_check.sh
```

```bash
#!/bin/bash
# Health Check — Marketplace SIS313
# Autor: Calatayud Mamani Alex Josué

LOGFILE="/var/log/health_check.log"
sudo touch $LOGFILE 2>/dev/null
TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')

echo "[$TIMESTAMP] === Inicio Health Check ===" | sudo tee -a $LOGFILE

# 1. Verificar Proxy (Nginx)
if curl -sk -o /dev/null -w "%{http_code}" --max-time 5 https://192.168.100.174 2>/dev/null | grep -qE "200|301|302"; then
    echo "[$TIMESTAMP] [OK] Proxy (Nginx) respondiendo." | sudo tee -a $LOGFILE
else
    echo "[$TIMESTAMP] [ALERTA] Proxy NO responde." | sudo tee -a $LOGFILE
fi

# 2. Verificar App1 local (PM2)
APP_NAME="app1"
if pm2 describe $APP_NAME 2>/dev/null | grep -q "online"; then
    echo "[$TIMESTAMP] [OK] $APP_NAME (PM2) online." | sudo tee -a $LOGFILE
else
    echo "[$TIMESTAMP] [ALERTA] $APP_NAME caída. Reiniciando..." | sudo tee -a $LOGFILE
    pm2 restart $APP_NAME 2>/dev/null
    sleep 3
    if pm2 describe $APP_NAME 2>/dev/null | grep -q "online"; then
        echo "[$TIMESTAMP] [OK] $APP_NAME reiniciada." | sudo tee -a $LOGFILE
    else
        echo "[$TIMESTAMP] [CRITICO] $APP_NAME no pudo reiniciarse." | sudo tee -a $LOGFILE
    fi
fi

# 3. Verificar App2 remota
if curl -s -o /dev/null -w "%{http_code}" --max-time 5 http://192.168.100.176:3000 2>/dev/null | grep -q "200"; then
    echo "[$TIMESTAMP] [OK] App2 respondiendo." | sudo tee -a $LOGFILE
else
    echo "[$TIMESTAMP] [ALERTA] App2 NO responde." | sudo tee -a $LOGFILE
fi

# 4. Verificar MariaDB
if nc -zw2 192.168.100.177 3306 2>/dev/null; then
    echo "[$TIMESTAMP] [OK] MariaDB accesible." | sudo tee -a $LOGFILE
else
    echo "[$TIMESTAMP] [ALERTA] MariaDB NO accesible." | sudo tee -a $LOGFILE
fi

# 5. Disco
DISK_USAGE=$(df -h / | awk 'NR==2 {print $5}' | sed 's/%//g')
if [ "$DISK_USAGE" -gt 85 ]; then
    echo "[$TIMESTAMP] [CRITICO] Disco al $DISK_USAGE%." | sudo tee -a $LOGFILE
else
    echo "[$TIMESTAMP] [OK] Disco al $DISK_USAGE%." | sudo tee -a $LOGFILE
fi

# 6. Memoria
MEM_AVAILABLE=$(free -m | grep Mem | awk '{print $7}')
echo "[$TIMESTAMP] [INFO] Memoria disponible: ${MEM_AVAILABLE}MB." | sudo tee -a $LOGFILE

echo "[$TIMESTAMP] === Fin Health Check ===" | sudo tee -a $LOGFILE
echo ""
echo "Últimas entradas:"
sudo tail -n 10 $LOGFILE
```

```bash
chmod +x ~/scripts/health_check.sh
~/scripts/health_check.sh
```

---

## 2A.9 Programar cron (cada 5 min)

```bash
crontab -e
```

```
*/5 * * * * /home/adming9/scripts/health_check.sh >> /var/log/health_check_cron.log 2>&1
```

---
---

# 🖥️ PASO 2B — VM App2 (server-176) · Jhon Christian

```bash
ssh adming9@192.168.100.176
```

---

## 2B.1 Instalar Node.js + PM2

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
\. "$HOME/.nvm/nvm.sh"
nvm install 22
npm install pm2@latest -g
```

---

## 2B.2 Clonar y configurar la aplicación

```bash
mkdir ~/apps && cd ~/apps
git clone https://github.com/marceloquispeortega/api-restful-crud-movies marketplace
cd ~/apps/marketplace && npm install
cp .env.example .env
nano .env
```

```env
PORT=3000
DB_HOST=192.168.100.177
DB_PORT=3306
DB_NAME=db_marketplace
DB_USER=usr_market
DB_PASSWORD=market2026
```

> También actualizar `app.js` con el marketplace-route.js (misma página bonita que App1)

---

## 2B.3 Lanzar con PM2

```bash
cd ~/apps/marketplace && pm2 start app.js --name app2
pm2 status
pm2 startup
pm2 save
```

---

## 2B.4 Instalar Node Exporter

```bash
sudo apt update
sudo apt install prometheus-node-exporter -y
sudo systemctl enable --now prometheus-node-exporter
```

---

## 2B.5 Menú Interactivo de Administración

```bash
mkdir -p ~/scripts
nano ~/scripts/admin_menu.sh
```

```bash
#!/bin/bash
# Menú de Administración — Marketplace SIS313
# Autor: Quispe Anagua Jhon Christian

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
CYAN='\033[0;36m'
NC='\033[0m'

IP_PROXY="192.168.100.174"
IP_APP1="192.168.100.175"
IP_APP2="192.168.100.176"
IP_DB="192.168.100.177"

show_header() {
    clear
    echo -e "${CYAN}"
    echo "╔══════════════════════════════════════════════════╗"
    echo "║     MARKETPLACE SIS313 — PANEL DE CONTROL       ║"
    echo "║              Grupo 9 · Feria 2026               ║"
    echo "╚══════════════════════════════════════════════════╝"
    echo -e "${NC}"
}

while true; do
    show_header
    echo -e "${YELLOW}  1)${NC} Estado de todos los servicios"
    echo -e "${YELLOW}  2)${NC} Estado de PM2"
    echo -e "${YELLOW}  3)${NC} Ver productos en la BD"
    echo -e "${YELLOW}  4)${NC} Ver comentarios"
    echo -e "${YELLOW}  5)${NC} Verificar balanceo de carga"
    echo -e "${YELLOW}  6)${NC} Uso de disco y memoria"
    echo -e "${YELLOW}  7)${NC} Reiniciar app local"
    echo -e "${YELLOW}  8)${NC} Ver backups disponibles"
    echo -e "${RED}  0)${NC} Salir"
    echo ""
    read -p "  Seleccione [0-8]: " opcion

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
                echo -e "  App1 (server-175):  ${GREEN}[ACTIVO]${NC}"
            else
                echo -e "  App1 (server-175):  ${RED}[CAÍDO]${NC}"
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
            echo -e "${CYAN}=== Productos ===${NC}"
            curl -s http://localhost:3000/products 2>/dev/null | python3 -m json.tool 2>/dev/null || echo "No se pudo obtener productos"
            echo ""
            read -p "  Enter para continuar..."
            ;;
        4)
            show_header
            echo -e "${CYAN}=== Comentarios ===${NC}"
            curl -s http://localhost:3000/comments 2>/dev/null | python3 -m json.tool 2>/dev/null || echo "Sin comentarios"
            echo ""
            read -p "  Enter para continuar..."
            ;;
        5)
            show_header
            echo -e "${CYAN}=== Balanceo de Carga ===${NC}"
            echo "  6 solicitudes al proxy..."
            echo ""
            for i in $(seq 1 6); do
                RESP=$(curl -sk https://$IP_PROXY/ 2>/dev/null | grep -o '"hostname":"[^"]*"' | head -1)
                echo -e "  Solicitud $i → ${GREEN}$RESP${NC}"
                sleep 0.5
            done
            echo ""
            read -p "  Enter para continuar..."
            ;;
        6)
            show_header
            echo -e "${CYAN}=== Recursos ===${NC}"
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
            echo -e "${YELLOW}Reiniciando app...${NC}"
            pm2 restart all
            sleep 2
            pm2 status
            echo ""
            read -p "  Enter para continuar..."
            ;;
        8)
            show_header
            echo -e "${CYAN}=== Backups (DB) ===${NC}"
            ssh -o ConnectTimeout=3 adming9@$IP_DB "ls -lh /var/backups/marketplace/" 2>/dev/null || echo -e "  ${RED}No se pudo conectar a la DB${NC}"
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

```bash
chmod +x ~/scripts/admin_menu.sh
~/scripts/admin_menu.sh
```

---
---

# 🖥️ PASO 3 — VM Proxy (server-174) · Janet

```bash
ssh adming9@192.168.100.174
```

---

## 3.1 Instalar Nginx

```bash
sudo apt update
sudo apt install nginx -y
sudo systemctl enable --now nginx
sudo systemctl status nginx
```

---

## 3.2 Instalar BIND9 (DNS)

```bash
sudo apt install bind9 bind9-utils -y
sudo systemctl enable --now named
```

### Configurar zona marketplace.local

```bash
sudo nano /etc/bind/named.conf.local
```

Agregar:

```
zone "marketplace.local" {
    type master;
    file "/etc/bind/db.marketplace.local";
};
```

### Crear archivo de zona

```bash
sudo nano /etc/bind/db.marketplace.local
```

```
$TTL    604800
@       IN      SOA     ns1.marketplace.local. admin.marketplace.local. (
                              2026060901         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
; Servidores de nombres
@       IN      NS      ns1.marketplace.local.
ns1     IN      A       192.168.100.174

; Registros A
@       IN      A       192.168.100.174
www     IN      A       192.168.100.174
grafana IN      A       192.168.100.174
app1    IN      A       192.168.100.175
app2    IN      A       192.168.100.176
db      IN      A       192.168.100.177
```

### Verificar y reiniciar BIND9

```bash
sudo named-checkconf
sudo named-checkzone marketplace.local /etc/bind/db.marketplace.local
sudo systemctl restart named
sudo systemctl status named
```

### Probar resolución DNS

```bash
dig @localhost marketplace.local
dig @localhost grafana.marketplace.local
dig @localhost app1.marketplace.local
```

---

## 3.3 Generar certificado TLS

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

## 3.4 Instalar Prometheus y Node Exporter

```bash
sudo apt install prometheus prometheus-node-exporter -y
sudo systemctl enable prometheus prometheus-node-exporter
sudo systemctl start prometheus prometheus-node-exporter
```

### Configurar targets

```bash
sudo nano /etc/prometheus/prometheus.yml
```

Reemplazar toda la sección `scrape_configs`:

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
      - targets: ['192.168.100.175:9100']

  - job_name: 'node-app2'
    static_configs:
      - targets: ['192.168.100.176:9100']

  - job_name: 'node-db'
    static_configs:
      - targets: ['192.168.100.177:9100']
```

```bash
sudo systemctl restart prometheus
```

Verificar targets:

```bash
curl -s http://localhost:9090/api/v1/targets | python3 -m json.tool | grep -E '"health"|"job"'
```

Todos deben mostrar `"health": "up"`.

---

## 3.5 Instalar Grafana

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
sudo systemctl status grafana-server
```

### Configurar Grafana

1. Abrir navegador: `http://192.168.100.174:3000`
2. Login: `admin` / `admin` → cambiar contraseña
3. **Connections → Data sources → Add → Prometheus**
4. URL: `http://localhost:9090`
5. Click **Save & test**
6. **Dashboards → New → Import → ID: 1860 → Load**
7. Configurar alerta CPU > 80%

---

## 3.6 Configurar Nginx con HTTPS + Balanceo + DNS

```bash
sudo nano /etc/nginx/sites-available/marketplace
```

```nginx
# Redirección HTTP → HTTPS
server {
    listen 80;
    server_name marketplace.local www.marketplace.local;
    return 301 https://$host$request_uri;
}

# Balanceo HTTPS
upstream marketplace_apps {
    server 192.168.100.175:3000;   # App 1 — Alex
    server 192.168.100.176:3000;   # App 2 — Jhon Christian
}

server {
    listen 443 ssl;
    server_name marketplace.local www.marketplace.local;

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

    server_tokens off;

    location / {
        proxy_pass http://marketplace_apps;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# Grafana por subdominio
server {
    listen 443 ssl;
    server_name grafana.marketplace.local;

    ssl_certificate /etc/nginx/ssl/nginx-selfsigned.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx-selfsigned.key;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Activar y reiniciar:

```bash
sudo rm -f /etc/nginx/sites-enabled/default
sudo ln -sf /etc/nginx/sites-available/marketplace /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

---

## 3.7 Verificación completa del Proxy

```bash
# DNS
dig @localhost marketplace.local
dig @localhost grafana.marketplace.local

# TLS
curl -k https://192.168.100.174

# Redirección
curl -I http://192.168.100.174

# Cabeceras de seguridad
curl -k -I https://192.168.100.174

# Servicios
systemctl status nginx named prometheus grafana-server --no-pager

# Prometheus targets
curl -s http://localhost:9090/api/v1/targets | python3 -m json.tool | grep -E '"health"|"job"'

# Balanceo
for i in {1..6}; do
    echo "--- Solicitud $i ---"
    curl -sk https://192.168.100.174/ | grep -o '"hostname":"[^"]*"'
    sleep 0.5
done
```

---
---

# 🌐 CONFIGURAR DNS EN LAS PCs DE LA FERIA

Para que los visitantes accedan por nombre de dominio, cada PC necesita apuntar su DNS al Proxy.

### Windows

1. **Panel de Control → Red e Internet → Centro de redes → Cambiar configuración del adaptador**
2. Clic derecho en WiFi → **Propiedades**
3. Seleccionar **Protocolo IPv4** → **Propiedades**
4. **Usar las siguientes direcciones DNS:**
   - DNS preferido: `192.168.100.174`
5. Aceptar

### Linux/Mac

```bash
sudo nano /etc/resolv.conf
```

```
nameserver 192.168.100.174
```

### Probar

```
https://marketplace.local         → Página del marketplace
https://grafana.marketplace.local → Dashboards de Grafana
```

---
---

# 📋 CHECKLIST FINAL

## Luis Fernando (DB — server-177)
- [ ] MariaDB activo con bind-address correcto
- [ ] Base de datos db_marketplace con 3 tablas
- [ ] 10 productos insertados
- [ ] UFW configurado (solo App1 y App2 al 3306)
- [ ] Node Exporter activo
- [ ] Script backup_db.sh funcional
- [ ] Script restore_db.sh funcional
- [ ] Cron programado (cada hora)

## Alex Josué (App1 — server-175)
- [ ] Node.js + PM2 con app online
- [ ] .env apuntando a la DB
- [ ] Página marketplace bonita
- [ ] Node Exporter activo
- [ ] Script health_check.sh funcional
- [ ] Cron cada 5 min

## Jhon Christian (App2 — server-176)
- [ ] Node.js + PM2 con app online
- [ ] .env apuntando a la DB
- [ ] Página marketplace bonita
- [ ] Node Exporter activo
- [ ] Menú interactivo con 8 opciones

## Janet (Proxy — server-174)
- [ ] Nginx con HTTPS + balanceo
- [ ] Certificado TLS autofirmado
- [ ] BIND9 con zona marketplace.local
- [ ] Prometheus con 5 targets UP
- [ ] Grafana con dashboard importado
- [ ] Alerta CPU > 80% configurada
- [ ] Cabeceras de seguridad activas

---

# 🎯 DEMO PARA LA FERIA

1. **Mostrar la página** → `https://marketplace.local` (productos, comentarios, visitas)
2. **Recargar varias veces** → hostname cambia (balanceo)
3. **Mostrar Grafana** → `https://grafana.marketplace.local` (4 VMs monitoreadas)
4. **Demostrar failover** → detener App1, página sigue funcionando
5. **Ejecutar health check** → Alex muestra el script
6. **Abrir menú interactivo** → Jhon muestra las opciones con colores
7. **Mostrar backup** → Luis muestra los backups y restaura la BD
8. **Mostrar DNS** → `dig marketplace.local` resuelve a 192.168.100.174
9. **Mostrar TLS** → `curl -k -I https://marketplace.local` (cabeceras de seguridad)
10. **Mostrar UFW** → Luis muestra que solo Apps acceden a MariaDB

---

*Guía del Proyecto Final · Grupo 9 · SIS313 · USFX · Feria 2026*
