# Informe - Laboratorio 6.2: Backups Automáticos, Rotación y Recuperación

**Universidad San Francisco Xavier de Chuquisaca**  
**Asignatura:** Infraestructura, Plataformas Tecnológicas y Redes (SIS313)  
**Docente:** Ing. Marcelo Quispe Ortega  
**Estudiante:** Luis F.  
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

![Configuración netplan VM DB](imagenes/img_01.png)

> Se configuró `enp0s8` con IP estática `192.168.50.3/29`, DNS apuntando a `192.168.50.2` y ruta por defecto vía `192.168.50.2`. Esto permite que la VM DB acceda a internet a través de la VM Backup.

---

### 1.2 Configuración de IP estática en VM Backup (`enp0s8`)

Se editó el archivo `/etc/netplan/50-cloud-init.yaml` en la VM Backup para asignar la IP estática `192.168.50.2/29`. El adaptador `enp0s3` mantiene DHCP para acceso a internet vía NAT.

![Configuración netplan VM Backup](imagenes/img_02.png)

> Se configuró `enp0s8` con IP `192.168.50.2/29` y DNS público `8.8.8.8`. La VM Backup actúa como gateway de la red interna.

---

### 1.3 Verificación de conectividad — VM Backup → VM DB

Tras aplicar `sudo netplan apply` en ambas VMs, se verificó la conectividad bidireccional con `ping`.

![ip a y ping desde Backup a DB](imagenes/img_03.png)

> Se confirma que `enp0s8` tiene asignada `192.168.50.2/29` en la VM Backup y que el ping a `192.168.50.3` responde sin pérdida de paquetes (0% packet loss), validando la red interna.

---

### 1.4 Verificación de conectividad — VM DB → VM Backup

![ip a y ping desde DB a Backup](imagenes/img_04.png)

> La VM DB tiene `enp0s8` con `192.168.50.3/29` y el ping a `192.168.50.2` responde correctamente. Al final se observa `sudo netplan apply` siendo ejecutado nuevamente para confirmar la configuración.

---

### 1.5 Habilitación de IP Forwarding y NAT en VM Backup

Se habilitó el reenvío de paquetes IPv4 editando `/etc/sysctl.conf` (descomentando `net.ipv4.ip_forward=1`), se aplicó con `sysctl -p`, se configuró MASQUERADE con `iptables` y se persistió con `netfilter-persistent`.

![IP Forwarding y NAT configurados](imagenes/img_05.png)

> `sysctl -p` confirma `net.ipv4.ip_forward = 1`. La regla `iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE` permite que el tráfico de la VM DB salga a internet usando la IP de la VM Backup. `netfilter-persistent save` persiste las reglas entre reinicios.

---

### 1.6 Verificación de acceso a internet desde VM DB

Con el forwarding activo, se verificó que la VM DB puede alcanzar internet a través de la VM Backup.

![Ping a 8.8.8.8 desde DB](imagenes/img_06.png)

> La VM DB logra hacer ping a `8.8.8.8` (DNS de Google) con TTL=62, confirmando que el enrutamiento funciona correctamente a través de la VM Backup.

---

### 1.7 Configuración de SSH sin contraseña (Backup → DB)

Se generó un par de claves `ed25519` dedicadas para backup, se copió la clave pública a la VM DB con `ssh-copy-id`, y se verificó la conexión sin contraseña.

![SSH keygen y ssh-copy-id](imagenes/img_07.png)

> Se generó el par de claves en `/home/luisf/.ssh/id_backup` sin passphrase (necesario para automatización con cron). `ssh-copy-id` instaló la clave pública en la VM DB. La prueba final devuelve `db` y `SSH sin contraseña OK` sin pedir credenciales, confirmando el canal seguro para backups automatizados.

---

## Ejercicio 2: Preparar el Servidor de Base de Datos

### 2.1 Instalación de MariaDB y Nginx — verificación de estado

Se instalaron MariaDB y Nginx en la VM DB y se habilitaron con `systemctl enable --now`. Ambos servicios quedaron activos.

![Status de MariaDB y Nginx](imagenes/img_08.png)

> MariaDB 10.11.14 y Nginx están en estado `active (running)` y marcados como `enabled`, lo que garantiza que arrancan automáticamente con el sistema.

---

### 2.2 Creación de base de datos y datos de prueba

Se creó la base de datos `lab62_db` con la tabla `productos` y se insertaron 3 registros de prueba.

![SELECT de la tabla productos](imagenes/img_09.png)

> La tabla `productos` contiene los 3 registros correctamente insertados: Laptop ($1200.00), Mouse ($25.50) y Teclado ($45.00). Estos datos serán el objetivo de los backups.

---

### 2.3 Archivo de credenciales y contenido web

Se creó `/etc/.my.cnf` con las credenciales para `mysqldump`, se protegió con `chmod 600` y se creó el contenido web de prueba en `/var/www/html/`.

![Credenciales .my.cnf y contenido web](imagenes/img_10.png)

> El archivo `/etc/.my.cnf` tiene permisos `rw-------` (solo root puede leerlo), evitando exposición de credenciales. El `cat index.html` confirma que el contenido web de prueba fue creado correctamente con el título y párrafo del laboratorio.

---

## Ejercicio 3: Backup Local de Archivos con `tar`

Se prepararon los directorios de trabajo en la VM Backup y se creó el script `file_backup.sh`.

![Directorios y script file_backup.sh](imagenes/img_11.png)

> Se crearon los directorios `/opt/backup_scripts`, `/var/backups/data_center` y `/var/backups/files`. El script `file_backup.sh` usa `tar -czf` para comprimir con gzip. El flag `-C $(dirname)` evita rutas absolutas dentro del archivo comprimido, facilitando restauraciones portables.

---

## Ejercicio 4: Backup Remoto de Base de Datos mediante SSH

### 4.1 Script `db_backup.sh`

Se creó el script que ejecuta `mysqldump` remotamente en la VM DB vía SSH y transmite el resultado comprimido a la VM Backup.

![Script db_backup.sh](imagenes/img_12.png)

> El script conecta por SSH usando la clave dedicada `id_backup`, ejecuta `mysqldump` en la VM DB, transmite la salida por el túnel SSH cifrado, la comprime con `gzip` en tiempo real y la guarda localmente. Nunca se almacena el SQL sin comprimir.

---

### 4.2 Configuración de sudoers para mysqldump sin contraseña

Para que el script automatizado pueda ejecutar `mysqldump` con sudo en la VM DB sin interacción, se agregó una regla en `visudo`.

![visudo con regla NOPASSWD para mysqldump](imagenes/img_13.png)

> La línea `luisf ALL=(ALL) NOPASSWD: /usr/bin/mysqldump` permite que únicamente ese comando se ejecute sin contraseña, aplicando el principio de mínimo privilegio — solo lo necesario para automatizar el backup.

---

### 4.3 Ejecución exitosa del backup remoto y verificación con `zcat`

Tras corregir el `.my.cnf` (usando `localhost` en lugar de `127.0.0.1` para conexión por socket), el backup se ejecutó correctamente y se inspeccionó su contenido.

![Backup completado y zcat](imagenes/img_14.png)

> El backup pesa 833 bytes y contiene SQL válido. `zcat | head -20` muestra el header de MariaDB dump, la base de datos `lab62_db` y el inicio de la estructura de la tabla `productos`, confirmando que el backup es funcional.

---

### 4.4 Corrección del `.my.cnf` — prueba local en VM DB

Se verificó que `mysqldump` funciona correctamente desde la VM DB con el archivo de credenciales corregido.

![Prueba local de mysqldump en DB](imagenes/img_15.png)

> `visudo: sudoers.tmp unchanged` confirma que el archivo sudoers es válido. La prueba local de `mysqldump --defaults-file=/etc/.my.cnf lab62_db | head -10` muestra el header SQL correctamente. El error `errno 32 on write` es normal: `head` cierra el pipe después de 10 líneas.

---

## Ejercicio 5: Verificación de Integridad de Backups

Se creó y ejecutó el script `verify_backup.sh` que valida 4 aspectos críticos del backup.

![verify_backup.sh ejecutado con todos los checks OK](imagenes/img_16.png)

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

![Reporte de backups y script](imagenes/img_17.png)

> El reporte muestra ambos backups existentes con sus metadatos completos: el backup de BD (`lab62_db-20260521_2105.sql.gz`, 833 bytes) y el backup de archivos (`web-20260521_2102.tar.gz`, 109 bytes). El uso de disco en `/var/backups` es del 46% sobre 12GB disponibles. El script usa `awk` para formatear columnas y `df -h` para el estado del disco.

---

## Ejercicio 7: Rotación de Logs de Nginx

### 7.1 Generación de tráfico y verificación de logs

Se generaron 20 peticiones HTTP al servidor Nginx para poblar el log antes de rotarlo.

![Generación de tráfico y verificación del log](imagenes/img_18.png)

> El bucle `for i in {1..20}; do curl -s http://localhost/` generó 20 entradas en el log de acceso. `wc -l` confirma 20 líneas y `ls -lh` muestra el archivo de 1.6K listo para ser rotado.

---

### 7.2 Ejecución de la rotación y verificación

Se creó y ejecutó `log_rotate.sh` en la VM DB, que archiva el log activo y crea uno nuevo vacío.

![Log rotado y archivado, nuevo access.log vacío](imagenes/img_19.png)

> El script renombra el log activo, lo comprime en `archive/access-20260521.tar.gz` (218 bytes), elimina el temporal y recrea `access.log` vacío con permisos `www-data:adm 640`. Nginx puede seguir escribiendo sin interrupción porque trabaja con el descriptor de archivo abierto, no con el nombre.

---

## Ejercicio 8: Retención de Datos y Limpieza Automática

Se creó y ejecutó `cleanup_old_backups.sh` que aplica la política de retención de 7 días.

![cleanup_old_backups.sh ejecutado](imagenes/img_20.png)

> El script reporta 0 eliminaciones porque todos los backups tienen menos de 7 días — comportamiento correcto. `find -mtime +7` busca archivos modificados hace más de 7 días; `-delete` los elimina. En producción, este script mantiene el almacenamiento controlado eliminando backups obsoletos automáticamente.

---

## Ejercicio 9: Planificación con Cron

Se configuraron 3 tareas programadas en el crontab de root de la VM Backup.

![Crontab con las 3 tareas programadas](imagenes/img_21.png)

> Las 3 tareas quedan registradas:
> - `0 0 * * *` → Backup de BD diario a medianoche
> - `0 2 * * 0` → Backup de archivos los domingos a las 02:00
> - `0 3 * * 1` → Limpieza de backups antiguos los lunes a las 03:00
>
> `> /dev/null 2>&1` suprime la salida para evitar emails de cron al administrador.

---

## Ejercicio 10: Restauración de Archivos desde `tar`

Se creó contenido web real, se hizo un nuevo backup, se simuló la pérdida y se restauró exitosamente.

![Simulación de pérdida y restauración completa](imagenes/img_22.png)

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

![DROP DATABASE y error de restauración sin archivo local](imagenes/img_23.png)

> `DROP DATABASE lab62_db` elimina completamente la base de datos. El intento de `zcat` falla con "No such file or directory" porque el backup reside en `/var/backups/data_center/` de la VM Backup, no en la VM DB. Esto ilustra la importancia de tener el backup accesible desde donde se necesita restaurar.

---

### 11.2 Transferencia del backup con `scp`

Se copió el backup desde la VM Backup hacia la VM DB usando `scp` con la clave SSH.

![SCP del backup desde Backup hacia DB](imagenes/img_24.png)

> `scp -i /home/luisf/.ssh/id_backup` transfiere el archivo `lab62_db-20260521_2105.sql.gz` a `/tmp/` de la VM DB usando el canal SSH seguro ya configurado. La transferencia es rápida al estar en red interna.

---

### 11.3 Primer intento de restauración — error sin base de datos destino

El primer intento de restaurar sin especificar base de datos destino falló porque el dump no incluye `CREATE DATABASE`.

![Error de restauración sin especificar BD](imagenes/img_25.png)

> El error `ERROR 1046: No database selected` ocurre porque `mysqldump` sin la opción `--databases` no incluye el comando `CREATE DATABASE` en el dump. Es necesario crear la base de datos vacía primero y luego restaurar especificando el destino.

---

### 11.4 Restauración exitosa y medición del RTO

Se creó la base de datos vacía y se restauró correctamente en **0.1 segundos**.

![Restauración exitosa con tiempo medido](imagenes/img_26.png)

> La secuencia correcta: `CREATE DATABASE lab62_db` + `zcat ... | sudo mysql lab62_db`. El tiempo real de restauración es de **0.100 segundos** — este es el **RTO (Recovery Time Objective)** medido para este servicio. El segundo intento confirma la restauración exitosa con el error `database exists`.

---

### 11.5 Verificación final — datos completamente recuperados

Se verificó que la base de datos fue restaurada con todos sus registros intactos.

![SHOW DATABASES y SELECT con los 3 registros](imagenes/img_27.png)

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
