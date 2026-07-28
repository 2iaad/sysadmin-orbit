<div align="center">

<img src="https://raw.githubusercontent.com/devicons/devicon/master/icons/docker/docker-original.svg" width="80" />
<img src="https://raw.githubusercontent.com/devicons/devicon/master/icons/nginx/nginx-original.svg" width="80" />
<img src="https://raw.githubusercontent.com/devicons/devicon/master/icons/wordpress/wordpress-plain.svg" width="80" />
<img src="https://raw.githubusercontent.com/devicons/devicon/master/icons/mysql/mysql-original.svg" width="80" />

## Descripción General

Pequeña infraestructura web con **Docker Compose**, **NGINX**, **WordPress + PHP-FPM** y **MariaDB**, construida manualmente desde `debian:bookworm`, conectada en una red bridge privada accesible únicamente a través de **TLS en el puerto 443** (El puerto 80 está CERRADO, el HTTP no existe aquí :P). Sin imágenes pre-construidas, sin etiquetas `latest`.


---

# Arquitectura

<img src="./diagram.png" width="800" />

</div>

---

# Redes de Docker: ¿cómo se comunican los contenedores?

A través de **docker0**, que es básicamente el bridge virtual predeterminado que Docker crea en el host. Su función es:

- Asignar IPs a cada contenedor (172.17.0.x)
- Enrutar el tráfico entre contenedores (nginx → wordpress → mariadb pasan todos por aquí)
- Punto de entrada para el tráfico entrante desde el host vía iptables
- Aislar los contenedores del resto de la red del host

Podemos decir en una línea que es el switch virtual que permite que los contenedores hablen entre sí y con el mundo exterior, echa un vistazo a esto:

<div align="center">
  <img src="./docker0.png" width="650" />
</div>

---

## Orden de Inicio

```
make up
  |
  +-- mkdir /root/data/wordpress   (directorios de volúmenes del host)
  +-- mkdir /root/data/mariadb
  |
  +--> el contenedor mariadb inicia
  |       se ejecuta mariadb.sh:
  |         1. iniciar servicio (modo configuración)
  |         2. crear DB + usuario + permisos (grant)
  |         3. detener servicio
  |         4. lanzar mysqld_safe con 0.0.0.0 para que otros contenedores puedan comunicarse con mariadb (primer plano, PID 1)
  |       healthcheck: mysqladmin ping cada 7s
  |
  +--> el contenedor wordpress inicia (espera: que mariadb esté healthy)
  |       se ejecuta wp-config.sh:
  |         1. descargar wp-cli
  |         2. si no hay wp-config.php:
  |              wp core download
  |              wp core config (apunta a mariadb:3306)
  |              wp core install (crea usuarios administrador + editor)
  |         3. parchear php-fpm para escuchar en 0.0.0.0:9000
  |         4. lanzar php-fpm8.2 -F (primer plano, PID 1)
  |
  +--> el contenedor nginx inicia (espera: que wordpress esté arriba)
          Dockerfile generó un certificado autofirmado en tiempo de construcción
          nginx.conf: escucha en 443 ssl, redirige *.php -> wordpress:9000
          CMD: nginx -g "daemon off;" (primer plano, PID 1)
```

---

## Mapa de Archivos

```
sysadmin-orbit/
├── Makefile                        <- comandos build/run/clean
├── srcs/
│   ├── docker-compose.yml          <- redes, volúmenes, servicios
│   ├── .env                        <- secretos (ignorados por git)
│   ├── .env.example                <- plantilla para .env
│   └── requirements/
│       ├── mariadb/
│       │   ├── Dockerfile          <- instala mariadb-server
│       │   └── tools/mariadb.sh   <- inicia DB + ejecuta mysqld_safe
│       ├── nginx/
│       │   ├── Dockerfile          <- instala nginx + openssl, genera cert TLS
│       │   └── nginx.conf          <- config del servidor (443 ssl, fastcgi)
│       └── wordpress/
│           ├── Dockerfile          <- instala php-fpm, php-mysql, curl
│           └── wp-config.sh        <- instalación de wp-cli + ejecuta php-fpm
```

---

## Volúmenes y Persistencia de Datos

```
Ruta en Host               Nombre vol Docker   Montado en
/root/data/mariadb  --> mariadb (bind)    --> mariadb:/var/lib/mysql
/root/data/wordpress -> wordpress (bind)  --> wordpress:/var/www/wordpress
                                          --> nginx:/var/www/wordpress (lectura)
```

Ambos volúmenes usan `driver: local` con `o: bind` — son directorios simples del host montados mediante bind en los contenedores. Los datos sobreviven a un `docker compose down` pero se borran con `make fclean` (que ejecuta `rm -rf /root/data`).

---

Por qué el volumen `wordpress` es compartido entre dos contenedores

```
                  Solicitud HTTPS para /wp-content/uploads/photo.jpg
                                       │
                                       ▼
                              ┌────────────────┐
                              │     NGINX      │
                              │   contenedor    │
                              └────────┬───────┘
                                       │
                            lee el archivo directamente desde
                            /var/www/wordpress/wp-content/uploads/photo.jpg
                                       │
                                       ▼
                              ╔═══════════════════════╗
                              ║  volumen wordpress    ║
                              ║  (= /root/data/       ║
                              ║       wordpress en    ║
                              ║       host)           ║
                              ╚═══════════════════════╝
                                       ▲
                            misma ruta, mismos archivos
                                       │
                              ┌────────┴───────┐
                              │   WORDPRESS    │
                              │   contenedor    │
                              │   (php-fpm)    │
                              └────────────────┘
                                       ▲
                                       │
            Solicitud HTTPS para /index.php → FastCGI a wordpress:9000
            php-fpm lee el archivo .php desde el mismo volumen compartido
            y lo ejecuta
```

Dos contenedores, un volumen, una ruta en disco. Si WP escribe una nueva subida vía PHP-FPM, NGINX la ve instantáneamente porque es literalmente el mismo archivo.

## Variables .env

| Variable           | Usado por      | Propósito                         |
|--------------------|----------------|-----------------------------------|
| `MYSQL_DB`         | MariaDB, WP    | Nombre de la base de datos        |
| `MYSQL_USER`       | MariaDB, WP    | Usuario de DB con el que conecta WP |
| `MYSQL_PASSWORD`   | MariaDB, WP    | Contraseña para ese usuario        |
| `DOMAIN_NAME`      | WordPress      | URL del sitio (ej. https://IP)    |
| `WP_TITLE`         | WordPress      | Título del sitio                  |
| `WP_ADMIN_N/P/E`   | WordPress      | Nombre usuario / pass / email admin|
| `WP_USER_NAME/EMAIL/PASS/ROLE` | WordPress | Segundo usuario (editor)    |

El archivo `.env` está ignorado por git. Copia `.env.example` y completa con los valores reales.

---

## Reglas de mejores prácticas seguidas (restricciones 1337)

- Un servicio por contenedor, sin poner dos procesos en una sola imagen.
- Sin etiquetas `latest`.
- Sin `network: host` ni `--privileged`.
- Cada servicio se ejecuta como PID 1 en primer plano (sin `daemon on`).
- El puerto 80 nunca se abre — el HTTP no existe.
- Los secretos residen en `.env`, nunca integrados en los Dockerfiles.
- WordPress solo inicia después de que MariaDB pase su healthcheck de `mysqladmin ping`.
