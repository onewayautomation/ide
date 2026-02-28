# Industrial Data Explorer

A web application for exploring industrial data published on **MQTT brokers** and exposed via **OPC UA** servers.

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) 24+ with the Compose plugin
- Ports **80**, **443**, and **81** available on the host (used by the built-in reverse proxy)

## Installation

There are two ways to install: pulling the image from Docker Hub, or loading a
pre-built image archive (useful for air-gapped or restricted environments).

---

### Option A — Pull from Docker Hub

#### 1. Download the compose file

```bash
curl -O https://github.com/onewayautomation/ide/blob/main/docker-compose.yml
```

#### 2. Create the environment file

```bash
curl -o .env https://github.com/onewayautomation/ide/blob/main/env.example
```

Open `.env` in a text editor and at minimum set:

| Variable | Description |
|---|---|
| `POSTGRES_PASSWORD` | Password for the built-in PostgreSQL database |
| `CREDENTIALS_ENCRYPTION_KEY` | Random 32-byte key used to encrypt stored credentials. Generate with `openssl rand -base64 32` |
| `APP_BASE_URL` | The public URL of your server, e.g. `https://ide.example.com`. Used for OAuth redirect URIs. |

#### 3. Start the application

```bash
docker compose up -d
```

Docker will pull the images automatically on first run.

---

### Option B — Load from offline image archive

Use this method if the host has no access to Docker Hub.

#### 1. Download the image archive

```bash
curl -L -o ide-image.zip https://onewayautomation.com/opcua-binaries/ide-image.zip
```

#### 2. Extract and load the image

```bash
unzip ide-image.zip
docker load -i ide-image.tar.gz
```

Verify the image was loaded:

```bash
docker images ogamma/ide
```

#### 3. Download the compose and environment files

```bash
curl -O https://github.com/onewayautomation/ide/blob/main/docker-compose.yml
curl -o .env https://github.com/onewayautomation/ide/blob/main/env.example
```

#### 4. Configure the environment

Open `.env` and set at minimum:

| Variable | Description |
|---|---|
| `POSTGRES_PASSWORD` | Password for the built-in PostgreSQL database |
| `CREDENTIALS_ENCRYPTION_KEY` | Generate with `openssl rand -base64 32` |
| `APP_BASE_URL` | Public URL of your server, e.g. `https://ide.example.com` |

#### 5. Start the application

```bash
docker compose up -d
```

---

## Configure the Reverse Proxy

Open the **Nginx Proxy Manager** admin UI in your browser:

```
http://<your-server-ip>:81
```

Default credentials (you will be prompted to change them):
- **Email:** `admin@example.com`
- **Password:** `changeme`

Add a **Proxy Host** with the following settings:

**Details tab:**

| Field | Value |
|---|---|
| Domain Names | Your domain, e.g. `ide.example.com` |
| Scheme | `http` |
| Forward Hostname / IP | `ide` |
| Forward Port | `8080` |
| Websockets Support | ✅ Enable |

> **Websockets Support** must be enabled. The application uses a persistent WebSocket connection (`/ws`) to stream live data from MQTT brokers and OPC UA servers to the browser. Without it, the address space tree will not populate and live values will not update.

**Advanced tab:**

Paste the following into the **Custom Nginx Configuration** box to prevent Nginx from closing idle WebSocket connections:

```nginx
proxy_read_timeout 3600s;
proxy_send_timeout 3600s;
```

> Do **not** add `proxy_set_header Upgrade` or `proxy_http_version` here — the **Websockets Support** toggle already injects those directives. Duplicating them will cause Nginx to fail and the application will not load.

**SSL tab:**

To enable HTTPS, request a free Let's Encrypt certificate. Once HTTPS is active, the WebSocket connection will automatically use `wss://` (WebSocket Secure).

## Open the Application

Navigate to your configured domain (or `http://<server-ip>` if you are not using a custom domain) and create your first account.

## Included Services

### pgAdmin — Database Management

pgAdmin is included for inspecting and managing the PostgreSQL database.

It is accessible directly on the host at:

```
http://<your-server-ip>:4888
```

It can also be exposed through the reverse proxy at the `/pgadmin` path. In the NPM admin UI, add a **Proxy Host** for your domain with the following settings:

**Details tab:**

| Field | Value |
|---|---|
| Domain Names | Your domain, e.g. `ide.example.com` |
| Scheme | `http` |
| Forward Hostname / IP | `pgadmin` |
| Forward Port | `80` |

**Advanced tab:**

```nginx
proxy_set_header X-Script-Name /pgadmin;
```

This header tells pgAdmin it is running under the `/pgadmin` sub-path so all its internal links and redirects are generated correctly.

Then set the **Custom Location** to `/pgadmin` in the proxy host so requests to `https://ide.example.com/pgadmin` are forwarded to the pgadmin container.

Default pgAdmin credentials are set by `PGADMIN_EMAIL` and `PGADMIN_PASSWORD` in `.env` (defaults: `admin@example.com` / `admin`). Change these before exposing pgAdmin publicly.

## Optional Features

### LinkedIn OAuth Login

1. Create a LinkedIn app at https://www.linkedin.com/developers/
2. Set the **Authorized Redirect URL** to `https://<your-domain>/api/auth/linkedin/callback`
3. Fill in `LINKEDIN_CLIENT_ID` and `LINKEDIN_CLIENT_SECRET` in `.env`
4. Restart: `docker compose restart ide`

### Email Verification on Registration

Fill in the `SMTP_*` variables in `.env` with your mail server details. When `SMTP_HOST` is set, new users must verify their email address before completing registration.

## Updating

```bash
docker compose pull
docker compose up -d
```

## Stopping

```bash
docker compose down
```

Data (database, proxy configuration, SSL certificates) is stored in Docker named volumes and is preserved across restarts and updates.

## Support

- Issues: https://github.com/ogamma/ide/issues
