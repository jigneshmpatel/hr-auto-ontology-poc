# HR Auto-Ontology POC

This project evaluates NVIDIA Auto-Ontology using a synthetic HR SQLite database.

The POC tests whether an automatically generated and human-reviewed ontology improves natural-language answers about leave, attendance, payroll, and performance.

## Dataset

This proof of concept uses synthetic HR training data. It does not contain
records from a production HR system or information about real employees.

The dataset originated from: <training provider/course name>.

Redistribution status: <permitted / pending verification>.

## Prerequisites

- Docker Desktop with Docker Compose v2
- Git with submodule support
- An NVIDIA API key beginning with `nvapi-` from <https://build.nvidia.com/>

Initialize the pinned NVIDIA Auto-Ontology dependency after cloning this
repository:

```bash
git submodule update --init --recursive
```

## Local configuration

Auto-Ontology reads its local configuration from
`vendor/auto-ontology/.env`. This file contains secrets and must not be
committed.

Create it from NVIDIA's example:

```bash
cp vendor/auto-ontology/.env.example vendor/auto-ontology/.env
```

Generate separate local secrets. Copy each generated value into the matching
setting in `vendor/auto-ontology/.env`:

```bash
openssl rand -hex 32  # AUTH_SECRET
openssl rand -hex 24  # POSTGRES_PASSWORD
openssl rand -hex 24  # AUTO_ONTOLOGY_ADMIN_PASSWORD
openssl rand -hex 24  # PGADMIN_PASSWORD
```

Set or update these values in `vendor/auto-ontology/.env`:

```dotenv
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_USER=postgres
POSTGRES_PASSWORD=<generated-postgres-password>
POSTGRES_DATABASE=auto_ontology

# SQLite source path inside the Docker containers.
CONNECTION_STRINGS=sqlite:////data/hr_database.db

DEFAULT_MODELS_API_KEY=nvapi-xxxxxxxx
APP_URL=http://localhost:3000

PGADMIN_EMAIL=admin@example.com
PGADMIN_PASSWORD=<generated-pgadmin-password>

AUTO_ONTOLOGY_ADMIN_EMAIL=admin@example.com
AUTO_ONTOLOGY_ADMIN_PASSWORD=<generated-admin-password>
AUTH_SECRET=<generated-auth-secret>
```

The example file also contains placeholder API keys for individual model
roles. Remove these placeholder lines so the roles inherit
`DEFAULT_MODELS_API_KEY`:

```dotenv
REASONING_API_KEY=nvapi-your-api-key-here
NON_REASONING_API_KEY=nvapi-your-api-key-here
EMBED_API_KEY=nvapi-your-api-key-here
RERANK_API_KEY=nvapi-your-api-key-here
```

Keep the role-specific endpoint and model settings, particularly the embedding
and reranking settings.

## HR database path

The Docker override mounts the host's `data/hr_database.db` into the backend
and ingestion containers at `/data/hr_database.db`. Define the host path before
running Docker Compose.

From the repository root:

```bash
export HR_DATABASE_PATH="$(pwd)/data/hr_database.db"
test -f "$HR_DATABASE_PATH" && echo "HR database found"
```

This variable lasts for the current terminal session. To make it available to
Docker Compose without exporting it each time, add the absolute path to the
uncommitted `vendor/auto-ontology/.env` file:

```dotenv
HR_DATABASE_PATH=/absolute/path/to/hr-auto-ontology-poc/data/hr_database.db
```

Never put a real API key, password, generated secret, or personal absolute path
in this README or another committed file.

## Build and run

The HR SQLite database replaces NVIDIA's Wide World Importers sample. Do not
clone `auto-ontology-eval` or run the Wide World Importers seed script for this
POC.

Run the stack from the Auto-Ontology submodule. If `HR_DATABASE_PATH` is not in
the `.env` file, export it in the same terminal first as shown above.

```bash
cd vendor/auto-ontology

docker compose \
  -f docker-compose.yml \
  -f ../../docker-compose.hr.yml \
  up -d --wait postgres

docker compose \
  -f docker-compose.yml \
  -f ../../docker-compose.hr.yml \
  up -d --build
```

Check the services and ingestion logs:

```bash
docker compose \
  -f docker-compose.yml \
  -f ../../docker-compose.hr.yml \
  ps

docker compose \
  -f docker-compose.yml \
  -f ../../docker-compose.hr.yml \
  logs -f ingestion-service
```

Open <http://localhost:3000>, sign in using
`AUTO_ONTOLOGY_ADMIN_EMAIL` and `AUTO_ONTOLOGY_ADMIN_PASSWORD`, and enable
**Settings -> Semantic Compilation**.

Stop the services without deleting the PostgreSQL/pgvector volume:

```bash
docker compose \
  -f docker-compose.yml \
  -f ../../docker-compose.hr.yml \
  down
```

Do not add `-v` unless the ontology, catalog, users, and other persisted local
state should be deleted.

## Secret checks before committing

Confirm that the local environment file is ignored and review the staged files:

```bash
git -C vendor/auto-ontology check-ignore -v .env
git status
git diff --cached
```
