# Explicación Docker Compose: Nginx Proxy + Let's Encrypt

Este documento explica línea por línea un archivo `docker-compose.yml` que configura un proxy reverso con Nginx y certificados SSL automáticos.

---

## PRIMER SERVICIO: nginx-proxy

### services:
Define los contenedores/servicios que Docker Compose va a crear.

### nginx-proxy:
Nombre del primer servicio.

### image: jwilder/nginx-proxy
Imagen de Docker a usar. Es un Nginx que detecta automáticamente otros contenedores.

### container_name: nginx-proxy
Nombre del contenedor (en vez de uno auto-generado).

### ports:
Mapea puertos de tu máquina → contenedor:
- `"80:80"` = HTTP (puerto 80 de tu máquina → puerto 80 del contenedor)
- `"443:443"` = HTTPS (puerto 443 de tu máquina → puerto 443 del contenedor)

### volumes:
Carpetas/archivos compartidos entre tu máquina y el contenedor.

#### - conf:/etc/nginx/conf.d
Volumen llamado `conf` se monta en la ruta `/etc/nginx/conf.d` (configuraciones de Nginx).

#### - vhost:/etc/nginx/vhost.d
Volumen `vhost` → configuraciones de virtual hosts.

#### - dhparam:/etc/nginx/dhparam
Volumen `dhparam` → parámetros Diffie-Hellman para seguridad SSL.

#### - certs:/etc/nginx/certs:ro
Volumen `certs` → certificados SSL.
- `:ro` = **read-only** (solo lectura, no puede modificar)

#### - html:/usr/share/nginx/html
Volumen `html` → archivos HTML estáticos.

#### - /var/run/docker.sock:/tmp/docker.sock:ro
**¡Importante!** Comparte el socket de Docker con el contenedor.
- Permite que nginx-proxy **detecte otros contenedores automáticamente**
- `:ro` = solo lectura (por seguridad)

### networks:
#### - proxy
Conecta este contenedor a la red `proxy` (para que se comunique con otros contenedores).

### restart: always
Si el contenedor se detiene, Docker lo reinicia automáticamente.

---

## SEGUNDO SERVICIO: letsencrypt

### letsencrypt:
Nombre del segundo servicio. Genera certificados SSL gratis de Let's Encrypt.

### image: jrcs/letsencrypt-nginx-proxy-companion
Imagen que trabaja con nginx-proxy para obtener certificados SSL automáticamente.

### container_name: nginx-proxy-le
Nombre del contenedor ("le" = Let's Encrypt).

### volumes_from:
#### - nginx-proxy
**Hereda todos los volúmenes** del servicio `nginx-proxy`.
Así puede acceder a los mismos certificados y configuraciones.

### volumes:

#### - certs:/etc/nginx/certs:rw
Vuelve a montar `certs` pero ahora con `:rw` = **read-write**.
Necesita escribir los certificados que genera.

#### - /var/run/docker.sock:/var/run/docker.sock:ro
También necesita acceso al socket de Docker para detectar contenedores.
Solo lectura.

#### - acme:/etc/acme.sh
Volumen para guardar datos del protocolo ACME (usado por Let's Encrypt).

#### - html:/usr/share/nginx/html
Comparte el HTML para validación de dominios.

### restart: always
Reinicio automático.

---

## VOLÚMENES

```yaml
volumes:
  conf:
  vhost:
  dhparam:
  certs:
  acme:
  html:
```

Define **volúmenes nombrados** que persisten los datos.
- Sin configuración específica = Docker los gestiona automáticamente
- Los datos **no se pierden** aunque borres los contenedores

---

## REDES

```yaml
networks:
  proxy:
    name: nginx-proxy
    external: true
```

Define la red `proxy`:
- `name: nginx-proxy` = el nombre real de la red en Docker
- `external: true` = la red ya existe (fue creada antes con `docker network create nginx-proxy`)

---

## ¿Para qué sirve todo esto?

Este setup crea un **proxy reverso automático**:

1. **nginx-proxy** detecta cuando levantas otros contenedores
2. Si esos contenedores tienen variables de entorno como `VIRTUAL_HOST=miapp.com`
3. Nginx automáticamente los configura para responder en ese dominio
4. **letsencrypt** detecta el nuevo dominio y obtiene un certificado SSL
5. Tu app queda accesible en `https://miapp.com` automáticamente

### Ejemplo de uso

```yaml
# Otro docker-compose.yml para tu app
my-app:
  image: node:18
  environment:
    - VIRTUAL_HOST=miapp.com
    - LETSENCRYPT_HOST=miapp.com
    - LETSENCRYPT_EMAIL=tu@email.com
  networks:
    - nginx-proxy
```

¡Y listo! Tu app tendrá HTTPS automáticamente.

---

## Conceptos clave

- **Socket de Docker**: Archivo especial que permite la comunicación entre el cliente Docker y el daemon
- **Daemon**: Programa que corre en segundo plano sin interacción directa del usuario
- **Volumen**: Espacio de almacenamiento persistente que sobrevive aunque se eliminen los contenedores
- **Red**: Permite la comunicación entre contenedores
- **Proxy Reverso**: Servidor que recibe peticiones y las redirige a los contenedores correctos según el dominio

---

## Comandos útiles

```bash
# Crear la red externa antes de levantar los servicios
docker network create nginx-proxy

# Levantar los servicios
docker-compose up -d

# Ver logs
docker-compose logs -f

# Detener los servicios
docker-compose down

# Ver contenedores corriendo
docker ps
```
