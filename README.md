# n8n with PostgreSQL and Cloudflare Tunnel

This project provides a `docker compose` setup to run a local instance of [n8n](https://n8n.io/), a free and open-source workflow automation tool, with a PostgreSQL database for data persistence. The n8n instance is securely exposed to the internet using [Cloudflare Tunnel](https://www.cloudflare.com/products/tunnel/).

This setup is great for deploying on a Network Attached Storage (NAS) device that supports Docker, allowing you to run a self-hosted, publicly accessible n8n instance from your home or office network.

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/)
- [Docker Compose](https://docs.docker.com/compose/install/)
- A [Cloudflare account](https://dash.cloudflare.com/sign-up)

## Setup

### 1. Configure Environment Variables

Create a `.env` file in the root of the project and fill in the required values. You can use the `.env.example` file as a template.

```env
# .env

# PostgreSQL settings
POSTGRES_DB=n8n
POSTGRES_USER=n8n
POSTGRES_PASSWORD=

# n8n settings
GENERIC_TIMEZONE=America/New_York
TZ=America/New_York
N8N_HOST=localhost
N8N_EDITOR_BASE_URL=http://localhost:5678
WEBHOOK_URL=http://localhost:5678
N8N_RUNNERS_AUTH_TOKEN=

# The number of reverse proxies n8n is running behind.
# This should be 1 when using the Cloudflare Tunnel setup.
N8N_PROXY_HOPS=1

# Cloudflare Tunnel token (only needed for tunnel mode)
#TUNNEL_TOKEN=
```

### 2. Set up Cloudflare Tunnel

To expose your local n8n instance to the internet, you need to set up a Cloudflare Tunnel.

**a. Create a Cloudflare Tunnel**

Follow the official Cloudflare documentation to [create a tunnel and get your token](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/install-and-setup/tunnel-guide/). This will give you the `TUNNEL_TOKEN` for your `.env` file.

**b. Configure the Public Hostname**

Once your tunnel is created, you need to configure a public hostname that routes traffic to your local n8n service.

1.  In your Cloudflare dashboard, navigate to **Zero Trust** -> **Access** -> **Tunnels**.
2.  Select your tunnel and go to the **Public Hostnames** tab.
3.  Click **Add a public hostname**.
4.  Configure the route:
    - **Subdomain:** Enter the subdomain for your n8n instance (e.g., `n8n`).
    - **Domain:** Select your domain.
    - **Service:**
      - **Type:** `HTTP`
      - **URL:** `n8n:5678` (This points to the n8n service defined in the `docker-compose.yml` file, at its internal port).

This will create a public URL (e.g., `https://n8n.your-domain.com`) that tunnels traffic to your local n8n container.

### 3. A Note on File Permissions

When mounting files from the host into containers, it's crucial to ensure the file permissions are correctly set. Docker containers run under a specific user ID (`1000` for both the n8n and runner containers in this setup), and if that user doesn't have permission to read a mounted file, the service will fail to start.

A common place this occurs is with the `n8n-task-runners.json` file. If this file was created by a `root` user on the host, it can lead to permission errors inside the container. To fix this, you can adjust the file's permissions to make it readable by all users:

```bash
sudo chmod 644 n8n-task-runners.json
```

This is especially important when running on systems like a NAS where default file permissions may be more restrictive.

## Environment Variables

The following environment variables need to be set in the `.env` file:

| Variable                 | Description                                                                                                                                                                                   |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `POSTGRES_USER`          | The username for the PostgreSQL database.                                                                                                                                                     |
| `POSTGRES_PASSWORD`      | The password for the PostgreSQL database.                                                                                                                                                     |
| `POSTGRES_DB`            | The name of the PostgreSQL database.                                                                                                                                                          |
| `GENERIC_TIMEZONE`       | The timezone for the n8n application (e.g., `America/New_York`). Find your timezone from this [list of tz database time zones](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones). |
| `TZ`                     | The timezone for the container (should match `GENERIC_TIMEZONE`).                                                                                                                             |
| `N8N_HOST`               | The public domain where n8n is accessible (e.g., `n8n.your-domain.com`).                                                                                                                      |
| `N8N_EDITOR_BASE_URL`    | The full base URL for the n8n editor (e.g., `https://n8n.your-domain.com/`).                                                                                                                  |
| `WEBHOOK_URL`            | The full URL for n8n webhooks (should be the same as `N8N_EDITOR_BASE_URL`).                                                                                                                  |
| `N8N_RUNNERS_AUTH_TOKEN` | A secure, secret token shared between n8n and its runners for authentication.                                                                                                                 |
| `N8N_PROXY_HOPS`         | The number of reverse proxies n8n is running behind. Defaults to `1`, which is correct for the Cloudflare Tunnel setup in this project.                                                       |
| `TUNNEL_TOKEN`           | Your Cloudflare Tunnel token.                                                                                                                                                                 |

## Adding Custom Packages

The task runner can be customized to include additional Node.js or Python packages for use in your workflows. This is a two-step process: you must install the package in the Docker image and then add it to the allowlist.

### Step 1: Install the Package in the Docker Image

Edit the `Dockerfile.n8n-runner` file to include installation commands for your desired packages.

**For Node.js (npm) packages:**

Add the package name to the `pnpm add` command. For example, to add `axios`:

```dockerfile
# Dockerfile.n8n-runner

# ... (other instructions)
RUN cd /opt/runners/task-runner-javascript && \
    pnpm add moment uuid axios
# ... (other instructions)
```

**For Python (pip) packages:**

Add the package name to the `pip install` command. For example, to add `scikit-learn`:

```dockerfile
# Dockerfile.n8n-runner

# ... (other instructions)
RUN cd /opt/runners/task-runner-python && \
    pip install numpy pandas scikit-learn
# ... (other instructions)
```

### Step 2: Add the Package to the Allowlist

Edit the `n8n-task-runners.json` file to add the package name to the `N8N_RUNNERS_EXTERNAL_ALLOW` list for the corresponding runner type.

**For Node.js (npm) packages:**

Add the package name to the `N8N_RUNNERS_EXTERNAL_ALLOW` list for the `javascript` runner.

```json
// n8n-task-runners.json
{
  "task-runners": [
    {
      "runner-type": "javascript",
      "env-overrides": {
        "N8N_RUNNERS_STDLIB_ALLOW": "moment",
        "N8N_RUNNERS_EXTERNAL_ALLOW": "uuid,axios"
      }
    }
    // ... (python runner config)
  ]
}
```

**For Python (pip) packages:**

Add the package name to the `N8N_RUNNERS_EXTERNAL_ALLOW` list for the `python` runner.

```json
// n8n-task-runners.json
{
  "task-runners": [
    // ... (javascript runner config)
    {
      "runner-type": "python",
      "env-overrides": {
        "PYTHONPATH": "/opt/runners/task-runner-python",
        "N8N_RUNNERS_STDLIB_ALLOW": "json",
        "N8N_RUNNERS_EXTERNAL_ALLOW": "numpy,pandas,scikit-learn"
      }
    }
  ]
}
```

### Step 3: Rebuild the Runner Image

After making these changes, you must rebuild your custom runner image for them to take effect. Run the following command:

```bash
docker compose build n8n-task-runners
```

After the build is complete, you can restart your services as usual.

## Usage

This setup is configured to run in two distinct modes using a Docker Compose override file: **Local Mode** and **Tunnel Mode**. This gives you the flexibility to run n8n for local development or expose it publicly for production use.

### Local Mode (Default)

This mode is for local development and testing. It runs n8n and its database, making the n8n UI accessible on your host machine. The Cloudflare Tunnel is **not** used.

**To Start:**
Run the standard `docker compose` command. n8n will be available at `http://localhost:5678`.

```bash
docker compose up -d
```

> **Note:** For this mode, ensure the `N8N_...` URL variables in your `.env` file are set to `http://localhost:5678`.

### Tunnel Mode (Publicly Accessible)

This mode is for production or any scenario where you need n8n to be reachable from the internet (e.g., for webhooks). It uses the `compose.tunnel.yml` override file to start the Cloudflare Tunnel and prevents n8n from being exposed directly on your local port.

**To Start:**
You must explicitly include the override file in your command:

```bash
docker compose -f docker-compose.yml -f compose.tunnel.yml up -d
```

> **Note:** For this mode, ensure the `N8N_...` URL variables in your `.env` file are set to your public Cloudflare domain (e.g., `https://n8n.your-domain.com`).

### Stopping the Services

To stop all services, use the corresponding command for the mode you are running:

- **If in Local Mode:**
  ```bash
  docker compose down
  ```
- **If in Tunnel Mode:**
  ```bash
  docker compose -f docker-compose.yml -f compose.tunnel.yml down
  ```

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
