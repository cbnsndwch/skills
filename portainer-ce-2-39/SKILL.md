---
name: portainer-ce-2-39
description: "Interact with the Portainer CE v2.39.0 REST API. Covers authentication, environments, stacks, containers, Docker proxy, Kubernetes resources, edge agents, registries, users/teams, and settings. Use when automating Portainer 2.39.x deployments or writing integration code against this version."
compatibility: "Requires network access to a Portainer CE 2.39.x instance. curl or an HTTP client must be available."
metadata:
  author: cbn
  portainer-version: "2.39.0"
  api-spec-format: openapi-3.0.1
---

# Portainer CE v2.39.0 API

You are interacting with the **Portainer Community Edition v2.39.0** REST API. This skill provides the authentication flow, key concepts, endpoint categories, common operations, and gotchas specific to this version.

For full request/response schemas, consult the OpenAPI spec at [references/openapi.yml](references/openapi.yml).

## Authentication

Most endpoints require a JWT token. Obtain one via `POST /auth`:

```bash
TOKEN=$(curl -s -X POST "https://<portainer>/api/auth" \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"<password>"}' \
  | jq -r '.jwt')
```

Then include it in every subsequent request:

```
Authorization: Bearer $TOKEN
```

Alternatively, Portainer supports **API keys** (created under user settings in the UI). Pass them with the same `Authorization: Bearer <key>` header.

### Logout

```
POST /auth/logout
```

Invalidates the current JWT.

## Access policies

Each endpoint documents one of these policies:

| Policy | Meaning |
|---|---|
| **Public** | No authentication needed |
| **Authenticated** | Valid JWT or API key required |
| **Restricted** | Authenticated; response may be filtered by user permissions |
| **Administrator** | Authenticated + admin role required |

## Core concepts

- **Environment (Endpoint)**: A Docker host, Swarm cluster, or Kubernetes cluster managed by Portainer. The API uses the term "endpoint" — the `{id}` in `/endpoints/{id}` is the environment ID.
- **Stack**: A Docker Compose or Kubernetes manifest deployment managed through Portainer.
- **Edge Agent**: A remotely-managed Portainer agent that phones home to the Portainer server.
- **Endpoint Group**: A logical grouping of environments for access control and tagging.
- **Resource Control**: Fine-grained ownership/access metadata on Docker resources (containers, volumes, etc.).

## Docker proxy

Portainer does **not** expose dedicated endpoints for every Docker resource. Instead it proxies requests to the Docker Engine API:

```
/endpoints/{id}/docker/<docker-api-path>
```

Any request to this path is forwarded to the Docker API of that environment. Request and response shapes match the [Docker Engine API v1.30+](https://docs.docker.com/engine/api/v1.30/).

Example — list containers on environment 1:

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://<portainer>/api/endpoints/1/docker/containers/json?all=true"
```

### Private registry header

When pulling images from a private registry registered in Portainer, include a Base64-encoded JSON object in the `X-Registry-Auth` header:

```bash
echo -n '{"registryId":1}' | base64
# eyJyZWdpc3RyeUlkIjoxfQ==
```

## Endpoint categories

### Auth
| Method | Path | Description |
|--------|------|-------------|
| POST | `/auth` | Authenticate (returns JWT) |
| POST | `/auth/logout` | Logout |
| POST | `/auth/oauth/validate` | Validate OAuth token |

### Environments (Endpoints)
| Method | Path | Description |
|--------|------|-------------|
| GET | `/endpoints` | List environments |
| POST | `/endpoints` | Create environment |
| GET | `/endpoints/{id}` | Inspect environment |
| PUT | `/endpoints/{id}` | Update environment |
| DELETE | `/endpoints/{id}` | Remove environment |
| POST | `/endpoints/snapshot` | Snapshot all environments |
| POST | `/endpoints/{id}/snapshot` | Snapshot one environment |
| PUT | `/endpoints/{id}/settings` | Update environment settings |
| GET | `/endpoints/{id}/registries` | List registries for environment |
| PUT | `/endpoints/{id}/registries/{registryId}` | Update registry access |
| POST | `/endpoints/delete` | Batch delete environments |

### Stacks
| Method | Path | Description |
|--------|------|-------------|
| GET | `/stacks` | List stacks |
| GET | `/stacks/{id}` | Inspect stack |
| PUT | `/stacks/{id}` | Update stack |
| DELETE | `/stacks/{id}` | Remove stack |
| POST | `/stacks/{id}/start` | Start stack |
| POST | `/stacks/{id}/stop` | Stop stack |
| GET | `/stacks/{id}/file` | Get stack file content |
| POST | `/stacks/{id}/migrate` | Migrate stack to another environment |
| PUT | `/stacks/{id}/git` | Update stack Git settings |
| PUT | `/stacks/{id}/git/redeploy` | Redeploy stack from Git |
| POST | `/stacks/{id}/associate` | Associate orphaned stack |
| GET | `/stacks/name/{name}` | Get stack by name |

Stack creation has separate endpoints per type:

| Type | Path |
|------|------|
| Standalone (file) | `POST /stacks/create/standalone/file` |
| Standalone (string) | `POST /stacks/create/standalone/string` |
| Standalone (repo) | `POST /stacks/create/standalone/repository` |
| Swarm (file) | `POST /stacks/create/swarm/file` |
| Swarm (string) | `POST /stacks/create/swarm/string` |
| Swarm (repo) | `POST /stacks/create/swarm/repository` |
| Kubernetes (string) | `POST /stacks/create/kubernetes/string` |
| Kubernetes (repo) | `POST /stacks/create/kubernetes/repository` |
| Kubernetes (URL) | `POST /stacks/create/kubernetes/url` |

### Docker helpers
| Method | Path | Description |
|--------|------|-------------|
| GET | `/docker/{environmentId}/dashboard` | Environment dashboard summary |
| GET | `/docker/{environmentId}/images` | List images with extended info |
| GET | `/docker/{environmentId}/containers/{containerId}/gpus` | Container GPU info |

### Kubernetes
| Method | Path | Description |
|--------|------|-------------|
| GET | `/kubernetes/config` | Get kubeconfig |
| GET | `/kubernetes/{id}/dashboard` | K8s dashboard summary |
| GET | `/kubernetes/{id}/applications` | List applications |
| GET | `/kubernetes/{id}/namespaces` | List all namespaces |
| GET | `/kubernetes/{id}/namespaces/{namespace}/services` | List services in namespace |
| GET | `/kubernetes/{id}/namespaces/{namespace}/ingresses` | List ingresses in namespace |
| GET | `/kubernetes/{id}/namespaces/{namespace}/volumes` | List volumes in namespace |
| GET | `/kubernetes/{id}/namespaces/{namespace}/configmaps/{configmap}` | Get configmap |
| GET | `/kubernetes/{id}/namespaces/{namespace}/secrets/{secret}` | Get secret |
| GET | `/kubernetes/{id}/configmaps` | List all configmaps |
| GET | `/kubernetes/{id}/secrets` | List all secrets |
| GET | `/kubernetes/{id}/services` | List all services |
| GET | `/kubernetes/{id}/ingresses` | List all ingresses |
| GET | `/kubernetes/{id}/volumes` | List all volumes |
| GET | `/kubernetes/{id}/events` | List events |
| GET | `/kubernetes/{id}/describe` | Describe resource |
| GET | `/kubernetes/{id}/metrics/nodes` | Node metrics |
| GET | `/kubernetes/{id}/metrics/pods/{namespace}` | Pod metrics by namespace |
| GET | `/kubernetes/{id}/metrics/pods/{namespace}/{name}` | Pod metrics by name **(new in 2.39)** |
| POST | `/kubernetes/{id}/nodes/{name}/drain` | Drain node **(new in 2.39)** |
| GET | `/kubernetes/{id}/rbac_enabled` | Check RBAC status |
| POST | `/kubernetes/{id}/ingresses/delete` | Batch delete ingresses |
| POST | `/kubernetes/{id}/services/delete` | Batch delete services |
| GET | `/kubernetes/{id}/ingresscontrollers` | List ingress controllers |
| GET/PUT | `/kubernetes/{id}/namespaces/{namespace}` | Get/update namespace |
| GET | `/kubernetes/{id}/max_resource_limits` | Get resource limits |
| GET | `/kubernetes/{id}/nodes_limits` | Get node limits |
| GET | `/kubernetes/{id}/roles` | List roles |
| GET | `/kubernetes/{id}/clusterroles` | List cluster roles |
| GET | `/kubernetes/{id}/rolebindings` | List role bindings |
| GET | `/kubernetes/{id}/clusterrolebindings` | List cluster role bindings |
| GET | `/kubernetes/{id}/serviceaccounts` | List service accounts |
| GET | `/kubernetes/{id}/cron_jobs` | List cron jobs |
| GET | `/kubernetes/{id}/jobs` | List jobs |

### Helm (via environment)
| Method | Path | Description |
|--------|------|-------------|
| GET | `/endpoints/{id}/kubernetes/helm` | List Helm releases |
| POST | `/endpoints/{id}/kubernetes/helm` | Install Helm chart |
| DELETE | `/endpoints/{id}/kubernetes/helm/{release}` | Uninstall release |
| GET | `/endpoints/{id}/kubernetes/helm/{release}/history` | Release history |
| PUT | `/endpoints/{id}/kubernetes/helm/{release}/rollback` | Rollback release |
| GET | `/endpoints/{id}/kubernetes/helm/{name}` | Get release by name **(new in 2.39)** |

### Edge
| Method | Path | Description |
|--------|------|-------------|
| GET | `/edge_groups` | List edge groups |
| POST | `/edge_groups` | Create edge group |
| GET/PUT/DELETE | `/edge_groups/{id}` | Manage edge group |
| GET | `/edge_stacks` | List edge stacks |
| GET/PUT/DELETE | `/edge_stacks/{id}` | Manage edge stack |
| POST | `/edge_stacks/create/file` | Create edge stack from file |
| POST | `/edge_stacks/create/string` | Create edge stack from string |
| POST | `/edge_stacks/create/repository` | Create edge stack from repo |
| GET | `/edge_jobs` | List edge jobs |
| GET/PUT/DELETE | `/edge_jobs/{id}` | Manage edge job |

### Registries
| Method | Path | Description |
|--------|------|-------------|
| GET | `/registries` | List registries |
| POST | `/registries` | Create registry |
| GET/PUT/DELETE | `/registries/{id}` | Manage registry |
| POST | `/registries/{id}/configure` | Configure registry access |
| POST | `/registries/ping` | Test registry connectivity **(new in 2.39)** |

### Users & Teams
| Method | Path | Description |
|--------|------|-------------|
| GET | `/users` | List users |
| POST | `/users` | Create user |
| GET/PUT/DELETE | `/users/{id}` | Manage user |
| GET | `/users/me` | Current user info |
| PUT | `/users/{id}/passwd` | Change password |
| GET/POST | `/users/{id}/tokens` | List/create API keys |
| DELETE | `/users/{id}/tokens/{keyID}` | Revoke API key |
| GET | `/teams` | List teams |
| POST | `/teams` | Create team |
| GET/PUT/DELETE | `/teams/{id}` | Manage team |
| GET | `/team_memberships` | List memberships |
| POST | `/team_memberships` | Create membership |
| PUT/DELETE | `/team_memberships/{id}` | Manage membership |

### Settings & System
| Method | Path | Description |
|--------|------|-------------|
| GET/PUT | `/settings` | Get/update settings |
| GET | `/settings/public` | Public settings (no auth) |
| GET | `/status` | Portainer status |
| GET | `/system/info` | System information |
| GET | `/system/version` | Version info |
| GET | `/system/status` | System status |
| GET | `/system/nodes` | Cluster nodes (HA) |
| POST | `/system/upgrade` | Trigger upgrade |
| GET/PUT | `/ssl` | Manage SSL certificate |
| GET | `/motd` | Message of the day |

### Other
| Method | Path | Description |
|--------|------|-------------|
| POST | `/backup` | Create backup |
| POST | `/restore` | Restore from backup |
| GET | `/roles` | List roles |
| GET | `/tags` | List tags |
| POST | `/tags` | Create tag |
| DELETE | `/tags/{id}` | Delete tag |
| GET | `/custom_templates` | List custom templates |
| POST | `/custom_templates/create/string` | Create template from string |
| POST | `/custom_templates/create/file` | Create template from file |
| POST | `/custom_templates/create/repository` | Create template from repo |
| GET/DELETE | `/custom_templates/{id}` | Manage template |
| POST | `/resource_controls` | Create resource control |
| PUT/DELETE | `/resource_controls/{id}` | Manage resource control |
| GET/POST | `/webhooks` | List/create webhooks |
| PUT/DELETE | `/webhooks/{id}` | Manage webhook |
| POST | `/stacks/webhooks/{webhookID}` | Trigger stack webhook |
| POST | `/upload/tls/{certificate}` | Upload TLS certificate |
| POST | `/ldap/check` | Test LDAP connectivity |
| POST | `/gitops/repo/file/preview` | Preview Git repo file |
| GET/POST | `/templates` | List/create app templates |
| GET | `/templates/helm` | List Helm templates |
| GET | `/endpoint_groups` | List endpoint groups |
| POST | `/endpoint_groups` | Create endpoint group |
| GET/PUT/DELETE | `/endpoint_groups/{id}` | Manage endpoint group |

## Changes from v2.33.7

v2.39.0 adds these endpoints not present in v2.33.7:

| Endpoint | Description |
|----------|-------------|
| `GET /endpoints/{id}/kubernetes/helm/{name}` | Get a Helm release by name |
| `GET /kubernetes/{id}/metrics/pods/{namespace}/{name}` | Get metrics for a specific pod |
| `GET /kubernetes/{id}/namespaces` | List all namespaces (top-level) |
| `POST /kubernetes/{id}/nodes/{name}/drain` | Drain a Kubernetes node |
| `POST /registries/ping` | Test registry connectivity |

## Common workflows

### Deploy a standalone Docker Compose stack

```bash
curl -s -X POST "https://<portainer>/api/stacks/create/standalone/string?endpointId=1" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "my-stack",
    "stackFileContent": "version: \"3\"\nservices:\n  web:\n    image: nginx:latest\n    ports:\n      - \"8080:80\""
  }'
```

### Redeploy a stack from Git

```bash
curl -s -X PUT "https://<portainer>/api/stacks/{id}/git/redeploy?endpointId=1" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"pullImage": true, "repositoryReferenceName": "refs/heads/main"}'
```

### List running containers on an environment

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://<portainer>/api/endpoints/1/docker/containers/json"
```

### Create a user

```bash
curl -s -X POST "https://<portainer>/api/users" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"username":"newuser","password":"securePass123!","role":2}'
```

Role values: `1` = administrator, `2` = standard user.

### Drain a Kubernetes node (new in 2.39)

```bash
curl -s -X POST "https://<portainer>/api/kubernetes/{id}/nodes/{nodeName}/drain" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json"
```

### Test registry connectivity (new in 2.39)

```bash
curl -s -X POST "https://<portainer>/api/registries/ping" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"my-registry","url":"https://registry.example.com","type":3}'
```

## Important notes

- All API paths are prefixed with `/api` in practice (e.g., `https://portainer.example.com/api/auth`).
- Environment IDs are integers. Use `GET /endpoints` to discover them.
- Stack operations require `endpointId` as a query parameter.
- The Docker proxy (`/endpoints/{id}/docker/...`) follows the Docker Engine API — consult Docker docs for those request/response shapes.
- Pagination: `GET /endpoints` and similar list endpoints accept `start`, `limit`, and `search` query parameters.
- Filtering: Many list endpoints support filter query parameters. Check the OpenAPI spec for details.
- WebSocket endpoints (`/websocket/exec`, `/websocket/attach`, `/websocket/pod`, `/websocket/kubernetes-shell`) require a WebSocket upgrade handshake.
