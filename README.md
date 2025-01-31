# Practica-IAW.3.1
Repositorio para la práctica 3.1
En esta práctica vamos a realizar la administración de un sitio WordPress desde el terminal con la utilidad WP-CLI.

Con la utilidad WP-CLI podemos realizar las mismas tareas que se pueden hacer desde el panel de administración web de WordPress, pero desde la línea de comandos

En primer lugar, tenemos que tener instalada y configurada la pila LAMP haciendo uso del scripts que diseñamos en la Actividad 1.1, ademas del script `setup_letsencrypt_certificate.sh`

> [!IMPORTANT]  
> Antes de empezar a configurar los archivos vamos a crear la siguiente estructura de archivos y directorios
>![](Imagenes/estructura.png)

## Creación de un nombre de dominio 
Lo primero que vamos a hacer es crear un nombre de dominio para nuestro servidor. Vamos a usar el proveedor de nombres de dominio gratuito `No-Ip`

Nos registramos en la pagina y nos iremos a la sección no ip hostnames
![](Imagenes/acceso_noip.png)

Una vez dentro le damos a crear nombre de host
![](Imagenes/noip_cambiar_nombrehost.png)

Y se nos abrira un menú para crear el nombre de host, en mi caso le he cambiado el nombre y dominio y he añadido mi dirección IP
![](Imagenes/noip_host_creado_.png)

Hecho esto, pasaremos a configurar el script de deploy de wordpress con la utilidad wpcli

## Configuración del archivo deploy_wordpress_with_wpcli

### Primeros pasos
Empezaremos añadiendo las siguientes lineas, siendo la primera para mostrar los comandos y finalizar si hay error y la segunda para importar el archivo de variables para tener acceso a estas

```bash
set -ex

source .env
```
> [!IMPORTANT]  
> Antes de seguir vamos a asegurarnos que tenemos el archivo .env configurado de esta manera. El archivo.env debe contener valores para estas variables pero en la captura se han eliminado para proteger esta información
>![](Imagenes/archivo_.env.png)

### Descarga y configuración de WordPress y wpcli

Empezamos eliminando descargas previas de wp-cli

```bash
rm -rf /tmp/wp-cli.par
```
Una vez eliminado, descargamos la herramienta
```bash
curl -O https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar
```
Le asignamos permisos de ejecución al archivo wp-cli.phar
```bash
chmod +x wp-cli.phar
```
Movemos el archivo wp-cli.phar al directorio /usr/local/bin/
```bash
mv wp-cli.phar /usr/local/bin/wp
```
Eliminamos instalaciones previas en /var/www/html
```bash
rm -rf $WORDPRESS_DIRECTORY/*
```
Descargamos el código fuente de Wordpress
```bash
wp core download --locale=es_ES --path=/var/www/html --allow-root
```
Creamos la base de datos y el usuario para WordPress.
Usaremos las variables definidas en el .env
```bash
mysql -u root <<< "DROP DATABASE IF EXISTS $WORDPRESS_DB_NAME"
mysql -u root <<< "CREATE DATABASE $WORDPRESS_DB_NAME"
mysql -u root <<< "DROP USER IF EXISTS $WORDPRESS_DB_USER@$IP_CLIENTE_MYSQL"
mysql -u root <<< "CREATE USER $WORDPRESS_DB_USER@$IP_CLIENTE_MYSQL IDENTIFIED BY '$WORDPRESS_DB_PASSWORD'"
mysql -u root <<< "GRANT ALL PRIVILEGES ON $WORDPRESS_DB_NAME.* TO $WORDPRESS_DB_USER@$IP_CLIENTE_MYSQL"
```
Creamos el archivo de configuración usando las variables del .env
```bash
wp config create \
  --dbname=$WORDPRESS_DB_NAME \
  --dbuser=$WORDPRESS_DB_USER \
  --dbpass=$WORDPRESS_DB_PASSWORD \
  --dbhost=$WORDPRESS_DB_HOST \
  --path=$WORDPRESS_DIRECTORY \
  --allow-root
```
Instalamos WordPress
```bash
wp core install \
  --url=$LE_DOMAIN\
  --title="$WORDPRESS_TITLE" \
  --admin_user=$WORDPRESS_ADMIN_USER \
  --admin_password=$WORDPRESS_ADMIN_PASS \
  --admin_email=$WORDPRESS_ADMIN_EMAIL \
  --path=$WORDPRESS_DIRECTORY \
  --allow-root  
```
`--url`: Dominio del sitio WordPress. En una instalación en una máquina local también podríamos poner localhost o la dirección IP del servidor

`--title`: Título del sitio WordPress

`--admin_user`: Nombre del usuario administrador

`--admin_password`: Contraseña del usuario administrador

`--admin_email`: Email del usuario administrador

Añadimos un comando para añadir un tema a la pagina, en mi caso he elegido el mindscape
```bash
wp theme install mindscape --activate --path=$WORDPRESS_DIRECTORY --allow-root
```
Instalamos el plugin de hide login, quese encargara de poner la pagina de administracion de wordpress en en elace que digamos
```bash
wp plugin install wps-hide-login --activate --path=$WORDPRESS_DIRECTORY --allow-root
```
Configuraremos el plugin añadiendo el enlace que vamos a usar para el login
```bash
wp option update whl_page "$WORDPRESS_HIDE_LOGIN" --path=$WORDPRESS_DIRECTORY --allow-root
```
Tambien vamos a configurar los enlaces permanentes
```bash
wp rewrite structure '/%postname%/' --path=$WORDPRESS_DIRECTORY --allow-root
```
`-/%postname%/`: indica el patrón de reescritura que queremos utilizar

Copiamos el archivo htacces
```bash
cp ../htaccess/.htaccess $WORDPRESS_DIRECTORY
```
Finalmente modificamos el propietario y el grupo del directorio
```bash
chown -R www-data:www-data $WORDPRESS_DIRECTORY
```
### Comprobaciones
Ahora vamos a comprobar que se ha configurado todo correctamente

Ponemos el dominio en el navegador y vemos que nos mete directamente en la pagina ya configurada
![](Imagenes/Acceso_wp.png)

Si ponemos wordpress-asir.zapto.org/nadaimportante nos llevara al incio de sesion de admin ya que ese es enlace que hemos puesto para el login
![](Imagenes/wp_nadaimportante.png)

y si ponemos wordpress-asir.zapto.org/wp-admin nos dirá que pagina no encontrada
![](Imagenes/wp_no_encuentra.png)

Hecho esto el script deberia de estar asi
```bash
#!/bin/bash

# Configuramos para mostrar los comandos y finalizar si hay error
set -ex

#Importamos las variables de entorno
source .env

# Eliminamos descargas previas de wp-cli
rm -rf /tmp/wp-cli.par

# Descargamos el wP-CLI
curl -O https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar

# Le asignamos permisos de ejecución al archivo wp-cli.phar.
chmod +x wp-cli.phar

# Movemos el archivo wp-cli.phar al directorio /usr/local/bin/
mv wp-cli.phar /usr/local/bin/wp

# Eliminamos instalaciones previas en /var/www/html
rm -rf $WORDPRESS_DIRECTORY/*

# Descargamos el código fuente de WordPress
wp core download --locale=es_ES --path=/var/www/html --allow-root

# Creamos la base de datos y el usuario para WordPress
mysql -u root <<< "DROP DATABASE IF EXISTS $WORDPRESS_DB_NAME"
mysql -u root <<< "CREATE DATABASE $WORDPRESS_DB_NAME"
mysql -u root <<< "DROP USER IF EXISTS $WORDPRESS_DB_USER@$IP_CLIENTE_MYSQL"
mysql -u root <<< "CREATE USER $WORDPRESS_DB_USER@$IP_CLIENTE_MYSQL IDENTIFIED BY '$WORDPRESS_DB_PASSWORD'"
mysql -u root <<< "GRANT ALL PRIVILEGES ON $WORDPRESS_DB_NAME.* TO $WORDPRESS_DB_USER@$IP_CLIENTE_MYSQL"

# Creamos el archivo de configuración
wp config create \
  --dbname=$WORDPRESS_DB_NAME \
  --dbuser=$WORDPRESS_DB_USER \
  --dbpass=$WORDPRESS_DB_PASSWORD \
  --dbhost=$WORDPRESS_DB_HOST \
  --path=$WORDPRESS_DIRECTORY \
  --allow-root

# Instalación de WordPress

wp core install \
  --url=$LE_DOMAIN\
  --title="$WORDPRESS_TITLE" \
  --admin_user=$WORDPRESS_ADMIN_USER \
  --admin_password=$WORDPRESS_ADMIN_PASS \
  --admin_email=$WORDPRESS_ADMIN_EMAIL \
  --path=$WORDPRESS_DIRECTORY \
  --allow-root  
 
# Instalamos y actiamos el theme mindscape
wp theme install mindscape --activate --path=$WORDPRESS_DIRECTORY --allow-root

#Instalamos un plugin
wp plugin install wps-hide-login --activate --path=$WORDPRESS_DIRECTORY --allow-root

#Configuramos el plugin de Url
wp option update whl_page "$WORDPRESS_HIDE_LOGIN" --path=$WORDPRESS_DIRECTORY --allow-root
  
# Enlaces permanentes
wp rewrite structure '/%postname%/' --path=$WORDPRESS_DIRECTORY --allow-root

# Copiamos el archivo .htaccess
cp ../htaccess/.htaccess $WORDPRESS_DIRECTORY
  
# Modificamos el propietario y el grupo del directio de /var/www/html
chown -R www-data:www-data $WORDPRESS_DIRECTORY
```
