# Udistrital Web Backend

Backend headless de la Universidad Distrital basado en Drupal 11, JSON:API,
Composer y Docker.

## Requisitos

- Docker Desktop o Docker Engine con Compose
- Git
- Composer 2.x y PHP 8.3+ solo si se trabajará fuera de Docker

## Instalación local

Desde el directorio de este repositorio:

```bash
cp .env.example .env
```

Edita `.env` y define contraseñas locales para `DB_PASSWORD`,
`DB_ROOT_PASSWORD` y un valor aleatorio para `HASH_SALT`.

Instala las dependencias si trabajarás directamente con PHP o tu IDE las
necesita:

```bash
composer install
```

Puedes construir la imagen del backend desde este repositorio:

```bash
docker build -t web_institucional_backend .
```

Para ejecutarlo necesitas conectarlo a una instancia MySQL accesible. Para
desarrollo local se recomienda usar el `docker-compose.yml` del repositorio de
infraestructura, porque también inicia MySQL y conecta el frontend.

## Integración con Docker Compose

La estructura esperada por el Compose local es:

```text
web_institucional/
├── docker-compose.yml
├── web_institucional_backend/
└── web_institucional_frontend/
```

Desde el directorio que contiene `docker-compose.yml`:

```bash
cp web_institucional_backend/.env.example web_institucional_backend/.env
cp web_institucional_backend/.env .env
# Edita web_institucional_backend/.env antes de continuar.
docker compose build backend
docker compose up -d db backend
```

El servicio queda disponible en:

- Drupal: http://localhost:8080
- JSON:API: http://localhost:8080/jsonapi
- Login administrativo: http://localhost:8080/user/login

## Primera instalación

Desde la raíz del entorno Compose, carga las variables y ejecuta la
instalación una sola vez:

```bash
set -a
source web_institucional_backend/.env
set +a

docker compose exec backend vendor/bin/drush site:install standard \
  --db-url="mysql://${DB_USERNAME}:${DB_PASSWORD}@db:3306/${DB_DATABASE}" \
  --site-name="Universidad Distrital" \
  --account-name=admin \
  --account-pass='cambia-esta-clave' \
  --locale=es -y

docker compose exec backend vendor/bin/drush en jsonapi -y
docker compose exec backend vendor/bin/drush cache:rebuild
```

Cambia la contraseña administrativa inmediatamente después de la instalación.

## Configuración

- `web/sites/default/settings.php` obtiene la conexión de base de datos,
  `hash_salt`, hosts confiables y la ruta de sincronización desde variables de
  entorno.
- `web/sites/default/services.yml` habilita CORS para el frontend local.
- `config/sync/` contiene la configuración exportada de Drupal.
- `web/sites/default/files/` contiene archivos subidos y no debe versionarse.

Para importar configuración versionada:

```bash
docker compose exec backend vendor/bin/drush config:import -y
docker compose exec backend vendor/bin/drush cache:rebuild
```

Para aplicar un despliegue completo:

```bash
docker compose exec backend vendor/bin/drush deploy
```

Para exportar cambios hechos en Drupal:

```bash
docker compose exec backend vendor/bin/drush config:export -y
docker compose exec backend tar -cf - -C /var/www/html config/sync | tar -xf -
```

Después copia los archivos exportados a `config/sync/` de este repositorio,
revísalos y haz commit.

## Contenido y archivos

La configuración exportada no incluye artículos, usuarios ni archivos
subidos. Para compartir contenido entre entornos se necesita un respaldo de
la base de datos y una copia de `web/sites/default/files/`.

No subas al repositorio:

- `.env`
- Dumps `.sql` o `.sql.gz`
- `web/sites/default/files/`
- Credenciales, certificados o claves privadas

## Variables principales

| Variable | Uso |
| --- | --- |
| `DB_DATABASE` | Nombre de la base de datos |
| `DB_USERNAME` | Usuario de Drupal |
| `DB_PASSWORD` | Contraseña del usuario de Drupal |
| `DB_ROOT_PASSWORD` | Contraseña root de MySQL local |
| `DB_HOST` | Host de MySQL; en Compose es `db` |
| `DB_PORT` | Puerto de MySQL; normalmente `3306` |
| `HASH_SALT` | Sal única para Drupal |
| `TRUSTED_HOST_PATTERN` | Hosts permitidos por Drupal |

En producción, las variables deben gestionarse mediante secretos de AWS, no
mediante archivos `.env` dentro de la imagen.
