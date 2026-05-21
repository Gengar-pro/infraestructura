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

## 🛠️ Arquitectura del Entorno

| VM | Hostname | Rol | IP Interna |
|---|---|---|---|
| Lab5.2-Server | `server` | Servidor Objetivo (Nginx, SSH, Fail2ban) | `192.168.30.2` |
| Lab5.2-Attacker | `attacker` | Máquina Atacante (Hydra, Nmap) | `192.168.30.3` |

- **Red interna:** `192.168.30.0/29`
- **Gateway:** La VM Server actúa como router para la VM Attacker

---

## 💻 Ejercicio 1: Configuración de Red Estática

### 1.1 Configuración de IP estática en la VM Server

Se editó el archivo `/etc/netplan/50-cloud-init.yaml` en la VM Server asignando la IP estática `192.168.30.2/29` a la interfaz `enp0s8`.

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

<img width="304" height="233" alt="ej1_01_netplan_server" src="https://github.com/user-attachments/assets/fe22da53-ccbe-44c5-8414-bba359a5f73b" />


La interfaz `enp0s3` usa DHCP para salida a Internet (NAT) y `enp0s8` recibe la IP estática `192.168.30.2/29` para la red interna del laboratorio.

---

### 1.2 Configuración de IP estática en la VM Attacker

Se configuró la VM Attacker con IP `192.168.30.3/29` apuntando al Server como gateway por defecto.

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```
<img width="380" height="329" alt="ej1_02_netplan_attacker" src="https://github.com/user-attachments/assets/8e4b59ac-eb48-472e-8133-da1dff5a50b6" />


La ruta `default via 192.168.30.2` indica que todo el tráfico del Attacker pasa por el Server, que actúa como router/gateway.

---

### 1.3 Verificación de interfaces en la VM Attacker

Se ejecutó `ip a` en la VM Attacker para confirmar que la IP `192.168.30.3/29` quedó asignada correctamente.

```bash
ip a
```

<img width="773" height="329" alt="ej1_03_ipa_attacker" src="https://github.com/user-attachments/assets/a5a6cfa2-7a38-429d-8f48-5dcad2c2a669" />


La interfaz `enp0s8` muestra la IP `192.168.30.3/29` activa, confirmando la configuración correcta de red en la VM Attacker.

---

### 1.4 Aplicación de netplan y verificación en la VM Server

Se aplicó la configuración con `netplan apply` y se verificó que la IP `192.168.30.2/29` quedó activa en `enp0s8`.

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
sudo netplan apply
ip a
```

<img width="773" height="363" alt="ej1_04_netplan_apply_server" src="https://github.com/user-attachments/assets/bc7e6dd2-b13a-43e8-b6e9-511a7e3739f5" />


La interfaz `enp0s8` del Server muestra `inet 192.168.30.2/29`, confirmando que la red interna está correctamente configurada.

---

### 1.5 Habilitación de IP Forwarding

Se habilitó el reenvío de paquetes editando `/etc/sysctl.conf` para que el Server pueda funcionar como router.

```bash
sudo nano /etc/sysctl.conf
sudo sysctl -p
```

<img width="773" height="72" alt="ej1_05_ipforward" src="https://github.com/user-attachments/assets/5080931c-9100-4e2f-9ef4-64fa593a4e67" />


La salida `net.ipv4.ip_forward = 1` confirma que el kernel está reenviando paquetes entre interfaces, requisito para que el Attacker tenga conectividad a través del Server.

---

### 1.6 Configuración de NAT Masquerade y persistencia

Se configuró la regla NAT para que los paquetes del Attacker salgan con la IP del Server, y se guardaron las reglas de forma persistente.

```bash
sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
sudo apt install iptables-persistent
sudo netfilter-persistent save
```

<img width="773" height="737" alt="ej1_06_nat_masquerade" src="https://github.com/user-attachments/assets/2a0c9b1d-a0ee-48e2-9368-445b6b128365" />


La instalación de `iptables-persistent` y la ejecución de `netfilter-persistent save` garantizan que las reglas NAT sobrevivan reinicios del servidor.

---

### 1.7 Verificación de conectividad entre VMs

Se realizaron pings desde la VM Attacker hacia el Server y hacia Internet para confirmar conectividad completa.

```bash
ping -c 3 192.168.30.2
ping -c 3 8.8.8.8
```

<img width="773" height="293" alt="ej1_07_ping" src="https://github.com/user-attachments/assets/e4663d2a-42cb-499f-8a98-0af2f3c0fad1" />


Ambos pings son exitosos con 0% de pérdida de paquetes, confirmando que la red interna funciona y que el Server enruta correctamente el tráfico del Attacker hacia Internet.

---

## 💻 Ejercicio 2: Instalación del Servidor Objetivo

### 2.1 Instalación y habilitación de Nginx y SSH

Se instalaron los servicios Nginx y OpenSSH Server, habilitándolos para que inicien automáticamente.

```bash
sudo apt install nginx openssh-server -y
sudo systemctl enable --now nginx
sudo systemctl enable --now ssh
```

<img width="773" height="149" alt="ej2_01_instalacion_nginx_ssh" src="https://github.com/user-attachments/assets/077cad76-940e-43c2-859d-e6b0c48b0575" />


Los servicios `nginx` y `ssh` quedaron habilitados con `enable --now`, lo que significa que arrancan automáticamente en cada inicio del sistema.

---

### 2.2 Creación del sitio web de prueba

Se creó una página HTML simple en `/var/www/html/index.html` como objetivo web del laboratorio.

```bash
sudo bash -c 'echo "<h1>Servidor de Prueba - Lab 5.2</h1>" > /var/www/html/index.html'
sudo bash -c 'echo "<p>Objetivo de simulación de intrusiones</p>" >> /var/www/html/index.html'
cat /var/www/html/index.html
```

<img width="773" height="102" alt="ej2_02_sitio_web" src="https://github.com/user-attachments/assets/a04ec900-16e0-4e88-85a3-9dd58e8d546f" />


El archivo `index.html` contiene el contenido de prueba que Nginx servirá como objetivo del reconocimiento web en el Ejercicio 5.

---

### 2.3 Verificación de servicios activos y puertos

Se verificó el estado de ambos servicios y los puertos en escucha.

```bash
sudo systemctl status nginx && sudo systemctl status sshd
sudo ss -tulnp | grep -E "nginx|sshd"
```

<img width="1062" height="648" alt="ej2_03_status_nginx_ssh" src="https://github.com/user-attachments/assets/6678796b-afc7-4fa7-88c2-f083f127c1ac" />


Nginx está `active (running)` en el puerto 80 y SSH en el puerto 22, ambos escuchando en todas las interfaces. Los logs del servicio SSH muestran las conexiones previas desde la PC anfitriona vía NAT.

---

## 💻 Ejercicio 3: Instalación y Configuración de Fail2ban

### 3.1 Instalación de Fail2ban

Se instaló Fail2ban como sistema de prevención de intrusiones basado en host (HIPS).

```bash
sudo apt install fail2ban -y
sudo systemctl enable --now fail2ban
sudo systemctl status fail2ban.service
```
<img width="1062" height="304" alt="ej3_01_instalacion_fail2ban" src="https://github.com/user-attachments/assets/ea69dc99-8f71-4ca6-bcfe-730c43248649" />


Fail2ban está `active (running)` con el servidor listo (`Server ready`), monitoreando los logs del sistema en tiempo real.

---

### 3.2 Configuración del archivo jail.local

Se creó el archivo de configuración con las reglas de detección y bloqueo para SSH y Nginx.

```bash
sudo nano /etc/fail2ban/jail.local
sudo cat /etc/fail2ban/jail.local
```

<img width="1062" height="345" alt="ej3_02_jail_local" src="https://github.com/user-attachments/assets/b34553a9-c765-45c8-a91d-d78a62d86ce0" />


La configuración establece: `bantime=600` (10 min de bloqueo), `findtime=600` (ventana de detección), `maxretry=3` (máximo 3 intentos). Se habilitaron las jails `[sshd]` y `[nginx-http-auth]`.

---

### 3.3 Verificación de jails activas

Se verificó que Fail2ban reconoció ambas jails y está listo para detectar ataques.

```bash
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

<img width="1062" height="285" alt="ej3_03_jails_activas" src="https://github.com/user-attachments/assets/b9b8f4ac-d8cf-45c1-9753-5a45959b47c2" />


Fail2ban muestra 2 jails activas: `nginx-http-auth` y `sshd`. La jail `sshd` aparece con 0 intentos fallidos y 0 IPs baneadas, estado inicial antes del ataque.

---

## 💻 Ejercicio 4: Simulación de Ataque de Fuerza Bruta a SSH

### 4.1 Ejecución del ataque con Hydra

Desde la VM Attacker se lanzó el ataque de fuerza bruta contra el SSH del Server usando el diccionario `passwords.txt`.

```bash
hydra -l luisf -P passwords.txt ssh://192.168.30.2 -t 4 -V
```

<img width="1565" height="285" alt="ej4_01_hydra_ataque" src="https://github.com/user-attachments/assets/bd63b7e3-c275-4b64-b698-9f9f856575a6" />


Hydra intentó 10 contraseñas del diccionario contra el usuario `luisf` en `192.168.30.2:22`. Todos los intentos fallaron (`0/0`), generando el tráfico malicioso que Fail2ban detectará.

---

### 4.2 Verificación de intentos fallidos en auth.log

Desde el Server se analizaron los registros para confirmar que el ataque fue capturado en los logs.

```bash
sudo grep "Failed password" /var/log/auth.log
sudo grep "Failed password" /var/log/auth.log | wc -l
```

<img width="1565" height="221" alt="ej4_02_authlog_failed" src="https://github.com/user-attachments/assets/52db17d1-1479-4b64-8634-91c7067a0b5b" />


El log registra 10 intentos de contraseña fallidos desde `192.168.30.3` en un intervalo de 2 segundos (16:28:42 a 16:28:44), patrón típico de un ataque automatizado de fuerza bruta.

---

### 4.3 Confirmación del bloqueo automático por Fail2ban

Se verificó que Fail2ban detectó los intentos y baneó la IP atacante automáticamente.

```bash
sudo grep "Failed password" /var/log/auth.log | wc -l
sudo fail2ban-client status sshd
```

<img width="1565" height="321" alt="ej4_03_fail2ban_banned" src="https://github.com/user-attachments/assets/d74390be-ee31-40f5-bf03-edab838c27a8" />


Fail2ban registra: `Total failed: 8`, `Currently banned: 1`, `Banned IP list: 192.168.30.3`. La IP atacante fue bloqueada automáticamente tras superar el `maxretry=3` dentro de la ventana `findtime=600`.

---

### 4.4 Verificación de regla iptables y desbaneo

Se confirmó la regla de firewall generada por Fail2ban y se procedió al desbaneo para continuar el laboratorio.

```bash
sudo iptables -L -n | grep f2b
sudo fail2ban-client set sshd unbanip 192.168.30.3
```

<img width="512" height="77" alt="ej4_04_iptables_desbaneo" src="https://github.com/user-attachments/assets/3f5a040f-2529-43c4-aa24-0bfed84d5d24" />


La regla `f2b` en iptables bloqueó el tráfico de `192.168.30.3`. El desbaneo devuelve `1` confirmando que la IP fue removida exitosamente de la lista de bloqueados.

---

## 💻 Ejercicio 5: Escaneo y Reconocimiento Web

### 5.1 Escaneo de puertos con Nmap

Desde la VM Attacker se realizó un escaneo de servicios para identificar puertos y versiones.

```bash
nmap -sV 192.168.30.2
```

<img width="671" height="221" alt="ej5_01_nmap" src="https://github.com/user-attachments/assets/337f8d42-9df0-44a3-a93e-bb1761e9ed8e" />


Nmap detectó los puertos **22/tcp (OpenSSH 9.6p1)** y **80/tcp (Nginx 1.24.0)** abiertos. Esta información simula la fase de reconocimiento de un atacante real, identificando servicios y versiones para planificar el ataque.

---

### 5.2 Análisis del access.log de Nginx

Desde el Server se analizaron los logs de Nginx para detectar las peticiones del escaneo.

```bash
sudo grep " 404 " /var/log/nginx/access.log | head -20
sudo grep " 404 " /var/log/nginx/access.log | awk '{print $1}' | sort | uniq -c | sort -nr
```

<img width="1330" height="375" alt="ej5_02_access_log_404" src="https://github.com/user-attachments/assets/93b9fcf1-dc01-4434-a678-ff847ff12c44" />


El log muestra 54 peticiones 404 desde `192.168.30.3`: 4 del Nmap Scripting Engine (rutas como `/HNAP1`, `/sdk`, `/evox/about`) y las primeras del loop curl (`/admin1` a `/admin16`). El conteo final confirma **54 peticiones** en total desde la IP atacante.

---

### 5.3 Reconocimiento web con curl (50 peticiones)

Desde el Attacker se generaron 50 peticiones a rutas inexistentes simulando un escáner de directorios.

```bash
for i in $(seq 1 50); do
  curl -s -o /dev/null -w "%{http_code}" http://192.168.30.2/admin$i
  echo " -> /admin$i"
done
```

<img width="1330" height="823" alt="ej5_03_curl_loop" src="https://github.com/user-attachments/assets/86877817-b68b-468f-a593-8b71b581ef83" />


Las 50 peticiones retornan código `404` para cada ruta `/admin1` a `/admin50`. Este patrón de peticiones consecutivas a rutas incrementales es un indicador claro de reconocimiento automatizado.

---

## 💻 Ejercicio 6: Ciclo de Respuesta a Incidentes

### FASE 1 — IDENTIFICAR

Se ejecutaron comandos forenses para determinar origen, tipo y alcance del ataque.

```bash
sudo grep " 404 " /var/log/nginx/access.log | awk '{print $1}' | sort | uniq -c | sort -nr
sudo lastb | head -20
sudo grep "Failed password" /var/log/auth.log | tail -20
sudo grep " 404 " /var/log/nginx/access.log | awk '{print $1}' | sort | uniq -c | sort -nr | head -10
```
<img width="1330" height="435" alt="ej6_01_identificar" src="https://github.com/user-attachments/assets/fa06df97-c605-4a9d-b4e6-7a50a9d54d2e" />


**Hallazgos:**
- `lastb` muestra 8 intentos de login fallidos desde `192.168.30.3` el 21/05/2026 a las 16:28
- `auth.log` registra los intentos fallidos con timestamps exactos
- El conteo de 404 confirma **54 peticiones** desde la IP atacante
- **Servicios atacados:** SSH y HTTP
- **IP atacante:** `192.168.30.3`
- **Tipo:** Fuerza bruta SSH + reconocimiento web

---

### FASE 2 — CONTENER

Fail2ban aplicó contención automática. Se intentó bloqueo adicional con UFW.

```bash
sudo ufw deny from 192.168.30.3
```

> **Nota:** UFW no estaba instalado en el servidor. La contención fue cubierta por las reglas iptables que Fail2ban generó automáticamente, siendo equivalente en efectividad.

---

### FASE 3 — ERRADICAR

Se verificó que no hubo accesos exitosos desde la IP atacante.

```bash
sudo grep "Accepted" /var/log/auth.log
```

<img width="1330" height="102" alt="ej6_02_erradicar_accepted" src="https://github.com/user-attachments/assets/19ff53dd-fa20-449e-8ea0-00476ab7078e" />


Los únicos accesos aceptados son desde `10.0.2.2` (PC anfitriona vía NAT port forwarding). **Ningún acceso exitoso** proviene de la IP atacante `192.168.30.3`, confirmando que el servidor no fue comprometido.

---

### FASE 4 — RECUPERAR

Se verificó que los tres servicios críticos operan correctamente tras el ataque.

```bash
sudo systemctl status sshd nginx fail2ban
sudo fail2ban-client set sshd unbanip 192.168.30.3
```

<img width="1330" height="863" alt="ej6_03_recuperar_servicios" src="https://github.com/user-attachments/assets/2cc34043-6124-4dd9-93c1-8fe1fd9cd0a1" />


Los tres servicios están en estado `active (running)`: SSH desde las 15:57 UTC, Nginx desde las 16:13 UTC y Fail2ban desde las 16:23 UTC. El servidor operó con normalidad durante y después del ataque.

---

### FASE 5 — DOCUMENTAR

Se creó el registro oficial del incidente en el servidor.

```bash
nano ~/incidente_001.md
```

<img width="1330" height="863" alt="ej6_04_documentar_incidente" src="https://github.com/user-attachments/assets/df04ddcc-f2b1-489e-bfe7-a3a45e2ada97" />


El archivo `incidente_001.md` documenta todos los campos del incidente: fecha, hora, IP atacante, servicios afectados, tipo de ataque, intentos fallidos, accesos exitosos, acciones tomadas y estado final.

---

## 📊 Resumen del Incidente #001

| Campo | Detalle |
|---|---|
| **Fecha** | 21 de Mayo de 2026 |
| **Hora de detección** | 16:28 UTC |
| **IP Atacante** | `192.168.30.3` |
| **Servicios afectados** | SSH (`sshd`) y HTTP (`nginx`) |
| **Tipo de ataque** | Fuerza bruta SSH + reconocimiento web (404 scan) |
| **Intentos fallidos SSH** | 8 registrados en `auth.log` |
| **Peticiones 404 web** | 54 (4 Nmap NSE + 50 curl) |
| **Accesos exitosos del atacante** | **Ninguno** |
| **Respuesta automática** | Fail2ban baneó `192.168.30.3` tras 3 intentos |
| **Estado final** | ✅ Resuelto |

---

## ✅ Conclusiones

1. **Fail2ban como HIPS:** La configuración de las jails `sshd` y `nginx-http-auth` demostró ser efectiva bloqueando el ataque automáticamente tras el tercer intento fallido, en los primeros segundos del ataque.

2. **Análisis de logs:** Los archivos `auth.log` y `access.log` son fuentes críticas de información. Permitieron identificar la IP atacante, el tipo de ataque, los timestamps exactos y confirmar que no hubo compromiso del sistema.

3. **Ciclo de respuesta a incidentes:** Las cinco fases (identificar, contener, erradicar, recuperar, documentar) permiten una respuesta estructurada que minimiza el tiempo de exposición y garantiza que no queden vectores de ataque activos.

4. **Integridad del servidor:** A pesar del ataque simulado, ninguna credencial fue comprometida y todos los servicios continuaron operando con normalidad, validando la efectividad de las medidas de seguridad implementadas.

---

*Informe elaborado por Luis F. — SIS313 — Universidad San Francisco Xavier de Chuquisaca — 21/05/2026*
