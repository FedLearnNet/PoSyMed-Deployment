# PosyMed Deployment

Docker Compose deployment for the PosyMed federated learning stack.

The stack contains:

- global learning API and database
- orchestrator API with access to the host Docker daemon
- frontend
- Keycloak and Keycloak database
- nginx reverse proxy
- user documentation
- local Docker registry on port `5000`

## Requirements

- Docker Engine with the Compose plugin
- access to the GitLab container registry used by the service images
- a host where port `8291` is available for the reverse proxy
- a host where port `5000` is available for the local Docker registry

## Configuration

Runtime configuration lives in `env/*.env`.

Before starting the stack, review and update at least:

- `env/orch-api.env`
- `env/global-api.env`
- `env/keycloak.env`
- `env/global-learning-db.env`
- `env/keycloak-postgres.env`
- `env/nginx.env`

The compose services currently pull PosyMed images from `gitlab.cosy.bio:5050`.
Log in on the deployment host before the first start:

```sh
docker login gitlab.cosy.bio:5050
```

## Local Docker Registry

The deployment includes a local registry:

```yaml
registry:
  image: registry:2
  ports:
    - "5000:5000"
```

Registry data is stored in the named Docker volume `posymed_registry-data`.
Using a named volume keeps registry data outside the repository and avoids
accidentally committing image layers.

The registry uses Docker Registry's native `htpasswd` authentication. For this
deployment, the password is treated like an API key: generate a long random
value once, store only its bcrypt hash in `registry-auth/htpasswd`, and put the
plain value into the orchestrator env.

Generate the local registry API key and htpasswd file:

```sh
mkdir -p registry-auth

REGISTRY_USER=posymed
REGISTRY_API_KEY="$(openssl rand -base64 32)"

docker run --rm --entrypoint htpasswd httpd:2 \
  -Bbn "$REGISTRY_USER" "$REGISTRY_API_KEY" > registry-auth/htpasswd

printf '%s\n' "$REGISTRY_API_KEY"
```

Copy the printed value into `env/orch-api.env`:

```env
ORCH_DOCKER__LOCAL__REGISTRY_USERNAME=posymed
ORCH_DOCKER__LOCAL__REGISTRY_PASSWORD=<printed registry api key>
```

Use the same values for manual Docker login:

```sh
docker login localhost:5000
```

The orchestrator is configured in `env/orch-api.env` with:

```env
ORCH_DOCKER__LOCAL__REGISTRY_URL=localhost:5000
ORCH_DOCKER__LOCAL__ENABLED=true
ORCH_DOCKER__LOCAL__REGISTRY_USERNAME=posymed
ORCH_DOCKER__LOCAL__REGISTRY_PASSWORD=<printed registry api key>
```

`localhost:5000` is intentional when the orchestrator talks to the host Docker
daemon through `/var/run/docker.sock`: the Docker daemon resolves and pulls the
image from the deployment host. If the registry is moved to another machine,
replace this with the registry host name or IP, for example
`registry.example.org:5000`.

Quick registry check:

```sh
curl -u posymed:<printed registry api key> http://localhost:5000/v2/
```

Expected result: `{}`.

## Orchestrator Registries

`env/orch-api.env` defines the registries known by the orchestrator:

```env
ORCH_DOCKER__LOCAL__REGISTRY_URL=localhost:5000
ORCH_DOCKER__LOCAL__ENABLED=true
ORCH_DOCKER__LOCAL__REGISTRY_USERNAME=posymed
ORCH_DOCKER__LOCAL__REGISTRY_PASSWORD=<printed registry api key>

ORCH_DOCKER__FEATURECLOUD__REGISTRY_URL=featurecloud.ai
ORCH_DOCKER__FEATURECLOUD__ENABLED=true
# ORCH_DOCKER__FEATURECLOUD__REGISTRY_USERNAME=simon.suewer@uni-hamburg.de
# ORCH_DOCKER__FEATURECLOUD__REGISTRY_PASSWORD=<insert featurecloud registry token>

ORCH_DOCKER__GITLAB__REGISTRY_URL=gitlab.cosy.bio
ORCH_DOCKER__GITLAB__ENABLED=true
# ORCH_DOCKER__GITLAB__REGISTRY_USERNAME=simon.suewer@uni-hamburg.de
ORCH_DOCKER__GITLAB__REGISTRY_PASSWORD=
```

Quarkus maps these environment variables to properties like
`orch.docker."gitlab".registry-url`.

## Secrets

Do not commit real secrets. Keep the values in the deployment host environment
or in the local `env/*.env` files used for that host.

Generate strong local passwords with:

```sh
openssl rand -base64 32
```

Create GitLab tokens in GitLab under user, project, or group access tokens.
Use the smallest scope that works for the deployment. Common examples are:

- `read_registry` for pulling private images
- `write_registry` only when pushing images
- `read_api` or `api` only if the pipeline integration requires API access

Insert secret values by editing the right variable after the `=` sign:

```env
ORCH_DOCKER__GITLAB__REGISTRY_PASSWORD=<gitlab registry token>
ORCH_DOCKER__LOCAL__REGISTRY_PASSWORD=<printed registry api key>
PIPELINE_DOCKER_PASSWORD=<gitlab registry token>
PIPELINE_REPO_TOKEN=<gitlab api/project token>
QUARKUS_OIDC_CREDENTIALS_SECRET=<keycloak client secret>
QUARKUS_KEYCLOAK_ADMIN_CLIENT_CLIENT_SECRET=<keycloak client secret>
QUARKUS_LANGCHAIN4J_OPENAI_API_KEY=<openai or ollama api key, if used>
KC_BOOTSTRAP_ADMIN_PASSWORD=<generated admin password>
```

For Keycloak client secrets, open the Keycloak admin UI after import and copy
the generated client secret for the configured client into `env/global-api.env`.

## Start

Start the stack:

```sh
docker compose up -d
```

Show service status:

```sh
docker compose ps
```

Follow logs:

```sh
docker compose logs -f
```

The reverse proxy publishes the application on host port `8291`.

## Update

Pull current images and recreate services:

```sh
docker compose pull
docker compose up -d
```

## Stop

Stop containers while keeping volumes:

```sh
docker compose down
```

Remove containers and volumes only when the deployment data can be deleted:

```sh
docker compose down -v
```
