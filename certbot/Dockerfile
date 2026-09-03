# Imagen oficial de certbot (mantenida por EFF/Let's Encrypt), sin version fija a proposito --
# a diferencia de nginx/mssql, la superficie de ataque real de este contenedor es minima (no
# expone ningun puerto, no atiende trafico externo, solo corre certbot en un loop) y la CA de
# Let's Encrypt a veces exige una version minima de certbot para hablar el protocolo ACME
# vigente -- seguir "latest" aqui evita romper renovaciones por un pin viejo.
FROM certbot/certbot:latest

# certbot-dns-ionos (PyPI, github.com/helgeerbe/certbot-dns-ionos): plugin de certbot para
# resolver el reto dns-01 contra la API de DNS "normal" de IONOS (developer.hosting.ionos.com) --
# la de dominios/hosting comprados directo en ionos.com/ionos.mx, DISTINTA de "IONOS Cloud DNS"
# (otro producto, con su propio plugin certbot-dns-ionos-cloud). Si el dominio se compro como
# hosting/dominio normal (no como parte de IONOS Cloud), este es el plugin correcto. Si usas otro
# proveedor de DNS, cambia este plugin por el correspondiente (certbot tiene uno para casi todos).
RUN pip install --no-cache-dir certbot-dns-ionos

# OJO: a proposito NO se pone aqui un ENTRYPOINT propio -- la imagen base ya trae
# ENTRYPOINT ["certbot"] (confirmado con "docker inspect"), y sobreescribirlo rompe la emision
# inicial manual (docker-compose run --rm certbot certonly ...) porque esos argumentos le
# llegarian a un shell-loop en vez de a certbot. El loop de renovacion para el uso de "docker-compose
# up -d" en cambio, se define a nivel de "entrypoint:"/"command:" en docker-compose.yml, no aqui --
# asi "docker-compose run --entrypoint certbot certbot certonly ..." sigue funcionando para la
# emision inicial sin pelearse con el loop.
