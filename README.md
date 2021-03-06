# ☁️ Un hogar seguro para todos tus datos ☁️

![alt text](https://github.com/JuanRodenas/Nextcloud_server/blob/main/NextcloudHub.jpg)

# Nextcloud
Seguridad🔒 Rendimiento🚀 Control☑️

* 📁 **Accede a tus datos** Puedes almacenar tus archivos, contactos, calendarios y más en un servidor de tu elección.
* 🔄 **Sincroniza tus datos** Mantén tus archivos, contactos, calendarios y más sincronizados entre tus dispositivos.
* 🙌 **Comparte tus datos** ...dando a otros acceso a las cosas que quieres que vean o con las que quieren colaborar.
* 🖥 [**Instalar** un servidor por ti mismo](https://nextcloud.com/install/#instructions-server) en tu propio hardware o utilizando una de nuestras **aplicaciones** listas para usar.
* 📦 Compra uno de los [impresionantes **dispositivos** que vienen con Nextcloud preinstalado](https://nextcloud.com/devices/)
* 🚀 [Documentación oficial](https://docs.nextcloud.com/server/stable/user_manual/es/)


## PREPARACIÓN DE LOS ARCHIVOS Y DIRECTORIOS

#### Crear los directorios donde se montarán los volúmenes de persistencia
Creamos los directorios donde se montarán los volúmenes de persistencia
~~~
mkdir /patch/to/data/nextcloud/nextcloud
mkdir /patch/to/data/nextcloud/mysql
mkdir /patch/to/data/nextcloud/redis
~~~~

#### Damos al user www-data y permisos a los archivos
Damos permiso de escritura/lectura y le damos permisos al usuario www-data a la carpeta donde están los documentos
<p>Le damos el grupo del usuario al usuario www-data:</p>
<p>  &nbsp;&nbsp;<code>sudo usermod -a -G ${USER} www-data</code></p>
<p>  Y le damos permisos al usuario www-data a los archivos:
<p>  &nbsp;&nbsp;<code>sudo chown -R www-data:${USER} /patch/to/data/nextcloud</code></p>
<p>  &nbsp;&nbsp;<code>sudo chmod -R 770 /patch/to/data/nextcloud</code></p>

Directorios:
* **files** Contendrá los archivos almacenados en nuestra nube. También contendrá los ficheros de configuración, ficheros de las aplicaciones instaladas, etc. Es importante realizar una copia de seguridad de este directorio/volumen de persistencia.
* **mysql** Contendrá la totalidad de ficheros de nuestra base de datos MySQL.
* **redis** Contiene las bases de datos que genera el servidor Redis. Obviamente también es interesante realizar una copia de seguridad de este directorio.
* **backup** Las copias de seguridad de la base de datos. En caso de utilizar un volumen llamado `/backup`, puede realizar una copia de seguridad de la base de datos y almacenarla en este directorio tan solo tenéis que ejecutar el comando `sudo docker-compose exec db backup`

#
<blockquote class="is-info"><p>Los pasos que se explican a continuación están basados en una red que puede diferir de la que tú tienes montada. Si sigues al pie de la letra todos los pasos, pueden no coincidir con la configuración de tu <em>red</em> y dejarla inservible. Adapta en todo momento lo que a continuación se expone para que cuadre con tu red.</p></blockquote>

#### Crear la red interna para comunicar con los demás contenedores
Creada la red interna, ya podemos levantar el contenedor
~~~~
docker network create nextcloud_internal
~~~~

## EXPLICACIÓN DE LOS PARÁMETROS USADOS EN EL DOCKER-COMPOSE PARA INSTALAR NEXTCLOUD CON TRAEFIK
Con el servicio Traefik v2 configurado de forma correcta tan solo tenemos que añadir las etiquetas pertinentes al Docker-Compose para configurar el servicio que levantaremos a nuestro gusto. Las etiquetas y parámetros introducidos en mi caso han sido los siguientes.

#### Definir los contenedores que serán accesibles desde el exterior de la red local a través de Traefik
Lo primero que tenemos que tener claro  son los contenedores que queremos exponer al exterior y los que no. En 
nuestro caso tenemos los contenedores nextcloud, db y redis. El único contenedor que tiene que ser accesible fuera de nuestra red local es el de nextcloud. Por lo tanto en las etiquetas (labels) de los contenedores de redis y db añadimos el siguiente código:
~~~~
      - traefik.enable=false
~~~~

En cambio en el contenedor de nextcloud añadimos la etiqueta:
~~~~
      - traefik.enable=true
~~~~

#### Definir la red en que se levantará cada uno de los contenedores
Al configurar Traefik v2 definimos que el contenedor nextcloud tiene que ser accesible desde fuera y dentro de nuestra red local. Por lo tanto mediante el siguiente código definimos que el contenedor nextcloud sea accesible a través de la red `internal` y la red `nextcloud`.
Para los contenedores que tienen que ser accesibles al exterior usamos el networks `nextcloud`. Si el contenedor tiene que acceder tambien a la red local, le añadimos la red `internal`:
~~~~
    networks:
      - internal
      - nextcloud
~~~~

El resto de contenedores no tienen que ser accesibles desde el exterior de nuestra red local. Por lo tanto tan solo tenemos que levantarlos en la red `internal` y esto lo hacemos mediante el siguiente código.
~~~~
    networks:
      - internal
~~~~

## LEVANTAR EL CONTENEDOR DE NEXTCLOUD
En la misma ubicación que hemos indicado la carpeta Nextcloud, descargamos el `docker-compose.yml`
☑️ [Docker-compose](https://github.com/JuanRodenas/Nextcloud_server/blob/main/docker-compose.yml)
```
  - Modificamos la ruta de los volúmenes a la ruta donde estén los archivos.
  - Modificamos la red `networks:`
  - Modificamos las passwords y usuarios del docker-compose de `MySQL, REDIS y NEXTCLOUD`
  - Introducimos la red que levante en la red internal de docker: `TRUSTED_PROXIES=172.19.0.0/16`
```

#### Definir las variables de entorno de nextcloud
Las variables de entorno de configuración de nextcloud:
~~~
    environment:
      - NEXTCLOUD_ADMIN_USER=admin
      - NEXTCLOUD_ADMIN_PASSWORD= 'YOUR_PASSWORD'
      - MYSQL_PASSWORD= 'YOUR_PASSWORD'
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_HOST=db:3306
      - TZ=Europe/Madrid
      - OVERWRITEPROTOCOL=https
      - OVERWRITEHOST='YOUR_DOMAIN'
      - REDIS_HOST=redis
      - REDIS_HOST_PASSWORD= 'YOUR_PASSWORD'
      - NEXTCLOUD_TRUSTED_DOMAINS= 'YOUR_DOMAIN'
      - TRUSTED_PROXIES=172.20.0.0/16
~~~

* Levantamos el contenedor con:
~~~
docker-compose up -d
~~~

Una vez ejecutado el comando se descargarán las imagenes del docker-compose y se crearán, levantarán los contenedores.

#### Ver el log del contenedor
* Vemos el contenedor:
~~~
docker logs nextcloud
~~~
* Vemos todos los contenedores del docker-compose:
~~~
docker-compose logs -f
~~~

## Configurar configuración de nextcloud
Para poder configurar el archivo `config.php` debemos acceder al contenedor de nextcloud como `root`
~~~
docker exec -u root -t -i nextcloud /bin/bash
~~~
* Una vez que hemos accedido al contenedor, tenemos que actualizar e instalar `sudo` y `nano` para poder modificar el archivo
~~~
apt update && apt install nano imagemagick sudo
~~~

### Ya instalado los paquetes accedemos al contenedor como `www-data` para poder modificar los archivos
~~~
docker exec -u www-data -t -i nextcloud /bin/bash
~~~
* Una vez instalado abrimos el archivo en la ruta, ponemos el servidor en modo mantenimiento
~~~
php occ maintenance:mode --on
~~~
* Creamos la carpeta para los archivos temporales
~~~
mkdir -p /var/www/html/var/tmp
~~~
* Una vez instalado abrimos el archivo en la ruta
~~~
nano config/config.php
~~~
#### Configuración del archivo config:
Añadimos al final del archivo
~~~
  'filelocking.enabled' => true,
  'overwritehost' => 'your_domain',
  'overwrite.cli.url' => 'https://your_domain',
  'htaccess.RewriteBase' => '/',
  'default_phone_region' => 'ES',
  'force_language' => 'es',
  'default_locale' => 'es_ES',
  'force_locale' => 'es_ES',
  'updater.release.channel' => 'stable',
  'maintenance' => false,
  'trashbin_retention_obligation' => 'auto',
  'tempdirectory' => '/var/www/html/var/tmp',
~~~
* Modificamos el dominio a utilizar en nextcloud:
```
  'trusted_domains' => 
  array (
    0 => 'localhost',
    1 => 'your_domain',
```
#### Problema con MEMORY_LIMIT
Para solucionar el problema del `Fatal error: Allowed memory size of 2097152 bytes exhausted (tried to allocate 446464 bytes) in /var/www/html/3rdparty/composer/autoload_real.php on line 37`
* Primero comprobamos el php:
```
php -i | grep memory_limit
```
* Accedemos al contenedor de docker y cambiar en `/usr/local/etc/php/conf.d/nextcloud.ini` los parámetros de: 
```
nano /usr/local/etc/php/conf.d/nextcloud.ini
```
* Modificar estas líneas:
```
memory_limit=2048M
upload_max_filesize=100G 
post_max_size=100G
```

#### Una vez realizado los cambios, reestructuramos de nuevo nextcloud
~~~
php occ maintenance:repair
~~~
~~~
php occ maintenance:update:htaccess
~~~
~~~
php occ maintenance:mimetype:update-db
~~~
* Una vez realizado los cambios, podremos quitar el modo mantenimiento y podemos acceder a configurar nuestra cuenta.
~~~
php occ maintenance:mode --off
~~~

### ACCEDER A LA WEB O DASHBOARD DE NEXTCLOUD
Con el contenedor levantado tan solo tenemos que abrir el navegador web e ingresar a la URL que indicamos en `TRUSTED_DOMAINS`.
Una vez ingresadas la credenciales tendremos acceso al panel de control. Fíjense que estamos accediendo de forma segura mediante https y TLS.
![alt text](https://github.com/JuanRodenas/Nextcloud_server/blob/main/nextcloud-interfaz-web.png)

### COPIAS DE SEGURIDAD
Si queremos realizar una copias de seguridad de la configuración o recuperar el backup, Pulsa en la imagen para visitar el repositorio de copias de seguridad.
<p align="center">
    <a href="https://github.com/JuanRodenas/Backup/blob/main/README.md">
        <img src="https://github.com/JuanRodenas/Pi-hole_list/blob/main/cloud-backup.png" width="400" height="200">
    </a>
    <br>
    <strong>Pulsa en la imagen para visitar el repositorio de copias de seguridad.</strong>
</p>

### HELP ME
<p> &nbsp;Si quieres contribuir a mejorar Nextcloud o tienes algún error, abre un <code>issue</code> y te ayudaré a solucionarlo:</p>
<sup>ábreme un problema aquí <A HREF="https://github.com/JuanRodenas/Nextcloud_server/issues"> ISSUE </A>.</sup>


### Credits
Si la guía te ha gustado y tienes funcionando un NAS personal, invítame a un café por mi trabajo.
#
<a href="https://www.paypal.com/donate/?hosted_button_id=HVJT2YDSHRZY2" target="_blank"><img src="https://cdn.buymeacoffee.com/buttons/v2/default-yellow.png" alt="Buy Me A Coffee" style="height: 60px !important;width: 217px !important;" ></a>

## 🎉 ¡Ready!
