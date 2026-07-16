# Self-hosted LiveKit for Palco

This runs the open-source `livekit-server` — a drop-in replacement for LiveKit
Cloud. `nora-api`'s `PalcoLiveKitService` (`src/modules/ws/modules/palco/palco.livekit.service.ts`)
already reads `LIVEKIT_API_URL` / `LIVEKIT_API_KEY` / `LIVEKIT_API_SECRET` /
`LIVEKIT_URL` from env with no LiveKit-Cloud-specific assumptions, so no
application code changes are needed — only pointing those env vars at your own
server.

## 1. Generate keys (once, per environment)

```bash
docker run --rm livekit/livekit-server generate-keys
```

Copy `.env.example` to `.env` in this folder and paste the output in as
`LIVEKIT_KEYS=<key>: <secret>`. Use a **different** key/secret pair for local
testing vs. the VPS — don't reuse the same one.

## 2. Local test (your own PC)

```bash
cd deployment/livekit
docker compose up
```

Then in `nora-api`'s root `.env`:

```
LIVEKIT_URL=ws://localhost:7880
LIVEKIT_API_URL=http://localhost:7880
LIVEKIT_API_KEY=<the key from your .env>
LIVEKIT_API_SECRET=<the secret from your .env>
```

Start `nora-api` (`npm run dev`) and trigger the Palco "open stage" flow. Watch
the console for `[PalcoLiveKit] === INICIALIZAÇÃO ===` — it should show the
URL/key as defined, not "NÃO DEFINIDA — usando fallback". Since there's no
frontend in this repo, the easiest end-to-end check is
[`livekit-cli`](https://github.com/livekit/livekit-cli):

```bash
lk room list --url http://localhost:7880 --api-key <key> --api-secret <secret>
```

This should show the `"palco"` room once the admin opens the stage.

## 3. VPS deploy (public IP, domain, Nginx Proxy Manager)

```bash
# on the VPS
mkdir -p ~/livekit && cd ~/livekit
# copy docker-compose.yml, livekit.yaml, and your filled-in .env here
docker compose up -d
```

### Nginx Proxy Manager

1. **Proxy Hosts → Add Proxy Host**
   - Domain: `livekit.yourdomain.com`
   - Forward to: the VPS's address and port `7880` (the container's signaling port)
   - **Enable "Websockets Support"** — required, this is how LiveKit signaling
     works
   - SSL tab: request a new Let's Encrypt certificate, enable "Force SSL"

2. **Firewall — do this in addition to the NPM proxy host, not instead of it.**
   NPM/nginx can only front the signaling port (plain HTTP/WS under the hood).
   It **cannot** proxy the media ports — those carry raw WebRTC/RTP, not HTTP.
   Open these directly on the VPS's public IP (e.g. `ufw allow`, and make sure
   the cloud provider's security group/firewall allows them too):
   - `7881/tcp`
   - `20000-20099/udp`

   The RTC UDP range is deliberately `20000-20099` rather than the more
   commonly cited `50000-60000` — on Linux the default ephemeral port range
   (`32768-60999`) overlaps that range, causing intermittent bind failures
   from unrelated local processes. `20000-20099` sits safely below it. If you
   need more concurrent media connections than ~100, widen the range in
   `livekit.yaml`/`docker-compose.yml` (keep it under 32768, or reserve the
   range on the host via `net.ipv4.ip_local_port_range`).

   The `docker-compose.yml` in this folder already publishes those ports
   directly on the host, bypassing NPM entirely — this step is just opening
   the OS/cloud firewall for them.

### Adding TURN later

If some users report audio never connects (typical on networks that block
direct UDP), uncomment the `turn:` block in `livekit.yaml` and open `3478/udp`
(and `5349/tcp` for TURN-over-TLS — note NPM already owns 443, so TURN/TLS
needs its own port and its own cert, e.g. via `certbot`/`acme.sh` for a
`turn.yourdomain.com` subdomain, or LiveKit's own ACME support). Not needed
for the first pass.

## 4. Point nora-api at the VPS instance

In `nora-api`'s `.env` (and the prod secrets — see the repo root's
`deployment/docker/docker-compose.yml` and `.github/workflows/prod.yml`):

```
LIVEKIT_URL=wss://livekit.yourdomain.com
LIVEKIT_API_URL=https://livekit.yourdomain.com
LIVEKIT_API_KEY=<the VPS key>
LIVEKIT_API_SECRET=<the VPS secret>
```
