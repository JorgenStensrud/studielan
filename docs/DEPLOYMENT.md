# Deployment notes

Live at <https://studielan.stens.app>. Hosted on Coolify on `kvitto-prod`
(public IP `77.42.36.80`, also reachable on Tailscale).

App UUID on Coolify: `kswgg8ckcgo444gsowocg00k`. Build pack:
`dockercompose` (Dockerfile + `docker-compose.yaml` at repo root). Volume:
`kswgg8ckcgo444gsowocg00k_studielan-data` mounted at `/app/data`.

Both fqdns currently configured:
- `https://studielan.stens.app` (canonical)
- `https://studielan.77-42-36-80.nip.io` (legacy — 301-redirects via
  middleware in `app/main.py`)

## Auto-deploy is **broken** until source is rebound

The Coolify GitHub App webhook does not currently fire for this app
because `applications.source_id` is `0` (orphaned). Pushes to `main`
will not redeploy automatically — you must force redeploy.

**Fix (one-time, dashboard only — the Coolify v4 API does not expose
this field):**

1. Open the Coolify dashboard
   (`https://kvitto-prod.taild4cf55.ts.net:8443/` over Tailscale).
2. Open the `studielan` application.
3. Configuration → Source → pick `jorgenstensrud-github` (the GitHub
   App with id `11`, same one `hitforhit` uses).
4. Save.

After that, webhooks fire on push, no manual redeploy needed.

Verify:
```bash
curl -sS -H "Authorization: Bearer $COOLIFY_API_TOKEN" \
  https://kvitto-prod.taild4cf55.ts.net:8443/api/v1/applications/kswgg8ckcgo444gsowocg00k \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['source_id'])"
```
Should print `11`.

## Force redeploy (until source is rebound)

Requires Tailscale and a Coolify API token.

```bash
COOLIFY_API_URL=https://kvitto-prod.taild4cf55.ts.net:8443
COOLIFY_API_TOKEN=<from coolify dashboard, or pass dev/coolify/api-token>
curl -H "Authorization: Bearer $COOLIFY_API_TOKEN" \
  -X POST "$COOLIFY_API_URL/api/v1/deploy?uuid=kswgg8ckcgo444gsowocg00k&force=true"
```

## docker-compose apps use a different domain field

Coolify treats compose apps separately from plain Nixpacks apps. The
plain `domains` field on the application returns
`422 "field cannot be used for dockercompose applications"`. Use
`docker_compose_domains` with `{name, domain}` per service:

```bash
curl -H "Authorization: Bearer $COOLIFY_API_TOKEN" -H "Content-Type: application/json" \
  -X PATCH "$COOLIFY_API_URL/api/v1/applications/kswgg8ckcgo444gsowocg00k" \
  -d '{"docker_compose_domains":{"backend":{"name":"backend","domain":"https://studielan.stens.app"}}}'
```

## Domain cutover history

- 2026-05-22: added `studielan.stens.app`, Let's Encrypt cert via
  Traefik HTTP-01, Cloudflare proxy ON.
- 2026-05-22: nip.io → stens.app 301 middleware shipped (PR #6,
  `app/main.py`).

After ~30 days when reinstalls have caught up, drop nip.io from the
app's `docker_compose_domains` config.
