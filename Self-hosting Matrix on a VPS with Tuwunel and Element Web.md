# Self-hosting Matrix on a VPS with Tuwunel and Element Web

A reusable, step-by-step runbook for a Debian/Ubuntu VPS. This guide uses Docker Compose, Tuwunel as the Matrix homeserver, Nginx as the reverse proxy, Let's Encrypt for HTTPS, and a self-hosted Element Web client. It is generic: replace the example domains with your own.

> **Example values used throughout**
>
> - Matrix homeserver/API hostname: `matrix.example.com`
> - Element Web hostname: `chat.example.com`
> - VPS public IPv4: `203.0.113.10` (documentation-only example; replace it)
> - Project directory: `/opt/matrix`
> - Tuwunel internal HTTP port: `8008`
> - Public HTTPS port: `443` (the standard choice when available)
> - Optional federation port: `8448`
>
> The simplest server name in this tutorial is `matrix.example.com`, so user IDs look like `@alice:matrix.example.com`. **Choose the Matrix `server_name` before the first account is created. It is an identity, not merely a web address, and changing it later is not a routine rename.** If you want IDs such as `@alice:example.com`, plan root-domain delegation (`.well-known`) before installing; see Tuwunel's official delegation documentation.

---

## 1. Understand the components

A Matrix deployment has at least two separate pieces:

- **Tuwunel** is the homeserver. It handles accounts, room state, messages, media, and the Matrix APIs. Its persistent database and media live in a data directory/volume.
- **Element Web** is a web client. It is the graphical interface people open in a browser; it is not the homeserver and does not store the server's authoritative account database.
- **Nginx** accepts public HTTP/HTTPS traffic, terminates TLS, chooses the correct website based on the requested hostname, and proxies requests to the correct container.
- **Let's Encrypt/Certbot** provides the TLS certificate used by Nginx.
- **Docker Compose** describes and runs the containers together.

Request flow:

```text
Browser or Matrix app
  | HTTPS https://chat.example.com
  v
Nginx :443 --reverse proxy--> Element Web container

Matrix app
  | HTTPS https://matrix.example.com
  v
Nginx :443 --reverse proxy--> Tuwunel :8008
                                      |
                                      v
                              persistent data volume
```

Only Nginx needs public web ports. Do not publish Tuwunel's port `8008` to the entire Internet. Docker containers on the same Compose network can reach Tuwunel by its service name.

### Public ports

- `80/tcp`: HTTP; used for redirecting visitors to HTTPS and, in this tutorial, the Let's Encrypt HTTP-01 challenge.
- `443/tcp`: HTTPS for Matrix clients and Element Web. Both hostnames can use the same port because Nginx selects a virtual host based on the hostname/SNI.
- `8448/tcp`: conventional Matrix federation port. It is only needed if federation is enabled and you intend other homeservers to communicate with yours.
- `8008/tcp`: Tuwunel's internal plain-HTTP listener. Keep it private to Docker.

If another application already owns ports 80/443, do not stop or reconfigure it blindly. Either integrate Matrix into the existing reverse proxy, give Matrix a separate IP, or deliberately use a non-standard port. A non-standard port means URLs and Matrix federation discovery need extra planning; the guide below assumes a fresh VPS where ports 80 and 443 are available.

## 2. Choose the permanent Matrix identity

Decide the Matrix server name before starting Tuwunel for the first time. This guide uses:

```text
matrix.example.com
```

That name appears after the colon in Matrix IDs:

```text
@alice:matrix.example.com
@bob:matrix.example.com
```

Element Web will be hosted at:

```text
https://chat.example.com
```

and configured to connect to:

```text
https://matrix.example.com
```

These two hostnames can point to the same VPS. The Element hostname does not become part of users' Matrix IDs.

**Do not change `TUWUNEL_SERVER_NAME` after registration begins.** Treat it as permanent. To use IDs on the root domain (`@alice:example.com`) while hosting the API at `matrix.example.com`, follow the official Tuwunel root-domain delegation guide rather than improvising a rename.

## 3. Prepare DNS

At your DNS provider, create these records:

| Type | Name | Value |
|---|---|---|
| A | `matrix` | your VPS public IPv4 |
| A | `chat` | your VPS public IPv4 |
| AAAA | `matrix` | your VPS IPv6, only if IPv6 is configured and reachable |
| AAAA | `chat` | your VPS IPv6, only if IPv6 is configured and reachable |

Do not publish an AAAA record for a server that does not actually serve these hostnames over IPv6; clients may try the broken IPv6 route even while IPv4 works.

Check DNS from the VPS or another computer:

```bash
dig +short matrix.example.com A
dig +short chat.example.com A
dig +short matrix.example.com AAAA
dig +short chat.example.com AAAA
```

If `dig` is unavailable, install `dnsutils` on Debian/Ubuntu or use `getent hosts matrix.example.com` for a basic lookup.

## 4. Prepare the VPS and Docker

Use a supported Debian or Ubuntu release, keep it patched, and ensure Docker Engine plus the Docker Compose plugin are installed. Installation details change by distribution; use the official Docker instructions for the VPS's OS rather than adding random third-party repositories.

Check Docker:

```bash
docker --version
docker compose version
```

Create a dedicated project directory:

```bash
sudo mkdir -p /opt/matrix/{nginx,element,secrets,letsencrypt}
sudo chown -R root:root /opt/matrix
sudo chmod 750 /opt/matrix
cd /opt/matrix
```

This guide uses root-owned configuration files. If you prefer to administer files as a non-root account, change the ownership intentionally and keep secret files private.

### Check port conflicts before starting

```bash
sudo ss -ltnp | grep -E ':(80|443|8448)\b' || true
docker ps --format 'table {{.Names}}\t{{.Ports}}'
```

If another service already binds the required ports, stop here and plan how the reverse proxy will coexist. Do not kill a process just because it occupies a port; it may be an existing production service.

## 5. Create registration tokens

For a small invited community, token-protected registration is a useful default. It allows people who have a valid token to create accounts without making registration open to every Internet visitor.

Create the token file:

```bash
sudo install -d -o root -g root -m 700 /opt/matrix/secrets
sudo touch /opt/matrix/secrets/registration_tokens.txt
sudo chmod 600 /opt/matrix/secrets/registration_tokens.txt
```

Generate a strong token and append it to the file:

```bash
TOKEN="$(openssl rand -hex 32)"
printf '%s\n' "$TOKEN" | sudo tee -a /opt/matrix/secrets/registration_tokens.txt >/dev/null
printf 'New registration token (share privately): %s\n' "$TOKEN"
unset TOKEN
```

Save the displayed token in a password manager and privately give it to the intended invitee. Repeat the token-generation command to create more tokens. You can use one token for a group or issue a separate token per invitee for easier revocation. Keep your mapping of token to recipient in a separate private note; the token file should contain only the token values, one per line.

Tuwunel's exact environment variable for the file is:

```yaml
TUWUNEL_REGISTRATION_TOKEN_FILE: "/etc/tuwunel/registration_tokens.txt"
```

**Do not confuse it with `TUWUNEL_REGISTRATION_TOKEN`.** The latter configures one static token; the `_FILE` variable tells Tuwunel to read multiple tokens from a file. Tuwunel accepts whitespace-separated tokens, so one token per line is appropriate. The file must be non-empty and readable by the Tuwunel process.

The standard rootful Docker image runs with its image-default user; the official container is minimal and has no shell or utilities such as `sh`, `id`, `ls`, or `cat`. Do not assume `docker exec tuwunel id` will work. Inspect a container's metadata from the host with `docker inspect` when needed. If using rootless Docker or a custom user mapping, review file permissions for that setup.

Registration tokens permit **account creation only**. They are not administrator credentials and do not give a user the ability to log in as another user. Someone who obtains a token may, however, create an account and potentially abuse your server. Keep tokens private, revoke leaked tokens, and monitor usage.

## 6. Obtain a TLS certificate

This tutorial uses Certbot's standalone HTTP-01 validation. It requires the domain names to resolve to the VPS and public TCP port 80 to be reachable. Port 80 must be free while Certbot's temporary web server runs.

Run the following before starting Nginx:

```bash
sudo docker run --rm -it \
  -p 80:80 \
  -v /opt/matrix/letsencrypt:/etc/letsencrypt \
  certbot/certbot:latest certonly \
  --standalone \
  --preferred-challenges http \
  --agree-tos \
  --email admin@example.com \
  --cert-name matrix.example.com \
  -d matrix.example.com \
  -d chat.example.com
```

Replace `admin@example.com` with an address you control. This certificate covers both hostnames, so Nginx can serve either hostname with the same certificate.

Verify the certificate and its Subject Alternative Names:

```bash
sudo openssl x509 \
  -in /opt/matrix/letsencrypt/live/matrix.example.com/fullchain.pem \
  -noout -subject -issuer -dates -ext subjectAltName
```

Look for both `DNS:matrix.example.com` and `DNS:chat.example.com`. The files under `live/` are typically symlinks into Certbot's `archive/` directory; that is normal.

If port 80 is already occupied, do not stop an unrelated service without planning. Use the existing reverse proxy's webroot support or a DNS-01 challenge instead.

## 7. Configure Tuwunel, Element, and Nginx with Docker Compose

Create `/opt/matrix/docker-compose.yml`:

```yaml
services:
  tuwunel:
    image: ghcr.io/matrix-construct/tuwunel:latest
    container_name: tuwunel
    restart: unless-stopped
    expose:
      - "8008"
    volumes:
      - tuwunel_data:/var/lib/tuwunel
      - ./secrets/registration_tokens.txt:/etc/tuwunel/registration_tokens.txt:ro
    environment:
      TUWUNEL_SERVER_NAME: "matrix.example.com"
      TUWUNEL_DATABASE_PATH: "/var/lib/tuwunel"
      TUWUNEL_ADDRESS: "0.0.0.0"
      TUWUNEL_PORT: "8008"
      TUWUNEL_ALLOW_REGISTRATION: "true"
      TUWUNEL_REGISTRATION_TOKEN_FILE: "/etc/tuwunel/registration_tokens.txt"
      TUWUNEL_ALLOW_FEDERATION: "false"
      TUWUNEL_IP_SOURCE: "rightmost_x_forwarded_for"

  element-web:
    image: ghcr.io/element-hq/element-web:v1.12.30
    container_name: element-web
    restart: unless-stopped
    volumes:
      - ./element/config.json:/app/config.json:ro
    expose:
      - "80"

  nginx:
    image: nginx:1.28
    container_name: matrix-nginx
    restart: unless-stopped
    depends_on:
      - tuwunel
      - element-web
    ports:
      - "80:80"
      - "443:443"
      # Optional: uncomment this if federation is enabled in Tuwunel.
      # - "8448:8448"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./letsencrypt:/etc/letsencrypt:ro

volumes:
  tuwunel_data:
```

### What each Compose section means

- `image` selects the software image. For production, pin and deliberately update known-good releases rather than blindly following a moving tag. The Element version shown here was current when this tutorial was prepared; check official releases before deploying later.
- `container_name` provides a predictable name for diagnostic commands. If you remove it, Compose may generate a project-specific container name.
- `restart: unless-stopped` asks Docker to bring services back after a host reboot unless you explicitly stopped them.
- `expose` documents an internal container port for the Compose network. It does **not** publish the port to the host Internet.
- `tuwunel_data` is a named Docker volume. This is the persistent server data. Recreating the container should not delete it.
- The token file is mounted read-only (`:ro`) so Tuwunel can read, but not modify, it.
- Environment variables configure Tuwunel. `SERVER_NAME` is the permanent Matrix identity; `DATABASE_PATH` is inside the container; `ADDRESS=0.0.0.0` lets Tuwunel listen on its container network; `PORT=8008` is its internal HTTP port.
- `ALLOW_REGISTRATION=true` plus `REGISTRATION_TOKEN_FILE` enables token-protected registration. Do not enable the special open-registration safety override for this setup.
- `ALLOW_FEDERATION=false` makes this an isolated/local community by default. See the federation section below to enable it deliberately.
- `IP_SOURCE=rightmost_x_forwarded_for` tells Tuwunel to use the client-IP header inserted by the trusted reverse proxy. This is safe only when public clients cannot bypass that proxy and the proxy overwrites/sets the header correctly.
- Nginx is the only service publishing public web ports. Tuwunel and Element do not publish their own ports.

The tutorial uses the official GHCR Tuwunel image. The `jevolk/tuwunel` Docker Hub image is also published by the project. Before copying this file for a real deployment, check the project's current releases and choose a deliberate image tag.

## 8. Configure Element Web

Create `/opt/matrix/element/config.json`:

```json
{
  "default_server_config": {
    "m.homeserver": {
      "base_url": "https://matrix.example.com",
      "server_name": "matrix.example.com"
    }
  },
  "disable_custom_urls": true,
  "disable_guests": true,
  "brand": "Example Chat"
}
```

Explanation:

- `base_url` is the HTTPS URL Element uses to call the Matrix Client-Server API. With ordinary HTTPS, do not append `:443` because it is the default port.
- `server_name` should match Tuwunel's permanent `TUWUNEL_SERVER_NAME`.
- `disable_custom_urls` removes the option to switch to a different homeserver, simplifying the experience for invitees. Remove it if you intentionally want users to choose other homeservers.
- `disable_guests` disables guest login in the client.
- `brand` changes the visible product label; it does not rebrand every asset automatically.

Element Web is static web content. It does not need its own public port; Nginx serves as the only public entry point.

## 9. Configure Nginx

Create `/opt/matrix/nginx/nginx.conf`:

```nginx
events {}

http {
    upstream tuwunel_backend {
        server tuwunel:8008;
    }

    upstream element_backend {
        server element-web:80;
    }

    # Send ordinary HTTP requests to HTTPS.
    server {
        listen 80;
        listen [::]:80;
        server_name matrix.example.com chat.example.com;
        return 301 https://$host$request_uri;
    }

    # Matrix Client-Server API and (optionally) federation.
    server {
        listen 443 ssl;
        listen [::]:443 ssl;
        # Optional federation listener. Only publish port 8448 if federation is enabled.
        # listen 8448 ssl;
        # listen [::]:8448 ssl;

        server_name matrix.example.com;
        ssl_certificate /etc/letsencrypt/live/matrix.example.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/matrix.example.com/privkey.pem;

        client_max_body_size 20M;

        location / {
            proxy_pass http://tuwunel_backend;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-For $remote_addr;
            proxy_set_header X-Forwarded-Proto https;
            proxy_read_timeout 300s;
            proxy_send_timeout 300s;
        }
    }

    # Element Web interface.
    server {
        listen 443 ssl;
        listen [::]:443 ssl;
        server_name chat.example.com;
        ssl_certificate /etc/letsencrypt/live/matrix.example.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/matrix.example.com/privkey.pem;

        location / {
            proxy_pass http://element_backend;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-For $remote_addr;
            proxy_set_header X-Forwarded-Proto https;
            proxy_read_timeout 300s;
            proxy_send_timeout 300s;
        }
    }
}
```

The two HTTPS server blocks both listen on port 443. Nginx chooses the right block based on the requested hostname: `matrix.example.com` goes to Tuwunel and `chat.example.com` goes to Element. This is why both DNS names can point to the same IP and use the same public port.

`proxy_pass` names the Docker service and internal port. Docker's Compose network resolves `tuwunel` and `element-web` as service hostnames.

The `Host`, `X-Forwarded-For`, and `X-Forwarded-Proto` headers tell the upstream service which hostname, client IP, and original HTTPS scheme were used. Use `$remote_addr` here so Nginx does not blindly append an untrusted client-supplied forwarding chain.

The `client_max_body_size` setting limits request bodies at Nginx. It must be compatible with Tuwunel's `max_request_size`; increasing only the Nginx limit does not increase Tuwunel's limit. If you need larger uploads, configure both to the intended size, following Tuwunel's current configuration reference.

### Federation listener (optional)

For federation, uncomment the `listen 8448 ssl` lines inside the Matrix server block, add `- "8448:8448"` under Nginx's `ports`, and enable federation in Tuwunel. The firewall must allow TCP 8448. Do not enable federation merely because the port is present; decide whether this server should communicate with arbitrary other homeservers.

## 10. Validate and start the services

Check the JSON file parses and Compose accepts the YAML before starting anything:

```bash
cd /opt/matrix
docker compose config
```

Start the stack:

```bash
docker compose up -d
```

Check status:

```bash
docker compose ps -a
```

Check logs:

```bash
docker compose logs --tail 80 tuwunel
docker compose logs --tail 80 element-web
docker compose logs --tail 80 nginx
```

Tuwunel's production container is intentionally minimal and does not include a normal shell or tools such as `id`, `ls`, and `cat`. If commands like `docker exec tuwunel id` fail with “executable file not found,” that is expected for this image. Use host-side Docker inspection instead:

```bash
CID="$(docker compose ps -q tuwunel)"
docker inspect "$CID" --format 'Configured user: {{printf "%q" .Config.User}}'
docker inspect "$CID" --format '{{range .Mounts}}{{println .Source " -> " .Destination " mode=" .Mode}}{{end}}'
docker inspect "$CID" --format '{{range .Config.Env}}{{println .}}{{end}}' | grep -E 'REGISTRATION|SERVER_NAME|DATABASE_PATH|FEDERATION'
```

The last command shows environment-variable names and values configured in Compose, so do not use it if you have also placed actual secret values directly in environment variables. In this token-file setup, the registration tokens are in the mounted file, not in the environment itself.

## 11. Verify the Matrix API and Element separately

### Test the Matrix API

```bash
curl -i https://matrix.example.com/_matrix/client/versions
```

A successful response includes HTTP `200` and JSON listing supported Matrix versions. Seeing JSON at this endpoint is normal; it is an API endpoint, not a web login page.

You can also test Tuwunel's version endpoint:

```bash
curl -i https://matrix.example.com/_tuwunel/server_version
```

### Test Element Web

Open:

```text
https://chat.example.com
```

It should show the Element interface. If it instead displays Tuwunel's plain response, the Element container may be healthy but Nginx may be routing the `chat` hostname to Tuwunel or may be running an old configuration.

Test Nginx syntax and active configuration:

```bash
docker exec matrix-nginx nginx -t
docker exec matrix-nginx nginx -T 2>&1 | grep -nE 'server_name|listen 443|proxy_pass'
```

After changing the mounted Nginx config, test and reload/restart the *Matrix Nginx container* (not some unrelated web proxy):

```bash
docker exec matrix-nginx nginx -t && docker exec matrix-nginx nginx -s reload
```

If reload fails or the process does not load the new config:

```bash
docker restart matrix-nginx
```

Test both hostnames from the VPS while forcing local DNS resolution if useful:

```bash
curl -i --resolve matrix.example.com:443:127.0.0.1 https://matrix.example.com/_matrix/client/versions
curl -i --resolve chat.example.com:443:127.0.0.1 https://chat.example.com/
```

The first should return Matrix JSON. The second should return HTML containing the Element page title. The `--resolve` option tests local Nginx routing while retaining the intended hostname and TLS SNI, making it useful for separating DNS problems from virtual-host problems.

If certificate verification errors appear, check the certificate includes both hostnames:

```bash
openssl s_client -connect matrix.example.com:443 -servername chat.example.com </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates -ext subjectAltName
```

## 12. Register the first account and invite users

With registration enabled and token-file configured:

1. Open `https://chat.example.com`.
2. Choose **Create account**.
3. Enter a username and password.
4. Enter one of the tokens from `registration_tokens.txt` when prompted.
5. The Matrix ID will look like `@username:matrix.example.com`.

Tuwunel grants administrator status to the first registered account by default. Therefore, create the administrator account first, and keep its credentials safe. After that, create a second test account and verify direct messaging and room creation.

The registration token is a registration/admission secret, not a login password and not an admin credential. Anyone holding a valid token can potentially create a new account, so do not publish it openly unless you intentionally want open registration. Existing users continue to log in with their own account credentials/session; they do not keep entering the registration token.

### How multiple tokens work

The correct Compose setting is:

```yaml
TUWUNEL_ALLOW_REGISTRATION: "true"
TUWUNEL_REGISTRATION_TOKEN_FILE: "/etc/tuwunel/registration_tokens.txt"
```

with this volume mount:

```yaml
- ./secrets/registration_tokens.txt:/etc/tuwunel/registration_tokens.txt:ro
```

Do **not** use `TUWUNEL_REGISTRATION_TOKEN` when the intention is to load multiple tokens from a file. The `_FILE` setting is the key point.

The file can contain multiple tokens separated by whitespace; one token per line is easy to maintain. When revoking a token, edit the host file and remove the relevant value, then restart Tuwunel so it rereads the file:

```bash
sudo nano /opt/matrix/secrets/registration_tokens.txt
cd /opt/matrix
docker compose restart tuwunel
```

Revoking a registration token prevents future registrations with that token; it does not delete or disable accounts already created with it. If you want stronger per-person attribution, issue a separate random token per invitee and maintain a private mapping outside the file.

### If you intentionally want fully open registration

That is a different security model. To permit un-tokened public registration, remove the token-file setting and use both `TUWUNEL_ALLOW_REGISTRATION: "true"` and the explicit safety override `TUWUNEL_YES_I_AM_VERY_VERY_SURE_I_WANT_AN_OPEN_REGISTRATION_SERVER_PRONE_TO_ABUSE: "true"`. Anyone who can reach the registration API can then attempt to create an account; a hidden URL is not an access-control mechanism. Prefer token-file registration for an invited community and configure abuse controls/monitoring before enabling open registration.

## 13. Test messages and end-to-end encryption

Create two users on the same homeserver, sign in from two different browser profiles/devices, start a direct chat, and send messages in both directions. This verifies more than the `/versions` test: it checks account authentication, sync, room state, and messaging.

Matrix supports end-to-end encryption (E2EE), but do not assume every room is encrypted merely because HTTPS is enabled. HTTPS protects traffic in transit between client and server. E2EE encrypts message contents on the client so the homeserver stores encrypted events rather than plaintext for that conversation. Start an encrypted direct chat or encrypted room and verify devices/keys as appropriate. Back up recovery keys securely and verify important contacts/devices; losing keys can mean losing access to historical encrypted content.

A compromised VPS can still expose metadata, unencrypted messages, account/session data, logs, and the server database. A compromised VPS hosting Element Web can also serve malicious JavaScript to clients. E2EE is valuable, but it does not protect against a compromised user's device or malicious client code delivered to that device.

## 14. Enable federation only when ready

With federation disabled, accounts on this homeserver can communicate locally, but they cannot normally federate with users on unrelated Matrix homeservers. That is a reasonable initial posture for a private community.

If you later want communication with the wider Matrix network:

1. Decide how federation should be reached. Standard deployments commonly expose TLS on 8448 for federation.
2. Enable the federation setting in Tuwunel (consult the current Tuwunel config for the exact variable spelling/version).
3. Configure Nginx to listen on 8448 with the Matrix hostname and correct certificate.
4. Publish 8448 through Docker and allow it through the host/provider firewall.
5. Verify federation using the current Matrix/Tuwunel documentation and federation testers.
6. Review abuse controls and moderation implications before accepting arbitrary remote homeservers.

If your public HTTPS port is non-standard, the federation route and `.well-known/matrix/server` delegation need extra care. Do not infer that external federation works merely because local Element login works.

## 15. TLS certificate renewal

A Let's Encrypt certificate expires. Since this tutorial obtained it with Certbot's standalone HTTP-01 mode, renewal requires port 80 to be free while Certbot validates the name. But the live Nginx service uses port 80 for HTTP-to-HTTPS redirection.

A simple renewal approach is to stop only this project's Nginx container briefly, run Certbot renewal, then start Nginx again. Create `/opt/matrix/renew-certificate.sh`:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
cd /opt/matrix

docker compose stop nginx
restart_proxy() { docker compose start nginx; }
trap restart_proxy EXIT

docker run --rm -p 80:80 \
  -v /opt/matrix/letsencrypt:/etc/letsencrypt \
  certbot/certbot:latest renew --standalone

docker compose start nginx
trap - EXIT
```

Then:

```bash
sudo chmod 700 /opt/matrix/renew-certificate.sh
sudo /opt/matrix/renew-certificate.sh
```

This causes a brief outage of the Matrix and Element web endpoints during the renewal attempt. Test it manually before scheduling it. A systemd timer/cron job can run the script periodically; Certbot renews only when a certificate is due. For a more polished deployment, use a DNS-01 challenge or a reverse proxy such as Caddy that can manage certificates automatically. If your certificate renewal method differs, do not use this exact script blindly.

## 16. Backups and safe upgrades

The container is replaceable; the persistent data is not. The `tuwunel_data` volume holds the homeserver database and related persistent data. Do not run `docker compose down -v`, and never remove `tuwunel_data` as a troubleshooting step unless you deliberately intend to erase the server.

Before an upgrade:

1. Check Tuwunel's release notes and breaking changes.
2. Make a backup using Tuwunel's supported backup procedure or a consistent filesystem/volume backup strategy.
3. Record the current image tags and Compose file.
4. Pull the intended images and update deliberately.
5. Recreate only the required services.
6. Check the status and logs, then test login and messaging.

Useful routine commands:

```bash
cd /opt/matrix
docker compose ps -a
docker compose logs --tail 100 tuwunel
docker compose logs --tail 100 element-web
docker compose logs --tail 100 nginx
docker compose pull
docker compose up -d
```

Do not assume a casual copy of a live RocksDB directory is consistent. Follow the current Tuwunel documentation for online backups or quiesce the service before a filesystem-level backup. Keep at least one backup off the VPS and periodically test restoring it.

## 17. Security checklist

- Keep the OS, Docker Engine, Tuwunel, Element Web, and Nginx updated on a deliberate schedule.
- Restrict SSH to keys where practical; disable password SSH login only after confirming key access and recovery.
- Allow only the necessary inbound ports in the provider firewall and host firewall: normally 80/443, plus 8448 only if federation is enabled. Do not publicly expose 8008.
- Keep `registration_tokens.txt` private and mount it read-only. Do not publish it in a website, public repository, or screenshots.
- Use strong unique account passwords and protect the administrator account.
- Use E2EE for conversations that need it and verify important devices/contacts. Learn the recovery-key process before depending on encrypted history.
- Keep backups of data and server configuration, and protect backups as sensitive data.
- Avoid logging secrets. Container inspection can reveal environment variables if secrets are placed directly in Compose.
- If the VPS is compromised, assume configuration, unencrypted data, sessions, and the hosted Element Web code may be affected. Isolate it, rotate credentials/secrets, inspect from a trusted environment, and restore from a known-clean backup if needed.

## 18. What self-hosting can and cannot do during Internet blackouts

Self-hosting Element removes the dependency on `app.element.io` for downloading the web interface: users open your own `https://chat.example.com`. It does not make the service available if users have no network route at all to the VPS.

- **Partial restrictions:** if users can still reach your VPS via a working route, direct IP path, permitted network, or proxy/VPN, the service may remain useful.
- **DNS disruption:** a cached DNS result or a prepared alternate route may help in some cases, but certificates, routing, and endpoint reachability still matter. Plan and test this before an outage; don't assume direct-IP HTTPS will work without hostname/SNI handling.
- **Total isolation:** if users cannot reach the VPS at all, an Internet-hosted Matrix server cannot deliver messages between them. Local LAN, mesh, Bluetooth, radio, or store-and-forward systems are separate designs.
- **Client persistence:** users can install a native Matrix client in advance as a second option. A self-hosted web interface is convenient but depends on reaching the server and loading trusted web code.

## 19. Common errors and their meaning

### The browser shows `hewwo from tuwunel woof!` instead of Element

The Matrix hostname is reaching Tuwunel correctly, but the Element hostname is probably using the wrong Nginx virtual host or the running Nginx has not reloaded the edited configuration. Check:

```bash
docker exec matrix-nginx nginx -t
docker exec matrix-nginx nginx -T 2>&1 | grep -nE 'server_name|proxy_pass'
```

You need `matrix.example.com` proxied to `http://tuwunel_backend` and `chat.example.com` proxied to `http://element_backend`. Then reload or restart only this project's Nginx container.

### `Invalid registration token`

Check the spelling of `TUWUNEL_REGISTRATION_TOKEN_FILE`, the read-only mount path, whether the file is non-empty, and whether you restarted Tuwunel after editing it. The correct path in Compose and the environment variable must match exactly:

```text
Host file:      /opt/matrix/secrets/registration_tokens.txt
Container file: /etc/tuwunel/registration_tokens.txt
Variable:       TUWUNEL_REGISTRATION_TOKEN_FILE=/etc/tuwunel/registration_tokens.txt
```

Do not paste token contents into support chats. Verify container mounts/environment from the host with `docker inspect`.

### `docker exec ... id/ls/cat: executable file not found`

The Tuwunel image is minimal and has no shell utilities. Inspect mounts, environment and user metadata with `docker inspect` on the host instead.

### `No such container`

The name used in the command does not match a current container. Run:

```bash
docker compose ps -a
docker ps -a
```

Then use the actual container name or Compose service name.

### Certificate error for `chat.example.com`

The certificate may only contain `matrix.example.com`. Inspect `subjectAltName` and issue a certificate that includes both hostnames. A successful TLS handshake alone does not prove the certificate matches the requested hostname.

### Matrix works locally but federation does not

Local Matrix API success does not prove federation. Check whether federation is enabled, TCP 8448 is published/reachable, the Nginx federation listener has the right certificate, DNS/delegation is correct, and logs show remote federation traffic.

---

## Official references

- Tuwunel project and deployment guide: <https://github.com/matrix-construct/tuwunel>
- Tuwunel example configuration, including `registration_token_file`: <https://github.com/matrix-construct/tuwunel/blob/main/tuwunel-example.toml>
- Tuwunel Nginx reverse-proxy guide: <https://github.com/matrix-construct/tuwunel/blob/main/docs/deploying/reverse-proxy-nginx.md>
- Tuwunel root-domain delegation: <https://github.com/matrix-construct/tuwunel/blob/main/docs/deploying/root-domain-delegation.md>
- Element Web installation: <https://github.com/element-hq/element-web/blob/develop/docs/install.md>
- Element Web configuration: <https://github.com/element-hq/element-web/blob/develop/docs/config.md>
- Docker installation guide: <https://docs.docker.com/engine/install/>
- Certbot documentation: <https://eff-certbot.readthedocs.io/en/stable/>

This is a deployment runbook, not a substitute for checking upstream release notes before an update. Configuration names, image tags, and recommendations can change; consult the linked official documentation when applying this guide to a new installation.
