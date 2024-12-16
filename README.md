# PracticaCMS 

## Índice
1. Descripción
    - Direccionamiento IP  
2. Pasos de Instalación y Configuración  
    - Capa 1: Balanceador 
    - Capa 2: Servidores Web y NFS  
    - Capa 3: Base de Datos 
    - Despliegue y Configuración del CMS
3. Resultado final


## 1.Descripción

La práctica consiste en desplegar un CMS (OwnCloud) sobre una infraestructura en alta disponibilidad basada en la pila LEMP (Linux, Nginx, MariaDB, PHP-FPM). Esta infraestructura está organizada en tres capas: un balanceador de carga con Nginx (capa 1), dos servidores web con Nginx y un servidor NFS con PHP-FPM (capa 2), y una base de datos MariaDB (capa 3). Con respecto a la la infraestructura, solo dispondremos de acceso al exterior desde la capa pública, es decir, desde la primera;también impediremos la conexion entre la primera y la tercera capa, además la segunda y la tercera capa no estarán expuestas a la red pública

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

### - Capa1: Balanceador

# Script de Shell: `balanceador.sh`


### - Capa2: Servidores Web y NFS

### - Capa3: Base de datos

### - Despliegue y Configuración del CMS

## 3.Resultado final

El único problema que tendremos para acceder a la práctica es que tendremos que entrar en la máquina balanceador para ver su ip pública, debido a que el reenvio de puertos no es posible.

![image](https://github.com/user-attachments/assets/48e5981b-7359-47d9-b5dc-51cf0606271a)

Y una vez dentro con el usuario admin y la contraseña 1234 el resultado es:

![image](https://github.com/user-attachments/assets/e2e46688-d7b4-408b-bc22-70eae9b58010)


