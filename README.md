# tuwunel-matrixrtc-compose

A Docker Compose stack for a self-hosted, private [Matrix](https://matrix.org) server with voice/video calling. It runs:

| Service | Image | Purpose |
| --- | --- | --- |
| `caddy` | Caddy 2.11.4 | Reverse proxy and automatic HTTPS |
| `matrix-config-init` | Alpine 3.23.5 | One-shot job that renders the config templates with your `.env` values |
| `matrix-server` | [tuwunel](https://github.com/matrix-construct/tuwunel) v1.8.2 | Matrix homeserver |
| `matrix-rtc-jwt` | lk-jwt-service 0.5.0 | Issues LiveKit tokens to Matrix users (MatrixRTC auth) |
| `matrix-rtc-livekit` | LiveKit server 1.13.4 | SFU for voice/video calls (Element Call) |
| `matrix-element-web` | Element Web v1.12.24 | Web client, with custom branding |

All images are pinned by digest. Each container has memory and CPU limits, `no-new-privileges`, and rotated logs (10 MB × 3).

Federation is switched on only so MatrixRTC works. Every remote server is blocked (`forbidden_remote_server_names = [".*"]`), so the server cannot talk to other Matrix servers.

## Subdomains

With `DOMAIN=example.com`, Caddy serves:

| Host | Routes to |
| --- | --- |
| `matrix.example.com` | tuwunel (client API and `.well-known`) |
| `matrix-rtc.example.com` | lk-jwt-service (`/sfu/get`, `/get_token`, `/healthz`) and LiveKit (everything else, including WebSockets) |
| `chat.example.com` | Element Web |

User IDs look like `@alice:matrix.example.com`.

## Requirements

- A Linux host with Docker and Docker Compose v2.20+ (needed for the volume `subpath` used by Element Web)
- A public IPv4 address
- DNS A/AAAA records for `matrix.`, `matrix-rtc.` and `chat.` pointing to the host
- These ports open in the firewall:
  - `80/tcp`, `443/tcp`: HTTP/HTTPS (Caddy)
  - `7881/tcp`: LiveKit ICE over TCP
  - `50100-50200/udp`: LiveKit media

## Setup

1. **Create the shared Docker network.** The compose file expects it to exist already:

   ```bash
   docker network create caddy
   ```

2. **Create your `.env`:**

   ```bash
   cp .env.example .env
   ```

   | Variable | Description |
   | --- | --- |
   | `DOMAIN` | Base domain, e.g. `example.com` (no subdomain) |
   | `REGISTRATION_TOKEN` | Token needed to register an account. Use a long random string. |
   | `MRTC_KEY` | LiveKit API key, shared by LiveKit and lk-jwt-service |
   | `MRTC_SECRET` | LiveKit API secret, at least 32 characters |

   You can generate the random values with:

   ```bash
   openssl rand -hex 32
   ```

3. **Create the config templates from the examples.** The real templates are gitignored so you can change them locally:

   ```bash
   cp tuwunel.toml.template.example tuwunel.toml.template
   ```

   ```bash
   cp element-web-config.json.template.example element-web-config.json.template
   ```

   `${DOMAIN}` and `${REGISTRATION_TOKEN}` in these files are filled in by `matrix-config-init` each time the stack starts.

4. **Set your server's public IP in `livekit.yml`.** Replace the `node_ip` value with the host's public IPv4 address. LiveKit gives this address to clients for media, so calls will fail if it is wrong.

5. **Start the stack:**

   ```bash
   docker compose up -d
   ```

   Caddy gets TLS certificates for the three subdomains automatically. Element Web is then at `https://chat.<DOMAIN>`.

## First steps after starting

- **Create the admin account.** Register at `https://chat.<DOMAIN>` using `REGISTRATION_TOKEN`. In tuwunel, the first account created on the server becomes the admin.
- **Give out single-use invite tokens** instead of the main token. In the admin room, run:

  ```
  !admin token issue <MAX_USES>
  ```

- **Create the welcome room.** If you turn on `auto_join_rooms`, new users are added to `#welcome:matrix.<DOMAIN>`. Create a public room with that alias first.

## Configuration

### tuwunel (`tuwunel.toml.template`)

The example file:

- turns on end-to-end encryption by default for every new room (`encryption_enabled_by_default_for_room_type = "all"`)
- leaves `auto_join_rooms` commented out
- adds 👋 to new users' display names
- limits uploads to about 20 MB (`max_request_size`)
- reads client IPs from Caddy's `X-Forwarded-For` header

Your changes take effect after you restart the stack. `matrix-config-init` renders the template again on each start:

```bash
docker compose up -d --force-recreate matrix-config-init matrix-server
```

> **Note:** `server_name` cannot be changed after the first start without wiping the database.

### Element Web (`element-web-config.json.template`)

Element Web is locked to this homeserver: custom server URLs, guests and 3PID (email/phone) login are all turned off. Branding is set in the `branding` block. The background and logo come from `custom-images/`, which is mounted at `/custom-images/` in the web client. To use your own branding, replace `bg.webp` and `skglogo.svg` (or point the config at new files) and change `brand`, `logo_link_url` and `auth_footer_links`.

### LiveKit (`livekit.yml`)

LiveKit uses TCP port 7881 and UDP ports 50100–50200 for media. If you change the port range, change it in `compose.yml` too. The API key and secret come from `.env` through `LIVEKIT_KEYS`.

## Data and backups

| Location | Contents |
| --- | --- |
| `db` volume | tuwunel database (all accounts, rooms and messages). **Back this up.** |
| `./data`, `./config` | Caddy certificates and state |
| `tuwunel-config`, `element-web-config` volumes | Rendered configs, rebuilt on each start |

Stop `matrix-server` before backing up the `db` volume so the copy is consistent.

## Updating

Images are pinned by digest. To upgrade a service, replace its digest in `compose.yml` with the new one, update the version comment above it, then run:

```bash
docker compose pull
```

```bash
docker compose up -d
```

## Troubleshooting

- **Containers can't start: `network caddy declared as external, but could not be found`.** Run `docker network create caddy`.
- **Calls connect but there is no audio or video.** Check that `node_ip` in `livekit.yml` is the public IP, and that `7881/tcp` and `50100-50200/udp` are open.
- **Check the logs:**

  ```bash
  docker compose logs -f matrix-server
  ```
