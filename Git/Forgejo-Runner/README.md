# Forgejo-Runner

The executable that runs **Forgejo Actions**, wrapped in a Docker compose stack.

## What this is

Forgejo (the self-hosted git server) does not run CI itself. It only schedules jobs. The actual work — checking out a repo, running the steps of a workflow, building a Docker image, pushing it to the registry — is done by a separate program called the **Forgejo Runner**.

The runner appears in the Forgejo web UI as **Actions → Nodes** (English: *Runners*). It fetches workflows from `.forgejo/workflows/*.yml` and executes them.

## Why we need it

A project (e.g. an application tracker) builds its own Docker image in CI and pushes it to the Forgejo **container registry**, so Portainer can later pull `latest` and deploy it.

Without a runner, the workflow files run nowhere: the "Actions" tab shows a failing/queued run that finishes in 0s and nothing is ever published. That is the state you are in before adding this service.

## How the pieces fit together

```
Forgejo (git + registry + job scheduler)
    │  (Actions service)
    ▼
Forgejo Runner  ──────────────►  Docker-in-Docker (dind)
   executes workflows              isolated Docker daemon that
   each job in its own             builds/pushes the images
   container
```

- `docker-in-docker` (`docker:dind`) is a private Docker daemon. The runner uses it to run each job in an isolated container and to perform `docker build` / `docker push` inside jobs. It **never** touches the host `/var/run/docker.sock`, so a compromised job cannot affect other services on the server.
- `runner` is the Forgejo Runner daemon. It connects back to the Forgejo instance using the node credentials you created in the UI, polls for jobs, and reports logs/status back.

## Secrets (important)

The runner authenticates to Forgejo with a node **uuid + token**. Those are secrets and must **never** be committed:

| File | Where it lives | Contents |
|------|----------------|----------|
| `runner-config.yml` | `$PATH_TO_SECRETS/Forgejo-Runner/` | REAL uuid + token + labels |
| `runner-config.yml.example` | this repo (public) | template with `CHANGE_ME` placeholders only |

The Compose file mounts the real config read-only into the container from `$PATH_TO_SECRETS/Forgejo-Runner/runner-config.yml`. Because it lives outside the repo (in the git-ignored secrets folder), the public repo stays clean.

## One-time setup

Forgejo Actions is enabled by default (`[actions] ENABLED = true`), so the "Actions" tab already appears. The only thing you must create is the **node** (runner identity):

1. Log into Forgejo (as the owner user).
2. Go to **Actions → Nodes** (user level; or site admin → same section).
3. Click **Create new node**, enter a **Name** (e.g. `server`) and optionally a **Description**, and confirm.
4. Forgejo then shows the node **UUID** and a confidential **Token**. Copy both — you will only see the token once.
5. Create `$PATH_TO_SECRETS/Forgejo-Runner/runner-config.yml` from the `.example` template and fill in `server.connections.forgejo.url` (your instance URL) and the `uuid` / `token` values.

> If you lose the token, use **Reset registration token** on the node and copy the new value.

## How job containers reach Docker (DIND)

Forgejo Runner runs each job inside a job container. For workflow steps that use Docker (`docker/setup-buildx-action`, etc.) to reach the isolated DIND daemon, the runner config needs **two things** (already present in `runner-config.yml.example`):

1. `runner.envs.DOCKER_HOST` — an environment variable injected into every job container so the `docker` CLI inside it knows which daemon to talk to. Its value must match the hostname set up in step 2.
2. `container.options` — an extra `--add-host=...:host-gateway` so that hostname resolves to the DIND container address from inside the job container (each job gets its own temporary network, so it cannot rely on the compose service name).

Without these, a `docker` command inside a job looks for `/var/run/docker.sock`, finds nothing, and fails with `failed to connect to the docker API at unix:///var/run/docker.sock`. That is exactly the failure seen in the `Set up Buildx` step before this was configured.

## Deploy

This stack is a normal repository compose, deployed on the server via Portainer (git repository method), like the rest of the setup.

- It joins the shared external network `apps-net`.
- `runner` reads its config from the secrets volume (mounted read-only).
- `runner` keeps its persistent identity/cache under `$PATH_TO_CONTAINERS/Forgejo-Runner/runner`.

Prerequisites before first `docker compose up`:

```bash
# persistent runner data dir (owned by uid 1001 = the runner container user)
mkdir -p "${PATH_TO_CONTAINERS}/Forgejo-Runner/runner"
chown 1001:1001 "${PATH_TO_CONTAINERS}/Forgejo-Runner/runner"
```

Then deploy the stack (Portainer → Stacks → Add stack → Repository → this repo, path `Git/Forgejo-Runner`).

### After editing `runner-config.yml`

The secrets config is mounted read-only at runtime; the container reads it **only once at startup**. So after editing `$PATH_TO_SECRETS/Forgejo-Runner/runner-config.yml`, you must restart the stack (Portainer → stack → Update, or restart the `runner` container) for the change to take effect.

You do **not** need to manage the `docker-in-docker` container separately — it is part of the same stack and the compose already declares `depends_on: docker-in-docker: condition: service_started`, so updating the stack always starts DIND before the runner.

## Verify it works

1. In Portainer, open the `runner` container logs. You should see `Starting runner daemon` and no errors fetching jobs.
2. In Forgejo → **Actions → Nodes**, the node should now appear **online/active** instead of idle/unknown.
3. Trigger the `Publish image` workflow in the target repo (push to the configured branch or *Run workflow*). This time the job executes and, on success, the image appears under **Packages → Containers**.
4. Only then deploy the app in Portainer (it pulls the image just published).

## Why not share the host Docker socket?

The simple alternative is to give the runner `docker_host: automount` (the host `/var/run/docker.sock`). That is less setup but a security risk: any Actions job that can run `docker` would control the whole host daemon and every container on it. Docker-in-Docker keeps CI isolated, at the cost of one extra container. Given the workflows build and push images automatically, isolation is the safer choice.

## References

- Forgejo Actions admin guide (v16.0): https://forgejo.org/docs/v16.0/admin/actions/
- Runner registration: https://forgejo.org/docs/v16.0/admin/actions/registration/
- Runner installation with Docker: https://forgejo.org/docs/v16.0/admin/actions/installation/docker/
- Docker access within Actions (dind vs socket): https://forgejo.org/docs/v16.0/admin/actions/docker-access/
