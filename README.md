# Practica-IAW.3.1
Repositorio para la práctica 3.1

(BORRADOR)
En esta práctica vamos a realizar la implantación de la aplicación web Moodle en dos instancias EC2 de Amazon Web Services (AWS) haciendo uso de playbooks de Ansible. En una de las instancias deberemos instalar Apache HTTP Server y los módulos necesarios de PHP y en la otra máquina deberá instalar MySQL Server.

> [!IMPORTANT]  
> Antes de empezar a configurar los archivos vamos a crear la siguiente estructura de archivos y directorios
>![](Imagenes/estructura.png)

## Creación de un nombre de dominio 
Lo primero que vamos a hacer es crear un nombre de dominio para nuestro servidor. Vamos a usar el proveedor de nombres de dominio gratuito `No-Ip`

Nos registramos en la pagina y nos iremos a la sección no ip hostnames
![](Imagenes/noip1.png)

Una vez dentro le damos a crear nombre de host
![](Imagenes/noip2.png)

Y se nos abrira un menú para crear el nombre de host, en mi caso le he cambiado el nombre y dominio y he añadido la dirección ip del frontal
![](Imagenes/noip3.png)

Hecho esto, pasaremos a configurar playbook que reunirá todos 

> [!IMPORTANT]  
> Antes de seguir vamos a asegurarnos que tenemos el archivo de variables configurado de esta manera. Este archivo sera importado por los diferentes playbooks
> ```bash
> Variables para el certificado de LetsEncrypt
>letsencrypt:
>  email: "admin@gmail.com"
>  domain: "practicaiawmoodle.zapto.org"
>
> Variables para Moodle
>moodle:
>  version: "4.3.1"
>  lang: "es"
>  wwwroot: "https://practicaiawmoodle.zapto.org"
>  dataroot: "/var/moodledata"
>db:
>    type: "mysqli"
>    host: "172.31.33.225"
>    name: "Moodle"
>    user: "admin"
>    pass: "Usuario?"
>  fullname: "Practica 3.1"
>  shortname: "MOODLE"
>  summary: "Esta es mi web de Moodle"
>  admin:
>    user: "admin"
>    pass: "Usuario0?"
>    email: "demo@demo.es"
>
> Direcciones IP
>ips:
>  frontend_private: "172.31.47.161"
>  backend_private: "172.31.33.225"
>
> Rutas
>routes:
>  apache_ini: "/etc/php/8.3/apache2/php.ini"
>  cli_ini: "/etc/php/8.3/cli/php.ini"
>  config_php: "/var/www/html/config.php"
>  moodle_directory: "/var/www/html/moodle"
>```

## Configuración del playbook main.yml
Este playbook recopila todos en uno solo para desde un solo playbook ejecutar todos. Main.yml deberá tener este codigo

```
---
- import_playbook: playbooks/install_lamp_frontend.yaml
- import_playbook: playbooks/install_lamp_backend.yaml
- import_playbook: playbooks/setup_letsencrypt_certificate.yaml
- import_playbook: playbooks/deploy_moodle_backend.yaml
- import_playbook: playbooks/deploy_moodle_frontend.yaml
```

Vamos a explicar los playbooks en el orden de que serán ejecutados

 A PARTIR DE AQUI ES BORRADOR. HAY QUE CAMBIAR CIERTAS COSAS
### Configuración de install install_lamp_frontend.yaml

Empezamos eliminando descargas previas de wp-cli

```bash
---
- name: Configuración de MySQL
  hosts: backend
  become: yes
  vars_files:
    - ../vars/variables.yml  # Archivo YAML con las variables necesarias

  tasks:
    - name: Actualizar los repositorios
      apt:
        update_cache: yes

    - name: Actualizar los paquetes instalados
      apt:
        upgrade: dist
        autoremove: yes

    - name: Instalar MySQL Server
      apt:
        name: mysql-server
        state: present

    - name: Instalamos el módulo de pymysql
      apt:
        name: python3-pymysql
        state: present

    - name: Configurar el archivo mysqld.cnf
      lineinfile:
        path: /etc/mysql/mysql.conf.d/mysqld.cnf
        regexp: "^bind-address\\s*=\\s*.*"
        line: "bind-address = {{ ips.backend_private }}"
        state: present
        
    - name: Reiniciar el servicio de MySQL
      service:
        name: mysql
        state: restarted
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
