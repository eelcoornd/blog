# blog.orn-dijkstra.no — Setup & Runbook

Hugo site on GitHub Pages with a Cloudflare-managed custom domain.

## Stack
- **Generator:** Hugo (extended)
- **Host:** GitHub Pages, repo `eelcoornd/blog`, build type = GitHub Actions workflow
- **Domain:** `blog.orn-dijkstra.no`
- **DNS:** Cloudflare — `CNAME blog → eelcoornd.github.io`, **DNS only (grey)**
- **TLS:** GitHub-provisioned Let's Encrypt cert (`CN=blog.orn-dijkstra.no`)

## How it deploys
Push to `main` → `.github/workflows` builds with Hugo and deploys to Pages.
Local preview: `hugo server -D` → http://localhost:1313

## The custom-domain / HTTPS gotcha (what bit us)
Symptom: browser said **"connection is not secure"** after proxying the CNAME.

Root cause: the Cloudflare **proxy (orange cloud) was enabled before GitHub had
provisioned the per-domain certificate**. GitHub then served its default
`*.github.io` cert, which doesn't match `blog.orn-dijkstra.no` → "not secure".
The Pages API showed `https_certificate.state: null`.

### Fix (order matters)
1. Cloudflare: set the `blog` CNAME to **DNS only (grey cloud)**.
2. Re-trigger cert provisioning by unsetting then re-setting the custom domain:
   ```bash
   gh api -X PUT repos/eelcoornd/blog/pages -f cname=""
   sleep 5
   gh api -X PUT repos/eelcoornd/blog/pages -f cname=blog.orn-dijkstra.no
   ```
3. Poll until approved:
   ```bash
   gh api repos/eelcoornd/blog/pages -q '.https_certificate.state'
   # authorization_pending → approved  (minutes, up to ~1h)
   ```
4. Enforce HTTPS:
   ```bash
   gh api -X PUT repos/eelcoornd/blog/pages -f https_enforced=true
   ```
5. Verify the live cert matches the host:
   ```bash
   echo | openssl s_client -servername blog.orn-dijkstra.no \
     -connect blog.orn-dijkstra.no:443 2>/dev/null \
     | openssl x509 -noout -subject -issuer
   # subject=CN=blog.orn-dijkstra.no  issuer=Let's Encrypt
   curl -sI https://blog.orn-dijkstra.no | head -1   # HTTP/2 200
   ```

## Cloudflare proxy — the rule
- **Grey (DNS only):** works fully; GitHub serves valid HTTPS directly. **Default choice.**
- **Orange (proxied):** only *after* the GitHub cert is approved, and you **must**
  set **SSL/TLS → Full** (never Flexible/Off, which causes redirect loops or
  "not secure"). Only worth it if you want Cloudflare CDN/WAF caching.

## Handy checks
```bash
# Is DNS grey (GitHub IPs) or proxied (Cloudflare IPs)?
dig +short blog.orn-dijkstra.no        # 185.199.108-111.153 = grey/GitHub

# Which repo owns the domain (avoids "domain already taken")?
for r in $(gh repo list eelcoornd --limit 100 --json name -q '.[].name'); do
  c=$(gh api repos/eelcoornd/$r/pages -q '.cname' 2>/dev/null)
  [ -n "$c" ] && [ "$c" != "null" ] && echo "$r -> $c"
done
```

## Gotcha: "custom domain already taken by another repository"
A domain can only be bound to one repo. If `PUT .../pages -f cname=...` returns
`Invalid cname ... already taken`, find the owning repo with the loop above and
free it there first.
