# nginx-server-docker

Proxy reverso central con TLS para **todo el servidor** — no para un repo en particular. Cualquier
proyecto (un monorepo de sitios Angular, una API .NET, una app Java, un sitio estático, lo que sea)
vive en su propio contenedor aislado y se une a la red compartida de este repo para que nginx le
pueda hacer `proxy_pass`. A nginx no le importa el stack: solo necesita "nombre de contenedor +
puerto interno".

Corre de forma **fija e independiente** — se levanta una sola vez en el servidor y queda ahí. Los
proyectos van y vienen (se agregan, se redespliegan, se bajan) sin que este stack se entere ni se
reinicie por eso.

## Arquitectura

- **Una sola red Docker compartida** (`edge-net`), creada por este repo (`docker-compose.yml`, sin
  `external: true`) — cualquier proyecto se conecta a ella declarándola `external: true` en su
  propio `docker-compose.yml`. Un contenedor puede pertenecer a varias redes de Docker a la vez:
  nada impide que un proyecto tenga ADEMÁS su propia red privada para sus propios contenedores
  internos (ej. API + base de datos) que nginx nunca toca. Si el día de mañana dos proyectos
  necesitan hablarse directo entre sí sin pasar por nginx, se suman a una red adicional puntual sin
  rediseñar nada de esto — no hace falta anticipar esa necesidad hoy.
- **Un archivo `.conf.template` por proyecto** en `resources/templates/`, ninguno sabe del otro.
  Cada uno con su propia zona de rate-limiting/variable de WebSocket upgrade con nombre único
  (derivado del slug del proyecto) — **obligatorio, no cosmético**: todos los `.template` de esta
  carpeta se cargan dentro del mismo bloque `http{}` de un único nginx, y dos `limit_req_zone` (o
  dos `map $http_upgrade`) con el mismo nombre chocan entre sí — nginx se niega a arrancar
  COMPLETO, no solo ese sitio. Ver `docs/example-site.conf.template` para la plantilla comentada de
  referencia (no vive en `resources/templates/` a propósito, para que la imagen de nginx no intente
  procesarla).
- **Un `.env` con variables por proyecto** (`APP_<SLUG>_*`) — agregar un proyecto nuevo es agregar
  su bloque a `.env` + su `.conf.template`, nunca tocar `docker-compose.yml` (usa `env_file`, no una
  lista de `environment:` que crecería sin límite).
- **Certificados**: `resources/ssl/` puede tener varias carpetas — un certificado wildcard manual
  para un dominio, certificados de Let's Encrypt para otros. Cada bloque `APP_<SLUG>_VOLUME_SSL`
  apunta a la carpeta que le corresponde; no hay un único certificado "del sistema".
- **`resources/templates/00-tuning.conf.template`**: 2 directivas de `http{}` (no un sitio) —
  `variables_hash_bucket_size`/`server_names_hash_bucket_size` en 128. El default de nginx (64) se
  queda corto ya con ~10 proyectos cargados (`could not build variables_hash_bucket_size` / mismo
  error con `server_names_hash_bucket_size`, nginx se niega a arrancar COMPLETO) — bug real
  encontrado al validar 12 sitios juntos. Subir de nuevo si nginx vuelve a quejarse con muchos más.

> [!WARNING]
> **`env_file:` resuelve `${VAR}` de forma SECUENCIAL, línea por línea** — a diferencia de
> `environment:` (arma una tabla completa antes de sustituir), una variable referenciada ANTES de
> definirse en `.env` queda vacía en silencio, sin ningún error de Compose. Síntoma real: nginx en
> loop de restart con `cannot load certificate ".../ssl//archivo.cer"` (doble `/`, la pista). Si
> agregás una variable base compartida (tipo `APP_CERT_FOLDER`), definila ANTES de cualquier otra
> que la referencie.

## Flujo de trabajo

1. Levantar este stack una sola vez, siempre antes que cualquier proyecto:
   ```bash
   cp .env.template .env   # completar valores reales
   docker compose up -d --build
   ```
2. En cada proyecto (otro repo, otra carpeta), su propio `docker-compose.yml` declara la red como
   externa:
   ```yaml
   networks:
     edge-net:
       external: true
   services:
     mi-servicio:
       networks: [edge-net]
   ```
3. Agregar el bloque `APP_<SLUG>_*` correspondiente a `.env` (ver comentarios en `.env.template`,
   incluye un ejemplo real de sitio Angular y uno de una API en otro dominio/stack) y copiar
   `docs/example-site.conf.template` a `resources/templates/<proyecto>.conf.template`,
   reemplazando `SLUG`/`SLUG_MAY` por el nombre real.
4. `docker compose restart nginx` (no hace falta rebuild, los templates se reprocesan al
   arrancar el contenedor).

## Resolución de DNS por request

Cada template usa `resolver 127.0.0.11 valid=10s;` (el DNS embebido de Docker) + una variable en
`proxy_pass`, en vez de un bloque `upstream {}` estático. Esto es intencional y necesario: un
`upstream {}` estático obliga a nginx a resolver el nombre del contenedor backend **al
arrancar/recargar**, y si el backend no existe todavía, nginx completo se niega a arrancar. Con
`resolver` + variable, nginx sigue sirviendo aunque el proyecto todavía no tenga contenedor
levantado (devuelve 502 hasta que exista), y recoge sola una IP nueva si el contenedor se recrea.

## Certificado

Dos mecanismos soportados, elegí el que corresponda por proyecto:

- **Opción A — manual**: copiar la carpeta del certificado (hoja + cadena intermedia concatenadas,
  más la key) a `resources/ssl/<Carpeta>/` y referenciarla desde el bloque `APP_<SLUG>_*` en
  `.env`. `COMPOSE_PROFILES` queda vacío — el servicio `certbot` ni se levanta.
- **Opción B — Let's Encrypt automático (dns-01 vía IONOS)**: `COMPOSE_PROFILES=letsencrypt` +
  completar `LETSENCRYPT_DOMAIN`/`LETSENCRYPT_EMAIL` en `.env` + `certbot/ionos.ini` (copiar de
  `ionos.ini.template`, credencial real de la API de IONOS — nunca se commitea). Emisión inicial
  (una sola vez, manual):
  ```bash
  cp certbot/ionos.ini.template certbot/ionos.ini   # completar con la API key real
  docker compose run --rm --entrypoint certbot certbot certonly \
    --dns-ionos --dns-ionos-credentials /etc/letsencrypt/ionos.ini \
    -d TU_DOMINIO -d "*.TU_DOMINIO" --email TU_CORREO --agree-tos --non-interactive
  docker compose up -d
  ```
  certbot escribe en `resources/ssl/live/TU_DOMINIO/{fullchain,privkey,chain}.pem` — referenciar
  esa ruta desde el bloque `APP_<SLUG>_*` del proyecto que la use. **nginx no relee el certificado
  solo tras una renovación** — necesita `docker compose exec nginx nginx -s reload` para tomar el
  archivo nuevo (no se automatiza vía socket de Docker a propósito, evita darle a este contenedor
  más privilegios de los que necesita).

## Fail2Ban

nginx corre en Docker, no en el host — Fail2Ban (que sí corre en el host) necesita
`resources/logs/` bind-montado (`NGINX_VOLUME_LOGS`) para poder leer `access.log`/`error.log`. Ver
la guía de despliegue del servidor para la configuración completa de Fail2Ban.

## Redespliegue automático (CI/CD)

`.github/workflows/deploy.yml` redespliega este stack solo (`docker compose up -d --build`) en cada
push a `main`. Usa los mismos 4 secrets que cualquier otro repo de este servidor: `DEPLOY_SSH_KEY`,
`DEPLOY_HOST`, `DEPLOY_USER`, `SSH_PORT` — ver la guía de despliegue del servidor para el detalle
completo de cómo se genera/comparte esa llave entre repos.

## ¿Por qué se separó de ngx-docs-markdown-kit?

Originalmente nginx vivía dentro de `ngx-docs-markdown-kit` (`docker/nginx/`), acoplado a los 9
sitios de ese monorepo. Eso funcionaba mientras nginx solo atendía ese repo, pero un proxy reverso
de servidor tiene que poder atender cualquier proyecto futuro (otro monorepo, un repo suelto,
cualquier stack) sin que ese proyecto tenga que vivir dentro de `ngx-docs-markdown-kit` para
beneficiarse de él. Este repo es el resultado de sacar esa pieza a su propio lugar, genérico desde
el diseño.
