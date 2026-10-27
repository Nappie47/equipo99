# Registro de decisiones — equipo01

Integrantes: Johan Muñoz

Completa **una fila por decisión**. Explica el porqué en una o dos frases: la defensa oral se basa en este documento.

## Decisiones técnicas (obligatorias)

| # | Decisión | Alternativas que consideraron | Por qué eligieron esta | Riesgo o costo que aceptan |
|---|----------|-------------------------------|------------------------|----------------------------|
| 1 | Imagen base de `catalog` y `worker` | `python:3.12` (completa), `python:3.12-alpine` | Usamos `python:3.12-slim` en multi-stage porque las librerías (`psycopg2-binary`) se instalan directo con wheels y queda liviana (~200 MB). Alpine a veces da problemas al compilar paquetes en C. | La imagen slim no trae herramientas como `curl`, así que para el healthcheck usamos Python directamente. |
| 2 | Imagen base de `orders` | `node:20` (pesa casi 1 GB), `node:20-slim` | Elegimos `node:20-alpine` con multi-stage porque Node corre bien sobre Alpine, trae el usuario no root `node` y la imagen final pesa ~200 MB con las dependencias instaladas. | Alpine usa `musl libc`, por lo que si más adelante se agregan librerías nativas complejas podría requerir dependencias extras. |
| 3 | Imagen base y usuario de `frontend` | `nginx:alpine` estándar, `httpd:alpine` | Usamos `nginxinc/nginx-unprivileged:alpine`. Viene lista para correr con usuario no root (UID 101) y escucha en el puerto 8080 sin tener que configurar permisos a mano. | No puede escuchar en puertos menores al 1024 dentro del contenedor, pero lo resolvemos mapeando el puerto en Compose (`${FRONTEND_PORT}:8080`). |
| 4 | Cómo se manejan los secretos (contraseña de la base) | Dejar las variables fijas en el Dockerfile con `ENV`, pasarlas por `ARG` | Cargamos las variables desde el archivo `.env` mediante `env_file` en compose, y dejamos `.env` en el `.dockerignore` para que no se filtre en las imágenes. | Cualquiera con acceso al archivo `.env` en la máquina puede ver la clave; en producción real convendría usar Docker Secrets o un gestor externo. |
| 5 | Diseño de redes: qué servicio ve a cuál | Dejar que compose cree una sola red donde todos los servicios se vean entre sí | Creamos dos redes (`frontend_net` y `backend_net`). El frontend solo ve a catalog y orders; la base de datos y redis están aislados en el backend para que nadie acceda a la base desde afuera. | Hay que acordarse de conectar cada servicio a la red correspondiente en Compose para evitar problemas de conectividad. |
| 6 | Healthcheck de `worker` y de `db`/`cache` | Probar si el proceso de worker está vivo con `ps`, o inventar un endpoint HTTP | Para el worker revisamos con un comando en Python que `/tmp/healthy` se haya actualizado en los últimos 15 s. Para postgres usamos `pg_isready` y para redis `redis-cli ping`. | Se ejecuta un comando cada 5 segundos en cada contenedor, consumiendo un poco de CPU. |
| 7 | Dónde viven los datos y cómo sobreviven a un reinicio | Carpetas locales montadas (bind mounts) o no usar volumen | Usamos un volumen nombrado `db_data` montado en `/var/lib/postgresql/data` para que los pedidos y datos se guarden aunque se apague el stack con `docker compose down`. | Si se ejecuta `docker compose down -v` por error, se borra el volumen con todos los datos de la base. |

## Problemas que encontraron y cómo los diagnosticaron

1. **Error al construir `orders` por nombre de imagen inválido:**
   Al correr `docker compose build orders` salía un error de que no se encontraba la imagen en Docker Hub. Revisamos el Dockerfile y vimos que el equipo anterior escribió `FROM node:20-alpne` (le faltaba la "i" a alpine). Se corrigió la errata a `node:20-alpine` y construyó normalmente.
2. **Contraseña expuesta en el Dockerfile de `catalog`:**
   En el Dockerfile de catalog venía una línea `ENV POSTGRES_PASSWORD=...` en texto claro, lo que representaba un riesgo de seguridad. Al revisar el código y correr el verificador estático nos alertó de secretos en capas. Lo solucionamos borrando esa línea y pasando la variable mediante el archivo `.env`.

## Incidentes

| Incidente | Qué pasó | Qué cambiaron | Cuánto demoraron |
|-----------|----------|---------------|------------------|
| 1 | `orders` fallaba al construir por imagen no encontrada | Se corrigió `node:20-alpne` a `node:20-alpine` en el Dockerfile | 5 min |
| 2 | Contraseña visible en las capas de `catalog` | Se quitó `ENV POSTGRES_PASSWORD` del Dockerfile y se dejó en `.env` | 5 min |
| 3 | Riesgo de acceso directo desde frontend a la base de datos | Se separó en dos redes (`frontend_net` y `backend_net`) | 10 min |

## Uso de IA

Ocupamos Gemini para desarrollar consultas sobre cómo estructurar el archivo `docker-compose.yml` (especialmente las dependencias por salud con `depends_on` y la separación de redes) y cómo reducir el tamaño de las imágenes usando multi-stage build con imágenes ligeras (slim y alpine). Tuvimos que corregir y descartar sugerencias donde nos proponía usar `curl` para los healthchecks en imágenes que no lo traían instalado, adaptándolo a comandos nativos de Python y `wget`.

## Qué haríamos distinto con más tiempo

1. Usar un sistema más formal de secretos (como Docker Secrets) en lugar de tener las contraseñas en un archivo `.env` en plano.
2. Agregar un panel de monitoreo básico para ver el estado de la cola en Redis y la actividad de la base de datos.
