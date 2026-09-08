# Day 1 — AI Gateway Runtime Foundation

## Objective

Build a reproducible AI Gateway development environment using:

- GitHub Codespaces
- Docker
- Docker Compose
- LiteLLM
- External configuration
- Health-check validation

We intentionally do **not** install LiteLLM directly on the local machine.

---

## 1. Target Architecture

    GitHub Repository
           |
           v
    GitHub Codespace
           |
           v
    Docker / Docker Compose
           |
           v
    LiteLLM v1.100.0
           |
           +---- config.yaml
           |
           v
    HTTP :4000
           |
           v
    /health/liveliness
           |
           v
    "I'm alive!"

### Configuration Flow

    Host
    /workspaces/.../docker/config.yaml
             |
             | Read-only bind mount
             v
    Container
    /app/config.yaml

---

## 2. Step 1 — Create / Use GitHub Repository

Repository:

    LLM_Gateway

Recommended structure:

    LLM_Gateway/
    |
    +-- docker/
    |   +-- docker-compose.yml
    |   +-- config.yaml
    |
    +-- config/
    +-- docs/
    +-- experiments/
    +-- scripts/
    +-- week-01/
    |
    +-- requirements.txt
    +-- README.md

### Learning

Separate:

- Application code
- Infrastructure configuration
- Runtime configuration

Docker-related files belong under `docker/`.

---

## 3. Step 2 — Create GitHub Codespace

From GitHub:

    Repository
        |
        v
    Code
        |
        v
    Codespaces
        |
        v
    Create codespace on main

Once inside the Codespace:

    pwd

The workspace path may differ depending on the repository name.

### Learning

Codespaces provides the development environment.

Docker provides runtime isolation.

Therefore:

    Development Environment
            |
            v
    GitHub Codespace

    Application Runtime
            |
            v
    Docker Container

---

## 4. Step 3 — Verify Docker

Check Docker:

    docker --version

Check Docker Compose:

    docker compose version

Check Docker engine:

    docker info

### Learning

Follow this engineering sequence:

    Inspect
       |
       v
    Verify
       |
       v
    Configure
       |
       v
    Run

Do not install software before checking whether the environment already provides it.

---

## 5. Step 4 — Create Docker Directory

From the repository root:

    mkdir -p docker

Verify:

    ls -la docker

---

## 6. Step 5 — Create LiteLLM Configuration

Create:

    docker/config.yaml

Initial configuration:

    model_list: []

Verify:

    cat docker/config.yaml

Expected:

    model_list: []

### Why is `model_list` empty?

We are intentionally not configuring an LLM provider yet.

Today's objective is to validate the gateway infrastructure.

The progression is:

    Gateway Infrastructure
            |
            v
    Provider Configuration
            |
            v
    Actual LLM Request
            |
            v
    Multiple Providers
            |
            v
    Fallback

---

## 7. Step 6 — Create Docker Compose Configuration

Create:

    docker/docker-compose.yml

Use the following configuration:

    services:
      litellm:
        image: ghcr.io/berriai/litellm:1.100.0

        ports:
          - "4000:4000"

        command:
          - "--config"
          - "/app/config.yaml"
          - "--port"
          - "4000"

        volumes:
          - ${PWD}/docker/config.yaml:/app/config.yaml:ro

---

## 8. Understanding docker-compose.yml

### LiteLLM Image

    image: ghcr.io/berriai/litellm:1.100.0

This tells Docker to use LiteLLM version `1.100.0`.

### Why pin the version?

Avoid:

    image: ghcr.io/berriai/litellm:latest

because `latest` can change over time.

Version pinning gives us:

    Same Configuration
            +
    Same Image Version
            =
    More Reproducible Environment

---

## 9. Port Mapping

Configuration:

    ports:
      - "4000:4000"

Meaning:

    Codespace Host
          |
          | :4000
          v
    Docker Container
          |
          | :4000
          v
       LiteLLM

Therefore LiteLLM is accessible through:

    http://localhost:4000

---

## 10. LiteLLM Command

Configuration:

    command:
      - "--config"
      - "/app/config.yaml"
      - "--port"
      - "4000"

This tells LiteLLM:

1. Use `/app/config.yaml`
2. Listen on port `4000`

---

## 11. Configuration Mount

Configuration:

    volumes:
      - ${PWD}/docker/config.yaml:/app/config.yaml:ro

Conceptually:

    HOST

    /workspaces/.../docker/config.yaml
                  |
                  | Bind Mount
                  v
    CONTAINER

    /app/config.yaml

The final parameter:

    :ro

means:

    Read Only

### Why use read-only?

The container should consume the configuration rather than modify the source configuration.

This is a useful security and operational practice.

---

## 12. Step 7 — Validate Configuration File

Check the file:

    cat docker/config.yaml

Expected:

    model_list: []

Check its type:

    ls -l docker/config.yaml

The path must represent a regular file.

### Important Troubleshooting Lesson

A Docker bind mount can behave unexpectedly if the host source path does not exist or is interpreted as a directory.

Therefore always verify the host file before troubleshooting the application.

---

## 13. Step 8 — Start LiteLLM

From the repository root:

    docker compose -f docker/docker-compose.yml up -d

Check the container:

    docker compose -f docker/docker-compose.yml ps

Expected state:

    litellm    running

---

## 14. Step 9 — Check Logs

View logs:

    docker compose -f docker/docker-compose.yml logs

Or follow logs:

    docker compose -f docker/docker-compose.yml logs -f

Exit log-follow mode:

    Ctrl + C

### Learning

A container being `Up` does not automatically prove that the application is healthy.

We need to validate:

    Container Running
           |
           v
    Application Running
           |
           v
    Application Responding

---

## 15. Step 10 — Verify the Configuration Mount

Find the container ID:

    docker compose -f docker/docker-compose.yml ps -q litellm

Inspect the container:

    docker inspect $(docker compose -f docker/docker-compose.yml ps -q litellm)

Look for the mount:

    Source:
    .../docker/config.yaml

    Destination:
    /app/config.yaml

    RW:
    false

The important relationship is:

    Host config.yaml
           |
           v
    /app/config.yaml
           |
           v
    Read-only

---

## 16. Step 11 — Health Check

Run:

    curl http://localhost:4000/health/liveliness

Expected:

    "I'm alive!"

This is the actual infrastructure validation.

---

## 17. Step 12 — Validate the Architecture

At this point the complete architecture is:

    GitHub
       |
       v
    GitHub Codespace
       |
       v
    Docker Engine
       |
       v
    Docker Compose
       |
       v
    LiteLLM v1.100.0
       |
       v
    Port 4000
       |
       v
    /health/liveliness
       |
       v
    "I'm alive!"

Configuration:

    docker/config.yaml
           |
           | Read-only Bind Mount
           v
    /app/config.yaml

---

## 18. Step 13 — Git Safety Check

Before committing:

    git status

Make sure there are:

- No API keys
- No passwords
- No tokens
- No credentials

Today's configuration contains only:

    model_list: []

Therefore no provider credentials are required.

---

## 19. Step 14 — Commit the Infrastructure

Stage the Docker configuration:

    git add docker/

Commit:

    git commit -m "Build LiteLLM gateway development environment"

Push:

    git push origin main

Now the infrastructure configuration is stored in Git.

---

## 20. Reproducibility

Another Codespace can recreate the environment using:

    git clone <repository>

Then:

    cd LLM_Gateway

Then:

    docker compose -f docker/docker-compose.yml up -d

The goal is:

    Code
      +
    Configuration
      +
    Pinned Runtime
      =
    Reproducible Environment

---

# 21. What I Learned Today

## Development Environment

GitHub Codespaces provides an isolated cloud development environment.

## Runtime Isolation

Docker runs LiteLLM independently from the host operating system.

## Orchestration

Docker Compose defines:

- Container
- Image
- Ports
- Commands
- Configuration mounts

## Configuration Management

LiteLLM configuration is externalized into:

    docker/config.yaml

and mounted into the container.

## Security

The configuration is mounted read-only:

    :ro

Provider credentials should not be hard-coded into Git.

## Validation

The container being `Up` is not enough.

The actual application must respond successfully:

    curl http://localhost:4000/health/liveliness

Expected:

    "I'm alive!"

---

# 22. Today's Engineering Principles

## Principle 1 — Inspect Before Installing

    Inspect
       |
       v
    Verify
       |
       v
    Configure
       |
       v
    Run

## Principle 2 — Pin Infrastructure Versions

Prefer:

    LiteLLM 1.100.0

over:

    latest

## Principle 3 — Separate Configuration

    Container Image
           +
    External Configuration

## Principle 4 — Minimize Privileges

Use:

    Read-only Configuration Mount

where appropriate.

## Principle 5 — Validate Behavior, Not Just Process State

    Container Up
         !=
    Application Healthy

Health checks provide stronger validation.

---

# 23. Definition of Done

| Component | Status |
|---|---|
| GitHub Repository | Complete |
| GitHub Codespace | Complete |
| Docker | Complete |
| Docker Compose | Complete |
| LiteLLM | Complete |
| LiteLLM version pinned | Complete — `1.100.0` |
| External configuration | Complete |
| Read-only configuration mount | Complete |
| Container starts | Complete |
| Health endpoint responds | Complete |
| Provider configured | Pending |
| Actual LLM request | Pending |
| Second provider | Pending |
| Provider fallback | Pending |

---

# 24. Milestone

## Milestone 1 — AI Gateway Runtime Foundation

**Status: COMPLETE**

We successfully established:

    GitHub Codespace
           |
           v
        Docker
           |
           v
    Docker Compose
           |
           v
        LiteLLM
           |
           v
       HTTP :4000
           |
           v
      Health Check
           |
           v
      "I'm alive!"

---

# 25. Important Troubleshooting Lesson

During the setup, the container initially failed because the LiteLLM configuration contained:

    model_lis: []

The correct configuration key is:

    model_list: []

The troubleshooting process demonstrated an important engineering practice:

    Application Fails
           |
           v
    Inspect Container
           |
           v
    Inspect Mounted File
           |
           v
    Verify Actual Contents
           |
           v
    Identify Configuration Error
           |
           v
    Fix
           |
           v
    Restart
           |
           v
    Health Check

Do not assume that the host configuration is correct simply because the file exists.

---

# 26. Next Milestone

The next step is not rate limiting, APISIX, agents, or LangGraph.

First prove the complete request path:

    Application
         |
         v
    LiteLLM Gateway
         |
         v
    LLM Provider
         |
         v
    LLM Model
         |
         v
      Response

## Milestone 2

    Configure ONE Provider
            |
            v
    Send ONE Real LLM Request
            |
            v
    Verify Response
            |
            v
    Understand Complete Request Path

Then progress toward:

    One Provider
         |
         v
    Second Provider
         |
         v
    Routing
         |
         v
    Fallback
         |
         v
    Rate / Token Governance
         |
         v
    Observability
         |
         v
    Failure Injection

This provides the correct progression from:

**Basic Gateway Infrastructure → Production-Grade AI Traffic Management**