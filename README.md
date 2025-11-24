### SillyTavern Kit (with Ollama CPU/GPU)

**SillyTavern Kit** is an open-source Docker Compose template that quickly sets up a complete local LLM chat environment featuring [SillyTavern](https://github.com/SillyTavern/SillyTavern) as the web UI and [Ollama](https://ollama.com) as the model backend, with both CPU and NVIDIA GPU variants.

---

### What's included

✅ [**SillyTavern**](https://github.com/SillyTavern/SillyTavern) – Web-based chat UI for local and remote LLMs, built for extensibility and power users

✅ [**Ollama (CPU profile)**](https://ollama.com) – Local LLM runtime optimized for CPUs, ideal for portability and low-end hardware

✅ [**Ollama (GPU profile)**](https://ollama.com) – Local LLM runtime with NVIDIA GPU acceleration via the `nvidia` driver for high performance inference

✅ **Model Pull Jobs** – Automated init jobs that pull your default LLM model on first run for each profile (CPU/GPU)

✅ **Hardened Docker Setup** – Opinionated security baselines, logging, resource limits and an isolated network (`main`)

---

### What you can build

⭐️ **Local LLM Chat UI** – Run SillyTavern in your browser and connect it directly to local Ollama models

⭐️ **GPU-Accelerated Inference** – Offload model inference to your NVIDIA GPU using the `ollama-gpu` profile

⭐️ **Portable CPU Deployment** – Use the `ollama-cpu` profile for environments without GPUs (home server, VPS)

⭐️ **Extensible Playground** – Mount your own configs, extensions and plugins into the SillyTavern container

⭐️ **Isolated LLM Sandbox** – All services run on a dedicated Docker network with restricted capabilities and logging

---

## Installation

### Prerequisites

- Docker and Docker Compose installed on your system
- (For GPU mode) NVIDIA drivers, NVIDIA Container Toolkit and `runtime: nvidia` configured on the host
- Basic familiarity with Docker and local networking

---

### Running SillyTavern Kit using Docker Compose

#### 1. Clone and prepare

```bash
git clone https://github.com/barrax63/sillytavern-kit.git
cd sillytavern-kit
cp .env.example .env
```

Update the `.env` file with your desired values, especially:

- `SILLYTAVERN_PORT` – Port on which the SillyTavern UI will be exposed
- `TZ` – Your timezone (e.g. `Europe/Berlin`)
- `OLLAMA_DEFAULT_MODEL` – The default model Ollama will pull on first run (e.g. `llama3.1:8b`)
- `CUDA_VISIBLE_DEVICES` – For GPU setups, which GPU(s) to expose to the container

The following folders will be created on first run:

- `./config` – SillyTavern configuration
- `./data` – SillyTavern data (chats, presets, etc.)
- `./plugins` – SillyTavern plugins
- `./extensions` – Third-party SillyTavern extensions
- `./ollama` – Ollama models and configuration

> [!IMPORTANT]
> The `./data` folder will be created using an encrypted mountpoint to preserve user data privacy.
> You have to create this mountpoint by yourself or changing the volume back to local storage.

---

#### 2. Start with CPU-only Ollama (default / portable)

```bash
docker compose --profile ollama-cpu up -d
```

This will:

- Start `sillytavern` and `ollama-cpu` on the `main` network
- Run the `ollama-pull-llm-model-cpu` init job which pulls `OLLAMA_DEFAULT_MODEL` once
- Apply resource limits for Ollama CPU (up to 4 CPUs, 8G RAM by default)

---

#### 3. Start with GPU-accelerated Ollama

If you have a compatible NVIDIA GPU and drivers:

```bash
docker compose --profile ollama-gpu up -d
```

This will:

- Start `sillytavern` and `ollama-gpu` with GPU access via the `nvidia` driver
- Apply GPU-aware reservations and CPU/memory limits (by default: 1 GPU, 2 CPU cores, 2G RAM)
- Run the `ollama-pull-llm-model-gpu` init job to pull `OLLAMA_DEFAULT_MODEL`

#### 4. Start without Ollama (using cloud LLMs)

```bash
docker compose up -d
```

---

## Quick start and usage

After starting the stack with your chosen profile, open SillyTavern in your browser:

- Local access: `http://localhost:<SILLYTAVERN_PORT>/`
- Or: `http://YOUR_SERVER_IP:<SILLYTAVERN_PORT>/`

### First-time setup in SillyTavern

1. **Open SillyTavern**  
   Navigate to the SillyTavern UI using the port defined by `SILLYTAVERN_PORT` in your `.env` file.

2. **Configure Ollama connector**  
   - In SillyTavern, go to the LLM / API settings panel.
   - Select the Ollama backend.
   - Set the Ollama endpoint (usually `http://ollama:11434`).

3. **Choose your model**  
   - Select the model you defined in `OLLAMA_DEFAULT_MODEL` or any other model you pulled manually.
   - Adjust temperature, max tokens, etc. according to your preferences.

4. **Start chatting**  
   - Create a new chat, pick a character or system prompt, and start interacting with the model.
   - Your chat history, configs and plugins are persisted in the bound volumes.

---

## DNS / Network architecture

The stack uses a single, isolated Docker network named `main` with a dedicated subnet:

```text
Client browser → SillyTavern (Web UI) → Ollama (LLM backend)
```

- `sillytavern` and `ollama-*` services communicate over the `main` bridge network (`10.254.3.0/24`).
- Only explicitly published ports (such as `SILLYTAVERN_PORT`) are exposed to the host; Ollama itself is only reachable inside the Docker network by default.

---

## Containers and access

The kit consists of multiple containers with restricted access and shared volumes for persistence.

| Container                     | Purpose                      | Hostname   | Ports (exposed)                      | Network accessible?           |
|------------------------------|------------------------------|-------------|--------------------------------------|-------------------------------|
| `sillytavern`                | Web LLM chat UI              | sillytavern | `${SILLYTAVERN_PORT}` on the host    | From LAN (via exposed port)   |
| `ollama-cpu` (profile)       | LLM runtime (CPU)            | ollama      | (internal only, default Ollama port) | From `main` network only      |
| `ollama-gpu` (profile)       | LLM runtime (GPU / NVIDIA)   | ollama      | (internal only, default Ollama port) | From `main` network only      |
| `ollama-pull-llm-model-cpu`  | One-shot model pull (CPU)    | —           | none                                 | Runs then exits               |
| `ollama-pull-llm-model-gpu`  | One-shot model pull (GPU)    | —           | none                                 | Runs then exits               |

Notes:

- Both Ollama services share the same base configuration (`x-ollama`), including volumes, healthchecks and environment.
- Only one Ollama profile is normally active at a time (CPU or GPU), depending on the profile you use when starting the stack.

---

## Upgrading

### Update all containers to the latest images

From your project directory:

```bash
git pull
docker compose pull
docker compose --profile ollama-cpu up -d      # or --profile ollama-gpu or no profile at all
```

This will:

- Pull the latest repository changes
- Download the latest SillyTavern and Ollama images
- Recreate containers with the new images
- Preserve all persistent data in volumes (`./config`, `./data`, `./plugins`, `./extensions`, `./ollama`)

---

### Enable auto-updates (optional)

You can enable automatic updates using a cron job. For example:

```bash
crontab -u <USER> -e
```

Add an entry similar to:

```bash
# Every Sunday at 02:30 run git pull and compose up with latest images (CPU profile)
30 2 * * 0 cd /home/docker/sillytavern-kit && /usr/bin/git pull && /usr/bin/docker compose build --no-cache && /usr/bin/docker compose up -d >> /home/docker/logs/sillytavern-kit-update.log 2>&1
```

Adjust the profile (`ollama-cpu` vs `ollama-gpu`) and schedule as needed.

---

## Configuration guide

### Environment variables

The `.env.example` file (to be copied to `.env`) contains all configurable options, including but not limited to:

- `SILLYTAVERN_PORT` – Port mapping for the SillyTavern UI
- `TZ` – Timezone for containers
- `OLLAMA_DEFAULT_MODEL` – Model to auto-pull on first run (used by the init jobs)
- `OLLAMA_KEEP_ALIVE`, `OLLAMA_NUM_PARALLEL`, `OLLAMA_KV_CACHE_TYPE`, `OLLAMA_CONTEXT_LENGTH`, `OLLAMA_FLASH_ATTENTION` – Ollama runtime tuning parameters
- `CUDA_VISIBLE_DEVICES` – Control GPU visibility for GPU profile

Edit these values to match your hardware, performance and networking needs.

---

### Security features

This setup includes several security-oriented defaults:

- **No new privileges** – `no-new-privileges:true` is set for services, preventing privilege escalation
- **AppArmor profile** – Containers use `apparmor=docker-default` for additional isolation
- **Capability dropping** – Most Linux capabilities are dropped via `cap_drop: [ALL]`, with minimal `cap_add` where needed (e.g. `CHOWN`, `SETUID`, `SETGID`, `FOWNER`)
- **Network isolation** – All services are on a dedicated bridge network (`main`) with its own subnet
- **Resource limits** – CPU and memory limits/reservations are defined for SillyTavern and Ollama services
- **Health checks** – Ollama services have a healthcheck that runs `ollama list` to verify readiness
- **Structured logging** – JSON-file logging driver with size and file rotation (`max-size: 10m`, `max-file: 5`) is enabled for all services

You can further harden the setup by adjusting the Compose file to your environment (e.g. more restrictive network exposure, different AppArmor profiles).

---

## Recommended reading

### Official documentation

- [SillyTavern GitHub](https://github.com/SillyTavern/SillyTavern)
- [Ollama Documentation](https://github.com/jmorganca/ollama)

### Helpful topics

#### SillyTavern

- Custom characters and presets
- Third-party extensions (mounted via `./extensions`)
- Plugin system (mounted via `./plugins`)
- UI themes and hotkeys

#### Ollama

- Managing multiple models (`ollama pull`, `ollama list`, `ollama show`)
- Model quantization and sizes
- Context length tuning (`OLLAMA_CONTEXT_LENGTH`)
- Performance tweaks (`OLLAMA_NUM_PARALLEL`, `OLLAMA_KV_CACHE_TYPE`, `OLLAMA_FLASH_ATTENTION`)

---

## License

This project is licensed under the Apache License 2.0 (or adapt to your actual project license) – see the `LICENSE` file for details.

---

## Support

### Community resources

- [SillyTavern Discussions](https://github.com/SillyTavern/SillyTavern/discussions)
- [Ollama GitHub Issues](https://github.com/jmorganca/ollama/issues)
- General-purpose LLM communities (Reddit, Discord, etc.)

### Getting help

- **Report bugs** – Use GitHub Issues for bug reports and troubleshooting
- **Feature requests** – Open an issue or discussion to propose new features or improvements
- **Documentation** – Check each project’s official docs for detailed guides and advanced configuration

---

## Contributing

Contributions are welcome! To contribute:

1. Fork this repository
2. Create your feature branch:  
   `git checkout -b feature/AmazingFeature`
3. Commit your changes:  
   `git commit -m "Add some AmazingFeature"`
4. Push to the branch:  
   `git push origin feature/AmazingFeature`
5. Open a Pull Request

---

## Acknowledgments

- [SillyTavern](https://github.com/SillyTavern/SillyTavern) – Fantastic front-end for chatting with local and remote LLMs
- [Ollama](https://ollama.com) – Local model runtime with an easy CLI and API
- [Docker](https://www.docker.com/) – Containerization platform that powers this stack