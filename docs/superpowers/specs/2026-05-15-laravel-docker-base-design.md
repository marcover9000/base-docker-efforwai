# Disseny: `base-docker` — plantilla d'infra Docker per projectes Laravel admin + api

**Data:** 2026-05-15
**Estat:** Validat amb l'usuari, pendent d'implementació

## Context i objectiu

L'equip treballa amb projectes Laravel amb una estructura constant:

```
<projecte>/
├── <projecte>-admin/   # Laravel front (Blade + Vite)
├── <projecte>-api/     # Laravel api REST
└── <projecte>-infra/   # infraestructura Docker
```

Avui cada projecte porta la seva configuració de Docker (Laravel Sail
amb `compose.yaml` propi a admin i api per separat, i un MariaDB dins
de l'api). No hi ha `/infra` centralitzat.

L'objectiu és crear `base-docker/`: una **plantilla** que es copia a
`<projecte>-infra/` cada cop que es comença un projecte nou. Tot està
pre-definit (Dockerfiles, compose, scripts d'ergonomia); només cal
editar `.env` per personalitzar-lo.

Referència real del que els projectes utilitzen avui: `~/dev/balearic-boats-admin`
i `~/dev/balearic-boats-api` (Laravel 12, PHP 8.2, MariaDB 10.3.39).

## Principis de disseny

- **Caixa negra**: els companys no han de tocar `/etc/hosts`, instal·lar
  PHP, ni configurar res al host més enllà de Docker.
- **Plug-and-play**: només `.env` canvia entre projectes. `docker-compose.yml`,
  Dockerfiles i scripts són idèntics entre projectes.
- **Independent de Sail**: Dockerfiles propis. Així no es trenca quan
  algú fa `composer update` a l'admin/api.
- **Basat en l'ús real**: només s'inclou el que els projectes
  actualment utilitzen. Sense Redis, Mailpit ni Nginx separat.

## Stack

| Servei | Imatge | Per què |
|---|---|---|
| admin  | `php:8.2-apache` + Node 20 (build propi) | Laravel front + Vite |
| api    | `php:8.2-apache` (build propi) | Laravel api |
| db     | `mariadb:10.3.39` | Coincideix amb el que fa servir l'API actual |

Apache (no Nginx) perquè ve integrat amb la imatge oficial de PHP, no
cal procés separat i el config és mínim. Sense Redis (avui usen
`CACHE_STORE=database`, `QUEUE_CONNECTION=database`). Sense Mailpit
(admin usa `MAIL_MAILER=log`, api usa SMTP real). Sense Nginx separat
ni dominis `.localhost` (l'usuari prefereix ports directes per
mantenir-ho "caixa negra" sense convencions extra).

## Estructura del repositori

```
base-docker/
├── bin/                       # scripts d'ergonomia (executables)
│   ├── up                     # docker compose up -d
│   ├── down                   # docker compose down
│   ├── fresh                  # reset volums + rebuild (interactiu)
│   ├── admin                  # shell al contenidor admin
│   ├── api                    # shell al contenidor api
│   ├── composer               # ./bin/composer admin|api <args>
│   └── artisan                # ./bin/artisan admin|api <args>
├── docker/
│   ├── php-admin/
│   │   ├── Dockerfile         # PHP 8.2-apache + Node 20 + extensions Laravel
│   │   └── php.ini            # memory_limit, upload size, etc.
│   ├── php-api/
│   │   ├── Dockerfile         # PHP 8.2-apache + extensions Laravel (sense Node)
│   │   └── php.ini
│   └── mariadb/
│       └── init/              # scripts SQL inicials (opcional, buit per defecte)
├── docs/                      # aquesta especificació i futurs plans
├── .env.example               # ÚNIC fitxer a editar quan es copia
├── .gitignore                 # ignora .env i .claude/
├── docker-compose.yml
└── README.md
```

## `docker-compose.yml`

```yaml
services:
  admin:
    build: ./docker/php-admin
    container_name: ${PROJECT_NAME}-admin
    ports:
      - "${APP_PORT:-8000}:80"
      - "${VITE_PORT:-5173}:5173"
    volumes:
      - ../${PROJECT_NAME}-admin:/var/www/html
    networks: [app]

  api:
    build: ./docker/php-api
    container_name: ${PROJECT_NAME}-api
    ports:
      - "${API_PORT:-8001}:80"
    volumes:
      - ../${PROJECT_NAME}-api:/var/www/html
    depends_on:
      db:
        condition: service_healthy
    networks: [app]

  db:
    image: mariadb:10.3.39
    container_name: ${PROJECT_NAME}-db
    ports:
      - "${DB_PORT:-3306}:3306"
    environment:
      MARIADB_ROOT_PASSWORD: ${DB_PASSWORD}
      MARIADB_DATABASE: ${DB_DATABASE}
      MARIADB_USER: ${DB_USERNAME}
      MARIADB_PASSWORD: ${DB_PASSWORD}
    volumes:
      - db_data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mariadb-admin", "ping", "-p${DB_PASSWORD}"]
      retries: 5
      timeout: 3s
    networks: [app]

volumes:
  db_data:

networks:
  app:
    driver: bridge
```

**Convenció clau:** els directoris del codi font han de dir-se
`${PROJECT_NAME}-admin` i `${PROJECT_NAME}-api` i estar al directori
germà de `<projecte>-infra/`.

**Comunicació admin → api:** dins de la xarxa Docker `app`, l'API és
accessible com a `http://api/`. El `.env` de l'admin ha de posar
`API_HOST=http://api/api/` (no `http://localhost:8001/api/`). Es
documentarà al README.

## Dockerfiles

### `docker/php-admin/Dockerfile`

```dockerfile
FROM php:8.2-apache

RUN apt-get update && apt-get install -y \
        git unzip libzip-dev libpng-dev libjpeg-dev libfreetype6-dev \
        libonig-dev libxml2-dev libicu-dev curl gnupg \
    && docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install pdo_mysql mbstring exif pcntl bcmath gd zip intl \
    && rm -rf /var/lib/apt/lists/*

# Node 20 per Vite
RUN curl -fsSL https://deb.nodesource.com/setup_20.x | bash - \
    && apt-get install -y nodejs && rm -rf /var/lib/apt/lists/*

COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

ENV APACHE_DOCUMENT_ROOT=/var/www/html/public
RUN sed -ri 's!/var/www/html!${APACHE_DOCUMENT_ROOT}!g' \
        /etc/apache2/sites-available/*.conf /etc/apache2/apache2.conf /etc/apache2/conf-available/*.conf \
    && a2enmod rewrite

COPY php.ini /usr/local/etc/php/conf.d/zz-overrides.ini

WORKDIR /var/www/html
EXPOSE 80 5173
```

### `docker/php-api/Dockerfile`

Idèntic a l'anterior **excepte** que no instal·la Node ni exposa 5173.

### `docker/php-{admin,api}/php.ini`

```ini
memory_limit = 512M
upload_max_filesize = 64M
post_max_size = 64M
max_execution_time = 120
```

## `.env.example`

```dotenv
# El nom del projecte ha de coincidir amb els directoris germans
# (../${PROJECT_NAME}-admin i ../${PROJECT_NAME}-api)
PROJECT_NAME=projecte-x

# Ports publicats al host
APP_PORT=8000
API_PORT=8001
VITE_PORT=5173
DB_PORT=3306

# Credencials de la base de dades
DB_DATABASE=projecte_x
DB_USERNAME=projecte_x
DB_PASSWORD=secret
```

Aquestes són les **úniques** dades que un company ha de tocar per
adaptar la plantilla al seu projecte.

## `.gitignore`

```
.env
.claude/
```

`.claude/` mai no es comparteix amb companys (preferència de l'usuari).

## Bin scripts

Tots són bash, tots `set -euo pipefail`, tots fan `cd` a la rel de
`base-docker/` abans d'executar Docker, perquè es puguin cridar des
de qualsevol path.

| Script | Comportament |
|---|---|
| `up` | `docker compose up -d "$@"` |
| `down` | `docker compose down "$@"` |
| `fresh` | Confirmació + `down -v --rmi local` + `build --no-cache` + `up -d` |
| `admin` | `docker compose exec admin bash`; `-r` per root |
| `api` | `docker compose exec api bash`; `-r` per root |
| `composer` | `docker compose exec <admin\|api> composer <args>` |
| `artisan` | `docker compose exec <admin\|api> php artisan <args>` |

Tots els comentaris dins dels scripts: en català, una línia, escuets.

## README.md

Curt, en català. Explica:
1. Estructura esperada al disc.
2. Setup primer cop (copiar, editar `.env`, posar `API_HOST=http://api/api/`
   al `.env` de l'admin, executar `./bin/up`).
3. Llistat dels bin scripts d'ús diari.
4. URLs (localhost:8000 admin, localhost:8001 api, localhost:3306 db).

## Flux d'ús per a un projecte nou

```bash
# 1. Crea l'estructura
mkdir -p ~/desenvolupament/projecte-x
cd ~/desenvolupament/projecte-x

# 2. Clona admin i api (o crea projectes Laravel)
git clone <admin-repo> projecte-x-admin
git clone <api-repo>   projecte-x-api

# 3. Copia la plantilla
cp -r ~/desenvolupament/base-docker projecte-x-infra
cd projecte-x-infra
cp .env.example .env
# editar PROJECT_NAME=projecte-x i credencials DB

# 4. Ajusta .env de l'admin perquè parli amb l'api per nom de servei
# API_HOST=http://api/api/

# 5. Aixeca
./bin/up
```

## Què queda fora d'aquest disseny (YAGNI)

- Redis: no l'usen.
- Mailpit: no l'usen (l'admin loga, l'api SMTP real).
- Nginx separat / dominis `.localhost`: l'usuari prefereix ports directes.
- Adminer/phpMyAdmin: poden afegir-se per projecte si calen.
- Múltiples versions de PHP: 8.2 és prou. Si en algun projecte canvia,
  s'edita el Dockerfile localment (decisió conscient, no via `.env`).
- Workers Laravel en contenidor separat: avui `QUEUE_CONNECTION=database`
  i sense `queue:work` permanent. Si en cal, s'afegirà.

## Riscos i decisions a recordar

- **Convenció de noms de directori**: si un projecte no segueix
  `<projecte>-admin` i `<projecte>-api`, el `docker-compose.yml` no els
  trobarà. Convenció estricta, no parametritzable (decisió de l'usuari
  per simplicitat).
- **`API_HOST` a `.env` admin**: cal canviar `localhost:8001` per
  `api` per anar via xarxa Docker. Fàcil d'oblidar. Documentat al README.
- **MariaDB 10.3.39**: és antic (2019). S'usa perquè coincideix amb
  producció. Si en algun moment es modernitza producció, també cal
  actualitzar la plantilla.
