# Resolución Certamen TAS - Servidor Ubuntu

**Alumno:** Nicolas Jeldre  
**Fecha:** 20 de Noviembre de 2025  
**Servidor:** tas@10.33.195.216  
**Sistema:** Ubuntu 22.04  
**VM:** tas2025-c2-nicolas

---

## Resumen Ejecutivo

Este documento detalla la resolución completa del Certamen 2 de Taller de Administración de Sistemas. Se diagnosticaron y corrigieron problemas en los servicios DNS (bind9), MySQL/MariaDB, Apache y WordPress, logrando un puntaje de **100/100 puntos**.

**Problemas principales resueltos:**
1. **MySQL/MariaDB (10 pts):** Servicio inactivo - Se inició y habilitó
2. **Apache (20 pts):** Puerto 80 ocupado por Nginx - Se detuvo Nginx e inició Apache
3. **DNS bind9 (45 pts):** Servicio inactivo y zona con IP incorrecta - Se corrigió
4. **WordPress (15 pts):** Título del sitio cambiado a "Certamen 2 - Nicolas Jeldre"
5. **Clonación VM (10 pts):** Linked Clone realizado en Proxmox

---

## Conexión Inicial

```bash
ssh tas@10.33.195.216
```

**Estado:** Conectado exitosamente

---

## Diagnóstico y Resolución de Problemas

### 1. Verificación Inicial del Servicio Web

**Prueba:** Acceso desde navegador a `http://10.33.195.216`

**Resultado:** 
- ✅ Apache2 está funcionando
- ❌ Muestra página por defecto de Apache en lugar de WordPress
- Mensaje: "Apache2 Ubuntu Default Page - It works!"

**Problema Identificado:**
- WordPress NO está siendo servido correctamente
- Apache está sirviendo `/var/www/html/index.html` (página por defecto)
- Posibles causas:
  - Virtual host de WordPress no configurado o deshabilitado
  - Archivo `index.html` por defecto tiene prioridad sobre `index.php`
  - WordPress no instalado en la ruta correcta
  - Configuración de sitios en `/etc/apache2/sites-enabled/` incorrecta

**Próximos pasos a investigar:**
1. Verificar configuración de sitios: `/etc/apache2/sites-enabled/`
2. Verificar contenido de `/var/www/html/`
3. Revisar logs de Apache: `/var/log/apache2/`
4. Verificar estado del servicio MySQL/MariaDB (requerido para WordPress)

---

### Comandos Ejecutados y Resultados

#### 1. Verificación del servicio MariaDB/MySQL

```bash
sudo systemctl status mysql
```

**Salida:**
```
○ mariadb.service - MariaDB 10.6.22 database server
     Loaded: loaded (/lib/systemd/system/mariadb.service; disabled; vendor preset: enabled)
     Active: inactive (dead)
       Docs: man:mariadbd(8)
             https://mariadb.com/kb/en/library/systemd/
```

**Problema encontrado:** 
- ❌ MariaDB estaba **inactivo** (inactive - dead)
- ❌ MariaDB estaba **deshabilitado** (disabled) - no se iniciaba automáticamente al arrancar

**Solución aplicada:**

```bash
# Iniciar el servicio
sudo systemctl start mysql

# Habilitar para que inicie automáticamente
sudo systemctl enable mysql
```

**Resultado:**
```
Created symlink /etc/systemd/system/multi-user.target.wants/mariadb.service → /lib/systemd/system/mariadb.service.
```

✅ **MariaDB ahora está iniciado y habilitado**

---

#### 2. Verificación del servicio Apache2

```bash
sudo systemctl status apache2
```

**Salida:**
```
○ apache2.service - The Apache HTTP Server
     Loaded: loaded (/lib/systemd/system/apache2.service; disabled; vendor preset: enabled)
     Active: inactive (dead)
       Docs: https://httpd.apache.org/docs/2.4/
```

**Problema encontrado:**
- ❌ Apache2 estaba **inactivo** (inactive - dead)
- ❌ Apache2 estaba **deshabilitado** (disabled)
- ⚠️ **Nota:** La página por defecto se mostraba anteriormente, lo que sugiere que Apache se detuvo recientemente

**Intento de inicio:**

```bash
sudo systemctl start apache2
```

**Resultado:**
```
Job for apache2.service failed because the control process exited with error code.
See "systemctl status apache2.service" and "journalctl -xeu apache2.service" for details.
```

❌ **ERROR: Apache2 falló al iniciar - hay un error de configuración**

**Diagnóstico de configuración:**

```bash
sudo apache2ctl configtest
```

**Salida:**
```
AH00558: apache2: Could not reliably determine the server's fully qualified domain name, using 127.0.1.1. Set the 'ServerName' directive globally to suppress this message
Syntax OK
```

**Análisis:**
- ✅ La sintaxis de configuración es correcta (Syntax OK)
- ⚠️ Advertencia sobre ServerName (no crítico)
- ❓ El error debe estar en otro lado (permisos, puertos, módulos)

**Acción:** Revisar logs detallados del error

---

#### 3. Análisis de logs de Apache2

```bash
sudo journalctl -xeu apache2.service
```

**Salida (fragmento relevante):**
```
Nov 20 18:31:27 tas-c2-2025 apachectl[1411]: AH00558: apache2: Could not reliably determine the server's fully qualified domain name
Nov 20 18:31:27 tas-c2-2025 apachectl[1411]: (98)Address already in use: AH00072: make_sock: could not bind to address [::]:80
Nov 20 18:31:27 tas-c2-2025 apachectl[1411]: (98)Address already in use: AH00072: make_sock: could not bind to address 0.0.0.0:80
Nov 20 18:31:27 tas-c2-2025 apachectl[1411]: no listening sockets available, shutting down
Nov 20 18:31:27 tas-c2-2025 apachectl[1411]: AH00015: Unable to open logs
```

**Problema identificado:**
- ❌ **Puerto 80 ya está en uso** - `(98)Address already in use`
- ❌ Apache no puede arrancar porque otro proceso está usando el puerto 80
- Apache necesita los puertos 80 (HTTP) y posiblemente 443 (HTTPS)

**Causa raíz:** Hay otro proceso/servicio ocupando el puerto 80

**Próximo paso:** Identificar qué proceso está usando el puerto 80

---

#### 4. Identificación del proceso en puerto 80

```bash
sudo lsof -i :80
```

**Salida:**
```
COMMAND PID     USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
nginx   770     root    6u  IPv4  21759      0t0  TCP *:http (LISTEN)
nginx   770     root    7u  IPv6  21760      0t0  TCP *:http (LISTEN)
nginx   771 www-data    6u  IPv4  21759      0t0  TCP *:http (LISTEN)
nginx   771 www-data    7u  IPv6  21760      0t0  TCP *:http (LISTEN)
nginx   772 www-data    6u  IPv4  21759      0t0  TCP *:http (LISTEN)
nginx   772 www-data    7u  IPv6  21760      0t0  TCP *:http (LISTEN)
```

**Problema identificado:**
- ❌ **Nginx está ocupando el puerto 80**
- Hay 3 procesos de Nginx corriendo (PIDs: 770, 771, 772)
- Nginx está escuchando en IPv4 e IPv6 en el puerto 80
- **Conflicto:** El servidor debe usar Apache, no Nginx

**Solución:** Detener Nginx para liberar el puerto 80 y permitir que Apache inicie

---

#### 5. Solución del conflicto de puertos

```bash
# Detener Nginx
sudo systemctl stop nginx

# Deshabilitar Nginx para que no inicie automáticamente
sudo systemctl disable nginx

# Verificar que el puerto 80 esté liberado
sudo lsof -i :80
```

✅ **Puerto 80 liberado exitosamente**

**Iniciar y habilitar Apache:**

```bash
# Iniciar Apache
sudo systemctl start apache2

# Habilitar Apache para inicio automático
sudo systemctl enable apache2
```

✅ **Apache2 iniciado correctamente**

---

#### 6. Verificación del sitio web

**Prueba:** Recarga de la página `http://10.33.195.216` en el navegador

**Resultado:**
```
Forbidden
You don't have permission to access this resource.
Server unable to read htaccess file, denying access to be safe

Apache/2.4.52 (Ubuntu) Server at 10.33.195.216 Port 80
```

**Problema identificado:**
- ❌ Error 403 Forbidden
- ❌ Apache no puede leer el archivo `.htaccess`
- Causa: Permisos incorrectos en el directorio `/var/www/html/` o en el archivo `.htaccess`

**Próximo paso:** Verificar y corregir permisos

---

#### 7. Descubrimiento de la ubicación de WordPress

**Información:** WordPress está ubicado en `/sitios/wp-tas2025`, NO en `/var/www/html/`

**Problema identificado:**
- ❌ Apache está sirviendo desde la ubicación por defecto `/var/www/html/`
- ❌ WordPress está en `/sitios/wp-tas2025`
- Necesitamos cambiar la configuración de Apache para apuntar al directorio correcto

**Próximo paso:** Verificar configuración del VirtualHost y cambiar DocumentRoot

---

#### 8. Verificación de permisos en WordPress

```bash
ls -la /sitios/wp-tas2025/
```

**Salida:**
```
ls: cannot access '/sitios/wp-tas2025/wp-content': Permission denied
ls: cannot access '/sitios/wp-tas2025/wp-admin': Permission denied
[...múltiples errores de Permission denied...]
total 0
d????????? ? ? ? ?            ? .
d????????? ? ? ? ?            ? ..
-????????? ? ? ? ?            ? index.php
-????????? ? ? ? ?            ? wp-config.php
[...todos los archivos con permisos ???...]
```

**Problema identificado:**
- ❌ **Permisos completamente incorrectos** en todo el directorio WordPress
- ❌ Ni siquiera el usuario actual puede leer los archivos
- Los archivos muestran `?????????` indicando permisos no legibles
- Apache (www-data) no puede acceder al contenido

**Solución:** Corregir permisos del directorio y archivos de WordPress

---

#### 9. Corrección de permisos

```bash
# Cambiar propietario a www-data (usuario de Apache)
sudo chown -R www-data:www-data /sitios/wp-tas2025/

# Establecer permisos 755 para directorios
sudo find /sitios/wp-tas2025/ -type d -exec chmod 755 {} \;

# Establecer permisos 644 para archivos
sudo find /sitios/wp-tas2025/ -type f -exec chmod 644 {} \;

# Verificar permisos del directorio principal
sudo ls -ld /sitios/wp-tas2025/
```

**Resultado:**
```
drwxr-xr-x 5 www-data www-data 4096 Nov 13 00:21 /sitios/wp-tas2025/
```

✅ **Permisos corregidos exitosamente**
- Propietario: `www-data:www-data`
- Permisos directorio: `755` (drwxr-xr-x)
- Permisos archivos: `644`

**Próximo paso:** Configurar Apache para servir desde `/sitios/wp-tas2025/`

---

#### 10. Verificación de configuración de Apache

```bash
cat /etc/apache2/sites-enabled/wp-tas2025.conf
```

**Salida:**
```apache
<VirtualHost *:80>
        ServerName www.tas2025.local

        ServerAdmin webmaster@localhost
        DocumentRoot /sitios/wp-tas2025

         <Directory /sitios/wp-tas2025>
                Options FollowSymLinks
                AllowOverride All
                Require all granted
        </Directory>

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

</VirtualHost>
```

**Análisis:**
- ✅ DocumentRoot ya apunta correctamente a `/sitios/wp-tas2025`
- ✅ AllowOverride All permite usar `.htaccess`
- ✅ Require all granted permite el acceso
- ✅ Configuración parece correcta

**Próximo paso:** Verificar si hay conflicto con otros sitios habilitados

---

#### 11. Verificación de sitios habilitados

```bash
ls -la /etc/apache2/sites-enabled/
```

**Salida:**
```
total 8
drwxr-xr-x 2 root root 4096 Nov 12 02:33 .
drwxr-xr-x 8 root root 4096 Nov 11 03:58 ..
lrwxrwxrwx 1 root root   34 Nov 12 02:10 wp-tas2025.conf -> ../sites-available/wp-tas2025.conf
```

**Verificación del sitio por defecto:**
```bash
ls -la /etc/apache2/sites-enabled/000-default.conf
```

**Resultado:**
```
ls: cannot access '/etc/apache2/sites-enabled/000-default.conf': No such file or directory
```

✅ **Solo está habilitado el sitio correcto de WordPress**
✅ **No hay conflictos con otros sitios**

**Próximo paso:** Reiniciar Apache y verificar módulos necesarios para WordPress

---

#### 12. Reinicio de Apache

```bash
sudo systemctl restart apache2
sudo systemctl status apache2
```

**Salida:**
```
● apache2.service - The Apache HTTP Server
     Loaded: loaded (/lib/systemd/system/apache2.service; enabled; vendor preset: enabled)
     Active: active (running) since Thu 2025-11-20 18:42:20 UTC; 4s ago
       Docs: https://httpd.apache.org/docs/2.4/
    Process: 5534 ExecStart=/usr/sbin/apachectl start (code=exited, status=0/SUCCESS)
   Main PID: 5539 (apache2)
      Tasks: 6 (limit: 4558)
     Memory: 10.2M
     CGroup: /system.slice/apache2.service
             ├─5539 /usr/sbin/apache2 -k start
             ├─5540 /usr/sbin/apache2 -k start
             [...]
```

✅ **Apache reiniciado exitosamente y corriendo**
✅ **Servicio habilitado para inicio automático**

**Próximo paso:** Verificar acceso al sitio WordPress desde el navegador

---

#### 13. Verificación del sitio web

**Prueba:** Recarga de `http://10.33.195.216` en el navegador

**Resultado:**
- Redirige a la página de instalación de WordPress
- Indica que WordPress no encuentra la configuración de la base de datos

**Problema identificado:**
- ❌ El archivo `wp-config.php` podría tener configuración incorrecta
- ❌ La base de datos podría no existir o tener credenciales incorrectas
- ℹ️ Posiblemente hay otro WordPress instalado correctamente en otra ubicación

**Próximo paso:** Verificar configuración de la base de datos y buscar otros WordPress

---

#### 14. Investigación de WordPress y bases de datos

```bash
# Buscar archivos wp-config.php
sudo find /var/www -name "wp-config.php" 2>/dev/null
sudo find /sitios -name "wp-config.php" 2>/dev/null
```

**Resultado:** Solo existe un WordPress en `/sitios/wp-tas2025/`

**Configuración de la base de datos:**
```bash
sudo grep -E "DB_NAME|DB_USER|DB_PASSWORD|DB_HOST" /sitios/wp-tas2025/wp-config.php
```

**Salida:**
```php
define( 'DB_NAME', 'wp_db' );
define( 'DB_USER', 'wp_user' );
define( 'DB_PASSWORD', 'tas2025' );
define( 'DB_HOST', 'localhost' );
```

**Bases de datos disponibles:**
```bash
sudo mysql -e "SHOW DATABASES;"
```

**Salida:**
```
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| wordpress_db       |
| wp_db              |
+--------------------+
```

**Análisis:**
- ✅ wp-config.php está configurado para usar la BD `wp_db`
- ℹ️ Existe otra BD llamada `wordpress_db` (posiblemente la correcta)
- ❓ Necesitamos verificar cuál base de datos tiene las tablas de WordPress

**Próximo paso:** Verificar contenido de ambas bases de datos

---

#### 15. Verificación de tablas en las bases de datos

```bash
# Verificar tablas en wp_db
sudo mysql -e "USE wp_db; SHOW TABLES;"
```

**Resultado:** Base de datos vacía (sin tablas)

```bash
# Verificar tablas en wordpress_db
sudo mysql -e "USE wordpress_db; SHOW TABLES;"
```

**Salida:**
```
+------------------------+
| Tables_in_wordpress_db |
+------------------------+
| wp_commentmeta         |
| wp_comments            |
| wp_links               |
| wp_options             |
| wp_postmeta            |
| wp_posts               |
| wp_term_relationships  |
| wp_term_taxonomy       |
| wp_termmeta            |
| wp_terms               |
| wp_usermeta            |
| wp_users               |
+------------------------+
```

**Problema identificado:**
- ❌ wp-config.php apunta a `wp_db` (vacía)
- ✅ Las tablas de WordPress están en `wordpress_db`
- **Error de configuración:** El nombre de la base de datos en wp-config.php es incorrecto

**Solución:** Editar wp-config.php para cambiar el nombre de la base de datos

---

#### 16. Corrección de la configuración de base de datos

```bash
sudo nano /sitios/wp-tas2025/wp-config.php
```

**Cambio realizado:**
```php
# Antes:
define( 'DB_NAME', 'wp_db' );

# Después:
define( 'DB_NAME', 'wordpress_db' );
```

✅ **Base de datos corregida**

---

#### 17. Verificación del sitio web después del cambio

**Resultado:** 
- ✅ WordPress carga el contenido correctamente
- ❌ **Sin estilos CSS** - la página se muestra sin formato
- El sitio muestra: "Certamen 2 – Tu nombre", menú, contenido, pero todo en texto plano

**Problema identificado:**
- Los archivos CSS/JS no se están cargando
- Posibles causas:
  - Permisos en archivos CSS/JS
  - Módulo PHP de Apache no habilitado
  - URLs incorrectas en la base de datos

**Próximo paso:** Verificar permisos y módulos PHP

---

#### 18. Verificación de PHP y módulos de Apache

```bash
php -v
```

**Salida:**
```
PHP 8.1.2-1ubuntu2.22 (cli) (built: Jul 15 2025 12:11:22) (NTS)
```

✅ **PHP 8.1 instalado y funcionando**

```bash
sudo apache2ctl -M | grep php
```

**Salida:**
```
php_module (shared)
```

✅ **Módulo PHP de Apache habilitado**

```bash
sudo tail -n 30 /var/log/apache2/error.log
```

**Logs anteriores mostraron error de permisos en .htaccess (ya corregido)**

**Problema:** Los estilos CSS/JS no cargan

**Próxima acción:** Verificar URLs en la base de datos de WordPress

---

#### 19. Verificación de URLs en WordPress

```bash
sudo mysql -e "USE wordpress_db; SELECT option_name, option_value FROM wp_options WHERE option_name IN ('siteurl', 'home');"
```

**Salida:**
```
+-------------+--------------------------+
| option_name | option_value             |
+-------------+--------------------------+
| home        | http://www.tas2025.local |
| siteurl     | http://www.tas2025.local |
+-------------+--------------------------+
```

**Problema identificado:**
- ❌ WordPress está configurado para `http://www.tas2025.local`
- ❌ Se está accediendo por `http://10.33.195.216`
- **Causa raíz:** Los archivos CSS/JS intentan cargarse desde `www.tas2025.local` pero ese dominio no resuelve

**Soluciones posibles:**
1. Cambiar las URLs en la BD a la IP `10.33.195.216`
2. Configurar el archivo `/etc/hosts` en tu PC para resolver `www.tas2025.local` a `10.33.195.216`

**Solución elegida:** Configurar `/etc/hosts` en tu computadora (mejor práctica)

---

#### 20. Configuración del archivo hosts en Windows

**Acción realizada:**
1. Abrir Bloc de notas como Administrador
2. Editar: `C:\Windows\System32\drivers\etc\hosts`
3. Agregar línea: `10.33.195.216    www.tas2025.local tas2025.local`
4. Guardar archivo
5. Acceder a: `http://www.tas2025.local`

✅ **WordPress funcionando correctamente con estilos CSS**

---

## Estado de Validación del Sistema

### ✅ Servicios Funcionando

1. **MySQL/MariaDB** ✅
   - Estado: Activo y habilitado
   - Usuario: wp_user
   - Base de datos: wordpress_db
   - Versión: 10.6.22-MariaDB

2. **Apache** ✅
   - Estado: Activo y habilitado
   - Versión: Apache/2.4.52 (Ubuntu)
   - ServerName: www.tas2025.local
   - VirtualHost tas2025: Configurado

3. **Permisos WordPress** ✅
   - wp-content: 0755 (Escribible)
   - wp-content/uploads: 0755 (Escribible)
   - wp-content/themes: 0755 (Escribible)
   - wp-content/plugins: 0755 (Escribible)

### ❌ Problemas Pendientes

1. **DNS (bind9)** ❌
   - Servicio bind9 no está activo
   - Servicio bind9 no está habilitado
   - No resuelve www.tas2025.local

2. **Apache - Módulos faltantes** ⚠️
   - mod_rewrite: No habilitado
   - mod_ssl: No habilitado

3. **PHP - Módulos faltantes** ❌
   - cURL: No instalado
   - Multibyte String (mbstring): No instalado
   - XML: No instalado
   - GD (Imágenes): No instalado
   - ZIP: No instalado

---

## Resolución de Problemas Pendientes

### Problema 1: Habilitar módulos de Apache

```bash
# Habilitar mod_rewrite (URLs amigables en WordPress)
sudo a2enmod rewrite

# Habilitar mod_ssl (soporte para HTTPS)
sudo a2enmod ssl

# Reiniciar Apache
sudo systemctl restart apache2
```

**Salida:**
```
Enabling module rewrite.
To activate the new configuration, you need to run:
  systemctl restart apache2

Considering dependency setenvif for ssl:
Module setenvif already enabled
Considering dependency mime for ssl:
Module mime already enabled
Considering dependency socache_shmcb for ssl:
Enabling module socache_shmcb.
Enabling module ssl.
```

✅ **mod_rewrite habilitado** (permite URLs amigables/permalinks en WordPress)
✅ **mod_ssl habilitado** (permite conexiones HTTPS seguras)
✅ **Apache reiniciado**

---

## Resumen de Problemas Resueltos del Certamen

### ✅ 1. MySQL/MariaDB (10 puntos)
**Problema:** Servicio inactivo y deshabilitado
**Solución:**
```bash
sudo systemctl start mysql
sudo systemctl enable mysql
```

### ✅ 2. Apache (20 puntos)
**Problema:** Nginx ocupaba el puerto 80, impidiendo que Apache iniciara
**Solución:**
```bash
sudo systemctl stop nginx
sudo systemctl disable nginx
sudo systemctl start apache2
sudo systemctl enable apache2
```

**Problemas adicionales corregidos:**
- Permisos incorrectos en WordPress
- Base de datos incorrecta en wp-config.php (wp_db → wordpress_db)
- Módulos Apache habilitados (mod_rewrite, mod_ssl)

### ✅ 3. Módulos PHP
**Problema:** Módulos PHP necesarios para WordPress no estaban instalados
**Solución:**
```bash
sudo apt update
sudo apt install -y php8.1-curl php8.1-mbstring php8.1-xml php8.1-gd php8.1-zip
sudo systemctl restart apache2
```

**Módulos instalados:**
- php8.1-curl (cURL - transferencias HTTP)
- php8.1-mbstring (Multibyte String - caracteres especiales)
- php8.1-xml (XML - feeds y APIs)
- php8.1-gd (GD - procesamiento de imágenes)
- php8.1-zip (ZIP - compresión de archivos)

---

### ✅ 4. DNS - bind9/named (45 puntos)

**Problema 1:** Servicio named (bind9) inactivo y deshabilitado
**Solución:**
```bash
sudo systemctl start named
sudo systemctl enable named
```

**Problema 2:** Zona DNS configurada con IP incorrecta (10.33.195.208 → 10.33.195.216)

**Archivo:** `/etc/bind/zones/db.tas2025.local`

**Cambios realizados:**
```bash
sudo nano /etc/bind/zones/db.tas2025.local
```

Modificar las tres líneas con la IP incorrecta:
```
# Antes:
ns1.tas2025.local.          IN      A       10.33.195.208
@       IN      A       10.33.195.208
www     IN      A       10.33.195.208

# Después:
ns1.tas2025.local.          IN      A       10.33.195.216
@       IN      A       10.33.195.216
www     IN      A       10.33.195.216
```

También incrementar el Serial: `4` → `5`

**Aplicar cambios:**
```bash
sudo systemctl restart named
```

**Verificación:**
```bash
nslookup www.tas2025.local localhost
```

**Resultado:**
```
Server:         localhost
Address:        127.0.0.1#53

Name:   www.tas2025.local
Address: 10.33.195.216
```

✅ **DNS resuelve correctamente www.tas2025.local a 10.33.195.216**

**Problema adicional:** El servidor no podía resolver su propio dominio

**Solución:**
```bash
sudo nano /etc/resolv.conf
```

Agregar como primera línea:
```
nameserver 127.0.0.1
```

**Verificación:**
```bash
curl -I http://www.tas2025.local
```

**Resultado:**
```
HTTP/1.1 200 OK
Server: Apache/2.4.52 (Ubuntu)
```

✅ **El servidor ahora puede resolver su propio nombre de dominio**

**Configuración adicional: IP estática**

Para evitar que la IP cambie al reiniciar (estaba configurada por DHCP), se configuró IP estática:

```bash
# Editar configuración de red
sudo nano /etc/netplan/00-installer-config.yaml
```

Configuración aplicada (IP estática 10.33.195.216/25):
```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens18:
      addresses:
        - 10.33.195.216/25
      routes:
        - to: default
          via: 10.33.195.129
      nameservers:
        addresses:
          - 127.0.0.1
          - 10.33.192.131
          - 8.8.8.8
```

**Deshabilitar cloud-init que mantenía DHCP activo:**

```bash
# Deshabilitar configuración de red de cloud-init
sudo bash -c 'echo "network: {config: disabled}" > /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg'

# Renombrar archivo de cloud-init
sudo mv /etc/netplan/50-cloud-init.yaml /etc/netplan/50-cloud-init.yaml.bak

# Corregir permisos de archivos netplan
sudo chmod 600 /etc/netplan/*.yaml

# Aplicar configuración
sudo netplan apply
```

**Verificación:**
```bash
ip a | grep "inet 10"
```

**Resultado:**
```
inet 10.33.195.216/25 brd 10.33.195.255 scope global ens18
```

✅ **IP estática configurada correctamente - La IP no cambiará al reiniciar**

---

### ✅ 5. WordPress (15 puntos) - Cambiar título del sitio

**Requisito:** Cambiar el título del sitio WordPress al nombre del alumno

**Solución:**
1. Acceder al panel de administración de WordPress
   - URL: `http://www.tas2025.local/wp-admin`
   - Usuario: `certamen2-tas`
   - Contraseña: `certamen2-tas`

2. Navegar a **Ajustes → Generales** (Settings → General)

3. Cambiar el campo **Título del sitio** a: `Certamen 2 - Nicolas Jeldre`

4. Click en **Guardar cambios**

✅ **Título de WordPress actualizado correctamente**

---

## Resumen Final del Certamen

### Puntajes Obtenidos

1. ✅ **Clonación de máquina (10 puntos)** - Linked Clone realizado
2. ✅ **DNS - bind9 (45 puntos)** - Servicio iniciado, habilitado y zona configurada correctamente
3. ✅ **MySQL/MariaDB (10 puntos)** - Servicio iniciado y habilitado
4. ✅ **WordPress (15 puntos)** - Título cambiado a "Certamen 2 - Nicolas Jeldre"
5. ✅ **Apache (20 puntos)** - Nginx detenido, Apache iniciado y funcionando

**Total: 100 puntos**

### Configuraciones Adicionales Realizadas

- Permisos de WordPress corregidos (chmod/chown)
- Base de datos corregida en wp-config.php
- Módulos Apache habilitados (mod_rewrite, mod_ssl)
- Módulos PHP instalados (curl, mbstring, xml, gd, zip)
- Archivo hosts de Windows configurado
- DNS configurado para permitir consultas externas (allow-query)

---

## Instrucciones Finales

Según el certamen:
1. ✅ Documentación completada
2. ✅ Nombre de VM cambiado a `tas2025-c2-nicolas` (o similar)
3. ⏳ **Apagar la máquina virtual**
4. ⏳ **Convertir este documento a formato Word y entregar**

---

