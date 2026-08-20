# CockroachDB Geneos docker run lab

Private learning lab: start an insecure single-node CockroachDB, a Geneos Gateway in **demo mode**, a Self-Announcing Netprobe, and a Collection Agent using **`docker run` only**.

There is no Docker Compose file and no start/stop script. Type (or copy-paste) each command so you can see how the containers connect.

This stack is **insecure** and for learning or a quick test environment only. Do not use it in production.

## What you will run

```
Active Console  -->  Gateway :7039 (demo, insecure)
                        ^
                        | Self-Announcing Netprobe (insecure)
                        |
                   Netprobe :7036  <--- TCP reporter :9137 ---  Collection Agent
                                                                    |
                                                                    | HTTP :8080
                                                                    | JDBC :26257
                                                                    v
                                                              CockroachDB (insecure)
```

1. Collection Agent polls CockroachDB HTTP (`/_admin/v1/cluster`, nodes API) and JDBC (`crdb_internal.*`).
2. Collection Agent sends datapoints to the Netprobe on TCP port `9137`.
3. Netprobe self-announces to the Gateway on insecure port `7039`.
4. Gateway maps datapoints with the built-in **CockroachDB V1** mapping.
5. Active Console connects to the Gateway at `localhost:7039`.

Plugin metrics and mapping: [CockroachDB Collection Agent plugin](https://docs.itrsgroup.com/docs/geneos/collection/cockroachdb/current/user-guide/cockroachdb/index.html).

## Prerequisites

- Git
- Access to this **private** GitHub repo (`agalorio/cockroachdb-plugin-demo`)
- Docker Engine (or Docker Desktop) with permission to run `docker` commands
- `curl` (to wait for CockroachDB HTTP)
- Geneos **Active Console 7.11 or later** from [ITRS Downloads](https://docs.itrsgroup.com/docs/geneos/7.11.0/visualization/active-console/download-and-setup/index.html)
- An ITRS website login (same credentials are used for `docker.itrsgroup.com`)
- Free host ports: `7039` (Gateway), `7036` (Netprobe), `9137` (TCP reporter), `26257` (CockroachDB SQL), `8080` (CockroachDB HTTP)

Image versions are pinned in [`.env.example`](.env.example). Gateway and Netprobe must be **7.11.x or higher**. Collection Agent must be **6.6.x or higher**.

## 1. Clone or pull this repository

This repo is private. Sign in to GitHub as a user who has been granted access, then clone it once.

HTTPS:

```bash
# Downloads the lab files (README, .env.example, and config/) into a new directory.
git clone https://github.com/agalorio/cockroachdb-plugin-demo.git
cd cockroachdb-plugin-demo
```

SSH (if your GitHub account uses an SSH key):

```bash
git clone git@github.com:agalorio/cockroachdb-plugin-demo.git
cd cockroachdb-plugin-demo
```

If you already cloned the repo, update it instead of cloning again:

```bash
# From the repo root: fetch and merge the latest main branch.
cd cockroachdb-plugin-demo
git pull origin main
```

All later commands assume your current directory is the repo root.

## 2. Copy environment variables

From the repo root:

```bash
# Create a local .env from the template. .env is gitignored so you can change tags locally.
cp .env.example .env
```

Load the variables into **this same shell** so `${GATEWAY_IMAGE}` and the other names expand in later commands:

```bash
# Export every assignment in .env into the current shell.
set -a
source .env
set +a
```

If you open a new terminal, run `set -a && source .env && set +a` again before any `docker run`.

## 3. Log in to the ITRS Docker registry

Gateway, Netprobe, and Collection Agent images live on `docker.itrsgroup.com`. Use the same username and password as the ITRS website. If you do not have credentials, request them from the [ITRS Registration page](https://www.itrsgroup.com/).

```bash
# Prompts for ITRS website username and password.
docker login docker.itrsgroup.com
```

See [Geneos containers](https://docs.itrsgroup.com/docs/geneos/7.11.0/administration/orchestrated-environments/geneos_containers/index.html). CockroachDB images on Docker Hub do not need this login.

## 4. Create a Docker network

A user-defined bridge gives containers DNS names (`cockroach`, `gateway`, `netprobe`, `collection-agent`).

```bash
# Create the lab bridge. Safe to skip if it already exists (error: already exists).
docker network create "${NETWORK_NAME}"
```

## 5. Start CockroachDB (insecure, single-node)

```bash
docker run -d \
  --name "${COCKROACH_CONTAINER}" \
  --hostname "${COCKROACH_CONTAINER}" \
  --network "${NETWORK_NAME}" \
  -p "${COCKROACH_SQL_PORT}:26257" \
  -p "${COCKROACH_HTTP_PORT}:8080" \
  "${COCKROACH_IMAGE}" \
  start-single-node \
  --insecure
```

What each part does:

- `-d` runs the container in the background.
- `--name` / `--hostname` set the DNS name other containers use (`cockroach`).
- `--network` attaches the container to `geneos-lab`.
- `-p 26257:26257` publishes SQL (JDBC) to your host.
- `-p 8080:8080` publishes the HTTP API and DB Console to your host.
- `start-single-node --insecure` starts one node with no TLS and no password (lab only).

Wait until the cluster UUID is available. The Collection Agent **will not start** if `/_admin/v1/cluster` fails.

```bash
# Repeat until you see a JSON document with cluster_id / clusterID.
curl -s "http://localhost:${COCKROACH_HTTP_PORT}/_admin/v1/cluster"
```

Optional: open the CockroachDB DB Console at [http://localhost:8080](http://localhost:8080).

## 6. Start Gateway (insecure, demo mode)

Demo mode needs the Gateway name **Demo Gateway** and the `-demo` flag. Demo mode allows at most two Netprobes and you cannot rename the Gateway. See [Gateway licensing - demo mode](https://docs.itrsgroup.com/docs/geneos/7.11.0/administration/gateway_licensing/index.html).

```bash
docker run -d \
  --name "${GATEWAY_CONTAINER}" \
  --hostname "${GATEWAY_CONTAINER}" \
  --network "${NETWORK_NAME}" \
  -p "${GATEWAY_PORT}:7039" \
  -v "$(pwd)/config/gateway.setup.xml:/gateway/persist/setup/gateway.setup.xml:ro" \
  -e GATEWAY_CONFIG="-resources-dir /opt/gateway/resources -demo -setup /gateway/persist/setup/gateway.setup.xml" \
  "${GATEWAY_IMAGE}"
```

What each part does:

- `--name gateway` is the hostname the Netprobe uses in `netprobe.setup.xml`.
- `-p 7039:7039` publishes the insecure listen port for Active Console.
- `-v ...gateway.setup.xml` mounts [config/gateway.setup.xml](config/gateway.setup.xml) (CockroachDB V1 mapping, unmanaged CA port 9137, SAN enabled).
- `GATEWAY_CONFIG` ... `-demo` runs without a licence daemon.
- `GATEWAY_CONFIG` ... `-setup` points at the mounted setup file.

Check the log:

```bash
docker logs "${GATEWAY_CONTAINER}"
```

You should see the Gateway start in demo mode and listen on 7039.

## 7. Start Netprobe (insecure, self-announcing)

```bash
docker run -d \
  --name "${NETPROBE_CONTAINER}" \
  --hostname "${NETPROBE_CONTAINER}" \
  --network "${NETWORK_NAME}" \
  -p "${NETPROBE_PORT}:7036" \
  -p "${TCP_REPORTER_PORT}:9137" \
  -v "$(pwd)/config/netprobe.setup.xml:/netprobe/persist/setup/netprobe.setup.xml:ro" \
  -e NETPROBE_CONFIG="-port 7036 -setup /netprobe/persist/setup/netprobe.setup.xml" \
  "${NETPROBE_IMAGE}"
```

What each part does:

- `--name netprobe` is the hostname the Collection Agent TCP reporter uses.
- `-p 7036:7036` publishes the Netprobe listen port (debugging).
- `-p 9137:9137` publishes the unmanaged Collection Agent TCP reporter port.
- `-v ...netprobe.setup.xml` mounts [config/netprobe.setup.xml](config/netprobe.setup.xml) (SAN to `gateway:7039`, mapping type + CA params names).
- `NETPROBE_CONFIG` sets the listen port and setup file.

Check that it connected:

```bash
docker logs "${NETPROBE_CONTAINER}"
```

Look for a successful connection to Gateway `gateway:7039`.

## 8. Start Collection Agent (insecure, unmanaged)

The Collection Agent is a **separate container**. It is not started by the Netprobe. Gateway collection agent parameters are therefore **unmanaged**.

The Collection Agent image already includes the plugins that ship with the matching Netprobe Standard package (this lab: CA `6.6.3` with Netprobe `7.11.1`). Point `CA_PLUGIN_DIR` at `/app/plugins` inside the image. Do not copy JARs from Netprobe.

```bash
docker run -d \
  --name "${COLLECTION_AGENT_CONTAINER}" \
  --hostname "${COLLECTION_AGENT_CONTAINER}" \
  --network "${NETWORK_NAME}" \
  -v "$(pwd)/config/collection-agent.yml:/app/config/config.yaml:ro" \
  -e CA_PLUGIN_DIR=/app/plugins \
  -e HEALTH_CHECK_PORT="${CA_HEALTH_PORT}" \
  -e TCP_REPORTER_PORT="${TCP_REPORTER_PORT}" \
  -e COCKROACH_HOST="${COCKROACH_CONTAINER}" \
  "${COLLECTION_AGENT_IMAGE}" \
  java -jar /app/collection-agent.jar /app/config/config.yaml
```

What each part does:

- `-v ...collection-agent.yml` mounts [config/collection-agent.yml](config/collection-agent.yml) as the Collection Agent YAML.
- `CA_PLUGIN_DIR=/app/plugins` uses the plugin JARs already in the Collection Agent image (`CockroachDbCollector`, `GeneosProcessor`, and the rest of the Standard set).
- `TCP_REPORTER_PORT` must match Gateway `reporterPort` (9137).
- `COCKROACH_HOST` is `cockroach` so HTTP and JDBC URLs resolve on the Docker network.
- The command runs the Collection Agent JAR with that YAML (same layout as ITRS Collection Agent images).

If the image uses a different JAR path, check with:

```bash
docker run --rm --entrypoint ls "${COLLECTION_AGENT_IMAGE}" /app
```

Then adjust the `java -jar` path.

Check the log:

```bash
docker logs "${COLLECTION_AGENT_CONTAINER}"
```

You want a successful HTTP read of `/_admin/v1/cluster` and a TCP connection to `netprobe:9137`. There must be **no** `tls` errors (this lab omits the `tls` block for an insecure cluster).

## 9. Connect Active Console to the Gateway

1. Install Active Console 7.11+ from ITRS Resources > Downloads > Active Console 2. Unzip and start it. See [Download and setup](https://docs.itrsgroup.com/docs/geneos/7.11.0/visualization/active-console/download-and-setup/index.html).
2. Go to **Active Console > Workspace settings > Connections**.
3. Click **Add**.
4. Hostname: `localhost` if Active Console is on the same machine as Docker. Otherwise use the Docker host IP or hostname.
5. Port: `7039` (insecure Gateway listen port from `.env`).
6. Click **Apply**, then **OK**.
7. The Gateway appears in the Gateways dockable as **Demo Gateway**.

You should then see:

- Probe `cockroachdb-probe` (Self-Announcing)
- A dynamic managed entity named with the CockroachDB **cluster UUID** (from `/_admin/v1/cluster`)
- Node dataviews such as `node_1` from the CockroachDB V1 mapping

Double-click the Gateway to open Gateway Setup Editor if you want to inspect [config/gateway.setup.xml](config/gateway.setup.xml).

## 10. Metrics you should see

Each collection cycle publishes HTTP node/store metrics and SQL side-channel metrics. Full list: [Metrics collected](https://docs.itrsgroup.com/docs/geneos/collection/cockroachdb/current/user-guide/cockroachdb/index.html).

HTTP Status and Admin API:

- `status` per node (`LIVE`, `DEAD`, and other liveness values)
- Allowlisted **node** gauges (essential metrics / alerts)
- Allowlisted **store** gauges, with a `store` dimension

SQL side-channel:

- Sessions: `count` per `(user_name, application_name)`
- Nonterminal jobs: `status`, `job_type`, `description`, `created`, `last_run`, `age`, `num_runs`

This lab does not configure custom JDBC queries. That is a separate JDBC Collection Agent plugin.

## 11. Useful docker commands (no scripts)

List the four containers:

```bash
docker ps --filter "network=${NETWORK_NAME}"
```

Follow a log:

```bash
docker logs -f "${GATEWAY_CONTAINER}"
docker logs -f "${NETPROBE_CONTAINER}"
docker logs -f "${COLLECTION_AGENT_CONTAINER}"
docker logs -f "${COCKROACH_CONTAINER}"
```

## 12. Tear down

Stop and remove the lab containers, then the network.

```bash
docker stop "${COLLECTION_AGENT_CONTAINER}" "${NETPROBE_CONTAINER}" "${GATEWAY_CONTAINER}" "${COCKROACH_CONTAINER}"
docker rm "${COLLECTION_AGENT_CONTAINER}" "${NETPROBE_CONTAINER}" "${GATEWAY_CONTAINER}" "${COCKROACH_CONTAINER}"
docker network rm "${NETWORK_NAME}"
```

## Troubleshooting

| Symptom | What to check |
| --- | --- |
| `docker pull` / `docker run` unauthorized for `docker.itrsgroup.com` | Run `docker login docker.itrsgroup.com` with ITRS website credentials. |
| Collection Agent exits immediately | `curl` the cluster UUID (step 5). Keep the CA image tag in line with Netprobe 7.11.1 so `/app/plugins` includes CockroachDB. Read `docker logs collection-agent`. |
| `Failed to create directory '/app/./Workflow/...'` | `/app` is not writable by user `java`. `workflow.storeDirectory` must be `/tmp/collection-agent-store` (already set in `collection-agent.yml`). Recreate the Collection Agent container after changing the YAML. |
| No dynamic entities in Active Console | Confirm SAN in Netprobe logs, unmanaged reporter `9137`, and built-in mapping **CockroachDB V1** on Gateway 7.11+. |
| Active Console cannot connect | Gateway published `7039`, Gateway name is still `Demo Gateway`, and you are using the insecure port (not 7038). |
| `port is already allocated` | Another process is using that host port. Change the left-hand side of `-p` in `.env` / the `docker run` (keep the container-side ports). |
| TCP reporter connection refused | Start Netprobe **before** Collection Agent. Reporter hostname must be `netprobe` on `geneos-lab`. |
| Forgot to `source .env` | Image variables are empty. Run `set -a && source .env && set +a` in the current shell. |

## Files in this repo

| File | Role |
| --- | --- |
| [`.env.example`](.env.example) | Image tags, names, and ports. Copy to `.env`. |
| [`config/gateway.setup.xml`](config/gateway.setup.xml) | Demo Gateway, SAN, CockroachDB V1 mapping, unmanaged CA port 9137. |
| [`config/netprobe.setup.xml`](config/netprobe.setup.xml) | Self-announce to `gateway:7039`. |
| [`config/collection-agent.yml`](config/collection-agent.yml) | Insecure `CockroachDbCollector` + TCP reporter to `netprobe`. |

## Official documentation

- [CockroachDB Collection Agent plugin](https://docs.itrsgroup.com/docs/geneos/collection/cockroachdb/current/user-guide/cockroachdb/index.html)
- [Geneos containers](https://docs.itrsgroup.com/docs/geneos/7.11.0/administration/orchestrated-environments/geneos_containers/index.html)
- [Docker Images release notes](https://docs.itrsgroup.com/docs/all/geneos/deployments/docker-images/index.html)
- [Run Collection Agent as a standalone Java process](https://docs.itrsgroup.com/docs/geneos/7.11.0/collection/collection-agent/run-ca-standalone-java/index.html)
- [Dynamic Entities](https://docs.itrsgroup.com/docs/geneos/7.11.0/processing/configuration/dynamic_entities/index.html)
- [Active Console download and setup](https://docs.itrsgroup.com/docs/geneos/7.11.0/visualization/active-console/download-and-setup/index.html)
- [Gateway demo mode](https://docs.itrsgroup.com/docs/geneos/7.11.0/administration/gateway_licensing/index.html)
- [CockroachDB Docker (insecure)](https://www.cockroachlabs.com/docs/stable/start-a-local-cluster-in-docker-linux)
