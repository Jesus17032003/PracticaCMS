# PracticaCMS 

## Índice
1. Descripción
    - Direccionamiento IP  
2. Pasos de Instalación y Configuración

    - Vagrantfile
    - Capa 1: Balanceador 
    - Capa 2: Servidores Web y NFS  
    - Capa 3: Base de Datos 
3. Resultado final


## 1.Descripción

La práctica consiste en desplegar un CMS (OwnCloud) sobre una infraestructura en alta disponibilidad basada en la pila LEMP (Linux, Nginx, MariaDB, PHP-FPM). Esta infraestructura está organizada en tres capas: un balanceador de carga con Nginx (capa 1), dos servidores web con Nginx y un servidor NFS con PHP-FPM (capa 2), y una base de datos MariaDB (capa 3). Con respecto a la infraestructura, solo dispondremos de acceso al exterior desde la capa pública, es decir, desde la primera; también impediremos la conexión entre la primera y la tercera capa. Además, la segunda y la tercera capa no estarán expuestas a la red pública.

El objetivo es garantizar modularidad y escalabilidad mediante la configuración de un balanceador de carga, almacenamiento centralizado con NFS y un sistema de bases de datos robusto. La práctica también incluye una personalización mínima del CMS desplegado y la documentación detallada del proceso, incluyendo configuraciones, capturas de pantalla y explicaciones técnicas.

### - Direccionamiento IP

| Nombre de la Máquina     | Hostname             | Dirección IP        | Red Asociada   | Script de Provisión          |
|--------------------------|----------------------|---------------------|----------------|------------------------------|
| SGBDJesusA              | SGBDJesusA          | 192.168.60.10       | red_Sgbd       | BDD.sh                       |
| NFSJesusA               | NFSJesusA           | 192.168.56.12       | red1           | nfs.sh                       |
|                          |                     | 192.168.60.13       | red_Sgbd       |                              |
| Web1JesusA              | Web1JesusA          | 192.168.56.10       | red1           | webs.sh                      |
|                          |                     | 192.168.60.11       | red_Sgbd       |                              |
| Web2JesusA              | Web2JesusA          | 192.168.56.11       | red1           | webs.sh                      |
|                          |                     | 192.168.60.12       | red_Sgbd       |                              |
| BalanceadorJesusA       | BalanceadorJesusA   | 192.168.56.1        | red1           | balanceador.sh               |
|                          |                     | (red pública)       | public_network |                              |


## 2.Pasos de la Instalación y Configuración

### - Vagrantfile

``` Vagrantfile
  config.vm.box = "debian/bullseye64"

  config.vm.define "SGBDJesusA" do |app|
    app.vm.hostname = "SGBDJesusA"
    app.vm.network "private_network", ip: "192.168.60.10", virtualbox_intnet: "red_Sgbd"
    app.vm.provision "shell", path: "BDD.sh"
  end

  config.vm.define "NFSJesusA" do |app|
    app.vm.hostname = "NFSJesusA"
    app.vm.network "private_network", ip: "192.168.56.12", virtualbox_intnet: "red1"
    app.vm.network "private_network", ip: "192.168.60.13", virtualbox_intnet: "red_Sgbd"
    app.vm.provision "shell", path: "nfs.sh"
  end

  config.vm.define "Web1JesusA" do |app|
    app.vm.hostname = "Web1JesusA"
    app.vm.network "private_network", ip: "192.168.56.10", virtualbox_intnet: "red1"
    app.vm.network "private_network", ip: "192.168.60.11", virtualbox_intnet: "red_Sgbd"
    app.vm.provision "shell", path: "webs.sh"
  end

  config.vm.define "Web2JesusA" do |app|
    app.vm.hostname = "Web2JesusA"
    app.vm.network "private_network", ip: "192.168.56.11", virtualbox_intnet: "red1"
    app.vm.network "private_network", ip: "192.168.60.12", virtualbox_intnet: "red_Sgbd"
    app.vm.provision "shell", path: "webs.sh"
  end

  config.vm.define "BalanceadorJesusA" do |app|
    app.vm.hostname = "BalanceadorJesusA"
    app.vm.network "public_network"
    app.vm.network "private_network", ip: "192.168.56.1", virtualbox_intnet: "red1"
    app.vm.provision "shell", path: "balanceador.sh"
  end 
```
Este Vagrantfile configura un entorno virtualizado con 5 máquinas virtuales (VM) que tienen roles específicos y están conectadas a diferentes redes. Primero, se define la VM SGBDJesusA, que actúa como el servidor de base de datos y está conectada a la red privada red_Sgbd con la IP 192.168.60.10. Luego, la VM NFSJesusA se configura para compartir archivos a través de NFS, conectándose a dos redes privadas (red1 con la IP 192.168.56.12 y red_Sgbd con la IP 192.168.60.13). La siguiente máquina, Web1JesusA, se conecta a las mismas redes privadas y tiene la IP 192.168.56.10 en red1 y 192.168.60.11 en red_Sgbd, funcionando como un servidor web. La VM Web2JesusA se configura de forma similar con la IP 192.168.56.11 en red1 y 192.168.60.12 en red_Sgbd. Finalmente, el BalanceadorJesusA está conectado a una red pública y a la red privada red1 con la IP 192.168.56.1, actuando como un balanceador de carga que distribuye el tráfico entre los servidores web.

Es importante que las máquinas estén definidas en este orden específico en el Vagrantfile para asegurar que cada VM pueda acceder a los servicios de las demás. El servidor de base de datos SGBDJesusA debe estar listo antes de que los servidores web Web1JesusA y Web2JesusA intenten acceder a la base de datos. Del mismo modo, NFSJesusA debe estar configurada antes de que las VMs web puedan montar y acceder a los archivos compartidos a través de NFS. Finalmente, el balanceador de carga BalanceadorJesusA debe configurarse después de los servidores web para asegurarse de que esté disponible para distribuir el tráfico adecuadamente.

### - Capa1: Balanceador

```bash
#!/bin/bash

# Actualizar repositorios e instalar nginx
sudo apt-get update -y
sudo apt-get install -y nginx

# Configuracion de Nginx como balanceador de carga
cat <<EOF > /etc/nginx/sites-available/default
upstream backend_servers {
    server 192.168.56.10;
    server 192.168.56.11;
}

server {
    listen 80;
    server_name localhost;

    location / {
        proxy_pass http://backend_servers;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
    }
}

EOF

sudo systemctl restart nginx
```

Este script de Bash automatiza la instalación y configuración de Nginx como un balanceador de carga en un sistema basado en Ubuntu. Primero, actualiza los repositorios del sistema e instala el servidor web Nginx. Luego, crea un archivo de configuración en /etc/nginx/sites-available/default que define un grupo de servidores backend (con direcciones IP 192.168.56.10 y 192.168.56.12) para distribuir las solicitudes entrantes. Finalmente, configura Nginx para escuchar en el puerto 80 y redirigir las solicitudes a los servidores backend, pasando las cabeceras adecuadas.


### - Capa2: Servidores Web y NFS

#### Servidor Web

```bash
#!/bin/bash

# Actualizar repositorios e instalar nginx, nfs-common y PHP 7.4
sudo apt-get update -y
sudo apt-get install -y nginx nfs-common php7.4 php7.4-fpm php7.4-mysql php7.4-gd php7.4-xml php7.4-mbstring php7.4-curl php7.4-zip php7.4-intl php7.4-ldap
sudo apt install mariadb-client 

# Crear directorio para montar la carpeta compartida por NFS
sudo mkdir -p /var/www/html

# Montar la carpeta NFS desde el servidor NFS
sudo mount -t nfs 192.168.56.12:/var/www/html /var/www/html

# Añadir entrada al /etc/fstab para montaje persistente
echo "192.168.56.12:/var/www/html /var/www/html nfs defaults 0 0" >> /etc/fstab

# Configuración de Nginx 
cat <<EOF > /etc/nginx/sites-available/default
server {
    listen 80;

    root /var/www/html/owncloud;
    index index.php index.html index.htm;

    location / {
        try_files \$uri \$uri/ /index.php?\$query_string;
    }

    location ~ \.php\$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass 192.168.56.12:9000;
        fastcgi_param SCRIPT_FILENAME \$document_root\$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ ^/(?:\.htaccess|data|config|db_structure\.xml|README) {
        deny all;
    }
}
EOF

nginx -t

sudo systemctl restart nginx

sudo systemctl restart php7.4-fpm

sudo ip route del default 
```
Este script de Bash automatiza la instalación y configuración de un servidor web Nginx con soporte para PHP y NFS en un sistema basado en Ubuntu. Primero, actualiza los repositorios y luego instala Nginx, PHP 7.4 con varios módulos (como soporte para bases de datos MySQL, GD, CURL, etc.), y el paquete nfs-common necesario para montar sistemas de archivos compartidos mediante NFS. A continuación, crea un directorio en /var/www/html y monta una carpeta compartida desde un servidor NFS (con la IP 192.168.56.12) en ese directorio, asegurando que el montaje sea persistente mediante una entrada en el archivo /etc/fstab. Luego, configura Nginx para que sirva contenido desde el directorio /var/www/html/owncloud y maneje archivos PHP mediante FastCGI, enviando las solicitudes PHP al servidor NFS. Finalmente, reinicia los servicios de Nginx y PHP-FPM para aplicar la configuración y elimina la ruta por defecto de red (ip route del default), probablemente para configurar una nueva ruta de red.

#### Servidor NFS

```bash
#!/bin/bash

# Actualizar repositorios e instalar NFS y PHP 7.4
sudo apt-get update -y
sudo apt-get install -y nfs-kernel-server php7.4 php7.4-fpm php7.4-mysql php7.4-gd php7.4-xml php7.4-mbstring php7.4-curl php7.4-zip php7.4-intl php7.4-ldap unzip

# Crear carpeta compartida para OwnCloud y configurar permisos
sudo mkdir -p /var/www/html
sudo chown -R www-data:www-data /var/www/html
sudo chmod -R 755 /var/www/html

# Configurar NFS para compartir la carpeta
echo "/var/www/html 192.168.56.11(rw,sync,no_subtree_check)" >> /etc/exports
echo "/var/www/html 192.168.56.10(rw,sync,no_subtree_check)" >> /etc/exports

# Reiniciar NFS para aplicar cambios
sudo exportfs -a
sudo systemctl restart nfs-kernel-server

# Descargar y configurar OwnCloud
cd /tmp
wget https://download.owncloud.com/server/stable/owncloud-10.9.1.zip
unzip owncloud-10.9.1.zip
mv owncloud /var/www/html/

# Configurar permisos de OwnCloud
sudo chown -R www-data:www-data /var/www/html/owncloud
sudo chmod -R 755 /var/www/html/owncloud

# Crear archivo de configuración inicial para OwnCloud
cat <<EOF > /var/www/html/owncloud/config/autoconfig.php
<?php
\$AUTOCONFIG = array(
  "dbtype" => "mysql",
  "dbname" => "owncloud",
  "dbuser" => "owncloud",
  "dbpassword" => "1234",
  "dbhost" => "192.168.60.10",
  "directory" => "/var/www/html/owncloud/data",
  "adminlogin" => "JesusA",
  "adminpass" => "1234"
);
EOF

# Modificar el archivo config.php 
echo "Añadiendo dominios de confianza a la configuración de OwnCloud..."
php -r "
  \$configFile = '/var/www/html/owncloud/config/config.php';
  if (file_exists(\$configFile)) {
    \$config = include(\$configFile);
    \$config['trusted_domains'] = array(
      'localhost',
      'localhost:8080',
      '192.168.56.10',
      '192.168.56.11',
      '192.168.56.12',
    );
    file_put_contents(\$configFile, '<?php return ' . var_export(\$config, true) . ';');
  } else {
    echo 'No se pudo encontrar el archivo config.php';
  }
"

sed -i 's/^listen = .*/listen = 192.168.56.12:9000/' /etc/php/7.4/fpm/pool.d/www.conf

sudo systemctl restart php7.4-fpm

sudo ip route del default 
```

### - Capa3: Base de datos

```bash
#!/bin/bash

# Actualizar repositorios e instalar MariaDB
sudo apt-get update -y
sudo apt-get install -y mariadb-server

# Configurar MariaDB para permitir acceso remoto desde los servidores web
sed -i 's/bind-address.*/bind-address = 192.168.60.10/' /etc/mysql/mariadb.conf.d/50-server.cnf

sudo systemctl restart mariadb

mysql -u root <<EOF
CREATE DATABASE owncloud;
CREATE USER 'owncloud'@'192.168.60.%' IDENTIFIED BY '1234';
GRANT ALL PRIVILEGES ON owncloud.* TO 'owncloud'@'192.168.60.%';
FLUSH PRIVILEGES;
EOF

sudo ip route del default 
```
Este script de Bash automatiza la instalación y configuración de MariaDB para ser utilizado por un servidor de bases de datos remoto en un entorno de servidores web. Primero, actualiza los repositorios e instala MariaDB. Luego, modifica la configuración de MariaDB para permitir conexiones remotas, específicamente desde las direcciones IP dentro del rango 192.168.60.%, al cambiar la directiva bind-address en el archivo de configuración /etc/mysql/mariadb.conf.d/50-server.cnf. Después de reiniciar el servicio de MariaDB, el script crea una base de datos llamada owncloud y un usuario owncloud con acceso desde los servidores web (en el rango de IP 192.168.60.%), asignándole todos los privilegios sobre la base de datos owncloud. Finalmente, ejecuta un comando ip route del default, lo que probablemente elimina una ruta de red por defecto, posiblemente para configurar una nueva ruta de red.


## 3.Resultado final

El único problema que tendremos para acceder a la práctica es que tendremos que entrar en la máquina balanceador para ver su ip pública, debido a que el reenvio de puertos no funciona.

![image](https://github.com/user-attachments/assets/21d4e9fc-de60-48bb-b399-eb20d9453e7f)


Y una vez dentro con el usuario JesusA y la contraseña 1234 el resultado es:

![image](https://github.com/user-attachments/assets/35bbbc2d-7993-48ab-ac77-15f33718858e)


