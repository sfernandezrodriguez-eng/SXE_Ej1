# Informe de Práctica: Instalación de WordPress por Sergio fernández Rodriguez

## 1. El Servidor

### 1.1 Requisitos Previos e Inspección de la Pila


| Elemento | Lo que dice la documentación                                         | Lo que necesita la VM | Otros (Comentarios relevantes)                                                                                                  | Fuente de info |
| :--- |:---------------------------------------------------------------------| :--- |:--------------------------------------------------------------------------------------------------------------------------------| :--- |
| **S.O.** | Linux (Distribución basada en Debian/Ubuntu recomendada)             | Ubuntu Server 24.04 LTS (o Ubuntu 22.04 LTS) | Instalación mínima sin entorno gráfico (Server Headless) para simular un VPS real.                                              | [wordpress.org/about/requirements](https://wordpress.org/about/requirements/) |
| **Servidor web** | Apache (con módulo mod_rewrite) o Nginx                              | Apache2 | Se opta por Apache2 por madurez, compatibilidad directa con archivos .htaccess y sencillez de configuración en pila LAMP.       | [wordpress.org/about/requirements](https://wordpress.org/about/requirements/) |
| **Versión de PHP** | PHP 8.1, 8.2 o superior                                              | PHP 8.3 (incluido en repositorios de Ubuntu 24.04) | Requiere extensiones adicionales (php-mysql, php-curl, php-gd, php-mbstring, php-xml, php-xmlrpc, php-soap, php-intl, php-zip). | [wordpress.org/about/requirements](https://wordpress.org/about/requirements/) |
| **Gestor de BBDD** | MySQL v8.0+ o MariaDB v10.5+                                         | MariaDB Server / MySQL Server 8.0 | Se crea una base de datos con cotejamiento utf8mb4_unicode_ci y un usuario dedicado con privilegios limitados a esa BBDD.       | [wordpress.org/about/requirements](https://wordpress.org/about/requirements/) |
| **Memoria y Disco** | Mínimo 512 MB RAM (Recomendado 1 GB - 2 GB), 10 GB de almacenamiento | 2 GB RAM, 20 GB Disco SSD, 2 Cores vCPU | Garantiza fluidez suficiente para la pila LAMP, proceso de compilación/ejecución de scripts PHP e instalación de plugins.       | Documentación técnica WordPress / Requisitos Ubuntu Server |

---

### 1.2 Configuración de la Máquina Virtual (VPS)

Se ha creado y configurado una máquina virtual con comportamiento equivalente a un servidor VPS contratado (acceso *headless*, sin interfaz gráfica de usuario y gestión exclusivamente remota).

- **Sistema Operativo:** Ubuntu Server 24.04 LTS (64-bit)
- **Procesador:** 2 Cores vCPU
- **Memoria RAM Total:** 2048 MB (2 GB)
- **Espacio en Disco:** 20 GB
- **Red:** Adaptador en modo Puente para asignación de IP alcanzable desde la red local anfitriona.
- **Acceso Remoto:** Servicio openssh-server activo.
- **Usuario administrador:** vpsadmin con privilegios sudo.

### 1.3 Conexión remota vía SSH

Cumpliendo con la restricción de administración de servidores reales (donde no se dispone de pantalla ni teclado físico conectado), todo el trabajo se realiza desde la terminal del equipo anfitrión conectándose por SSH.

ssh vpsadmin@192.168.1.50

## 2. La Instalación

### 2.1 Guía de Referencia
La instalación se ha llevado a cabo siguiendo las pautas de la guía oficial de DigitalOcean:  
 **Tutorial utilizado:** [How To Install WordPress with LAMP on Ubuntu](https://www.digitalocean.com/community/tutorials/install-wordpress-on-ubuntu)

### 2.2 Paso a Paso de la Instalación

#### Paso 1: Actualización del Sistema
Antes de instalar cualquier paquete, se actualizan los repositorios y paquetes del sistema operativo:

sudo apt update && sudo apt upgrade -y

#### Paso 2: Instalación del Servidor Web Apache
Se instala Apache2 y se verifica el estado del servicio:

sudo apt install apache2 -y
sudo systemctl enable apache2
sudo systemctl status apache2

Se activa además el módulo mod_rewrite necesario para las URLs amigables (permalinks) de WordPress:

sudo a2enmod rewrite
sudo systemctl restart apache2

#### Paso 3: Instalación y Configuración del Gestor de BBDD (MySQL/MariaDB)
Se instala el servidor de base de datos MySQL/MariaDB y se ejecuta el script de seguridad inicial:

sudo apt install mysql-server -y
sudo mysql_secure_installation

A continuación, se accede a la consola de MySQL y se crea la base de datos y el usuario específico para WordPress:


CREATE DATABASE wordpress_db DEFAULT CHARACTER SET utf8 COLLATE utf8_unicode_ci;
CREATE USER 'wp_user'@'localhost' IDENTIFIED BY 'PasswordSegura123!';  <-En password segura 123 pones la contraseña que quieras, el resto no es obligatorio aunque si recomendable
GRANT ALL PRIVILEGES ON wordpress_db.* TO 'wp_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;

#### Paso 4: Instalación de PHP y Extensiones Necesarias
Se instala PHP junto a los módulos exigidos por WordPress para procesamiento gráfico, manejo de cadenas, conexión a BD y peticiones HTTP:

sudo apt install php libapache2-mod-php php-mysql php-curl php-gd php-mbstring php-xml php-xmlrpc php-soap php-intl php-zip -y

Se reinicia Apache para cargar los módulos de PHP:

sudo systemctl restart apache2

#### Paso 5: Descarga y Configuración de WordPress
Se descarga la última versión estable de WordPress desde la web oficial, se descomprime y se mueve al directorio raíz del servidor web `/var/www/html`:


cd /tmp
curl -O https://wordpress.org/latest.tar.gz
tar xzvf latest.tar.gz
sudo cp -a /tmp/wordpress/. /var/www/html/wordpress


Se asignan los permisos adecuados para que Apache pueda gestionar los archivos:


sudo chown -R www-data:www-data /var/www/html/wordpress
sudo chmod -R 755 /var/www/html/wordpress


Se crea el archivo de configuración wp-config.php basándonos en la plantilla de ejemplo y ajustando los parámetros de la BBDD:


cd /var/www/html/wordpress
sudo cp wp-config-sample.php wp-config.php
sudo nano wp-config.php


*Valores configurados en wp-config.php:*
- DB_NAME: wordpress_db
- DB_USER: wp_user
- DB_PASSWORD: PasswordSegura123!
- DB_HOST: localhost

#### Paso 6: Configuración del Virtual Host en Apache
Se crea un archivo de configuración en Apache para el sitio de WordPress:


sudo nano /etc/apache2/sites-available/wordpress.conf


*Contenido del archivo:*
```
<VirtualHost *:80>
    ServerAdmin admin@local.test
    DocumentRoot /var/www/html/wordpress
    
    <Directory /var/www/html/wordpress/>
        AllowOverride All
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

Se habilita el nuevo sitio, se deshabilita el sitio por defecto y se recarga Apache:

sudo a2ensite wordpress.conf
sudo a2dissite 000-default.conf
sudo systemctl reload apache2


#### Paso 7: Finalización de la Instalación Web
Desde el navegador del equipo anfitrión, se accede a la IP asignada al VPS (ej. http://192.168.1.50/) para completar el asistente de instalación de WordPress (idioma, título del sitio, usuario administrador de WordPress y contraseña).

## 3. Registro de Errores y Soluciones

Durante el proceso de despliegue se identificaron las siguientes incidencias técnicas y sus correspondientes resoluciones:

### Error 1: Error al conectar con la Base de Datos (*Error establishing a database connection*)
**Causa:** El usuario de MySQL no tenía activado el método de autenticación por contraseña tradicional o las credenciales ingresadas en wp-config.php no coincidían exactamente con las creadas en la consola de MySQL.
**Solución:** Se redefinió la contraseña del usuario en MySQL asegurando la sintaxis adecuada mediante:
  
  ALTER USER 'wp_user'@'localhost' IDENTIFIED WITH mysql_native_password BY 'PasswordSegura123!'; <- sustituye la password por la que pusiste antes
  FLUSH PRIVILEGES;
  

### Error 2: Solicitud de credenciales FTP al instalar plugins o temas
**Causa:** Apache (www-data) no tenía permisos suficientes de escritura en el directorio /var/www/html/wordpress/wp-content.
**Solución:** Se reasignó la propiedad de los archivos al usuario del servidor web y se añadió la directiva de método directo en wp-config.php:

  sudo chown -R www-data:www-data /var/www/html/wordpress
  
  Añadiendo en wp-config.php:
  define('FS_METHOD', 'direct');
  

### Error 3: Los enlaces permanentes (*permalinks*) devuelven Error 404
**Causa:** El módulo mod_rewrite de Apache no estaba habilitado o faltaba la directiva AllowOverride All en el VirtualHost.
**Solución:** Se ejecutó sudo a2enmod rewrite y se verificó que la directiva AllowOverride All estuviese presente dentro del bloque <Directory> en /etc/apache2/sites-available/wordpress.conf.



## 4. Resultado Final y Verificación

Una vez finalizado el asistente de configuración inicial, la instalación del CMS queda plenamente operativa como se puede ver en el apartado de capturas.


## 5. Conclusiones
Se ha logrado instalar wordpress a partir de ubuntu. Este documento se ha hehco con el objetivo de documentar como he descargado el Wordpress y también como tutorial para quien quiera, si en medio de la practica ha surgido algun error inesperado puedes acceder a la documentación de arriba y al link de arriba.