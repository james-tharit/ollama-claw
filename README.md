# OpenClaw + Ollama Podman Setup

This repository contains a functional Podman configuration for running **OpenClaw** alongside **Ollama**. Podman is used here for its rootless capabilities and better security model.

## 🚀 Quick Start

1.  **Ensure Ollama has the model:**

    ```bash
    podman exec -it ollama ollama pull qwen2.5:2b
    ```

2.  **Start the stack:**
    Using Podman Compose (or the `podman-compose` python tool):

    ```bash
    podman-compose up -d
    ```

3.  **Get the Pairing Code:**

    ```bash
    podman exec -it openclaw sh // shell into container
    openclaw devices // list all devices
    openclaw devices pair <id> // pair with selected device
    ```

4.  **Access the UI:**
    Go to [http://localhost:18789](https://www.google.com/search?q=http://localhost:18789)

      - **Username:** `admin`
      - **Password:** `test_1234`

## 🛠 Podman Specific Adjustments

### 1\. The Network Alias Fix

In Podman, the `aliases` under `networks` work similarly to Docker, but you must ensure the `dnsname` plugin is enabled (default in most modern Podman versions) so that Nginx can resolve the `browser` hostname.

### 2\. Permissions (Rootless Podman)

If you are running Podman in **rootless mode**, ensure the local directories have the correct permissions. Podman maps your user to the container's root user via `subuid`/`subgid`.
If you get "Permission Denied" on your volumes, run:

```bash
podman unshare chown -R 1000:1000 ./openclaw_data ./ollama_data
```

### 3\. The "Systemd" Conflict

Podman handles systemd inside containers slightly differently than Docker. By not overriding the `command`, we allow the OpenClaw entrypoint to run as the primary PID, avoiding the systemd service manager requirement entirely.

## 📂 Configuration (container-compose.yml)

```yaml
services:
  ollama:
    image: docker.io/ollama/ollama:latest
    container_name: ollama
    volumes:
      - ./ollama_data:/root/.ollama
    ports:
      - "11434:11434"

  openclaw:
    image: docker.io/coollabsio/openclaw:latest
    container_name: openclaw
    depends_on:
      - ollama
    networks:
      default:
        aliases:
          - browser
    environment:
      - AUTH_PASSWORD=test_1234
      - AUTH_USERNAME=admin
      - OPENCLAW_GATEWAY_TOKEN=my-secret-token
      - OPENCLAW_ALLOWED_ORIGINS=*
      - OLLAMA_BASE_URL=http://ollama:11434
      - OPENCLAW_STATE_DIR=/data/.openclaw
      - OPENCLAW_WORKSPACE_DIR=/data/workspace
      - OPENCLAW_GATEWAY_BIND=lan
    volumes:
      - ./openclaw_data:/data:Z # Added :Z for SELinux contexts
    ports:
      - "18789:8080"
    restart: unless-stopped

networks:
  default:
    driver: bridge
```

## 📋 Useful Podman Commands

  - **View Logs:** `podman logs -f openclaw`
  - **Check Internal Ports:** `podman exec -it openclaw ss -tulpn`
  - **Clean Start:** \`\`\`bash
    podman-compose down
    podman volume prune \# Warning: deletes unused volumes
    podman-compose up -d
    ```
    
    ```