# Práctica docker-compose para AWEB

Conjunto de cinco prácticas con **Docker Compose** sobre Windows. Cada práctica amplía la anterior y permite entender cómo se integran servidores web, lenguajes de servidor y bases de datos en una arquitectura basada en contenedores.

**Autor:** Badr Diboune El Wasmi
**Módulo:** Aplicaciones Web (AWEB), Tarea módulo 7

| Práctica | Objetivo |
|---|---|
| 1 | Servidor web sencillo con Nginx |
| 2 | Nginx + PHP-FPM |
| 3 | Añadir MySQL y conectar PHP a la base de datos |
| 4 | Desplegar WordPress conectado a MySQL |
| 5 | Desplegar Joomla usando el mismo servidor MySQL |

---

## Requisitos

- Windows con **Docker Desktop** instalado y el motor en marcha
- PowerShell
- Un editor de texto (VS Code o Bloc de notas)

## Estructura final del proyecto

```
16-11-2026/
├── docker-compose.yml
├── html/
│   ├── index.html
│   ├── index.php
│   ├── wordpress/
│   └── joomla/
├── nginx/
│   └── default.conf
└── php/
    └── Dockerfile
```

## Preparación

1. Abre PowerShell en la carpeta del proyecto (en el Explorador, clic en la barra de direcciones, escribe `powershell` y pulsa Enter).
2. Comprueba el entorno:

```powershell
pwd
docker --version
docker compose version
```

3. Crea las carpetas:

```powershell
mkdir html, nginx, php
```

---

## Práctica 1: Servidor web con Nginx

**`html/index.html`**

```html
<h1>Practica 1: Nginx funcionando</h1>
```

**`docker-compose.yml`**

```yaml
services:
  web:
    image: nginx:latest
    container_name: nginx
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html
```

**Ejecución**

```powershell
docker compose up -d
docker ps
```

**Comprobación:** abre <http://localhost:8080>.

---

## Práctica 2: Nginx + PHP

**`nginx/default.conf`**: Nginx envía los archivos `.php` al contenedor PHP-FPM.

```nginx
server {
    listen 80;
    root /var/www/html;
    index index.php index.html;
    client_max_body_size 64M;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass php:9000;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
```

**`html/index.php`**

```php
<?php phpinfo();
```

**`docker-compose.yml`** (reemplaza al anterior)

```yaml
services:
  web:
    image: nginx:latest
    container_name: nginx
    ports:
      - "8080:80"
    volumes:
      - ./html:/var/www/html
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - php

  php:
    image: php:8.3-fpm
    container_name: php
    volumes:
      - ./html:/var/www/html
```

**Ejecución**

```powershell
docker compose down
docker compose up -d
docker exec -it php ls /var/www/html
```

**Comprobación:** abre <http://localhost:8080/index.php>. Debe verse la página de `phpinfo()`.

---

## Práctica 3: MySQL y conexión desde PHP

**`php/Dockerfile`**: PHP necesita las extensiones para hablar con MySQL.

```dockerfile
FROM php:8.3-fpm

RUN docker-php-ext-install mysqli pdo pdo_mysql
```

**`docker-compose.yml`** (reemplaza al anterior)

```yaml
services:
  web:
    image: nginx:latest
    container_name: nginx
    ports:
      - "8080:80"
    volumes:
      - ./html:/var/www/html
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - php

  php:
    build: ./php
    container_name: php
    volumes:
      - ./html:/var/www/html
    depends_on:
      - db

  db:
    image: mysql:8.0
    container_name: mysql
    environment:
      MYSQL_ROOT_PASSWORD: root123
      MYSQL_DATABASE: practica
      MYSQL_USER: usuario
      MYSQL_PASSWORD: usuario123
    volumes:
      - db_data:/var/lib/mysql

volumes:
  db_data:
```

**`html/index.php`** (reemplaza el contenido)

```php
<?php
$conn = new mysqli("db", "usuario", "usuario123", "practica");
if ($conn->connect_error) {
    die("Error de conexion: " . $conn->connect_error);
}
echo "Conexion correcta a MySQL, version " . $conn->server_info;
```

El host es `db` porque Docker Compose resuelve el nombre del servicio dentro de su red.

**Ejecución**

```powershell
docker compose up -d --build
docker compose ps
```

**Comprobación**

- Navegador: <http://localhost:8080/index.php>
- Consulta SQL desde el contenedor:

```powershell
docker exec -it mysql mysql -uusuario -pusuario123 practica -e "SELECT VERSION(); SHOW TABLES;"
```

> La primera vez MySQL tarda unos segundos en arrancar. Si hay error de conexión, espera y recarga.

---

## Práctica 4: WordPress con MySQL

**1. Reemplaza `php/Dockerfile`** (WordPress necesita más extensiones)

```dockerfile
FROM php:8.3-fpm

RUN apt-get update && apt-get install -y \
      libzip-dev libpng-dev libjpeg-dev libfreetype6-dev libicu-dev unzip \
 && docker-php-ext-configure gd --with-freetype --with-jpeg \
 && docker-php-ext-install mysqli pdo pdo_mysql zip gd intl opcache \
 && rm -rf /var/lib/apt/lists/*
```

**2. Reconstruir**

```powershell
docker compose up -d --build
```

**3. Descargar WordPress**, descomprimirlo en `html/`, borrar el zip y dar permisos al usuario de PHP:

```powershell
Invoke-WebRequest -Uri https://wordpress.org/latest.zip -OutFile wp.zip
Expand-Archive wp.zip -DestinationPath html
Remove-Item wp.zip
docker exec -it php chown -R www-data:www-data /var/www/html
```

Se crea `html/wordpress/`.

**4. Crear la base de datos**

```powershell
docker exec -it mysql mysql -uroot -proot123 -e "CREATE DATABASE wordpress; GRANT ALL PRIVILEGES ON wordpress.* TO 'usuario'@'%'; FLUSH PRIVILEGES;"
```

**5. Instalar desde el navegador:** <http://localhost:8080/wordpress/>

| Campo | Valor |
|---|---|
| Nombre de la base de datos | `wordpress` |
| Usuario | `usuario` |
| Contraseña | `usuario123` |
| Servidor de la base de datos | `db` |
| Prefijo de tablas | `wp_` |

Después se completa el asistente (título del sitio, usuario administrador y contraseña).

**6. Verificar las tablas**

```powershell
docker exec -it mysql mysql -uusuario -pusuario123 wordpress -e "SHOW TABLES;"
```

Deben aparecer las tablas `wp_*`.

---

## Práctica 5: Joomla con la misma base de datos

**1. Descargar Joomla** desde <https://www.joomla.org/> (*Full Package*, ZIP) y descomprimirlo:

```powershell
mkdir html\joomla
Expand-Archive "$env:USERPROFILE\Downloads\Joomla_*.zip" -DestinationPath html\joomla
docker exec -it php chown -R www-data:www-data /var/www/html
```

**2. Crear su base de datos** en el mismo servidor MySQL:

```powershell
docker exec -it mysql mysql -uroot -proot123 -e "CREATE DATABASE joomla; GRANT ALL PRIVILEGES ON joomla.* TO 'usuario'@'%'; FLUSH PRIVILEGES;"
```

**3. Instalar desde el navegador:** <http://localhost:8080/joomla/>

| Campo | Valor |
|---|---|
| Tipo de base de datos | MySQLi |
| Nombre del host | `db` |
| Usuario | `usuario` |
| Contraseña | la de la base de datos MySQL |
| Nombre de la base de datos | `joomla` |

> Joomla exige una contraseña de **mínimo 12 caracteres** para el superusuario del sitio. Es independiente de la contraseña de MySQL.

**4. Verificar que conviven ambas aplicaciones**

```powershell
docker exec -it mysql mysql -uroot -proot123 -e "SHOW DATABASES;"
```

Deben aparecer `practica`, `wordpress` y `joomla` en el mismo servidor.

---

## Prueba final de los servidores

| Servicio | URL |
|---|---|
| HTML | <http://localhost:8080/index.html> |
| PHP + MySQL | <http://localhost:8080/index.php> |
| WordPress | <http://localhost:8080/wordpress/> |
| Joomla | <http://localhost:8080/joomla/> |
| Administración de Joomla | <http://localhost:8080/joomla/administrator/> |
| Administración de WordPress | <http://localhost:8080/wordpress/wp-admin/> |

---

## Comandos útiles

```powershell
docker compose ps                # estado de los servicios
docker compose logs -f php       # logs de un servicio
docker compose restart web       # reiniciar Nginx tras cambiar default.conf
docker compose down              # parar y eliminar contenedores (conserva los datos)
docker compose down -v           # además borra el volumen de MySQL
docker exec -it php bash         # entrar al contenedor PHP
docker exec -it mysql bash       # entrar al contenedor MySQL
```

## Problemas frecuentes

- **502 Bad Gateway:** Nginx no alcanza el contenedor PHP. Revisa `docker compose ps -a` y `docker compose logs php`.
- **`ERR_CONNECTION_REFUSED`:** falta el puerto en la URL (`:8080`) o los contenedores están parados. Ejecuta `docker compose up -d`.
- **`Access denied for user 'usuario'`:** la contraseña de MySQL no coincide. Se puede restablecer con:
  ```powershell
  docker exec -it mysql mysql -uroot -proot123 -e "ALTER USER 'usuario'@'%' IDENTIFIED BY 'usuario123'; FLUSH PRIVILEGES;"
  ```
- **Puerto 8080 ocupado:** cambia `"8080:80"` por otro, por ejemplo `"8081:80"`.
- **Tras reiniciar el PC:** con Docker Desktop abierto, ejecuta `docker compose up -d` en la carpeta del proyecto.

## Aviso de seguridad

Las contraseñas de este repositorio (`root123`, `usuario123`, etc.) son **solo para prácticas en local**. En un entorno real deben guardarse en un archivo `.env` fuera del control de versiones y ser robustas.
