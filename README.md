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
