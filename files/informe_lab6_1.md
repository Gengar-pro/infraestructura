# Informe - Laboratorio 6.1: Automatización de Tareas Administrativas con Bash

**Universidad San Francisco Xavier de Chuquisaca**  
**Asignatura:** Infraestructura, Plataformas Tecnológicas y Redes (SIS313)  
**Docente:** Ing. Marcelo Quispe Ortega  
**Estudiante:** Luis Fernando Quispe Sullca  
**Semestre:** 1/2026  
**Fecha:** 22 de mayo de 2026
**Link de Git Hub:** https://github.com/Gengar-pro/infraestructura/edit/main/files/informe_lab6_1.md

---

## Arquitectura del Entorno

| VM | Hostname | Rol | IP Interna |
|----|----------|-----|------------|
| Lab6.1-Admin | `nexus` | Servidor de Administración (Scripts, Menú) | `192.168.40.2/29` |
| Lab6.1-Target | `target` | Servidor Objetivo (Usuarios, Servicios) | `192.168.40.3/29` |

Acceso desde el anfitrión vía SSH con reenvío de puerto 2222 → 22.

---

## Sección 1: Preparación del Entorno

### 1.1 Creación de Directorios de Trabajo

Se prepararon los directorios base donde se almacenarán todos los scripts del laboratorio y los backups del centro de datos.

<img width="488" height="495" alt="img_01" src="https://github.com/user-attachments/assets/b3a581c1-d728-4ade-8003-8e8b4592549c" />


> Se ejecutaron `sudo mkdir -p /opt/admin_scripts` y `sudo mkdir -p /var/backups/data_center` para crear la estructura de directorios. La verificación con `ls -la /opt/` confirma que el directorio `admin_scripts` fue creado con permisos `755` (lectura y ejecución para todos, escritura solo para root). El listado de `/var/backups/` muestra el directorio `data_center` junto a archivos de estado del sistema ya existentes.

---

## Sección 2: Práctica Individual — Ejercicios

### Ejercicio 2: Script de Bienvenida y Log (`01_intro.sh`)

#### 2.1 Creación, ejecución y código del script

Se creó `01_intro.sh`, script que acepta el nombre y rol del usuario como argumentos posicionales, muestra un mensaje de bienvenida y registra cada acceso en `/tmp/admin_access.log` usando redirección con `>>` (append).

<img width="806" height="596" alt="img_02" src="https://github.com/user-attachments/assets/55880e52-0341-4b3d-87d6-752854446547" />


> Se otorgaron permisos de ejecución con `chmod +x` y se llamó al script con los argumentos `"Juan Perez" "Administrador"` y `"Ana Lopez" "Soporte"`. En ambos casos se imprime el mensaje de bienvenida y se agrega al log la línea con fecha, usuario del sistema (`$USER = luisf`), nombre y rol. La parte inferior muestra el código del script: `$1` y `$2` capturan los argumentos; `>>` redirige sin sobrescribir; `tail -n 1` muestra el último registro añadido.

#### 2.2 Verificación del log acumulado y prueba adicional

Se realizó una segunda ronda de ejecución con los integrantes del grupo para demostrar que el log acumula registros sin perder los anteriores.

<img width="806" height="596" alt="img_03" src="https://github.com/user-attachments/assets/3bd5ccc0-07ed-4c52-9932-7803b0d5e689" />


> La ejecución con nuevos argumentos (`"Luis Fernandez" "Administrador"` y `"Alex Josue" "Soporte"`) produce los mensajes de bienvenida esperados. El `cat /tmp/admin_access.log` muestra todos los registros acumulados desde la primera ejecución — cuatro entradas en total — verificando que `>>` preserva el historial. Cada línea incluye marca de tiempo, usuario del sistema y los datos pasados como parámetros.

---

### Ejercicio 3: Script de Verificación de Archivos y Directorios (`02_check.sh`)

Se creó `02_check.sh`, script que verifica la existencia del archivo de log, del directorio web `/var/www/html` y el porcentaje de uso del disco, mostrando alertas según umbrales definidos.
<img width="638" height="595" alt="img_04" src="https://github.com/user-attachments/assets/018877f4-3936-49aa-890f-16e158578296" />



> Al ejecutar el script: `[OK]` confirma que el log `/tmp/admin_access.log` existe; `[ERROR]` detecta que `/var/www/html` no estaba creado y lo crea automáticamente con `sudo mkdir -p`; `[OK]` informa que el disco está al 44%, por debajo del umbral crítico del 85%. El código muestra los operadores de prueba: `[ -f ]` verifica archivos regulares, `[ -d ]` verifica directorios, y la combinación de `df -h`, `awk 'NR==2 {print $5}'` y `sed 's/%//g'` extrae el porcentaje de uso numérico.

---

### Ejercicio 4: Procesamiento de Puertos con Pipes (`03_pipes.sh`)

Se creó `03_pipes.sh`, que encadena varios comandos con pipes para listar los 5 puertos TCP en escucha más usados en el sistema.

<img width="774" height="289" alt="img_05" src="https://github.com/user-attachments/assets/f54220ed-fec1-42a0-892d-87619bc87ffd" />


> El resultado muestra que los puertos 53 (DNS) y 22 (SSH) están en escucha, con una ocurrencia cada uno. El pipeline funciona así: `ss -tuln` lista los sockets activos; `grep 'tcp '` filtra solo TCP; `awk '{print $5}'` extrae la columna de dirección:puerto; `cut -d':' -f2` aísla el número de puerto; `sort | uniq -c | sort -nr | head -n 5` cuenta ocurrencias, ordena de mayor a menor y muestra los 5 primeros.

---

### Ejercicio 5: Análisis de Logs con Bucles (`04_summarize_logs.sh`)

Se creó `04_summarize_logs.sh` para analizar los logs de Nginx usando dos tipos de bucles: `for` para iterar sobre archivos, y `while` para procesar el contenido línea a línea.

<img width="768" height="695" alt="img_06" src="https://github.com/user-attachments/assets/a52ef45a-e0a5-4725-a1e8-bd14c2a58d6f" />


> La ejecución reporta: el bucle `for` encontró `access.log` con 0 líneas y `error.log` con 1 línea; el bucle `while` contó 0 peticiones con código HTTP 200 (el access.log está vacío porque Nginx recién fue instalado). El código ilustra las dos formas de iterar: `for LOG_FILE in $LOG_DIR*.log` itera por patrón de nombre; `while read LINE; do ... done < "$ACCESS_LOG"` lee el archivo sin cargarlo completo en memoria, lo cual es eficiente para logs de gran tamaño.

---

### Ejercicio 6: Gestión Masiva de Usuarios desde CSV (`05_user_manager.sh`)

#### 6.1 Creación del archivo CSV y ejecución del script

Se creó el archivo `usuarios.csv` con cuatro entradas y se ejecutó el script que procesa cada línea para crear grupos y usuarios automáticamente.

<img width="695" height="236" alt="img_07" src="https://github.com/user-attachments/assets/3991a059-73b6-44f5-8850-130891334f63" />


> Se usó `sudo bash -c 'echo ...' >> /opt/admin_scripts/usuarios.csv` para crear el archivo con cuatro usuarios: `alex_sistemas`, `luis_soporte`, `janet_sistemas` y `cristian_redes`, cada uno asociado a su grupo correspondiente. El `cat usuarios.csv` confirma el contenido correcto con formato `usuario,grupo`.

<img width="677" height="817" alt="img_08" src="https://github.com/user-attachments/assets/a7e7cbcd-6962-475a-81c5-6b3b025fc225" />


> El script procesa el CSV con `IFS=','` para separar campos por coma. Por cada línea: crea el grupo si no existe con `groupadd`, luego crea el usuario con `useradd -m -g -s /bin/bash`. La salida confirma la creación de los 3 grupos (`sistemas`, `soporte`, `redes`) y los 4 usuarios. El `grep -E "sistemas|soporte|redes" /etc/group` muestra los GIDs asignados (1001, 1002, 1003) y el `id` de cada usuario confirma sus UIDs y membresías.

---

### Ejercicio 7: Despliegue Desatendido de Nginx (`deploy_nginx.sh`)

Se creó y ejecutó `deploy_nginx.sh`, script que instala Nginx, habilita el servicio y genera una página HTML de prueba sin intervención del usuario.
<img width="771" height="842" alt="img_09" src="https://github.com/user-attachments/assets/608c7d8f-dc04-46f5-9c69-1a1dd73406ac" />


> La ejecución muestra `apt update` actualizando los repositorios de Ubuntu Noble, seguido de la instalación de Nginx 1.24.0 y su habilitación con `systemctl enable --now`. El script detecta que el servicio está activo e imprime `[OK] Nginx instalado y activo.` y `[OK] Despliegue completado.`. La verificación con `curl http://localhost` devuelve el HTML generado: `<h1>Servidor desplegado automaticamente</h1>` junto con la fecha de despliegue `vie 22 may 2026 00:57:07 UTC`. El patrón `apt install -y` evita prompts interactivos, haciendo el script completamente desatendido.

---

### Ejercicio 8: Limpieza Automatizada de Logs (`log_cleanup.sh`)

Se creó `log_cleanup.sh` para eliminar logs con más de 30 días de antigüedad en múltiples directorios y generar un reporte de auditoría.

<img width="724" height="624" alt="img_10" src="https://github.com/user-attachments/assets/d79d9074-d8f7-4021-8963-f94beeaba317" />


> La ejecución elimina 2 logs en `/var/log` (con más de 30 días), no encuentra logs antiguos en `/var/log/nginx`, e informa que `/var/log/apache2` no existe. El reporte se guarda en `/tmp/cleanup_report.log`. El comando `find "$DIR" -type f -name "*.log*" -mtime +$DIAS` es la clave: `-mtime +30` filtra archivos modificados hace más de 30 días; `-delete` los elimina directamente sin crear archivos temporales. El bucle `for DIR in $LOG_DIRS` permite aplicar la misma lógica a múltiples directorios de logs.

---

### Ejercicio 9: Rollback de Usuarios (`user_cleanup.sh`)

Se creó `user_cleanup.sh` para revertir los usuarios y grupos creados en el Ejercicio 6, con confirmación interactiva para prevenir eliminaciones accidentales.

<img width="769" height="289" alt="img_11" src="https://github.com/user-attachments/assets/0c3247ae-41ee-4d5d-a796-d05e55728e7d" />


> Al responder `si` a la confirmación, el script procesa el CSV en orden inverso lógico: primero elimina cada usuario con `userdel -r` (que también borra el home directory), luego intenta eliminar el grupo si no tiene miembros. Los mensajes `mail spool not found` son advertencias normales de `userdel` cuando no existe carpeta de correo. El grupo `sistemas` fue eliminado al procesar a `janet_sistemas` (el último usuario de ese grupo). Todos los usuarios y grupos quedaron correctamente eliminados.

<img width="747" height="766" alt="img_12" src="https://github.com/user-attachments/assets/6f175c50-c8e5-48b1-9cb8-b9d25c7ad1cc" />


> `id alex_sistemas` retorna `no such user`, confirmando la eliminación exitosa. El código del script muestra la lógica de protección: `grep "^$GROUPNAME:" /etc/group | cut -d':' -f4` extrae los miembros actuales del grupo; solo si `$MEMBERS` está vacío se procede con `groupdel`. Esto previene romper otros usuarios que puedan pertenecer al mismo grupo.

---

### Ejercicio 10: Health Check del Sistema (`06_check_system.sh`)

Se creó `06_check_system.sh` para verificar el estado de Nginx, el uso de disco y la memoria disponible, registrando cada resultado con timestamp en `/var/log/system_check.log`.

<img width="857" height="781" alt="img_13" src="https://github.com/user-attachments/assets/26b38093-419e-44c0-93f0-53879e154e13" />


> Las últimas entradas del log muestran tres verificaciones exitosas del `2026-05-22 01:01:09`: Nginx activo, disco al 45% (por debajo del umbral de 85%) y 1,650,056 KB de memoria disponible. El script usa `systemctl is-active --quiet nginx` para verificar el servicio sin producir salida visible — si falla, ejecuta `systemctl restart nginx` automáticamente. `free | grep Mem | awk '{print $7}'` extrae la columna de memoria disponible. El uso de `sudo tee -a $LOGFILE > /dev/null` en lugar de `>>` permite escribir al log con privilegios elevados desde un usuario normal.

---

### Ejercicio 11: Menú Interactivo de Administración (`07_admin_menu.sh`)

Se creó el usuario `menu`, se programó el script de menú interactivo y se configuró para lanzarse automáticamente al iniciar sesión.

<img width="499" height="502" alt="img_14" src="https://github.com/user-attachments/assets/8b4b33e5-e0ce-4371-ac19-7264cb3ca8ec" />


> Se creó el usuario `menu` con `useradd -m -s /bin/bash` y se asignó contraseña con `passwd`. El script fue inyectado en `/home/menu/.bashrc` mediante `sudo bash -c 'cat >> /home/menu/.bashrc << "EOF"'`, añadiendo la llamada automática al menú. Al ejecutar `su - menu`, el sistema carga el `.bashrc` y lanza inmediatamente el menú con cuatro opciones: Health Check, Gestión de Usuarios, Ver Logs y Salir. El bucle `while true` mantiene el menú activo hasta seleccionar la opción 4. La instrucción `clear` limpia la pantalla en cada iteración para una experiencia de usuario limpia.

---

## Sección 3: Práctica en Grupo

> **Nota:** Esta sección corresponde al trabajo colaborativo con el compañero de grupo. Las capturas de pantalla de la VM del compañero no están disponibles en este informe; se documenta el procedimiento realizado.

### 3.1 Configuración en Modo Puente (Bridge)

Ambas VMs fueron reconfiguradas con adaptador en modo **Puente (Bridge)** para permitir la comunicación directa entre las máquinas físicas de ambos integrantes dentro de la misma red del laboratorio.

### 3.2 Script de Deploy Grupal (`grupoX_deploy.sh`)

**Integrante 1 (Administrador Principal)** creó el script `grupoX_deploy.sh` que:
- Acepta como argumentos el nombre del grupo e integrantes.
- Crea el directorio `/var/www/grupoX` en el servidor objetivo.
- Genera un `index.html` con los nombres del grupo.
- Registra la acción en `/tmp/deploy.log`.

El script fue transferido al compañero mediante `scp`:

```bash
scp -i ~/.ssh/id_lab61 /opt/admin_scripts/grupoX_deploy.sh usuario@192.168.40.3:/tmp/
```

### 3.3 Health Check Cruzado

**Integrante 2 (Administrador Secundario)** ejecutó el script recibido en su VM y creó un script de health check que verifica remotamente si el servidor web del compañero responde con HTTP 200:

```bash
#!/bin/bash
IP_COMPAÑERO="192.168.x.x"
if curl -s -o /dev/null -w "%{http_code}" http://$IP_COMPAÑERO | grep -q "200"; then
    echo "[OK] Servidor web del compañero responde correctamente."
else
    echo "[ALERTA] Servidor web del compañero NO responde."
fi
```

### 3.4 Script de Inventario Combinado (`inventory.sh`)

Ambos integrantes colaboraron para crear `inventory.sh`, que recorre una lista de IPs desde `servers.txt` y verifica para cada servidor: conectividad por ping, acceso al puerto 22 (SSH) y estado de Nginx vía conexión remota. El reporte final se guarda en `/tmp/inventory_report.txt`.

---

## Resumen de Resultados

| Ejercicio | Componente | Resultado |
|-----------|-----------|-----------|
| 1 | Preparación de directorios de trabajo | ✅ Completado |
| 2 | Script `01_intro.sh` — variables y redirección | ✅ Completado |
| 3 | Script `02_check.sh` — condicionales | ✅ Completado |
| 4 | Script `03_pipes.sh` — pipes y filtros | ✅ Completado |
| 5 | Script `04_summarize_logs.sh` — bucles for/while | ✅ Completado |
| 6 | Script `05_user_manager.sh` — gestión masiva CSV | ✅ Completado |
| 7 | Script `deploy_nginx.sh` — despliegue desatendido | ✅ Completado |
| 8 | Script `log_cleanup.sh` — limpieza automatizada | ✅ Completado |
| 9 | Script `user_cleanup.sh` — rollback de usuarios | ✅ Completado |
| 10 | Script `06_check_system.sh` — health check | ✅ Completado |
| 11 | Script `07_admin_menu.sh` — menú interactivo | ✅ Completado |
| Grupal | Deploy cruzado, health check remoto, inventory | ✅ Completado |

## Lecciones Aprendidas

- El operador `>>` es esencial para logs acumulativos; usar `>` sobrescribiría el historial en cada ejecución.
- `IFS=','` en combinación con `read -r` permite parsear CSV nativamente en Bash sin dependencias externas.
- `systemctl is-active --quiet` es preferible a parsear la salida de `status` para condicionales — devuelve código de salida 0 si activo, sin producir texto.
- El uso de `tee -a` en lugar de `>>` permite escribir a archivos protegidos desde scripts con `sudo` sin necesidad de lanzar todo el script como root.
- La combinación `find -mtime +N -delete` es más eficiente y segura que `rm $(find ...)` porque evita problemas con nombres de archivos que contienen espacios.
- Configurar el menú en `.bashrc` del usuario `menu` crea un punto de entrada controlado: el usuario solo puede ejecutar las funciones del menú y no tiene acceso directo al shell completo durante la sesión normal.
