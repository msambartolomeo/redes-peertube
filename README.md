# Peertube

## Requerimientos

- Docker o Podman para deployear contenedores
- Red con posibilidad de hostear públicamente (en su defecto, se puede utilizar alguna nube)

## Compose

Para levantar los servicios necesarios de forma simple y eficiente se utilizarán contenedores.
Peertube provee un archivo `docker-compose.yml` oficial para levantar la instancia el cuál puede encontrarse en https://github.com/Chocobozzz/PeerTube/blob/master/support/docker/production/docker-compose.yml, pero el mismo no podía utilizarse directamente ya que tiene limitaciónes sobre la configuración de reduncancia de instancias. Por esta razón se aprovechó para crear un archivo compose nuevo basado en el mismo para saltearnos estas limitaciónes.

A continuación se detallan los servicios levantados y en el anexo se encontrará el archivo completo para su utilización.

### Peertube

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

Es el servicio que levanta la aplicación principal, utilizando la imagen oficial de peertube para deployear en producción. Siguiendo la documentación de la misma, esta se configura por variables de entorno, como suele hacerse normalmente con contenedores. Desafortunadamente, la imagen no provee una forma de realizar la configuración de redundancia mediante las mismas, por lo que optamos por agregar la configuración completa con modificaciónes nuestras como un volumen más. Este es el archivo production.yaml montado. Sobre la configuración de este archivo hablaremos más adelante.

Además de este archivo, también se montan volumenes para la data y la configuración, de forma de poder persistirlas entre reinicios de la aplicación.

En el caso de la network, peertube tiene la particularidad de que no reaccióna agradablemente a los cambios de ip entre reinicios, por lo que la configuramos para que tenga una ip fija en la red virtual de docker creada.

Por último, el servicio de peertube depende de dos bases de datos que utilizará para guardar información, una base de datos relacional `postgresql` y una base de datos para cache `redis`. Marcamos que el servicio depende de las mismas para evitar que peertube arranque antes que las bases de datos y produzca errores.

### Postgres

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

Postgres es la base de datos relacional que necesita el servicio de peertube para guardar información. En este caso le pasamos las variables de entorno para definir la base de datos, configuramos un volúmen para persistir la data y le agregamos un healthcheck para que peertube sepa cuando la misma está habilitada y lista para funcionar, y evitar errores de sincronización.

### Redis

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

Redis es una base de datos clave valor, normalmente utilizada como cache, y para persistir información en memoria. Igual que postgres, le configuramos un healthcheck y montamos los volumenes necesarios para persistir la data de la misma.

### Caddy

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

Caddy es un reverse proxy similar a nginx, pero más simple de configurar y a su vez contiene la lógica de negociación de certificados con la plataforma `Let's Encrypt` ya incluída. De esta forma simplemente con utilizar esta imagen, y especificar el comando para que cree un proxy reverso desde el servicio que peertube abre en el puerto 9000 de la red interna de docker hacia un dominio especificado por nosotros, podemos lograr que la aplicación nos provea https para poder utilizarla mediante internet con seguridad.

Es importante persistir la data de los certificados en un volumen e informarle a docker que abra los puertos 80 y 443 de este contenedor; 433 para proveer la salida https, y 80 para permitirle al mismo resolver los desafíos que le presenta `Let's Encrypt` para poder negociar el certificado ssl gratuito.

### Redes

```yml
networks:
  default:
    ipam:
      driver: default
      config:
        - subnet: 172.18.0.0/16
```

Es importante aclarar que de la misma forma que a peertube tuvimos que especificarle una ip fija, debemos especificar la subnet para que docker utilice para sus interfaces internas, de forma que la misma funcione. Naturalmente, la ip fija definida en el servicio de peertube debe ser parte de esta subnet.

### Postfix

```yml
postfix:
  image: mwader/postfix-relay
  env_file:
    - .env
  volumes:
    - ./docker-volume/opendkim/keys:/etc/opendkim/keys
  restart: "always"
```

El archivo compose oficial contiene también configuración para un servicio de mails necesario para la operación del servicio real. El mismo no es necesario para la prueba de concepto realizada y además no poseemos un dominio propio como para poder utilizarlo adecuadamente, por lo que se decidió no incluirlo en el archivo compose final. Se presenta aquí por completitud pero se obviará su configuración más adelante.

## Configuración

Como se mencionó antes la configuración del servidor de peertube se realizará mediante el archivo de configuración completo `production.yml` en lugar de utilizar las variables de entorno provistas por la imagen oficial. La configuración de ejemplo puede obtenerse en https://github.com/Chocobozzz/PeerTube/blob/master/config/production.yaml.example.

De todas formas, esta configuración está diseñada para cuándo el corredor no está corriendo en docker, por lo que se le deben hacer algunas modificaciónes:

- En `listen`, debe configurarse para que escuche en `0.0.0.0`, es decir en cualquier ip.
- Deben modificarse los hostnames de `database` y de `redis` a que sean `postgres` y `redis` respectivamente en lugar de `127.0.0.1` para que puedan detectarse en la red de docker.
- En `database`, debe modificarse la clave suffix por una que diga name
- En `storage`, deben modifiarse todos los paths a que comiencen con `/data/` en lugar de `/var/www/peertube/storage/`

A partir de estos cambios estructurales, se deben hacer las configuraciones pertinentes a la instancia:

- En `webserver`, configurar el hostname con un dominio asociado a la ip publica en donde correra el servidor.
- En `secrets`, escribir un secreto creado con el comando `openssl rand -hex 32`
- En `database`, configurar `name`, `username` y `password` con los valores a utilizar para la base de datos postgres
- En `admin`, configurar el email del usuario que luego será root en la instancia
- En `redundancy`, descomentar la estrategia de redundancy entre servidores a utilizar,

Como hacer todos estos cambios consume tiempo, se provee una version del archivo `production.placeholder.yaml` ya configurada pero con valores placeholder para los campos configurables. En el mismo se optó por la estrategia de redundancia de trending.

Para completar el archivo production.yaml, se provee un archivo `.env` que solamente incluya la configuración a estos valores. Los elementos que deben estár presentes en este archivo `.env` son los siguientes:

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

Luego de llenar el archivo `.env` se debe cargar esas variables de entorno en la consola y reemplazarlas en archivo de placeholder. Esto puede hacerse con el siguiente script si la shell utilizada es `bash` o `zsh`

```sh
set -a
source .env
envsubst < production.placeholder.yaml > production.yaml
set +a
```

## Deploy

Una vez realizada la configuración se puede proseguir con el deploy del servicio. El mismo puede ser realizado de manera local o en la nube. En caso de hacerlo local se deben realizar port forwarding a los puertos 80 y 443 del router hacia la pc que corra el servidor de forma que otros puedan conectarse. Si el servidor está corriendo algún tipo de firewall para estos puertos el mismo puede deshabilitarse. Esto puede hacerse con los comandos `firewall-cmd --add-port=443/tcp` y `firewall-cmd --add-port=80/tcp`. Si el contenedor se corre sin permisos de root (en el caso de que no se utilice docker, y se use podman por ejemplo) se debe permitir a los usuarios no root realizar bind a los puertos privilegiados con el comando `sysctl net.ipv4.ip_unprivileged_port_start=80`

Para el deploy es necesario un dominio que apunte a la ip pública en donde va a estar corriendo el servidor. Puede conseguirse uno de forma gratuita en https://duckdns.org.

Para deployear simplemente se debe correr `docker compose --env-file .env up -d`

Una vez peertube esté corriendo en los logs se puede observar la contraseña del usuario administrador o la misma se puede cambiar utilizando el comando `docker compose exec -u peertube peertube npm run reset-password -- -u root`

## Anexo

A continuación se especifican los archivos de `.env`, `production.placeholder.yaml`, `docker-compose.yml` y `podman-compose.yml`
Se puede usar tanto `docker-compose.yml` o `podman-compose.yml` dependiendo del motor de contenedores que se prefiera

### docker-compose.yml

```yml
services:
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

  peertube:
    image: chocobozzz/peertube:production-bookworm
    networks:
      default:
        ipv4_address: 172.18.0.42
    volumes:
      - ./data/data:/data
      - ./production.yaml:/config/production.yaml
      - ./data/config:/config
    links:
      - postgres
      - redis
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped

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

networks:
  default:
    ipam:
      driver: default
      config:
        - subnet: 172.18.0.0/16
```

### podman-compose.yml

```yaml
services:
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
      - ./data/caddy:/data:z
    links:
      - peertube
    depends_on:
      peertube:
        condition: service_started

  peertube:
    image: chocobozzz/peertube:production-bookworm
    networks:
      default:
        ipv4_address: 172.18.0.42
    volumes:
      - ./data/data:/data:z
      - ./production.yaml:/config/production.yaml:z
      - ./data/config:/config:z
    links:
      - postgres
      - redis
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped

  postgres:
    image: postgres:13-alpine
    environment:
      POSTGRES_PASSWORD: "${POSTGRES_PASSWORD}"
      POSTGRES_USER: "${POSTGRES_USER}"
      POSTGRES_DB: "${POSTGRES_DB}"
    volumes:
      - ./data/db:/var/lib/postgresql/data:Z
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -d ${POSTGRES_DB} -U ${POSTGRES_USER}"]
      interval: 1s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  redis:
    image: redis:6-alpine
    volumes:
      - ./data/redis:/data:z
    healthcheck:
      test: ["CMD-SHELL", "redis-cli ping | grep PONG"]
      interval: 1s
      timeout: 5s
      retries: 5
    restart: unless-stopped

networks:
  default:
    ipam:
      driver: host-local
      config:
        - subnet: 172.18.0.0/16
```

### .env

```
POSTGRES_USER=...
POSTGRES_PASSWORD=...
POSTGRES_DB=peertube
PEERTUBE_DOMAIN=...
PEERTUBE_ADMIN_EMAIL=...
PEERTUBE_SECRET=...
```

### production.placeholder.yaml

```yaml
listen:
  hostname: "0.0.0.0"
  port: 9000

# Correspond to your reverse proxy server_name/listen configuration (i.e., your public PeerTube instance URL)
webserver:
  https: true
  hostname: $PEERTUBE_DOMAIN
  port: 443

# Secrets you need to generate the first time you run PeerTube
secrets:
  # Generate one using `openssl rand -hex 32`
  peertube: $PEERTUBE_SECRET

rates_limit:
  api:
    # 50 attempts in 10 seconds
    window: 10 seconds
    max: 50
  login:
    # 15 attempts in 5 min
    window: 5 minutes
    max: 15
  signup:
    # 2 attempts in 5 min (only succeeded attempts are taken into account)
    window: 5 minutes
    max: 2
  ask_send_email:
    # 3 attempts in 5 min
    window: 5 minutes
    max: 3
  receive_client_log:
    # 10 attempts in 10 min
    window: 10 minutes
    max: 10
  plugins:
    # 500 attempts in 10 seconds (we also serve plugin static files)
    window: 10 seconds
    max: 500
  well_known:
    # 200 attempts in 10 seconds
    window: 10 seconds
    max: 200
  feeds:
    # 50 attempts in 10 seconds
    window: 10 seconds
    max: 50
  activity_pub:
    # 500 attempts in 10 seconds (we can have many AP requests)
    window: 10 seconds
    max: 500
  client: # HTML files generated by PeerTube
    # 500 attempts in 10 seconds (to not break crawlers)
    window: 10 seconds
    max: 500

oauth2:
  token_lifetime:
    access_token: "1 day"
    refresh_token: "2 weeks"

# Proxies to trust to get real client IP
# If you run PeerTube just behind a local proxy (nginx), keep 'loopback'
# If you run PeerTube behind a remote proxy, add the proxy IP address (or subnet)
trust_proxy:
  - "loopback"

# Your database name will be database.name OR 'peertube'+database.suffix
database:
  hostname: "postgres"
  port: 5432
  ssl: false
  name: $POSTGRES_DB
  username: $POSTGRES_USER
  password: $POSTGRES_PASSWORD
  pool:
    max: 5

# Redis server for short time storage
# You can also specify a 'socket' path to a unix socket but first need to
# set 'hostname' and 'port' to null
redis:
  hostname: "redis"
  port: 6379
  auth: null # Used by both standalone and sentinel
  db: 0
  sentinel:
    enabled: false
    enable_tls: false
    master_name: ""
    sentinels:
      - hostname: ""
        port: 26379

# SMTP server to send emails
smtp:
  # smtp or sendmail
  transport: smtp
  # Path to sendmail command. Required if you use sendmail transport
  sendmail: null
  hostname: null
  port: 465 # If you use StartTLS: 587
  username: null
  password: null
  tls: true # If you use StartTLS: false
  disable_starttls: false
  ca_file: null # Used for self signed certificates
  from_address: "admin@example.com"

email:
  body:
    signature: "PeerTube"
  subject:
    prefix: "[PeerTube]"

# Update default PeerTube values
# Set by API when the field is not provided and put as default value in client
defaults:
  # Change default values when publishing a video (upload/import/go Live)
  publish:
    download_enabled: true

    # enabled = 1, disabled = 2, requires_approval = 3
    comments_policy: 1

    # public = 1, unlisted = 2, private = 3, internal = 4
    privacy: 1

    # CC-BY = 1, CC-SA = 2, CC-ND = 3, CC-NC = 4, CC-NC-SA = 5, CC-NC-ND = 6, Public Domain = 7
    # You can also choose a custom licence value added by a plugin
    # No licence by default
    licence: null

  p2p:
    # Enable P2P by default in PeerTube client
    # Can be enabled/disabled by anonymous users and logged in users
    webapp:
      enabled: true

    # Enable P2P by default in PeerTube embed
    # Can be enabled/disabled by URL option
    embed:
      enabled: true

# From the project root directory
storage:
  tmp: "/data/tmp/" # Use to download data (imports etc), store uploaded files before and during processing...
  tmp_persistent: "/data/tmp-persistent/" # As tmp but the directory is not cleaned up between PeerTube restarts
  bin: "/data/bin/"
  avatars: "/data/avatars/"
  web_videos: "/data/web-videos/"
  streaming_playlists: "/data/streaming-playlists/"
  original_video_files: "/data/original-video-files/"
  redundancy: "/data/redundancy/"
  logs: "/data/logs/"
  previews: "/data/previews/"
  thumbnails: "/data/thumbnails/"
  storyboards: "/data/storyboards/"
  torrents: "/data/torrents/"
  captions: "/data/captions/"
  cache: "/data/cache/"
  plugins: "/data/plugins/"
  well_known: "/data/well-known/"
  # Overridable client files in client/dist/assets/images:
  # - logo.svg
  # - favicon.png
  # - default-playlist.jpg
  # - default-avatar-account.png
  # - default-avatar-video-channel.png
  # - and icons/*.png (PWA)
  # Could contain for example assets/images/favicon.png
  # If the file exists, peertube will serve it
  # If not, peertube will fallback to the default file
  client_overrides: "/data/client-overrides/"

static_files:
  # Require and check user authentication when accessing private files (internal/private video files)
  private_files_require_auth: true

object_storage:
  enabled: false

  # Without protocol, will default to HTTPS
  # Your S3 provider must support virtual hosting of buckets as PeerTube doesn't support path style requests
  endpoint: "" # 's3.amazonaws.com' or 's3.fr-par.scw.cloud' for example

  region: "us-east-1"

  upload_acl:
    # Set this ACL on each uploaded object of public/unlisted videos
    # Use null if your S3 provider does not support object ACL
    public: "public-read"
    # Set this ACL on each uploaded object of private/internal videos
    # PeerTube can proxify requests to private objects so your users can access them
    # Use null if your S3 provider does not support object ACL
    private: "private"

  proxy:
    # If private files (private/internal video files) have a private ACL, users can't access directly the ressource
    # PeerTube can proxify requests between your object storage service and your users
    # If you disable PeerTube proxy, ensure you use your own proxy that is able to access the private files
    # Or you can also set a public ACL for private files in object storage if you don't want to use a proxy
    proxify_private_files: true

  credentials:
    # You can also use AWS_ACCESS_KEY_ID env variable
    access_key_id: ""
    # You can also use AWS_SECRET_ACCESS_KEY env variable
    secret_access_key: ""

  # Maximum amount to upload in one request to object storage
  max_upload_part: 100MB

  streaming_playlists:
    bucket_name: "streaming-playlists"

    # Allows setting all buckets to the same value but with a different prefix
    prefix: "" # Example: 'streaming-playlists:'

    # Base url for object URL generation, scheme and host will be replaced by this URL
    # Useful when you want to use a CDN/external proxy
    base_url: "" # Example: 'https://mirror.example.com'

    # PeerTube makes many small requests to the object storage provider to upload/delete/update live chunks
    # which can be a problem depending on your object storage provider
    # You can also choose to disable this feature to reduce live streams latency
    # Live stream replays are not affected by this setting, so they are uploaded in object storage as regular VOD videos
    store_live_streams: true

  web_videos:
    bucket_name: "web-videos"
    prefix: ""
    base_url: ""

  user_exports:
    bucket_name: "user-exports"
    prefix: ""
    base_url: ""

  # Same settings but for original video files
  original_video_files:
    bucket_name: "original-video-files"
    prefix: ""
    base_url: ""

log:
  level: "info" # 'debug' | 'info' | 'warn' | 'error'

  rotation:
    enabled: true # Enabled by default, if disabled make sure that 'storage.logs' is pointing to a folder handled by logrotate
    max_file_size: 12MB
    max_files: 20

  anonymize_ip: false

  log_ping_requests: true
  log_tracker_unknown_infohash: true

  # If you have many concurrent requests, you can disable HTTP requests logging to reduce PeerTube CPU load
  log_http_requests: true

  prettify_sql: false

  # Accept warn/error logs coming from the client
  accept_client_log: true

# Support of Open Telemetry metrics and tracing
# For more information: https://docs.joinpeertube.org/maintain/observability
open_telemetry:
  metrics:
    enabled: false

    # How often viewers send playback stats to server
    playback_stats_interval: "15 seconds"

    http_request_duration:
      # You can disable HTTP request duration metric that can have a high tag cardinality
      enabled: false

    # Create a prometheus exporter server on this port so prometheus server can scrape PeerTube metrics
    prometheus_exporter:
      hostname: "127.0.0.1"
      port: 9091

  tracing:
    # If tracing is enabled, you must provide --experimental-loader=@opentelemetry/instrumentation/hook.mjs flag to the node binary
    enabled: false

    # Send traces to a Jaeger compatible endpoint
    jaeger_exporter:
      endpoint: ""

trending:
  videos:
    interval_days: 7 # Compute trending videos for the last x days for 'most-viewed' algorithm

    algorithms:
      enabled:
        - "hot" # Adaptation of Reddit's 'Hot' algorithm
        - "most-viewed" # Number of views in the last x days
        - "most-liked" # Global views since the upload of the video

      default: "most-viewed"

# Cache remote videos on your server, to help other instances to broadcast the video
# You can define multiple caches using different sizes/strategies
# Once you have defined your strategies, choose which instances you want to cache in admin -> manage follows -> following
redundancy:
  videos:
    check_interval: "1 hour" # How often you want to check new videos to cache
    strategies: # Just uncomment strategies you want
      #      -
      #        size: '10GB'
      #        # Minimum time the video must remain in the cache. Only accept values > 10 hours (to not overload remote instances)
      #        min_lifetime: '48 hours'
      #        strategy: 'most-views' # Cache videos that have the most views
      - size: "10GB"
        # Minimum time the video must remain in the cache. Only accept values > 10 hours (to not overload remote instances)
        min_lifetime: "48 hours"
        strategy: "trending" # Cache trending videos
#      -
#        size: '10GB'
#        # Minimum time the video must remain in the cache. Only accept values > 10 hours (to not overload remote instances)
#        min_lifetime: '48 hours'
#        strategy: 'recently-added' # Cache recently added videos
#        min_views: 10 # Having at least x views

# Other instances that duplicate your content
remote_redundancy:
  videos:
    # 'nobody': Do not accept remote redundancies
    # 'anybody': Accept remote redundancies from anybody
    # 'followings': Accept redundancies from instance followings
    accept_from: "anybody"

csp:
  enabled: false
  report_only: true # CSP directives are still being tested, so disable the report only mode at your own risk!
  report_uri:

security:
  # Set the X-Frame-Options header to help to mitigate clickjacking attacks
  frameguard:
    enabled: true

  # Set x-powered-by HTTP header to "PeerTube"
  # Can help remote software to know this is a PeerTube instance
  powered_by_header:
    enabled: true

tracker:
  # If you disable the tracker, you disable the P2P on your PeerTube instance
  enabled: true
  # Only handle requests on your videos
  # If you set this to false it means you have a public tracker
  # Then, it is possible that clients overload your instance with external torrents
  private: true
  # Reject peers that do a lot of announces (could improve privacy of TCP/UDP peers)
  reject_too_many_announces: false

history:
  videos:
    # If you want to limit users videos history
    # -1 means there is no limitations
    # Other values could be '6 months' or '30 days' etc (PeerTube will periodically delete old entries from database)
    max_age: -1

views:
  videos:
    # PeerTube creates a database entry every hour for each video to track views over a period of time
    # This is used in particular by the Trending page
    # PeerTube could remove old remote video views if you want to reduce your database size (video view counter will not be altered)
    # -1 means no cleanup
    # Other values could be '6 months' or '30 days' etc (PeerTube will periodically delete old entries from database)
    remote:
      max_age: "30 days"

    # PeerTube buffers local video views before updating and federating the video
    local_buffer_update_interval: "30 minutes"

    # How long does it take to count again a view from the same user
    view_expiration: "1 hour"

    # Minimum amount of time the viewer has to watch the video before PeerTube adds a view
    count_view_after: "10 seconds"

    # Player can send a session id string to track the user
    # Since this can be spoofed by users to create fake views, you have the option to disable this feature
    # If disabled, PeerTube will use the IP address to track the same user (default behavior before PeerTube 6.1)
    trust_viewer_session_id: true

    # How often the web browser sends "is watching" information to the server
    # Increase the value or set null to disable it if you plan to have many viewers
    watching_interval:
      # Non logged-in viewers
      anonymous: "5 seconds"

      # Logged-in users of your instance
      # Unlike anonymous viewers, this endpoint is also used to store the "last watched video timecode" for your users
      # Increasing this value reduces the accuracy of the video resume
      users: "5 seconds"

# Used to get country location of views of local videos
geo_ip:
  enabled: true

  country:
    database_url: "https://dbip.mirror.framasoft.org/files/dbip-country-lite-latest.mmdb"

  city:
    database_url: "https://dbip.mirror.framasoft.org/files/dbip-city-lite-latest.mmdb"

plugins:
  # The website PeerTube will ask for available PeerTube plugins and themes
  # This is an unmoderated plugin index, so only install plugins/themes you trust
  index:
    enabled: true
    check_latest_versions_interval: "12 hours" # How often you want to check new plugins/themes versions
    url: "https://packages.joinpeertube.org"

federation:
  # Some federated software such as Mastodon may require an HTTP signature to access content
  sign_federated_fetches: true

  videos:
    federate_unlisted: false

    # Add a weekly job that cleans up remote AP interactions on local videos (shares, rates and comments)
    # It removes objects that do not exist anymore, and potentially fix their URLs
    cleanup_remote_interactions: true

peertube:
  check_latest_version:
    # Check and notify admins of new PeerTube versions
    enabled: true
    # You can use a custom URL if your want, that respect the format behind https://joinpeertube.org/api/v1/versions.json
    url: "https://joinpeertube.org/api/v1/versions.json"

webadmin:
  configuration:
    edition:
      # Set this to false if you don't want to allow config edition in the web interface by instance admins
      allowed: true

# XML, Atom or JSON feeds
feeds:
  videos:
    # Default number of videos displayed in feeds
    count: 20

  comments:
    # Default number of comments displayed in feeds
    count: 20

remote_runners:
  # Consider jobs that are processed by a remote runner as stalled after this period of time without any update
  stalled_jobs:
    live: "30 seconds"
    vod: "2 minutes"

thumbnails:
  # When automatically generating a thumbnail from the video
  generation_from_video:
    # How many frames to analyze at the middle of the video to select the most appropriate one
    # Increasing this value will increase CPU and memory usage when generating the thumbnail, especially for high video resolution
    # Minimum value is 2
    frames_to_analyze: 50

stats:
  # Display registration requests stats (average response time, total requests...)
  registration_requests:
    enabled: true

  # Display abuses stats (average response time, total abuses...)
  abuses:
    enabled: true

  total_moderators:
    enabled: true

  total_admins:
    enabled: true

###############################################################################
#
# From this point, almost all following keys can be overridden by the web interface
# (local-production.json file). If you need to change some values, prefer to
# use the web interface because the configuration will be automatically
# reloaded without any need to restart PeerTube
#
# /!\ If you already have a local-production.json file, modification of some of
# the following keys will have no effect /!\
#
###############################################################################

cache:
  previews:
    size: 500 # Max number of previews you want to cache
  captions:
    size: 500 # Max number of video captions/subtitles you want to cache
  torrents:
    size: 500 # Max number of video torrents you want to cache
  storyboards:
    size: 500 # Max number of video storyboards you want to cache

admin:
  # Used to generate the root user at first startup
  # And to receive emails from the contact form
  email: $PEERTUBE_ADMIN_EMAIL

contact_form:
  enabled: true

signup:
  enabled: false

  limit: 10 # When the limit is reached, registrations are disabled. -1 == unlimited

  minimum_age: 16 # Used to configure the signup form

  # Users fill a form to register so moderators can accept/reject the registration
  requires_approval: true
  requires_email_verification: false

  filters:
    cidr: # You can specify CIDR ranges to whitelist (empty = no filtering) or blacklist
      whitelist: []
      blacklist: []

user:
  history:
    videos:
      # Enable or disable video history by default for new users.
      enabled: true

  # Default value of maximum video bytes the user can upload
  # Does not take into account transcoded files or account export archives (that can include user uploaded files)
  # Byte format is supported ("1GB" etc)
  # -1 == unlimited
  video_quota: -1
  video_quota_daily: -1

  default_channel_name: "Main $1 channel" # The placeholder $1 is used to represent the user's username

video_channels:
  max_per_user: 20 # Allows each user to create up to 20 video channels.

# If enabled, the video will be transcoded to mp4 (x264) with `faststart` flag
# In addition, if some resolutions are enabled the mp4 video file will be transcoded to these new resolutions
# Please, do not disable transcoding since many uploaded videos will not work
transcoding:
  enabled: true

  original_file:
    # If false the uploaded file is deleted after transcoding
    # If yes it is not deleted but moved in a dedicated folder or object storage
    keep: false

  # Allow your users to upload .mkv, .mov, .avi, .wmv, .flv, .f4v, .3g2, .3gp, .mts, m2ts, .mxf, .nut videos
  allow_additional_extensions: true

  # If a user uploads an audio file, PeerTube will create a video by merging the preview file and the audio file
  allow_audio_files: true

  # Enable remote runners to transcode your videos
  # If enabled, your instance won't transcode the videos itself
  # At least 1 remote runner must be configured to transcode your videos
  remote_runners:
    enabled: false

  # Amount of threads used by ffmpeg for 1 local transcoding job
  threads: 1
  # Amount of local transcoding jobs to execute in parallel
  concurrency: 1

  # Choose the local transcoding profile
  # New profiles can be added by plugins
  # Available in core PeerTube: 'default'
  profile: "default"

  resolutions: # Only created if the original video has a higher resolution, uses more storage!
    0p: false # audio-only (creates mp4 without video stream, always created when enabled)
    144p: false
    240p: false
    360p: false
    480p: false
    720p: false
    1080p: false
    1440p: false
    2160p: false

  # Transcode and keep original resolution, even if it's above your maximum enabled resolution
  always_transcode_original_resolution: true

  # Generate videos in a web compatible format
  # If you also enabled the hls format, it will multiply videos storage by 2
  # If disabled, breaks federation with PeerTube instances < 2.1
  web_videos:
    enabled: false

  # /!\ Requires ffmpeg >= 4.1
  # Generate HLS playlists and fragmented MP4 files. Better playback than with Web Videos:
  #     * Resolution change is smoother
  #     * Faster playback in particular with long videos
  #     * More stable playback (less bugs/infinite loading)
  # If you also enabled the web videos format, it will multiply videos storage by 2
  hls:
    enabled: true

live:
  enabled: false

  # Limit lives duration
  # -1 == unlimited
  max_duration: -1 # For example: '5 hours'

  # Limit max number of live videos created on your instance
  # -1 == unlimited
  max_instance_lives: 20

  # Limit max number of live videos created by a user on your instance
  # -1 == unlimited
  max_user_lives: 3

  # Allow your users to save a replay of their live
  # PeerTube will transcode segments in a video file
  # If the user daily/total quota is reached, PeerTube will stop the live
  # /!\ transcoding.enabled (and not live.transcoding.enabled) has to be true to create a replay
  allow_replay: true

  # Allow your users to change latency settings (small latency/default/high latency)
  # Small latency live streams cannot use P2P
  # High latency live streams can increase P2P ratio
  latency_setting:
    enabled: true

  # Your firewall should accept traffic from this port in TCP if you enable live
  rtmp:
    enabled: true

    # Listening hostname/port for RTMP server
    # '::' to listen on IPv6 and IPv4, '0.0.0.0' to listen on IPv4
    # Use null to automatically listen on '::' if IPv6 is available, or '0.0.0.0' otherwise
    hostname: null
    port: 1935

    # Public hostname of your RTMP server
    # Use null to use the same value than `webserver.hostname`
    public_hostname: null

  rtmps:
    enabled: false

    # Listening hostname/port for RTMPS server
    # '::' to listen on IPv6 and IPv4, '0.0.0.0' to listen on IPv4
    # Use null to automatically listen on '::' if IPv6 is available, or '0.0.0.0' otherwise
    hostname: null
    port: 1936

    # Absolute paths
    key_file: ""
    cert_file: ""

    # Public hostname of your RTMPS server
    # Use null to use the same value than `webserver.hostname`
    public_hostname: null

  # Allow to transcode the live streaming in multiple live resolutions
  transcoding:
    enabled: true

    # Enable remote runners to transcode your videos
    # If enabled, your instance won't transcode the videos itself
    # At least 1 remote runner must be configured to transcode your videos
    remote_runners:
      enabled: false

    # Amount of threads used by ffmpeg per live when using local transcoding
    threads: 2

    # Choose the local transcoding profile
    # New profiles can be added by plugins
    # Available in core PeerTube: 'default'
    profile: "default"

    resolutions:
      144p: false
      240p: false
      360p: false
      480p: false
      720p: false
      1080p: false
      1440p: false
      2160p: false

    # Also transcode original resolution, even if it's above your maximum enabled resolution
    always_transcode_original_resolution: true

video_studio:
  # Enable video edition by users (cut, add intro/outro, add watermark etc)
  # If enabled, users can create transcoding tasks as they wish
  enabled: false

  # Enable remote runners to transcode studio tasks
  # If enabled, your instance won't transcode the videos itself
  # At least 1 remote runner must be configured to transcode your videos
  remote_runners:
    enabled: false

video_file:
  update:
    # Add ability for users to replace the video file of an existing video
    enabled: false

import:
  # Add ability for your users to import remote videos (from YouTube, torrent...)
  videos:
    # Amount of import jobs to execute in parallel
    concurrency: 1

    # Set a custom video import timeout to not block import queue
    timeout: "2 hours"

    # Classic HTTP or all sites supported by youtube-dl https://rg3.github.io/youtube-dl/supportedsites.html
    http:
      # We recommend to use a HTTP proxy if you enable HTTP import to prevent private URL access from this server
      # See https://docs.joinpeertube.org/maintain/configuration#security for more information
      enabled: false

      youtube_dl_release:
        # Direct download URL to youtube-dl binary
        # Github releases API is also supported
        # Examples:
        #   * https://api.github.com/repos/ytdl-org/youtube-dl/releases
        #   * https://api.github.com/repos/yt-dlp/yt-dlp/releases
        #   * https://yt-dl.org/downloads/latest/youtube-dl
        url: "https://api.github.com/repos/yt-dlp/yt-dlp/releases"

        # Release binary name: 'yt-dlp' or 'youtube-dl'
        name: "yt-dlp"

        # Path to the python binary to execute for youtube-dl or yt-dlp
        python_path: "/usr/bin/python3"

      # IPv6 is very strongly rate-limited on most sites supported by youtube-dl
      force_ipv4: false

    # Magnet URI or torrent file (use classic TCP/UDP/WebSeed to download the file)
    torrent:
      # We recommend to only enable magnet URI/torrent import if you trust your users
      # See https://docs.joinpeertube.org/maintain/configuration#security for more information
      enabled: false

  # Add ability for your users to synchronize their channels with external channels, playlists, etc
  video_channel_synchronization:
    enabled: false

    max_per_user: 10

    check_interval: 1 hour

    # Number of latest published videos to check and to potentially import when syncing a channel
    videos_limit_per_synchronization: 10

    # Max number of videos to import when the user asks for full sync
    full_sync_videos_limit: 1000

  users:
    # Video quota is checked on import so the user doesn't upload a too big archive file
    # Video quota (daily quota is not taken into account) is also checked for each video when PeerTube is processing the import
    enabled: true

export:
  users:
    # Allow users to export their PeerTube data in a .zip for backup or re-import
    # Only one export at a time is allowed per user
    enabled: true

    # Max size of the current user quota to accept or not the export
    # Goal of this setting is to not store too big archive file on your server disk
    max_user_video_quota: 10GB

    # How long PeerTube should keep the user export
    export_expiration: "2 days"

auto_blacklist:
  # New videos automatically blacklisted so moderators can review before publishing
  videos:
    of_users:
      enabled: false

# Instance settings
instance:
  name: "PeerTube"
  short_description: "PeerTube, an ActivityPub-federated video streaming platform using P2P directly in your web browser."
  description: "Welcome to this PeerTube instance!" # Support markdown
  terms: "No terms for now." # Support markdown
  code_of_conduct: "" # Supports markdown

  # Who moderates the instance? What is the policy regarding NSFW videos? Political videos? etc
  moderation_information: "" # Supports markdown

  # Why did you create this instance?
  creation_reason: "" # Supports Markdown

  # Who is behind the instance? A single person? A non profit?
  administrator: "" # Supports Markdown

  # How long do you plan to maintain this instance?
  maintenance_lifetime: "" # Supports Markdown

  # How will you pay the PeerTube instance server? With your own funds? With users donations? Advertising?
  business_model: "" # Supports Markdown

  # If you want to explain on what type of hardware your PeerTube instance runs
  # Example: '2 vCore, 2GB RAM...'
  hardware_information: "" # Supports Markdown

  # What are the main languages of your instance? To interact with your users for example
  # Uncomment or add the languages you want
  # List of supported languages: https://peertube.cpy.re/api/v1/videos/languages
  languages:
  #    - en
  #    - es
  #    - fr

  # You can specify the main categories of your instance (dedicated to music, gaming or politics etc)
  # Uncomment or add the category ids you want
  # List of supported categories: https://peertube.cpy.re/api/v1/videos/categories
  categories:
  #    - 1  # Music
  #    - 2  # Films
  #    - 3  # Vehicles
  #    - 4  # Art
  #    - 5  # Sports
  #    - 6  # Travels
  #    - 7  # Gaming
  #    - 8  # People
  #    - 9  # Comedy
  #    - 10 # Entertainment
  #    - 11 # News & Politics
  #    - 12 # How To
  #    - 13 # Education
  #    - 14 # Activism
  #    - 15 # Science & Technology
  #    - 16 # Animals
  #    - 17 # Kids
  #    - 18 # Food

  default_client_route: "/videos/trending"

  # Whether or not the instance is dedicated to NSFW content
  # Enabling it will allow other administrators to know that you are mainly federating sensitive content
  # Moreover, the NSFW checkbox on video upload will be automatically checked by default
  is_nsfw: false
  # By default, `do_not_list` or `blur` or `display` NSFW videos
  # Could be overridden per user with a setting
  default_nsfw_policy: "do_not_list"

  customizations:
    javascript: "" # Directly your JavaScript code (without <script> tags). Will be eval at runtime
    css: "" # Directly your CSS code (without <style> tags). Will be injected at runtime
  # Robot.txt rules. To disallow robots to crawl your instance and disallow indexation of your site, add `/` to `Disallow:`
  robots: |
    User-agent: *
    Disallow:
  # /.well-known/security.txt rules. This endpoint is cached, so you may have to wait a few hours before viewing your changes
  # To discourage researchers from testing your instance and disable security.txt integration, set this to an empty string
  securitytxt: |
    Contact: https://github.com/Chocobozzz/PeerTube/blob/develop/SECURITY.md
    Expires: 2025-12-31T11:00:00.000Z'

services:
  # Cards configuration to format video in Twitter/X
  # All other social media (Facebook, Mastodon, etc.) are supported out of the box
  twitter:
    # Indicates the Twitter/X account for the website or platform where the content was published
    # This is just an information injected in HTML that is required by Twitter/X
    username: "@Chocobozzz"

followers:
  instance:
    # Allow or not other instances to follow yours
    enabled: true
    # Whether or not an administrator must manually validate a new follower
    manual_approval: false

followings:
  instance:
    # If you want to automatically follow back new instance followers
    # If this option is enabled, use the mute feature instead of deleting followings
    # /!\ Don't enable this if you don't have a reactive moderation team /!\
    auto_follow_back:
      enabled: false

    # If you want to automatically follow instances of the public index
    # If this option is enabled, use the mute feature instead of deleting followings
    # /!\ Don't enable this if you don't have a reactive moderation team /!\
    auto_follow_index:
      enabled: false
      # Host your own using https://framagit.org/framasoft/peertube/instances-peertube#peertube-auto-follow
      index_url: ""

theme:
  default: "default"

broadcast_message:
  enabled: false
  message: "" # Support markdown
  level: "info" # 'info' | 'warning' | 'error'
  dismissable: false

search:
  # Add ability to fetch remote videos/actors by their URI, that may not be federated with your instance
  # If enabled, the associated group will be able to "escape" from the instance follows
  # That means they will be able to follow channels, watch videos, list videos of non followed instances
  remote_uri:
    users: true
    anonymous: false

  # Use a third party index instead of your local index, only for search results
  # Useful to discover content outside of your instance
  # If you enable search_index, you must enable remote_uri search for users
  # If you do not enable remote_uri search for anonymous user, your instance will redirect the user on the origin instance
  # instead of loading the video locally
  search_index:
    enabled: false
    # URL of the search index, that should use the same search API and routes
    # than PeerTube: https://docs.joinpeertube.org/api-rest-reference.html
    # You should deploy your own with https://framagit.org/framasoft/peertube/search-index,
    # and can use https://search.joinpeertube.org/ for tests, but keep in mind the latter is an unmoderated search index
    url: ""
    # You can disable local search in the client, so users only use the search index
    disable_local_search: false
    # If you did not disable local search in the client, you can decide to use the search index by default
    is_default_search: false

# PeerTube client/interface configuration
client:
  videos:
    miniature:
      # By default PeerTube client displays author username
      prefer_author_display_name: false
      display_author_avatar: false

    resumable_upload:
      # Max size of upload chunks, e.g. '90MB'
      # If null, it will be calculated based on network speed
      max_chunk_size: null

  menu:
    login:
      # If you enable only one external auth plugin
      # You can automatically redirect your users on this external platform when they click on the login button
      redirect_on_single_external_auth: false

storyboards:
  # Generate storyboards of local videos using ffmpeg so users can see the video preview in the player while scrubbing the video
  enabled: true
```
