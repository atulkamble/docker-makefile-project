Awesome — here’s a clean, ready-to-use **Docker project with a Makefile** so you can build, run, test, and ship with single commands. Copy this structure into a repo and you’re good to go.

---

# Project layout

```
docker-makefile-project/
├─ Makefile
├─ Dockerfile
├─ docker-compose.yml
├─ .dockerignore
├─ .env.example
├─ README.md
├─ app/
│  ├─ app.py
│  └─ requirements.txt
└─ tests/
   └─ test_health.sh
```

---

# Makefile

```makefile
# ====== Config ======
APP_NAME       ?= docker-makefile-project
IMAGE          ?= $(APP_NAME)
REGISTRY       ?=                    # e.g. ghcr.io/youruser or your-dockerhub-user
TAG            ?= latest
IMAGE_REF      := $(if $(REGISTRY),$(REGISTRY)/$(IMAGE):$(TAG),$(IMAGE):$(TAG))

PORT           ?= 8080
CONTAINER_NAME ?= $(APP_NAME)
COMPOSE_FILE   ?= docker-compose.yml

PYTEST_CMD     ?= echo "⏭️ Add real tests or run ./tests/test_health.sh"
SHELL_CMD      ?= sh                # use bash if your base image has it

# ====== Helpers ======
.SILENT:
.ONESHELL:
MAKEFLAGS += --no-builtin-rules
.DEFAULT_GOAL := help

# Detect if docker is available
DOCKER := $(shell command -v docker 2>/dev/null)

# ====== Targets ======
.PHONY: help
help: ## Show this help
	@grep -E '^[a-zA-Z0-9_%-]+:.*?## ' Makefile | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "  \033[36m%-20s\033[0m %s\n", $$1, $$2}'

# --- Image lifecycle ---
.PHONY: build
build: ## Build local image
	$(DOCKER) build -t $(IMAGE_REF) .

.PHONY: rebuild
rebuild: ## Rebuild image without cache
	$(DOCKER) build --no-cache -t $(IMAGE_REF) .

.PHONY: tag
tag: ## Tag image to a new TAG (use TAG=newtag)
	@if [ -z "$(TAG)" ]; then echo "TAG is required"; exit 1; fi
	$(DOCKER) tag $(IMAGE):latest $(IMAGE_REF)

.PHONY: push
push: ## Push image to registry (set REGISTRY=your-registry)
	@if [ -z "$(REGISTRY)" ]; then echo "Set REGISTRY to push (e.g. REGISTRY=ghcr.io/you)"; exit 1; fi
	$(DOCKER) push $(IMAGE_REF)

# --- Run bare container ---
.PHONY: run
run: ## Run container (detached) mapping $(PORT)
	$(DOCKER) run -d --rm --name $(CONTAINER_NAME) -p $(PORT):8080 $(IMAGE_REF)

.PHONY: stop
stop: ## Stop running container
	-$(DOCKER) stop $(CONTAINER_NAME)

.PHONY: logs
logs: ## Tail container logs
	$(DOCKER) logs -f $(CONTAINER_NAME)

.PHONY: sh
sh: ## Open a shell inside the running container
	$(DOCKER) exec -it $(CONTAINER_NAME) $(SHELL_CMD)

# --- Docker Compose (single service) ---
.PHONY: up
up: ## docker-compose up -d
	DOCKER_DEFAULT_PLATFORM= $(DOCKER) compose -f $(COMPOSE_FILE) --env-file .env up -d --build

.PHONY: down
down: ## docker-compose down
	$(DOCKER) compose -f $(COMPOSE_FILE) --env-file .env down

.PHONY: ps
ps: ## docker-compose ps
	$(DOCKER) compose -f $(COMPOSE_FILE) --env-file .env ps

.PHONY: compose-logs
compose-logs: ## docker-compose logs -f
	$(DOCKER) compose -f $(COMPOSE_FILE) --env-file .env logs -f

# --- Test ---
.PHONY: test
test: ## Run tests
	$(PYTEST_CMD)

.PHONY: test-health
test-health: ## Curl /health of the running service
	./tests/test_health.sh $(PORT)

# --- Clean ---
.PHONY: clean
clean: ## Remove dangling images/containers
	-$(DOCKER) rm -f $(CONTAINER_NAME) 2>/dev/null || true
	-$(DOCKER) image prune -f
	-$(DOCKER) builder prune -f

# --- Optional: Multi-arch via buildx ---
.PHONY: buildx
buildx: ## Build multi-arch image (linux/amd64, linux/arm64) and load
	$(DOCKER) buildx create --use 2>/dev/null || true
	$(DOCKER) buildx build --platform linux/amd64,linux/arm64 -t $(IMAGE_REF) --load .

.PHONY: release
release: ## Buildx and push (requires REGISTRY)
	@if [ -z "$(REGISTRY)" ]; then echo "Set REGISTRY to push"; exit 1; fi
	$(DOCKER) buildx create --use 2>/dev/null || true
	$(DOCKER) buildx build --platform linux/amd64,linux/arm64 -t $(IMAGE_REF) --push .
```

---

# Dockerfile

```dockerfile
# Tiny Python web service (Flask) listening on 8080
FROM python:3.12-slim

ENV PYTHONUNBUFFERED=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

WORKDIR /app

# System deps (optional) and security best practices
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl ca-certificates && \
    rm -rf /var/lib/apt/lists/*

COPY app/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app/ ./app
WORKDIR /app

EXPOSE 8080
HEALTHCHECK --interval=10s --timeout=3s --retries=3 CMD curl -fsS http://localhost:8080/health || exit 1

CMD ["python", "app.py"]
```

---

# docker-compose.yml

```yaml
services:
  web:
    build: .
    image: ${REGISTRY:-}${REGISTRY:+/}${IMAGE:-docker-makefile-project}:${TAG:-latest}
    container_name: ${CONTAINER_NAME:-docker-makefile-project}
    ports:
      - "${PORT:-8080}:8080"
    environment:
      - APP_ENV=${APP_ENV:-dev}
    healthcheck:
      test: ["CMD-SHELL", "curl -fsS http://localhost:8080/health || exit 1"]
      interval: 10s
      timeout: 3s
      retries: 3
```

---

# .dockerignore

```
.git
.gitignore
__pycache__/
*.pyc
*.pyo
*.pyd
*.log
*.sqlite
.env
venv/
.env.*
.coverage
dist/
build/
Dockerfile*
README.md
tests/
```

---

# .env.example

```
# Copy to .env and adjust
REGISTRY=
IMAGE=docker-makefile-project
TAG=latest
CONTAINER_NAME=docker-makefile-project
PORT=8080
APP_ENV=dev
```

---

# app/requirements.txt

```
flask==3.0.3
```

---

# app/app.py

```python
from flask import Flask, jsonify

app = Flask(__name__)

@app.get("/")
def root():
    return jsonify(message="Hello from Flask + Docker + Makefile!", path="/")

@app.get("/health")
def health():
    return jsonify(status="ok")

if __name__ == "__main__":
    # Bind to 0.0.0.0:8080 for Docker
    app.run(host="0.0.0.0", port=8080)
```

---

# tests/test\_health.sh

```bash
#!/usr/bin/env bash
set -euo pipefail

PORT="${1:-8080}"

echo "Checking http://localhost:${PORT}/health ..."
status=$(curl -fsS "http://localhost:${PORT}/health" | tr -d '\n')
echo "Response: $status"
echo "✅ Health OK"
```

> Make it executable:

```bash
chmod +x tests/test_health.sh
```

---

# README.md (quick start)

````markdown
# Docker + Makefile Starter

One-command Docker workflows using a clean Makefile.

## Prereqs
- Docker (and optionally Buildx)
- Make

## First run
```bash
cp .env.example .env         # optional but recommended
make build                   # build image
make run                     # run container
open http://localhost:8080   # (macOS) or use your browser
````

### Using Docker Compose

```bash
make up
make ps
make compose-logs
make down
```

### Useful commands

```bash
make help            # list targets
make rebuild         # no-cache build
make logs            # tail container logs
make sh              # shell into running container
make test-health     # curl /health
make clean           # prune
```

### Push to a registry

```bash
make tag TAG=v1
make push REGISTRY=ghcr.io/<your-user>
# or multi-arch:
make release REGISTRY=ghcr.io/<your-user> TAG=v1
```

### Customizing

* Change `IMAGE`, `TAG`, `REGISTRY` in `.env`.
* Replace `app/` with your application code.

```

---

If you want this scaffold tailored for a **multi-service stack** (e.g., web + Redis + Postgres) or wired into **GitHub Actions** for CI/CD and release tags, say the word and I’ll extend it.
```
