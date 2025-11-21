# Comandos Esenciales - Certamen TAS

## 1. MySQL/MariaDB - Iniciar y Habilitar Servicio

```bash
# Ver estado del servicio
sudo systemctl status mysql

# Iniciar servicio
sudo systemctl start mysql

# Habilitar para inicio automático
sudo systemctl enable mysql
```

---

## 2. Apache - Resolver Conflicto de Puerto 80

```bash
# Identificar qué usa el puerto 80
sudo lsof -i :80

# Detener Nginx (si está ocupando el puerto)
sudo systemctl stop nginx
sudo systemctl disable nginx

# Iniciar Apache
sudo systemctl start apache2
sudo systemctl enable apache2

# Verificar estado
sudo systemctl status apache2

# Verificar configuración
sudo apache2ctl configtest

# Ver logs de error
sudo journalctl -xeu apache2.service

# Habilitar módulos necesarios
sudo a2enmod rewrite
sudo a2enmod ssl

# Reiniciar Apache
sudo systemctl restart apache2
```

---

## 3. Permisos de WordPress

```bash
# Cambiar propietario a www-data
sudo chown -R www-data:www-data /sitios/wp-tas2025/

# Permisos 755 para directorios
sudo find /sitios/wp-tas2025/ -type d -exec chmod 755 {} \;

# Permisos 644 para archivos
sudo find /sitios/wp-tas2025/ -type f -exec chmod 644 {} \;

# Verificar permisos
ls -la /sitios/wp-tas2025/
sudo ls -ld /sitios/wp-tas2025/
```

---

## 4. MySQL - Base de Datos

```bash
# Listar bases de datos
sudo mysql -e "SHOW DATABASES;"

# Ver tablas de una base de datos
sudo mysql -e "USE wordpress_db; SHOW TABLES;"

# Ver configuración en wp-config.php
sudo grep -E "DB_NAME|DB_USER|DB_PASSWORD|DB_HOST" /sitios/wp-tas2025/wp-config.php

# Editar wp-config.php
sudo nano /sitios/wp-tas2025/wp-config.php

# Ver opciones de WordPress (URLs)
sudo mysql -e "USE wordpress_db; SELECT option_name, option_value FROM wp_options WHERE option_name IN ('siteurl', 'home');"
```

---

## 5. DNS - bind9/named

```bash
# Ver estado del servicio
sudo systemctl status named
sudo systemctl status bind9

# Iniciar y habilitar
sudo systemctl start named
sudo systemctl enable named

# Buscar archivos de configuración
sudo grep -r "tas2025.local" /etc/bind/

# Ver configuración de zonas
cat /etc/bind/named.conf.local
cat /etc/bind/named.conf.options

# Editar zona DNS
sudo nano /etc/bind/zones/db.tas2025.local

# Verificar sintaxis de zona
sudo named-checkzone tas2025.local /etc/bind/zones/db.tas2025.local

# Reiniciar servicio
sudo systemctl restart named

# Probar resolución DNS
nslookup www.tas2025.local localhost
nslookup www.tas2025.local 10.33.195.216
dig @localhost www.tas2025.local

# Ver en qué puertos escucha
sudo ss -tulpn | grep named

# Configurar servidor para usar su propio DNS
sudo nano /etc/resolv.conf
# Agregar: nameserver 127.0.0.1

# Verificar que resuelve internamente
curl -I http://www.tas2025.local
```

---

## 6. PHP - Módulos

```bash
# Ver versión de PHP
php -v

# Verificar módulos de Apache
sudo apache2ctl -M | grep php

# Instalar módulos PHP
sudo apt update
sudo apt install -y php8.1-curl php8.1-mbstring php8.1-xml php8.1-gd php8.1-zip

# Reiniciar Apache después de instalar PHP
sudo systemctl restart apache2
```

---

## 7. Apache - Configuración VirtualHost

```bash
# Ver sitios habilitados
ls -la /etc/apache2/sites-enabled/

# Ver configuración de un sitio
cat /etc/apache2/sites-enabled/wp-tas2025.conf

# Ver todos los VirtualHosts configurados
sudo apache2ctl -S

# Habilitar un sitio
sudo a2ensite wp-tas2025.conf

# Deshabilitar un sitio
sudo a2dissite 000-default.conf
```

---

## 8. Red - Configuración IP Estática

```bash
# Ver IP actual
ip a
ip a | grep "inet 10"

# Ver archivos de configuración de red
ls /etc/netplan/

# Ver configuración actual
sudo cat /etc/netplan/00-installer-config.yaml

# Editar configuración de red
sudo nano /etc/netplan/00-installer-config.yaml

# Deshabilitar cloud-init
sudo bash -c 'echo "network: {config: disabled}" > /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg'

# Renombrar archivo cloud-init
sudo mv /etc/netplan/50-cloud-init.yaml /etc/netplan/50-cloud-init.yaml.bak

# Corregir permisos
sudo chmod 600 /etc/netplan/*.yaml

# Aplicar configuración
sudo netplan apply
```

---

## 9. Diagnóstico General

```bash
# Ver logs del sistema
sudo journalctl -xe
sudo journalctl -u nombre_servicio -n 50

# Ver logs de Apache
sudo tail -n 50 /var/log/apache2/error.log
sudo tail -n 50 /var/log/apache2/access.log

# Ver procesos escuchando en puertos
sudo lsof -i :80
sudo lsof -i :443
sudo lsof -i :53
sudo ss -tulpn

# Ver servicios activos
systemctl list-units --type=service --state=running

# Verificar conectividad
ping 8.8.8.8
curl -I http://www.tas2025.local
```

---

## 10. Archivos Importantes a Conocer

```bash
# Apache
/etc/apache2/apache2.conf              # Configuración principal
/etc/apache2/sites-available/          # Sitios disponibles
/etc/apache2/sites-enabled/            # Sitios habilitados
/var/log/apache2/error.log             # Logs de error
/var/log/apache2/access.log            # Logs de acceso

# DNS (bind9)
/etc/bind/named.conf                   # Configuración principal
/etc/bind/named.conf.local             # Zonas locales
/etc/bind/named.conf.options           # Opciones
/etc/bind/zones/                       # Archivos de zona

# Red
/etc/netplan/                          # Configuración de red
/etc/resolv.conf                       # Servidores DNS
/etc/hosts                             # Resolución local

# WordPress
/sitios/wp-tas2025/wp-config.php       # Configuración de WordPress
/sitios/wp-tas2025/                    # Directorio de WordPress
```

---

## 11. Windows - Configuración

```powershell
# Limpiar caché DNS
ipconfig /flushdns

# Probar resolución DNS
nslookup www.tas2025.local 10.33.195.216
Resolve-DnsName -Name www.tas2025.local -Server 10.33.195.216

# Archivo hosts de Windows
C:\Windows\System32\drivers\etc\hosts
# Agregar: 10.33.195.216    www.tas2025.local tas2025.local
```

---

## 12. Comandos Útiles para Recordar

```bash
# Apagar servidor
sudo poweroff

# Reiniciar servidor
sudo reboot

# Ver usuario actual
whoami

# Ver directorio actual
pwd

# Buscar archivos
sudo find /ruta -name "archivo.txt"
sudo grep -r "texto" /ruta/

# Editar archivos
sudo nano archivo.txt
# Guardar: Ctrl+O, Enter
# Salir: Ctrl+X

# Ver contenido de archivos
cat archivo.txt
sudo cat archivo.txt
less archivo.txt

# Copiar archivos
sudo cp origen destino

# Mover/renombrar
sudo mv origen destino

# Permisos
chmod 755 directorio/
chmod 644 archivo.txt
chown usuario:grupo archivo
```

---

## Secuencia Típica de Resolución de Problemas

1. **Verificar estado del servicio**
   ```bash
   sudo systemctl status nombre_servicio
   ```

2. **Si está inactivo, iniciar**
   ```bash
   sudo systemctl start nombre_servicio
   sudo systemctl enable nombre_servicio
   ```

3. **Si falla al iniciar, ver logs**
   ```bash
   sudo journalctl -xeu nombre_servicio
   ```

4. **Identificar el problema específico** (puerto ocupado, permisos, configuración)

5. **Aplicar solución específica**

6. **Reiniciar servicio**
   ```bash
   sudo systemctl restart nombre_servicio
   ```

7. **Verificar que funciona**
   ```bash
   sudo systemctl status nombre_servicio
   ```
