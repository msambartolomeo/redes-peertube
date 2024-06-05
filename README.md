# Peertube

## Requerimientos

- [Docker](https://docs.docker.com/engine/install/) o [Podman](https://podman.io/docs/installation), junto a [docker-compose](https://docs.docker.com/compose/install/) para deployear contenedores
- Red con posibilidad de hostear públicamente (en su defecto, se puede utilizar alguna nube)

## Quick start

1. Clonar el repositorio con el comando `git clone https://github.com/msambartolomeo/redes-peertube.git`

2. Completar el archivo [`.env`](.env) con las siguientes variables:

- POSTGRES_USER
  - Usuario de postgres
- POSTGRES_PASSWORD
  - Usuario de postgres
- POSTGRES_DB
  - Base de datos de postgres
- PEERTUBE_DOMAIN
  - Dominio apuntando a la ip del servidor (puede conseguirse uno gratuito en [duckdns.org](https://duckdns.org))
- PEERTUBE_ADMIN_EMAIL
  - Email del usuario que será administrador
- PEERTUBE_SECRET
  - Secreto aleatorio creado con `openssl rand -hex 32`

3. Correr el siguiente script en una `shell` `bash` o similar para popular las variables del archivo [`production.placeholder.yaml`](production.placeholder.yaml) y transformarlo en el archivo `production.yaml` que es el archivo de configuración de `Peertube`. El archivo tiene activada redundancia de videos entre instancias sincronizadas por default.

```sh
set -a
source .env
envsubst < production.placeholder.yaml > production.yaml
set +a
```

4. Asegurarse que los puertos 80 y 443 estén abiertos con `port forwarding` en el router y apuntando a la ip privada de la PC que vaya a ejecutar el servidor. También asegurarse de que el firewall de la misma PC no esté bloqueando los pedidos en esos puertos.

5. Ejecutar `docker compose up -d` para ejecutar los servicios necesarios para el servidor de `Peertube` con la configuración necesaria.

6. Ejecutar `docker compose exec -u peertube peertube npm run reset-password -- -u root` para modificar la contraseña del usuario `root` en la nueva instancia. Una vez hecho esto puede conectarse al dominio configurado anteriormente y comenzar a utilizarla.

7. Para configurar la sincronización entre instancias, una vez iniciado sesión con el usuario `root` y la contraseña definida anteriormente, se puede acceder en la interfaz de usuario a `Administration > Federation > Following > Follow` e ingresar el dominio de la otra instancia con el formato `peertube@DOMAIN`para seguirla y que sus videos aparezcan en la nuestra.

## Detalles

### Compose

Para levantar los servicios necesarios de forma simple y eficiente se utilizarán contenedores.
Peertube provee un archivo `docker-compose.yml` oficial para levantar la instancia el cuál puede encontrarse en https://github.com/Chocobozzz/PeerTube/blob/master/support/docker/production/docker-compose.yml, pero el mismo no podía utilizarse directamente ya que tiene limitaciones sobre la configuración de redundancia de instancias. Por esta razón se aprovechó para crear un archivo compose nuevo basado en el mismo para saltarnos estas limitaciones.

A continuación se detallan los servicios levantados y en el anexo se encontrará el archivo completo para su utilización.

#### Peertube

```yml
peertube:
  image: chocobozzz/peertube:production-bookworm
  volumes:
    - ./production.yaml:/config/production.yaml
    - ./data/data:/data
    - ./data/config:/config
  networks:
    default:
      ipv4_address: 172.18.0.42
  links:
    - postgres
    - redis
  depends_on:
    postgres:
      condition: service_healthy
    redis:
      condition: service_healthy
  restart: unless-stopped
```

Es el servicio que levanta la aplicación principal, utilizando la imagen oficial de peertube para deployear en producción. Siguiendo la documentación de la misma, esta se configura por variables de entorno, como suele hacerse normalmente con contenedores. Desafortunadamente, la imagen no provee una forma de realizar la configuración de redundancia mediante las mismas, por lo que optamos por agregar la configuración completa con modificaciones nuestras como un volumen más. Este es el archivo production.yaml montado. Sobre la configuración de este archivo hablaremos más adelante.

Además de este archivo, también se montan volúmenes para la data y la configuración, de forma de poder persistirlas entre reinicios de la aplicación.

En el caso de la network, peertube tiene la particularidad de que no reacciona agradablemente a los cambios de ip entre reinicios, por lo que la configuramos para que tenga una ip fija en la red virtual de docker creada.

Por último, el servicio de peertube depende de dos bases de datos que utilizará para guardar información, una base de datos relacional `postgresql` y una base de datos para cache `redis`. Marcamos que el servicio depende de las mismas para evitar que peertube arranque antes que las bases de datos y produzca errores.

#### Postgres

```yml
postgres:
  image: postgres:13-alpine
  environment:
    POSTGRES_PASSWORD: "${POSTGRES_PASSWORD}"
    POSTGRES_USER: "${POSTGRES_USER}"
    POSTGRES_DB: "${POSTGRES_DB}"
  volumes:
    - ./data/db:/var/lib/postgresql/data
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -d ${POSTGRES_DB} -U ${POSTGRES_USER}"]
    interval: 1s
    timeout: 5s
    retries: 5
  restart: unless-stopped
```

Postgres es la base de datos relacional que necesita el servicio de peertube para guardar información. En este caso le pasamos las variables de entorno para definir la base de datos, configuramos un volumen para persistir la data y le agregamos un healthcheck para que peertube sepa cuando la misma está habilitada y lista para funcionar, y evitar errores de sincronización.

#### Redis

```yml
redis:
  image: redis:6-alpine
  volumes:
    - ./data/redis:/data
  healthcheck:
    test: ["CMD-SHELL", "redis-cli ping | grep PONG"]
    interval: 1s
    timeout: 5s
    retries: 5
  restart: unless-stopped
```

Redis es una base de datos clave valor, normalmente utilizada como cache, y para persistir información en memoria. Igual que postgres, le configuramos un healthcheck y montamos los volúmenes necesarios para persistir la data de la misma.

#### Caddy

```yml
caddy:
  image: caddy:2.8-alpine
  restart: unless-stopped
  command: "caddy reverse-proxy --from https://${PEERTUBE_DOMAIN}:443 --to http://peertube:9000"
  cap_add:
    - NET_ADMIN
  ports:
    - 80:80
    - 443:443
  volumes:
    - ./data/caddy:/data
  links:
    - peertube
  depends_on:
    peertube:
      condition: service_started
```

Caddy es un reverse proxy similar a nginx, pero más simple de configurar y a su vez contiene la lógica de negociación de certificados con la plataforma `Let's Encrypt` ya incluida. De esta forma simplemente con utilizar esta imagen, y especificar el comando para que cree un proxy reverso desde el servicio que peertube abre en el puerto 9000 de la red interna de docker hacia un dominio especificado por nosotros, podemos lograr que la aplicación nos provea https para poder utilizarla mediante internet con seguridad.

Es importante persistir la data de los certificados en un volumen e informarle a docker que abra los puertos 80 y 443 de este contenedor; 433 para proveer la salida https, y 80 para permitirle al mismo resolver los desafíos que le presenta `Let's Encrypt` para poder negociar el certificado ssl gratuito.

#### Redes

```yml
networks:
  default:
    ipam:
      driver: default
      config:
        - subnet: 172.18.0.0/16
```

Es importante aclarar que de la misma forma que a peertube tuvimos que especificarle una ip fija, debemos especificar la subnet para que docker utilice para sus interfaces internas, de forma que la misma funcione. Naturalmente, la ip fija definida en el servicio de peertube debe ser parte de esta subnet.

#### Postfix

```yml
postfix:
  image: mwader/postfix-relay
  env_file:
    - .env
  volumes:
    - ./docker-volume/opendkim/keys:/etc/opendkim/keys
  restart: "always"
```

El archivo compose oficial contiene también configuración para un servicio de mails necesario para la operación del servicio real. El mismo no es necesario para la prueba de concepto realizada y además no poseemos un dominio propio en el cuál se pueda configurar email como para poder utilizarlo adecuadamente, por lo que se decidió no incluirlo en el archivo compose final. Se presenta aquí por completitud pero se obviará su configuración más adelante.

### Configuración

Como se mencionó antes la configuración del servidor de peertube se realizará mediante el archivo de configuración completo `production.yml` en lugar de utilizar las variables de entorno provistas por la imagen oficial. La configuración de ejemplo puede obtenerse en https://github.com/Chocobozzz/PeerTube/blob/master/config/production.yaml.example.

De todas formas, esta configuración está diseñada para cuándo el servicio no está corriendo en docker, por lo que se le deben hacer algunas modificaciones:

- En `listen`, debe configurarse para que escuche en `0.0.0.0`, es decir en cualquier ip.
- Deben modificarse los hostnames de `database` y de `redis` a que sean `postgres` y `redis` respectivamente en lugar de `127.0.0.1` para que puedan detectarse en la red interna de docker.
- En `database`, debe modificarse la clave suffix por una que diga name
- En `storage`, deben modificarse todos los paths a que comiencen con `/data/` en lugar de `/var/www/peertube/storage/`

A partir de estos cambios estructurales, se deben hacer las configuraciones pertinentes a la instancia:

- En `webserver`, configurar el hostname con un dominio asociado a la ip publica en donde correrá el servidor.
- En `secrets`, escribir un secreto creado con el comando `openssl rand -hex 32`
- En `database`, configurar `name`, `username` y `password` con los valores a utilizar para la base de datos postgres
- En `admin`, configurar el email del usuario que luego será root en la instancia
- En `redundancy`, descomentar la estrategia de redundancy entre servidores a utilizar,

Como hacer todos estos cambios consume tiempo, se provee una version del archivo [`production.placeholder.yaml`](production.placeholder.yaml) ya configurada pero con valores placeholder para los campos configurables. En el archivo ya provisto, se optó por utilizar todas las estrategias de redundancia entre instancias posibles, pero esto se podría modificar a mano si deseara.

Para conseguir el archivo final `production.yaml`, se provee un archivo [`.env`](.env) que solamente incluya la configuración a estos valores. Los elementos a configurar este archivo son los siguientes:

- POSTGRES_USER
  - Usuario de postgres
- POSTGRES_PASSWORD
  - Usuario de postgres
- POSTGRES_DB
  - Base de datos de postgres
- PEERTUBE_DOMAIN
  - Dominio apuntando a la ip del servidor
- PEERTUBE_ADMIN_EMAIL
  - Email del usuario que será administrador
- PEERTUBE_SECRET
  - Secreto aleatorio creado con `openssl rand -hex 32`

Luego de llenar el archivo se debe cargar esas variables de entorno en la consola y reemplazarlas en archivo de placeholder. Esto puede hacerse con el siguiente script si la shell utilizada es `bash` o `zsh`

```sh
set -a
source .env
envsubst < production.placeholder.yaml > production.yaml
set +a
```

Si se quisiera hacer alguna modificación extra a la configuración de la instancia se puede hacer en la configuración del `production.yaml` creado

### Deploy

Una vez realizada la configuración se puede proseguir con el deploy del servicio. El mismo puede ser realizado de manera local o en la nube. En caso de hacerlo local se deben realizar `port forwarding` a los puertos 80 y 443 del router hacia la pc que corra el servidor de forma que otros puedan conectarse. Si el servidor está corriendo algún tipo de firewall para estos puertos el mismo debe deshabilitarse. Esto puede hacerse en linux con los comandos `firewall-cmd --add-port=443/tcp` y `firewall-cmd --add-port=80/tcp`. Si el contenedor se corre sin permisos de root (en el caso de que no se utilice docker, y se use podman por ejemplo) se debe permitir a los usuarios no root realizar `bind` a los puertos privilegiados con el comando `sysctl net.ipv4.ip_unprivileged_port_start=80` o correr configurar el contenedor para que corra como root.

Para el deploy es necesario un dominio que apunte a la ip pública en donde va a estar corriendo el servidor. Puede conseguirse uno de forma gratuita en https://duckdns.org.

Como se mencionó antes, ya se provee el archivo [`docker-compose.yml`](docker-compose.yml) con los servicios necesarios. Por defecto, el mismo utiliza los valores del archivo [`.env`](.env) configurado para la creación de los servicios.

De esta forma, para deployear simplemente se debe correr `docker compose up -d` y todos los recursos necesarios serán creados.

En caso de utilizar `podman` en lugar de `docker` también se provee un archivo [`podman-compose.yml`](podman-compose.yml) con las modificaciones necesarias para utilizar este runtime de contenedores. Se pueden deployear todos los recursos utilizando el comando `podman compose -f podman-compose.yml up -d

Una vez peertube esté corriendo en los logs se puede observar la contraseña del usuario administrador o la misma se puede cambiar utilizando el comando `docker compose exec -u peertube peertube npm run reset-password -- -u root`

Para configurar la sincronización entre instancias, una vez iniciado sesión con el usuario `root` y la contraseña definida anteriormente, se puede acceder en la interfaz de usuario a `Administration > Federation > Following > Follow` e ingresar el dominio de la otra instancia con el formato `peertube@DOMAIN`para seguirla y que sus videos aparezcan en la nuestra. Como la redundancia de videos está habilitada, automáticamente se sincronizarán los videos de las instancias para distribuir la carga entre los servidores cuándo se esté mirando un video.
