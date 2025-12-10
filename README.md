# n8n with PostgreSQL and Cloudflare Tunnel

This project provides a `docker-compose` setup to run a local instance of [n8n](https://n8n.io/), a free and open-source workflow automation tool, with a PostgreSQL database for data persistence. The n8n instance is securely exposed to the internet using [Cloudflare Tunnel](https://www.cloudflare.com/products/tunnel/).

This setup is great for deploying on a Network Attached Storage (NAS) device that supports Docker, allowing you to run a self-hosted, publicly accessible n8n instance from your home or office network.

## Prerequisites

*   [Docker](https://docs.docker.com/get-docker/)
*   [Docker Compose](https://docs.docker.com/compose/install/)
*   A [Cloudflare account](https://dash.cloudflare.com/sign-up)

## Setup

### 1. Configure Environment Variables

Create a `.env` file in the root of the project and fill in the required values. You can use the `.env.example` file as a template.

```env
# .env

# PostgreSQL settings
POSTGRES_DB=n8n
POSTGRES_USER=n8nuser
POSTGRES_PASSWORD=yoursecretpassword

# n8n settings
GENERIC_TIMEZONE=America/New_York
TZ=America/New_York
N8N_HOST=your-n8n-subdomain.your-domain.com
N8N_EDITOR_BASE_URL=https://your-n8n-subdomain.your-domain.com/
WEBHOOK_URL=https://your-n8n-subdomain.your-domain.com/
# This specifies the number of reverse proxies n8n is running behind.
# The default of 1 is correct for this project's Cloudflare Tunnel setup.
N8N_PROXY_HOPS=1

# Cloudflare Tunnel token
TUNNEL_TOKEN=your-tunnel-token
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
    *   **Subdomain:** Enter the subdomain for your n8n instance (e.g., `n8n`).
    *   **Domain:** Select your domain.
    *   **Service:**
        *   **Type:** `HTTP`
        *   **URL:** `n8n:5678` (This points to the n8n service defined in the `docker-compose.yml` file, at its internal port).

This will create a public URL (e.g., `https://n8n.your-domain.com`) that tunnels traffic to your local n8n container.

## Environment Variables

The following environment variables need to be set in the `.env` file:

| Variable              | Description                                                                                             |
| --------------------- | ------------------------------------------------------------------------------------------------------- |
| `POSTGRES_USER`       | The username for the PostgreSQL database.                                                               |
| `POSTGRES_PASSWORD`   | The password for the PostgreSQL database.                                                               |
| `POSTGRES_DB`         | The name of the PostgreSQL database.                                                                    |
| `GENERIC_TIMEZONE`    | The timezone for the n8n application (e.g., `America/New_York`). Find your timezone from this [list of tz database time zones](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones). |
| `TZ`                  | The timezone for the container (should match `GENERIC_TIMEZONE`).                                       |
| `N8N_HOST`            | The public domain where n8n is accessible (e.g., `n8n.your-domain.com`).                                |
| `N8N_EDITOR_BASE_URL` | The full base URL for the n8n editor (e.g., `https://n8n.your-domain.com/`).                             |
| `WEBHOOK_URL`         | The full URL for n8n webhooks (should be the same as `N8N_EDITOR_BASE_URL`).                            |
| `N8N_PROXY_HOPS`      | The number of reverse proxies n8n is running behind. Defaults to `1`, which is correct for the Cloudflare Tunnel setup in this project. |
| `TUNNEL_TOKEN`        | Your Cloudflare Tunnel token.                                                                           |

## Usage

This setup uses Docker Compose Profiles to give you the flexibility to run your n8n instance with or without the Cloudflare Tunnel.

### Running with Cloudflare Tunnel (Publicly Accessible)

To start all services, including the Cloudflare Tunnel for public access, run the following command. This is ideal for production or when you need webhooks to be reachable from the internet.

```bash
docker compose --profile tunnel up -d
```

### Running Locally Only

To start only the n8n and PostgreSQL services for local development or testing (without exposing n8n publicly), use the standard command:

```bash
docker compose up -d
```

### Stopping the Services

To stop all running services, use:

```bash
docker compose down
```

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
