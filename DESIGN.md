# ephem

**Declarative ephemeral environments with a machine-readable API.**

Define your stack in Nix, spin up isolated copies per branch, and let both humans and AI agents interact with them programmatically. Docker-compose for the age of agentic development.

---

## The Problem

Modern development and AI agent workflows need isolated, reproducible copies of multi-service environments. Today's options fall short:

- **docker-compose** binds ports once. You can't run two copies of your stack without manual port remapping. No real isolation — containers share a kernel and network namespace. Builds are layer-cached, not content-addressed, so reproducibility is best-effort.
- **Kubernetes (namespaces, vcluster)** solves isolation but carries enormous operational overhead. Not practical for a developer laptop or for lightweight agent workflows.
- **AI agents** have no structured way to observe, interact with, or verify running environments. They're stuck parsing logs and screenshots rather than querying machine-readable APIs.

## The Solution

`ephem` is an environment manager that:

1. Takes a **Nix-based service definition** (postgres, valkey, your app services — anything NixOS can run)
2. Builds a **deterministic, cacheable NixOS VM image** from that definition
3. Boots it in a **lightweight VM** (QEMU with hardware acceleration on both macOS and Linux)
4. Exposes a **structured API** for both human CLIs and AI agents to interact with the running environment
5. Supports **multiple concurrent environments** mapped to git branches, with full network isolation

---

## Architecture

```
┌─────────────────────────────────────────────────┐
│                 Orchestrator API                 │
│           (gRPC/HTTP — for Conductor,            │
│           AI agents, CI pipelines)               │
├──────────────────┬──────────────────────────────┤
│    CLI (ephem)   │        MCP Server            │
│  (for humans)    │  (for LLM tool calling)      │
├──────────────────┴──────────────────────────────┤
│              Environment Manager                 │
│     branch tracking, networking, state,          │
│     health checks, port forwarding               │
├─────────────────────────────────────────────────┤
│              Nix Image Builder                   │
│    flake eval → NixOS VM image,                  │
│    service module resolution,                    │
│    binary cache, incremental rebuilds            │
├─────────────────────────────────────────────────┤
│              VM Runtime (thin)                   │
│    QEMU process lifecycle, networking            │
│    ~200 lines, swappable backend                 │
│    macOS: HVF accel | Linux: KVM accel           │
└─────────────────────────────────────────────────┘
```

### Why QEMU?

- Works on **both macOS and Linux** with hardware-accelerated virtualization (HVF / KVM)
- Same CLI and disk image format on both platforms
- Installable as a Nix derivation — no external dependencies
- Boot times of a few seconds for a minimal NixOS image on Apple Silicon
- The VM layer is intentionally thin (~200 lines) and swappable if a better runtime emerges

### Why NixOS inside the VM?

- NixOS is already a **declarative service composition system** with well-maintained modules for postgres, redis/valkey, nginx, and dozens of other services
- **Content-addressed builds** mean only changed derivations rebuild. Change one Python dependency and only that derivation is rebuilt — postgres, valkey, and your frontend are cached
- **Deterministic and diffable.** Two environments built from the same Nix expression are byte-identical. You can diff environments at the derivation level before building them
- The entire system (kernel, services, config, dependencies) is defined in one place and version-controlled

---

## Service Definition

Users define their stack in a Nix file. `ephem` provides convenience modules on top of standard NixOS modules.

### Example: `ephem.nix`

```nix
{
  # VM resource allocation
  environment = {
    memory = 2048;  # MB
    cpus = 2;
  };

  # Well-known services — these map to NixOS modules directly
  services.postgresql = {
    enable = true;
    initialScript = ./migrations/init.sql;
  };

  services.valkey = {
    enable = true;
  };

  # Application services — generic "process" type
  services.backend = {
    type = "process";
    command = "python -m uvicorn main:app --host 0.0.0.0 --port 8080";
    workdir = ./backend;
    env = {
      DATABASE_URL = "postgres://localhost:5432/app";
      VALKEY_URL = "valkey://localhost:6379";
    };
    ports = [ 8080 ];
    healthcheck = "http://localhost:8080/health";
  };

  services.frontend = {
    type = "process";
    command = "deno run --allow-net --allow-env main.ts";
    workdir = ./frontend;
    env.API_URL = "http://localhost:8080";
    ports = [ 3000 ];
  };
}
```

### Design Principles

- **NixOS modules are the foundation.** Postgres, valkey, nginx, etc. are already NixOS modules with years of production hardening. `ephem` doesn't reinvent service definitions — it wraps them with convenience and lifecycle management.
- **`type = "process"` is the escape hatch.** Any service that isn't a well-known NixOS module can be run as a supervised process. This covers custom backends, frontends, and anything else.
- **All services run inside a single VM.** Inter-service communication is just localhost. This keeps things simple, avoids service mesh complexity, and maps naturally to how developers think about their local stack.

---

## CLI Interface

```bash
# Lifecycle
ephem up                         # start environment for current git branch
ephem up feature/new-auth        # start environment for a specific branch
ephem down                       # stop current branch's environment
ephem down feature/new-auth      # stop a specific environment
ephem restart                    # rebuild and restart current environment
ephem destroy --all              # tear down all environments

# Inspection
ephem status                     # show all running environments
# BRANCH              STATUS    MEMORY   SERVICES                  PORTS
# main                running   1.8 GB   postgres, valkey, backend  3000, 8080
# feature/new-auth    running   1.8 GB   postgres, valkey, backend  3000, 8080

ephem logs                       # tail logs for current environment
ephem logs backend               # tail logs for a specific service
ephem logs -f feature/new-auth   # follow logs for a specific branch

# Interaction
ephem exec -- psql               # run a command inside the environment
ephem exec backend -- python     # run a command in the context of a service
ephem open frontend              # open the frontend's exposed port in browser
ephem ssh                        # drop into a shell inside the VM

# Diffing
ephem diff main feature/new-auth # diff the Nix derivations between two branches
                                 # (shows what's different BEFORE building)

# Agent API
ephem agent start                # start the MCP server for LLM tool access
ephem agent status               # show connected agents
```

---

## Agent Sidecar API

Every environment runs a lightweight agent sidecar service (port 9999 inside the VM, forwarded to host). This provides a structured, machine-readable interface designed for LLM consumption.

### Endpoints

#### `GET /status` — Environment state

```json
{
  "branch": "feature/new-auth",
  "git_ref": "abc1234",
  "uptime": "2m34s",
  "services": {
    "postgresql": { "state": "running", "healthy": true, "connections": 3 },
    "valkey": { "state": "running", "healthy": true, "keys": 0 },
    "backend": { "state": "running", "healthy": true, "port": 8080, "last_error": null },
    "frontend": { "state": "running", "healthy": true, "port": 3000 }
  }
}
```

#### `POST /exec` — Run a command

```json
// Request
{ "command": "python -m pytest tests/", "timeout": 30 }

// Response
{
  "exit_code": 1,
  "stdout": "...",
  "stderr": "2 failed, 8 passed",
  "duration_ms": 4200
}
```

#### `POST /http` — Make an HTTP request to any internal service

```json
// Request
{ "method": "POST", "url": "http://localhost:8080/api/users", "body": {"name": "alice"} }

// Response
{ "status": 201, "body": {"id": 1, "name": "alice"}, "latency_ms": 45 }
```

#### `POST /sql` — Query the database

```json
// Request
{ "query": "SELECT count(*) FROM users" }

// Response
{ "rows": [{"count": 1}], "duration_ms": 2 }
```

#### `POST /valkey` — Interact with valkey/redis

```json
// Request
{ "command": ["KEYS", "*"] }

// Response
{ "result": ["session:abc123"] }
```

#### `GET /diff?since=<timestamp>` — What changed? (the killer feature)

```json
{
  "since": "2025-02-28T10:30:00Z",
  "db_changes": {
    "users": { "inserts": 1, "updates": 0, "deletes": 0 }
  },
  "files_changed": [
    { "path": "backend/logs/app.log", "bytes_added": 1240 }
  ],
  "http_traffic": [
    { "from": "frontend", "to": "backend", "method": "POST", "path": "/api/users", "status": 201 }
  ],
  "valkey_operations": [
    { "command": "SET", "key": "session:abc123" }
  ],
  "service_state_changes": [],
  "stderr_lines": []
}
```

### MCP Server

The agent API is also exposed as an **MCP (Model Context Protocol) server**, making it natively usable by Claude, Cursor, Windsurf, and any MCP-compatible tool.

MCP tools provided:

| Tool | Description |
|---|---|
| `environment_status` | Get structured state of all services |
| `exec_command` | Run a command inside the environment |
| `http_request` | Make an HTTP request to an internal service |
| `query_database` | Execute SQL against postgres |
| `query_valkey` | Execute valkey/redis commands |
| `get_diff` | Get structured diff of all changes since a timestamp |
| `list_environments` | List all running environments |
| `create_environment` | Spin up a new environment for a git ref |
| `destroy_environment` | Tear down an environment |

This means an AI agent can manage and verify environments without any custom integration — it just connects to the MCP endpoint.

---

## Resource Footprint

Target: comfortable on a 16GB MacBook.

| Component | Memory |
|---|---|
| NixOS base system | ~150-200 MB |
| PostgreSQL (small dataset) | ~100-200 MB |
| Valkey | ~50 MB |
| Python backend (uvicorn) | ~100-300 MB |
| Deno frontend | ~50-150 MB |
| Agent sidecar | ~10-20 MB |
| **Total per environment** | **~500 MB - 1 GB** |

With 2 GB allocated per VM, 3-4 concurrent environments run comfortably on a 16GB machine.

Disk: NixOS images are ~1.5-2 GB per environment before Nix store deduplication. The Nix binary cache means shared dependencies (postgres, valkey, etc.) are stored once.

---

## MVP Milestones

### Phase 0: Proof of Concept
> *Can we boot a NixOS VM with postgres running on both macOS and Linux?*

- [ ] Nix flake that builds a minimal NixOS VM image with QEMU
- [ ] QEMU wrapper that boots the image with HVF (macOS) / KVM (Linux)
- [ ] PostgreSQL running and accessible from the host via port forwarding
- [ ] Basic `ephem up` / `ephem down` commands

**Exit criteria:** `ephem up` boots a VM with postgres accessible at localhost:5432 in under 60 seconds (warm cache) on a MacBook.

### Phase 1: Service Definition & Multi-Service
> *Support the target stack: postgres + valkey + backend + frontend*

- [ ] NixOS module for `type = "process"` services (arbitrary commands)
- [ ] Service dependency ordering and health checks
- [ ] `ephem.nix` config file parsing and validation
- [ ] Log aggregation — `ephem logs [service]`
- [ ] File mounting — sync local source code into the VM
- [ ] `ephem exec` and `ephem ssh`

**Exit criteria:** a user can define the example stack (postgres, valkey, python backend, deno frontend), run `ephem up`, and access all services from their host machine.

### Phase 2: Branch-Aware Multi-Environment
> *Run multiple copies of the stack simultaneously*

- [ ] Branch tracking — map git branches to VM instances
- [ ] Per-environment networking (unique IP or port range per VM)
- [ ] Concurrent environment lifecycle management
- [ ] `ephem status` dashboard
- [ ] `ephem diff` — Nix derivation diffing between branches
- [ ] Automatic cleanup of stale environments

**Exit criteria:** a developer can run `ephem up main` and `ephem up feature/x` simultaneously and access both independently.

### Phase 3: Agent Sidecar & API
> *Make environments machine-readable*

- [ ] Agent sidecar service (Go or Rust binary) bundled into the NixOS image
- [ ] `/status`, `/exec`, `/http`, `/sql`, `/valkey` endpoints
- [ ] `/diff` endpoint with change tracking (DB triggers, filesystem watch, HTTP interception)
- [ ] Host-side HTTP API for environment lifecycle (create, list, destroy)

**Exit criteria:** an external script can programmatically create an environment, run tests, query the database, get a structured diff, and tear down — all via HTTP.

### Phase 4: MCP Integration
> *Let AI agents use environments natively*

- [ ] MCP server exposing environment tools
- [ ] Integration with orchestrators (Conductor task worker, or generic webhook)
- [ ] Multi-turn agent workflow demo (create → test → verify → teardown)

**Exit criteria:** Claude (or another MCP-compatible agent) can manage and verify an environment through tool calls alone.

---

## Technical Decisions

### Language for the CLI / daemon

**Recommendation: Go**

- Excellent cross-compilation for macOS (arm64, amd64) and Linux
- Good process management primitives (for managing QEMU processes)
- Strong HTTP/gRPC libraries for the agent API
- Fast compilation, single binary distribution
- Reasonable Nix integration (shell out to `nix build`)

Rust is also viable but slower to iterate on for an MVP. Python is too slow for the daemon and creates its own dependency management headaches.

### Language for the agent sidecar

**Recommendation: Go**

- Same language as the host daemon — shared types, one build toolchain
- Small static binary (~10 MB) that's easy to include in the NixOS image
- Good HTTP server performance for the API endpoints

### Nix flake structure

```
ephem/
├── flake.nix                    # project flake
├── flake.lock
├── cmd/
│   └── ephem/                   # CLI entry point
├── internal/
│   ├── vm/                      # QEMU lifecycle management
│   ├── nix/                     # Nix build invocation and caching
│   ├── network/                 # Port forwarding, IP management
│   ├── environment/             # Branch tracking, state management
│   └── api/                     # HTTP/gRPC API server
├── sidecar/
│   └── cmd/agent/               # Agent sidecar binary
├── nix/
│   ├── base-system.nix          # Minimal NixOS VM configuration
│   ├── modules/
│   │   ├── process-service.nix  # Generic process service module
│   │   ├── agent-sidecar.nix    # Agent sidecar NixOS module
│   │   └── ...
│   └── overlays/                # Package overlays if needed
└── docs/
    ├── getting-started.md
    ├── service-definition.md
    └── agent-api.md
```

### Networking model

Each VM gets a unique IP on a host-only network.

- **macOS:** QEMU user-mode networking with port forwarding (simplest), or vmnet-bridged for real IPs
- **Linux:** tap device on a bridge, real IP per VM

For the MVP, user-mode networking with port forwarding is sufficient. Each environment gets a port range on the host (e.g., environment 1 maps 3000→13000, 8080→18080; environment 2 maps 3000→23000, 8080→28080). `ephem status` shows the mappings.

### File sync strategy

Local source code needs to be available inside the VM. Options:

1. **9p/virtio-fs mount** — share a host directory into the VM. Low latency, works well for source code. This is the recommended approach for development workflows.
2. **Bake into the image** — copy source code into the NixOS image at build time. Better reproducibility but requires a rebuild on every code change. Good for CI, not for interactive development.
3. **rsync on change** — watch for file changes and sync into the VM. Middle ground.

**MVP approach:** 9p mount for development, bake-into-image for CI/agent workflows.

---

## Non-Goals (for now)

- **Multi-machine environments.** Each environment runs on a single VM on a single host. Distributed testing is out of scope for the MVP.
- **Replacing Kubernetes in production.** This is a development and testing tool, not a deployment platform.
- **Full NixOS module compatibility.** The MVP supports a curated set of well-known services plus generic process types. Full NixOS flexibility comes later.
- **GUI.** CLI-first. A web dashboard could come later but isn't in scope.
- **Windows support.** macOS and Linux only. WSL2 on Windows is a reasonable workaround.

---

## Open Questions

1. **Config format:** Should users write raw Nix, or should we provide a simpler YAML/TOML frontend that generates Nix? Raw Nix is more powerful but has a learning curve. Projects like devenv.sh have shown that a friendlier wrapper increases adoption significantly.

2. **Image build strategy:** Should `ephem up` build the NixOS image locally, or should we support remote builders / binary caches for teams? Local-only is simpler for MVP but slower for first builds.

3. **Hot reload:** When a developer changes source code, should the service restart automatically inside the VM? This requires file watching and service restart coordination. Could be a Phase 1.5 feature.

4. **State persistence:** When you `ephem down` and `ephem up` the same branch, should database state be preserved? Ephemeral-by-default is cleaner, but some workflows need persistent state across restarts.

5. **Naming:** Is `ephem` the right name? Short, memorable, suggests ephemeral. Alternatives considered: `nenv`, `nixvm`, `flakeup`, `devvm`. Open to suggestions.

---

## Getting Started (future)

```bash
# Install ephem (requires Nix)
nix profile install github:yourorg/ephem

# Initialize a project
cd my-project
ephem init    # generates a starter ephem.nix

# Start your environment
ephem up

# In another terminal, or from an AI agent
curl http://localhost:9999/status
```

---

## License

TBD — considering MIT or Apache 2.0 for maximum adoption.
