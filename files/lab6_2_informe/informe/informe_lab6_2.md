# Informe - Laboratorio 6.2: Backups Automáticos, Rotación y Recuperación

**Universidad San Francisco Xavier de Chuquisaca**  
**Asignatura:** Infraestructura, Plataformas Tecnológicas y Redes (SIS313)  
**Docente:** Ing. Marcelo Quispe Ortega  
**Estudiante:** Luis Fernando Quispe Sullca
**Semestre:** 1/2026  
**Fecha:** 21 de mayo de 2026  

---

## Arquitectura del Entorno

| VM | Hostname | Rol | IP Interna |
|----|----------|-----|------------|
| Lab6.2-Backup | `backup` | Servidor de Backups | `192.168.50.2/29` |
| Lab6.2-DB | `db` | Servidor de Base de Datos | `192.168.50.3/29` |

Acceso desde el anfitrión vía SSH:
- VM Backup: `ssh luisf@127.0.0.1 -p 2224`
- VM DB: `ssh luisf@127.0.0.1 -p 2225`

---

## Ejercicio 1: Configuración de Red y Acceso SSH por Clave

### 1.1 Configuración de IP estática en VM DB (`enp0s8`)

Se editó el archivo `/etc/netplan/50-cloud-init.yaml` en la VM DB para asignar la IP estática `192.168.50.3/29`, con gateway apuntando a la VM Backup (`192.168.50.2`) que actúa como router hacia internet.

<img width="480" height="270" alt="img_01" src="https://github.com/user-attachments/assets/83094ac2-1d3b-41a7-ae22-7648ab9e2bdb" />


> Se configuró `enp0s8` con IP estática `192.168.50.3/29`, DNS apuntando a `192.168.50.2` y ruta por defecto vía `192.168.50.2`. Esto permite que la VM DB acceda a internet a través de la VM Backup.

---

### 1.2 Configuración de IP estática en VM Backup (`enp0s8`)

Se editó el archivo `/etc/netplan/50-cloud-init.yaml` en la VM Backup para asignar la IP estática `192.168.50.2/29`. El adaptador `enp0s3` mantiene DHCP para acceso a internet vía NAT.

<img width="480" height="270" alt="img_02" src="https://github.com/user-attachments/assets/ae75c348-d79f-4497-89af-c1b00bf07764" />


> Se configuró `enp0s8` con IP `192.168.50.2/29` y DNS público `8.8.8.8`. La VM Backup actúa como gateway de la red interna.

---

### 1.3 Verificación de conectividad — VM Backup → VM DB

Tras aplicar `sudo netplan apply` en ambas VMs, se verificó la conectividad bidireccional con `ping`.

<img width="755" height="449" alt="img_03" src="https://github.com/user-attachments/assets/eddc6c4d-f607-4151-852b-ce646445e3bd" />


> Se confirma que `enp0s8` tiene asignada `192.168.50.2/29` en la VM Backup y que el ping a `192.168.50.3` responde sin pérdida de paquetes (0% packet loss), validando la red interna.

---

### 1.4 Verificación de conectividad — VM DB → VM Backup

<img width="755" height="449" alt="img_04" src="https://github.com/user-attachments/assets/9f48c7c2-efba-4f58-8a59-446a809239dc" />


> La VM DB tiene `enp0s8` con `192.168.50.3/29` y el ping a `192.168.50.2` responde correctamente. Al final se observa `sudo netplan apply` siendo ejecutado nuevamente para confirmar la configuración.

---

### 1.5 Habilitación de IP Forwarding y NAT en VM Backup

Se habilitó el reenvío de paquetes IPv4 editando `/etc/sysctl.conf` (descomentando `net.ipv4.ip_forward=1`), se aplicó con `sysctl -p`, se configuró MASQUERADE con `iptables` y se persistió con `netfilter-persistent`.

<img width="755" height="116" alt="img_05" src="https://github.com/user-attachments/assets/bdfd84f6-55d7-43e2-bda5-91980e35d763" />


> `sysctl -p` confirma `net.ipv4.ip_forward = 1`. La regla `iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE` permite que el tráfico de la VM DB salga a internet usando la IP de la VM Backup. `netfilter-persistent save` persiste las reglas entre reinicios.

---

### 1.6 Verificación de acceso a internet desde VM DB

Con el forwarding activo, se verificó que la VM DB puede alcanzar internet a través de la VM Backup.

<img width="755" height="161" alt="img_06" src="https://github.com/user-attachments/assets/7c317f56-9cea-45f0-bd40-5e83e6010810" />


> La VM DB logra hacer ping a `8.8.8.8` (DNS de Google) con TTL=62, confirmando que el enrutamiento funciona correctamente a través de la VM Backup.

---

### 1.7 Configuración de SSH sin contraseña (Backup → DB)

Se generó un par de claves `ed25519` dedicadas para backup, se copió la clave pública a la VM DB con `ssh-copy-id`, y se verificó la conexión sin contraseña.

<img width="755" height="582" alt="img_07" src="https://github.com/user-attachments/assets/6713e18c-f3fb-49c8-8b91-0fc1629bb4b3" />


> Se generó el par de claves en `/home/luisf/.ssh/id_backup` sin passphrase (necesario para automatización con cron). `ssh-copy-id` instaló la clave pública en la VM DB. La prueba final devuelve `db` y `SSH sin contraseña OK` sin pedir credenciales, confirmando el canal seguro para backups automatizados.

---

## Ejercicio 2: Preparar el Servidor de Base de Datos

### 2.1 Instalación de MariaDB y Nginx — verificación de estado

Se instalaron MariaDB y Nginx en la VM DB y se habilitaron con `systemctl enable --now`. Ambos servicios quedaron activos.

<img width="1166" height="694" alt="img_08" src="https://github.com/user-attachments/assets/7fbd9265-8484-44a2-965b-5161fd2063c9" />


> MariaDB 10.11.14 y Nginx están en estado `active (running)` y marcados como `enabled`, lo que garantiza que arrancan automáticamente con el sistema.

---

### 2.2 Creación de base de datos y datos de prueba

Se creó la base de datos `lab62_db` con la tabla `productos` y se insertaron 3 registros de prueba.

<img width="1070" height="191" alt="img_09" src="https://github.com/user-attachments/assets/0140047a-14ad-4668-b670-83ac69ed2824" />


> La tabla `productos` contiene los 3 registros correctamente insertados: Laptop ($1200.00), Mouse ($25.50) y Teclado ($45.00). Estos datos serán el objetivo de los backups.

---

### 2.3 Archivo de credenciales y contenido web

Se creó `/etc/.my.cnf` con las credenciales para `mysqldump`, se protegió con `chmod 600` y se creó el contenido web de prueba en `/var/www/html/`.
<img width="1070" height="323" alt="img_10" src="https://github.com/user-attachments/assets/1b3cc930-2873-4d46-b2b8-e35e98202f11" />


> El archivo `/etc/.my.cnf` tiene permisos `rw-------` (solo root puede leerlo), evitando exposición de credenciales. El `cat index.html` confirma que el contenido web de prueba fue creado correctamente con el título y párrafo del laboratorio.

---

## Ejercicio 3: Backup Local de Archivos con `tar`

Se prepararon los directorios de trabajo en la VM Backup y se creó el script `file_backup.sh`.

<img width="1070" height="448" alt="img_11" src="https://github.com/user-attachments/assets/fe6448d0-e103-4107-b70a-99a47f870dbe" />


> Se crearon los directorios `/opt/backup_scripts`, `/var/backups/data_center` y `/var/backups/files`. El script `file_backup.sh` usa `tar -czf` para comprimir con gzip. El flag `-C $(dirname)` evita rutas absolutas dentro del archivo comprimido, facilitando restauraciones portables.

---

## Ejercicio 4: Backup Remoto de Base de Datos mediante SSH

### 4.1 Script `db_backup.sh`

Se creó el script que ejecuta `mysqldump` remotamente en la VM DB vía SSH y transmite el resultado comprimido a la VM Backup.
<img width="1070" height="420" alt="img_12" src="https://github.com/user-attachments/assets/d36b1ef0-f932-4787-978a-177bac50cce9" />


> El script conecta por SSH usando la clave dedicada `id_backup`, ejecuta `mysqldump` en la VM DB, transmite la salida por el túnel SSH cifrado, la comprime con `gzip` en tiempo real y la guarda localmente. Nunca se almacena el SQL sin comprimir.

---

### 4.2 Configuración de sudoers para mysqldump sin contraseña

Para que el script automatizado pueda ejecutar `mysqldump` con sudo en la VM DB sin interacción, se agregó una regla en `visudo`.
<img width="1070" height="420" alt="img_13" src="https://github.com/user-attachments/assets/f572ad1e-4e21-4d04-be97-13dead4091c2" />


> La línea `luisf ALL=(ALL) NOPASSWD: /usr/bin/mysqldump` permite que únicamente ese comando se ejecute sin contraseña, aplicando el principio de mínimo privilegio — solo lo necesario para automatizar el backup.

---

### 4.3 Ejecución exitosa del backup remoto y verificación con `zcat`

Tras corregir el `.my.cnf` (usando `localhost` en lugar de `127.0.0.1` para conexión por socket), el backup se ejecutó correctamente y se inspeccionó su contenido.

<img width="1070" height="411" alt="img_14" src="https://github.com/user-attachments/assets/70da24a6-8017-4945-b044-7e46f17ec3ff" />


> El backup pesa 833 bytes y contiene SQL válido. `zcat | head -20` muestra el header de MariaDB dump, la base de datos `lab62_db` y el inicio de la estructura de la tabla `productos`, confirmando que el backup es funcional.

---

### 4.4 Corrección del `.my.cnf` — prueba local en VM DB

Se verificó que `mysqldump` funciona correctamente desde la VM DB con el archivo de credenciales corregido.

<img width="1070" height="251" alt="img_15" src="https://github.com/user-attachments/assets/dcecafc2-d16b-4c37-9d98-11ba8fbea6fa" />


> `visudo: sudoers.tmp unchanged` confirma que el archivo sudoers es válido. La prueba local de `mysqldump --defaults-file=/etc/.my.cnf lab62_db | head -10` muestra el header SQL correctamente. El error `errno 32 on write` es normal: `head` cierra el pipe después de 10 líneas.

---

## Ejercicio 5: Verificación de Integridad de Backups

Se creó y ejecutó el script `verify_backup.sh` que valida 4 aspectos críticos del backup.
<img width="505" height="795" alt="img_16" src="https://github.com/user-attachments/assets/20b860ac-ec5d-4c1b-8ffa-5e9229384d28" />


> Los 4 checks pasan exitosamente:
> - **Tamaño:** 833 bytes (no vacío)
> - **gzip válido:** el archivo no está corrupto
> - **CREATE TABLE presente:** contiene estructura SQL
> - **INSERT INTO presente:** contiene datos reales
>
> `gzip -t` testea la integridad sin extraer el archivo. `zcat | grep` inspecciona el contenido en memoria sin crear archivos temporales.

---

## Ejercicio 6: Reporte de Estado de Backups

Se creó y ejecutó el script `backup_report.sh` que genera un reporte consolidado.
<img width="891" height="810" alt="img_17" src="https://github.com/user-attachments/assets/76cb82ec-12a6-4bf0-ae60-461040b4830a" />


> El reporte muestra ambos backups existentes con sus metadatos completos: el backup de BD (`lab62_db-20260521_2105.sql.gz`, 833 bytes) y el backup de archivos (`web-20260521_2102.tar.gz`, 109 bytes). El uso de disco en `/var/backups` es del 46% sobre 12GB disponibles. El script usa `awk` para formatear columnas y `df -h` para el estado del disco.

---

## Ejercicio 7: Rotación de Logs de Nginx

### 7.1 Generación de tráfico y verificación de logs

Se generaron 20 peticiones HTTP al servidor Nginx para poblar el log antes de rotarlo.

<img width="891" height="109" alt="img_18" src="https://github.com/user-attachments/assets/3f668783-9e4b-45db-aa78-3e1883c37531" />


> El bucle `for i in {1..20}; do curl -s http://localhost/` generó 20 entradas en el log de acceso. `wc -l` confirma 20 líneas y `ls -lh` muestra el archivo de 1.6K listo para ser rotado.

---

### 7.2 Ejecución de la rotación y verificación

Se creó y ejecutó `log_rotate.sh` en la VM DB, que archiva el log activo y crea uno nuevo vacío.
<img width="891" height="491" alt="img_19" src="https://github.com/user-attachments/assets/a64b90f6-2b57-422c-9169-7b5d63ca50e8" />


> El script renombra el log activo, lo comprime en `archive/access-20260521.tar.gz` (218 bytes), elimina el temporal y recrea `access.log` vacío con permisos `www-data:adm 640`. Nginx puede seguir escribiendo sin interrupción porque trabaja con el descriptor de archivo abierto, no con el nombre.

---

## Ejercicio 8: Retención de Datos y Limpieza Automática

Se creó y ejecutó `cleanup_old_backups.sh` que aplica la política de retención de 7 días.

<img width="891" height="591" alt="img_20" src="https://github.com/user-attachments/assets/bea847ba-3d6c-441b-b32b-9f4ca9c94f4c" />


> El script reporta 0 eliminaciones porque todos los backups tienen menos de 7 días — comportamiento correcto. `find -mtime +7` busca archivos modificados hace más de 7 días; `-delete` los elimina. En producción, este script mantiene el almacenamiento controlado eliminando backups obsoletos automáticamente.

---

## Ejercicio 9: Planificación con Cron

Se configuraron 3 tareas programadas en el crontab de root de la VM Backup.

<img width="891" height="591" alt="img_21" src="https://github.com/user-attachments/assets/5ecfba7b-4e7a-4486-a224-ea01563a40fa" />


> Las 3 tareas quedan registradas:
> - `0 0 * * *` → Backup de BD diario a medianoche
> - `0 2 * * 0` → Backup de archivos los domingos a las 02:00
> - `0 3 * * 1` → Limpieza de backups antiguos los lunes a las 03:00
>
> `> /dev/null 2>&1` suprime la salida para evitar emails de cron al administrador.

---

## Ejercicio 10: Restauración de Archivos desde `tar`

Se creó contenido web real, se hizo un nuevo backup, se simuló la pérdida y se restauró exitosamente.

<img width="891" height="362" alt="img_22" src="https://github.com/user-attachments/assets/d1526ccd-bb7e-487b-b156-ffd8853499f9" />


> La secuencia completa demuestra:
> 1. Creación del contenido web (`index.html` + `docs/secreto.txt`)
> 2. Backup con `file_backup.sh` → `web-20260521_2114.tar.gz` (279 bytes)
> 3. Eliminación simulada con `rm -rf /var/www/html/*` → directorio vacío
> 4. Restauración con `tar -xzf ... -C /var/www/` → recupera `docs/` e `index.html`
> 5. `cat index.html` confirma el contenido restaurado correctamente

---

## Ejercicio 11: Restauración de Base de Datos desde SQL

### 11.1 Simulación del desastre — DROP DATABASE

Se eliminó completamente la base de datos `lab62_db` para simular una pérdida total. El intento de restaurar directamente desde la VM DB falló porque el backup estaba en la VM Backup.

<img width="891" height="472" alt="img_23" src="https://github.com/user-attachments/assets/b9471dfe-0a7b-49f2-90c9-5f82deb54fae" />


> `DROP DATABASE lab62_db` elimina completamente la base de datos. El intento de `zcat` falla con "No such file or directory" porque el backup reside en `/var/backups/data_center/` de la VM Backup, no en la VM DB. Esto ilustra la importancia de tener el backup accesible desde donde se necesita restaurar.

---

### 11.2 Transferencia del backup con `scp`

Se copió el backup desde la VM Backup hacia la VM DB usando `scp` con la clave SSH.

<img width="891" height="87" alt="img_24" src="https://github.com/user-attachments/assets/9146479b-f1ef-4e2f-bd75-70b4c34b8381" />


> `scp -i /home/luisf/.ssh/id_backup` transfiere el archivo `lab62_db-20260521_2105.sql.gz` a `/tmp/` de la VM DB usando el canal SSH seguro ya configurado. La transferencia es rápida al estar en red interna.

---

### 11.3 Primer intento de restauración — error sin base de datos destino

El primer intento de restaurar sin especificar base de datos destino falló porque el dump no incluye `CREATE DATABASE`.

<img width="526" height="391" alt="img_25" src="https://github.com/user-attachments/assets/24f1b8ed-1742-4964-b103-3ca8ca263cea" />


> El error `ERROR 1046: No database selected` ocurre porque `mysqldump` sin la opción `--databases` no incluye el comando `CREATE DATABASE` en el dump. Es necesario crear la base de datos vacía primero y luego restaurar especificando el destino.

---

### 11.4 Restauración exitosa y medición del RTO

Se creó la base de datos vacía y se restauró correctamente en **0.1 segundos**.
<img width="617" height="281" alt="img_26" src="https://github.com/user-attachments/assets/b7a73c4f-ea59-41ec-b61b-1b8161b3326d" />


> La secuencia correcta: `CREATE DATABASE lab62_db` + `zcat ... | sudo mysql lab62_db`. El tiempo real de restauración es de **0.100 segundos** — este es el **RTO (Recovery Time Objective)** medido para este servicio. El segundo intento confirma la restauración exitosa con el error `database exists`.

---

### 11.5 Verificación final — datos completamente recuperados

Se verificó que la base de datos fue restaurada con todos sus registros intactos.

<img width="617" height="292" alt="img_27" src="https://github.com/user-attachments/assets/615ceaeb-809a-4f92-82ed-e62655e79152" />


> `SHOW DATABASES` confirma que `lab62_db` existe nuevamente. `SELECT * FROM lab62_db.productos` muestra los 3 registros originales completamente recuperados: Laptop (1200.00), Mouse (25.50) y Teclado (45.00). La restauración fue 100% exitosa.

---

## Resumen de Resultados

| Ejercicio | Componente | Resultado |
|-----------|-----------|-----------|
| 1 | Red interna + IP Forwarding + SSH por clave | ✅ Completado |
| 2 | MariaDB + Nginx + datos de prueba | ✅ Completado |
| 3 | Script `file_backup.sh` con tar | ✅ Completado |
| 4 | Script `db_backup.sh` remoto via SSH | ✅ Completado |
| 5 | Script `verify_backup.sh` — 4 checks OK | ✅ Completado |
| 6 | Script `backup_report.sh` | ✅ Completado |
| 7 | Rotación de logs Nginx | ✅ Completado |
| 8 | Política de retención 7 días | ✅ Completado |
| 9 | Cron — 3 tareas programadas | ✅ Completado |
| 10 | Restauración de archivos desde tar | ✅ Completado |
| 11 | Restauración de BD — RTO medido | ✅ Completado |

## Análisis de RTO

| Servicio | Tiempo de Restauración (RTO) |
|---------|------------------------------|
| Base de datos `lab62_db` | **~0.1 segundos** |
| Archivos web `/var/www/html` | **< 1 segundo** |

El RTO extremadamente bajo se debe al tamaño reducido de los datos en este entorno de laboratorio. En producción, con bases de datos de varios GB, el RTO podría variar entre minutos y horas dependiendo del tamaño del backup y el ancho de banda disponible.

## Lecciones Aprendidas

- El archivo de credenciales `/etc/.my.cnf` debe usar `host = localhost` (socket Unix) y no `127.0.0.1` (TCP) en MariaDB sobre Ubuntu/Debian, donde root usa autenticación por socket.
- Los backups deben almacenarse en una ubicación diferente al servidor de datos — en este caso la VM Backup actúa como repositorio externo.
- La verificación de integridad (`gzip -t`, `zcat | grep`) es tan importante como el proceso de backup en sí.
- Para restaurar un dump sin `--databases`, es necesario crear primero la base de datos destino manualmente.
