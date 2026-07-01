---
name: cloudflare-tunnel
description: Expose a local dev server port publicly using a Cloudflare quick tunnel (no account needed)
---

# Expose a port via Cloudflare Tunnel

Use this when running inside a GitHub Codespace (or any remote environment) and you need a public URL to test a local server externally.

## Steps

1. Download `cloudflared` if not already present:
   ```bash
   curl -s https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 -Lo /tmp/cloudflared
   chmod +x /tmp/cloudflared
```
   

2. Start the tunnel pointing at your local port (e.g. 3100):
   ```bash
   /tmp/cloudflared tunnel --url http://localhost:3100 > /tmp/cloudflared.log 2>&1 &
```

3. Wait ~8 seconds, then extract the public URL:
   ```bash
   grep -o 'https://[^ ]*trycloudflare.com' /tmp/cloudflared.log | head -1
```   


## Notes

- The URL is randomly generated each run (e.g. https://marsh-requirements-expert-offered.trycloudflare.com)
- No Cloudflare account required — this uses trycloudflare.com quick tunnels
- No uptime guarantee; tunnel lives as long as the process runs
- If using Next.js dev mode, add the tunnel domain to allowedDevOrigins in next.config.ts so RSC and HMR requests aren't blocked:
  ts
  const nextConfig = {
    allowedDevOrigins: ["*.trycloudflare.com"],
  };
    Then restart the dev server.
- Cloudflare shows a one-time interstitial on first visit — users just click through it
```

