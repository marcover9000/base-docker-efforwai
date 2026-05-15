# Pla d'implementació: `base-docker`

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Construir la plantilla `base-docker/` que es pot copiar a `<projecte>-infra/` per aixecar l'entorn Docker (admin Laravel + api Laravel + MariaDB) d'un projecte amb un sol `./bin/up`.

**Architecture:** Plantilla auto-continguda. `docker-compose.yml` orquestra 3 serveis (`admin`, `api`, `db`) dins d'una xarxa Docker `app`. Dockerfiles propis basats en `php:8.2-apache` (admin afegeix Node 20 per Vite). Tota la personalització per projecte va a `.env`. Scripts a `bin/` per ergonomia diària.

**Tech Stack:** Docker Compose, PHP 8.2-Apache, Node 20, MariaDB 10.3.39, Bash.

**Spec de referència:** `docs/superpowers/specs/2026-05-15-laravel-docker-base-design.md`

---

## Estructura final de fitxers

```
base-docker/
├── bin/
│   ├── up
│   ├── down
│   ├── fresh
│   ├── admin
│   ├── api
│   ├── composer
│   └── artisan
├── docker/
│   ├── php-admin/
│   │   ├── Dockerfile
│   │   └── php.ini
│   └── php-api/
│       ├── Dockerfile
│       └── php.ini
├── docs/                         # ja existeix (spec + pla)
├── .env.example
├── .gitignore                    # ja existeix amb .env i .claude/
├── docker-compose.yml
└── README.md
```

Cada fitxer té una responsabilitat clara i petita. Cap dependència entre fitxers que pugui crear ambigüitat: el compose llegeix `.env`, els Dockerfiles són independents un de l'altre, els bin scripts són autoexplicatius.

---

## Task 1: `.env.example` + verificació

**Files:**
- Create: `/home/marc/desenvolupament/base-docker/.env.example`

- [ ] **Step 1: Escriure `.env.example`**

```dotenv
# Nom del projecte. Ha de coincidir amb els directoris germans:
# ../${PROJECT_NAME}-admin i ../${PROJECT_NAME}-api
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

- [ ] **Step 2: Verificar**

Run: `cat /home/marc/desenvolupament/base-docker/.env.example | wc -l`
Expected: 13 (o similar, no buit)

- [ ] **Step 3: Commit**

```bash
cd /home/marc/desenvolupament/base-docker
git add .env.example
git commit -m "feat: .env.example amb les vars a editar per projecte"
```

---

## Task 2: `docker-compose.yml` + validació de sintaxi

**Files:**
- Create: `/home/marc/desenvolupament/base-docker/docker-compose.yml`

- [ ] **Step 1: Escriure `docker-compose.yml`**

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

- [ ] **Step 2: Crear un `.env` provisional per validar el compose**

```bash
cd /home/marc/desenvolupament/base-docker
cp .env.example .env
```

- [ ] **Step 3: Validar la sintaxi del compose**

Run: `cd /home/marc/desenvolupament/base-docker && docker compose config --quiet`
Expected: cap sortida (exit 0). Si surt error sobre `build` que no existeix encara, és normal — anem a verificar només el parse i la resolució de vars amb:
Run: `cd /home/marc/desenvolupament/base-docker && docker compose config | grep -E "(container_name|image|driver)"`
Expected: veure `projecte-x-admin`, `projecte-x-api`, `projecte-x-db`, `mariadb:10.3.39`, `bridge`.

- [ ] **Step 4: Eliminar `.env` provisional (no s'ha de commitejar mai)**

Run: `cd /home/marc/desenvolupament/base-docker && rm .env`
Verifica: `git status` no ha de mostrar `.env`.

- [ ] **Step 5: Commit**

```bash
cd /home/marc/desenvolupament/base-docker
git add docker-compose.yml
git commit -m "feat: docker-compose amb admin, api i db (MariaDB 10.3.39)"
```

---

## Task 3: Imatge PHP admin (Apache + Node 20)

**Files:**
- Create: `/home/marc/desenvolupament/base-docker/docker/php-admin/Dockerfile`
- Create: `/home/marc/desenvolupament/base-docker/docker/php-admin/php.ini`

- [ ] **Step 1: Crear el directori**

Run: `mkdir -p /home/marc/desenvolupament/base-docker/docker/php-admin`

- [ ] **Step 2: Escriure `docker/php-admin/php.ini`**

```ini
memory_limit = 512M
upload_max_filesize = 64M
post_max_size = 64M
max_execution_time = 120
```

- [ ] **Step 3: Escriure `docker/php-admin/Dockerfile`**

```dockerfile
FROM php:8.2-apache

# Dependències d'OS + extensions de PHP que Laravel necessita
RUN apt-get update && apt-get install -y \
        git unzip libzip-dev libpng-dev libjpeg-dev libfreetype6-dev \
        libonig-dev libxml2-dev libicu-dev curl gnupg \
    && docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install pdo_mysql mbstring exif pcntl bcmath gd zip intl \
    && rm -rf /var/lib/apt/lists/*

# Node 20 (cal per Vite a l'admin)
RUN curl -fsSL https://deb.nodesource.com/setup_20.x | bash - \
    && apt-get install -y nodejs \
    && rm -rf /var/lib/apt/lists/*

# Composer
COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

# DocumentRoot apunta a public/ (Laravel)
ENV APACHE_DOCUMENT_ROOT=/var/www/html/public
RUN sed -ri 's!/var/www/html!${APACHE_DOCUMENT_ROOT}!g' \
        /etc/apache2/sites-available/*.conf \
        /etc/apache2/apache2.conf \
        /etc/apache2/conf-available/*.conf \
    && a2enmod rewrite

COPY php.ini /usr/local/etc/php/conf.d/zz-overrides.ini

WORKDIR /var/www/html
EXPOSE 80 5173
```

- [ ] **Step 4: Crear `.env` provisional i fer build per validar**

```bash
cd /home/marc/desenvolupament/base-docker
cp .env.example .env
docker compose build admin
```

Expected: build correcte, ha de descarregar `php:8.2-apache`, instal·lar les extensions i acabar amb "Successfully built ...". Si falla la instal·lació d'algun paquet apt, mirar la línia exacta.

- [ ] **Step 5: Verificar que el contenidor pot arrencar**

Run: `docker run --rm $(docker compose images -q admin 2>/dev/null || docker images -q | head -1) php -m | grep -E "pdo_mysql|gd|zip|intl|bcmath"`

Una alternativa més directa, dins de l'arrel de base-docker:
Run: `docker compose run --rm --no-deps admin php -m`
Expected: la sortida inclou `pdo_mysql`, `gd`, `zip`, `intl`, `bcmath`, `mbstring`.

També verificar Node:
Run: `docker compose run --rm --no-deps admin node --version`
Expected: `v20.x.x`

- [ ] **Step 6: Netejar `.env` provisional**

```bash
cd /home/marc/desenvolupament/base-docker
rm .env
```

- [ ] **Step 7: Commit**

```bash
cd /home/marc/desenvolupament/base-docker
git add docker/php-admin/
git commit -m "feat: imatge PHP admin (Apache + Node 20 per Vite)"
```

---

## Task 4: Imatge PHP api (Apache, sense Node)

**Files:**
- Create: `/home/marc/desenvolupament/base-docker/docker/php-api/Dockerfile`
- Create: `/home/marc/desenvolupament/base-docker/docker/php-api/php.ini`

- [ ] **Step 1: Crear el directori**

Run: `mkdir -p /home/marc/desenvolupament/base-docker/docker/php-api`

- [ ] **Step 2: Escriure `docker/php-api/php.ini`**

```ini
memory_limit = 512M
upload_max_filesize = 64M
post_max_size = 64M
max_execution_time = 120
```

- [ ] **Step 3: Escriure `docker/php-api/Dockerfile`**

```dockerfile
FROM php:8.2-apache

# Dependències d'OS + extensions de PHP que Laravel necessita
RUN apt-get update && apt-get install -y \
        git unzip libzip-dev libpng-dev libjpeg-dev libfreetype6-dev \
        libonig-dev libxml2-dev libicu-dev \
    && docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install pdo_mysql mbstring exif pcntl bcmath gd zip intl \
    && rm -rf /var/lib/apt/lists/*

# Composer
COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

# DocumentRoot apunta a public/ (Laravel)
ENV APACHE_DOCUMENT_ROOT=/var/www/html/public
RUN sed -ri 's!/var/www/html!${APACHE_DOCUMENT_ROOT}!g' \
        /etc/apache2/sites-available/*.conf \
        /etc/apache2/apache2.conf \
        /etc/apache2/conf-available/*.conf \
    && a2enmod rewrite

COPY php.ini /usr/local/etc/php/conf.d/zz-overrides.ini

WORKDIR /var/www/html
EXPOSE 80
```

- [ ] **Step 4: Build per validar**

```bash
cd /home/marc/desenvolupament/base-docker
cp .env.example .env
docker compose build api
```

Expected: build correcte.

- [ ] **Step 5: Verificar extensions PHP dins de l'api**

Run: `docker compose run --rm --no-deps api php -m`
Expected: inclou `pdo_mysql`, `gd`, `zip`, `intl`, `bcmath`, `mbstring`. **No** ha de tenir Node (Node no és una extensió PHP, però verifiquem que el contenidor és lleuger):
Run: `docker compose run --rm --no-deps api which node || echo "no node, correcte"`
Expected: `no node, correcte`

- [ ] **Step 6: Netejar `.env` provisional**

```bash
cd /home/marc/desenvolupament/base-docker
rm .env
```

- [ ] **Step 7: Commit**

```bash
cd /home/marc/desenvolupament/base-docker
git add docker/php-api/
git commit -m "feat: imatge PHP api (Apache, sense Node)"
```

---

## Task 5: Scripts `up`, `down`, `fresh`

**Files:**
- Create: `/home/marc/desenvolupament/base-docker/bin/up`
- Create: `/home/marc/desenvolupament/base-docker/bin/down`
- Create: `/home/marc/desenvolupament/base-docker/bin/fresh`

- [ ] **Step 1: Crear el directori**

Run: `mkdir -p /home/marc/desenvolupament/base-docker/bin`

- [ ] **Step 2: Escriure `bin/up`**

```bash
#!/usr/bin/env bash
# aixeca tot en mode detached
set -euo pipefail
cd "$(dirname "$0")/.."
docker compose up -d "$@"
```

- [ ] **Step 3: Escriure `bin/down`**

```bash
#!/usr/bin/env bash
# atura tot (manté volums)
set -euo pipefail
cd "$(dirname "$0")/.."
docker compose down "$@"
```

- [ ] **Step 4: Escriure `bin/fresh`**

```bash
#!/usr/bin/env bash
# reset complet: borra volums i imatges locals, reconstrueix
set -euo pipefail
cd "$(dirname "$0")/.."
read -rp "Això esborrarà volums i imatges. Segur? [s/N] " r
[[ "$r" =~ ^[sS]$ ]] || exit 0
docker compose down -v --rmi local
docker compose build --no-cache
docker compose up -d
```

- [ ] **Step 5: Fer-los executables**

```bash
chmod +x /home/marc/desenvolupament/base-docker/bin/up
chmod +x /home/marc/desenvolupament/base-docker/bin/down
chmod +x /home/marc/desenvolupament/base-docker/bin/fresh
```

- [ ] **Step 6: Validar sintaxi bash**

Run: `bash -n /home/marc/desenvolupament/base-docker/bin/up && bash -n /home/marc/desenvolupament/base-docker/bin/down && bash -n /home/marc/desenvolupament/base-docker/bin/fresh && echo OK`
Expected: `OK`

- [ ] **Step 7: Commit**

```bash
cd /home/marc/desenvolupament/base-docker
git add bin/up bin/down bin/fresh
git commit -m "feat: scripts up, down i fresh"
```

---

## Task 6: Scripts `admin` i `api` (shells als contenidors)

**Files:**
- Create: `/home/marc/desenvolupament/base-docker/bin/admin`
- Create: `/home/marc/desenvolupament/base-docker/bin/api`

- [ ] **Step 1: Escriure `bin/admin`**

```bash
#!/usr/bin/env bash
# shell al contenidor admin. usa -r per entrar com a root
set -euo pipefail
cd "$(dirname "$0")/.."
if [[ "${1:-}" == "-r" ]]; then
    shift
    docker compose exec -u root admin bash "$@"
else
    docker compose exec admin bash "$@"
fi
```

- [ ] **Step 2: Escriure `bin/api`**

```bash
#!/usr/bin/env bash
# shell al contenidor api. usa -r per entrar com a root
set -euo pipefail
cd "$(dirname "$0")/.."
if [[ "${1:-}" == "-r" ]]; then
    shift
    docker compose exec -u root api bash "$@"
else
    docker compose exec api bash "$@"
fi
```

- [ ] **Step 3: Fer-los executables**

```bash
chmod +x /home/marc/desenvolupament/base-docker/bin/admin
chmod +x /home/marc/desenvolupament/base-docker/bin/api
```

- [ ] **Step 4: Validar sintaxi bash**

Run: `bash -n /home/marc/desenvolupament/base-docker/bin/admin && bash -n /home/marc/desenvolupament/base-docker/bin/api && echo OK`
Expected: `OK`

- [ ] **Step 5: Commit**

```bash
cd /home/marc/desenvolupament/base-docker
git add bin/admin bin/api
git commit -m "feat: scripts admin i api (shell als contenidors)"
```

---

## Task 7: Scripts `composer` i `artisan`

**Files:**
- Create: `/home/marc/desenvolupament/base-docker/bin/composer`
- Create: `/home/marc/desenvolupament/base-docker/bin/artisan`

- [ ] **Step 1: Escriure `bin/composer`**

```bash
#!/usr/bin/env bash
# composer dins d'admin o api
# ús: ./bin/composer admin install   |   ./bin/composer api require xx/yy
set -euo pipefail
cd "$(dirname "$0")/.."
target="${1:?cal indicar admin|api}"
shift
docker compose exec "$target" composer "$@"
```

- [ ] **Step 2: Escriure `bin/artisan`**

```bash
#!/usr/bin/env bash
# php artisan dins d'admin o api
# ús: ./bin/artisan api migrate   |   ./bin/artisan admin route:list
set -euo pipefail
cd "$(dirname "$0")/.."
target="${1:?cal indicar admin|api}"
shift
docker compose exec "$target" php artisan "$@"
```

- [ ] **Step 3: Fer-los executables**

```bash
chmod +x /home/marc/desenvolupament/base-docker/bin/composer
chmod +x /home/marc/desenvolupament/base-docker/bin/artisan
```

- [ ] **Step 4: Validar sintaxi**

Run: `bash -n /home/marc/desenvolupament/base-docker/bin/composer && bash -n /home/marc/desenvolupament/base-docker/bin/artisan && echo OK`
Expected: `OK`

- [ ] **Step 5: Verificar que el target obligatori s'aplica**

Run: `/home/marc/desenvolupament/base-docker/bin/composer 2>&1 | head -2`
Expected: missatge d'error tipus `cal indicar admin|api` (set -u + `${1:?...}` dispara abans de cridar Docker).

- [ ] **Step 6: Commit**

```bash
cd /home/marc/desenvolupament/base-docker
git add bin/composer bin/artisan
git commit -m "feat: scripts composer i artisan"
```

---

## Task 8: `README.md`

**Files:**
- Create: `/home/marc/desenvolupament/base-docker/README.md`

- [ ] **Step 1: Escriure el `README.md`**

````markdown
# base-docker

Plantilla d'infraestructura Docker per a projectes Laravel amb la
forma `xxxx-admin` (front) + `xxxx-api` (back). Tot va per Docker;
només cal editar `.env`.

## Estructura esperada al disc

```
projecte-x/
├── projecte-x-admin/   # Laravel front
├── projecte-x-api/     # Laravel api
└── projecte-x-infra/   # ← còpia d'aquesta plantilla
```

## Setup d'un projecte nou

1. Copia la carpeta `base-docker/` a `<projecte>-infra/`.
2. `cp .env.example .env` i edita `PROJECT_NAME`, credencials i ports.
3. Al `.env` de l'**admin**, posa-hi `API_HOST=http://api/api/`. Dins de
   la xarxa Docker l'API es diu `api`, no `localhost`.
4. `./bin/up`

## Ús diari

| Comanda | Què fa |
|---|---|
| `./bin/up` | Aixeca tot |
| `./bin/down` | Atura tot (manté la DB) |
| `./bin/fresh` | Reset complet (borra DB i imatges, reconstrueix) |
| `./bin/admin` | Shell al contenidor admin (`-r` per root) |
| `./bin/api` | Shell al contenidor api (`-r` per root) |
| `./bin/composer api install` | Composer dins de l'api |
| `./bin/artisan api migrate` | Artisan dins de l'api |

## URLs

- Admin: `http://localhost:8000`
- API: `http://localhost:8001`
- DB: `localhost:3306` (des del host, p.ex. amb DBeaver)

## Què conté

- 3 contenidors: `admin`, `api`, `db`
- PHP 8.2 + Apache (Dockerfiles propis, sense Sail)
- MariaDB 10.3.39
- Node 20 dins de l'admin per a Vite
````

- [ ] **Step 2: Verificar**

Run: `head -3 /home/marc/desenvolupament/base-docker/README.md`
Expected: comença amb `# base-docker`.

- [ ] **Step 3: Commit**

```bash
cd /home/marc/desenvolupament/base-docker
git add README.md
git commit -m "docs: README amb instruccions d'ús"
```

---

## Task 9: Verificació end-to-end amb projectes reals

Aquesta tasca valida que la plantilla funciona aixecant-la contra els
projectes reals que ja existeixen a `~/dev/balearic-boats-*`.
**Important:** és una verificació, no es committeja res aquí.

**Files:** cap (només execució i verificació)

- [ ] **Step 1: Preparar un workspace de test**

```bash
mkdir -p /tmp/base-docker-test
cd /tmp/base-docker-test
cp -r ~/dev/balearic-boats-admin balearic-boats-admin
cp -r ~/dev/balearic-boats-api balearic-boats-api
cp -r /home/marc/desenvolupament/base-docker balearic-boats-infra
cd balearic-boats-infra
rm -rf .git docs   # només volem la part Docker per al test
```

- [ ] **Step 2: Configurar `.env`**

```bash
cd /tmp/base-docker-test/balearic-boats-infra
cp .env.example .env
sed -i 's/^PROJECT_NAME=.*/PROJECT_NAME=balearic-boats/' .env
sed -i 's/^DB_DATABASE=.*/DB_DATABASE=balearic_boats/' .env
sed -i 's/^DB_USERNAME=.*/DB_USERNAME=balearic/' .env
sed -i 's/^DB_PASSWORD=.*/DB_PASSWORD=secret/' .env
```

- [ ] **Step 3: Ajustar `.env` de l'admin perquè parli amb l'API via xarxa Docker**

```bash
cd /tmp/base-docker-test/balearic-boats-admin
# guarda valor original per restaurar després
grep -E "^API_HOST=" .env
sed -i "s|^API_HOST=.*|API_HOST='http://api/api/'|" .env
sed -i "s|^API_HOST_STORAGE=.*|API_HOST_STORAGE='http://api/storage/'|" .env
```

- [ ] **Step 4: Ajustar `.env` de l'api perquè apunti al servei `db`**

```bash
cd /tmp/base-docker-test/balearic-boats-api
sed -i 's/^DB_HOST=.*/DB_HOST=db/' .env
sed -i 's/^DB_DATABASE=.*/DB_DATABASE=balearic_boats/' .env
sed -i 's/^DB_USERNAME=.*/DB_USERNAME=balearic/' .env
sed -i 's/^DB_PASSWORD=.*/DB_PASSWORD=secret/' .env
```

- [ ] **Step 5: Aixecar**

```bash
cd /tmp/base-docker-test/balearic-boats-infra
./bin/up
sleep 15   # dona temps a que db sigui healthy i Apache arrenqui
docker compose ps
```

Expected: els 3 serveis `Up`. `db` amb `(healthy)`.

- [ ] **Step 6: Instal·lar dependències de Laravel a admin i api**

```bash
cd /tmp/base-docker-test/balearic-boats-infra
./bin/composer admin install
./bin/composer api install
./bin/artisan api migrate --force
```

Expected: composer i migrate sense errors. Si `migrate` falla per "no
tables yet", normal — vol dir que la DB està viva i que les migracions
es creen.

- [ ] **Step 7: Verificar que admin i api responen**

```bash
curl -sI http://localhost:8000 | head -1
curl -sI http://localhost:8001 | head -1
```

Expected: `HTTP/1.1 200 OK` (o 302 si redirigeix al login). Si `500`,
mirar `docker compose logs admin` i `docker compose logs api`.

- [ ] **Step 8: Netejar**

```bash
cd /tmp/base-docker-test/balearic-boats-infra
./bin/down -v
cd /
rm -rf /tmp/base-docker-test
```

- [ ] **Step 9: Restaurar `.env`s originals dels projectes reals**

Si has tocat directament `~/dev/balearic-boats-*/.env` (en comptes de
còpies a `/tmp`), restaura'ls amb git:

```bash
cd ~/dev/balearic-boats-admin && git checkout .env 2>/dev/null || true
cd ~/dev/balearic-boats-api && git checkout .env 2>/dev/null || true
```

(Si `.env` no està al git, restaura manualment l'`API_HOST` i credencials).

---

## Task 10 (opcional): Configurar remot GitHub

**Files:** cap

Aquesta tasca **NO s'executa fins que l'usuari ho demani explícitament**.
El destí és: https://github.com/marcover9000/base-docker-efforwai

- [ ] **Step 1: Afegir el remot**

```bash
cd /home/marc/desenvolupament/base-docker
git remote add origin https://github.com/marcover9000/base-docker-efforwai.git
```

- [ ] **Step 2: Verificar branca principal**

Run: `cd /home/marc/desenvolupament/base-docker && git branch --show-current`
Expected: `main` o `master`. Si és `master` i el repo a GitHub espera `main`:
Run: `cd /home/marc/desenvolupament/base-docker && git branch -m master main`

- [ ] **Step 3: Push (només quan l'usuari ho demani)**

```bash
cd /home/marc/desenvolupament/base-docker
git push -u origin main
```

---

## Self-review (fet)

- **Cobertura spec → pla**: tots els fitxers de l'spec tenen tasca: `.env.example` (T1), `docker-compose.yml` (T2), Dockerfile + php.ini admin (T3), Dockerfile + php.ini api (T4), bin scripts (T5-7), README (T8). El `.gitignore` ja existeix (commit anterior, junt amb el spec).
- **Consistència de noms**: `${PROJECT_NAME}-admin`, `${PROJECT_NAME}-api`, `${PROJECT_NAME}-db` consistents arreu. Servei `admin`/`api`/`db` consistents als bin scripts (`docker compose exec admin/api`).
- **Sense placeholders**: cada step té el codi exacte o la comanda exacta. Cap "TBD" ni "similar a l'anterior".
- **Excloses (YAGNI)**: `docker/mariadb/init/` (no necessari ara, ho deixem fora; si en algun projecte cal, es crea localment).
