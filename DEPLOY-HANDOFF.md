# Handoff: deploy mitori.mn to the production server

Written 2026-10-19 for the Claude session that works on our server (the same session/setup that deployed ozzeel.mn). Bataa is the decision maker; ask him before anything that could affect ozzeel.mn.

## What this is

- **Site:** company website of Mitori LLC (Митори ХХК), the legal entity that will publish the ozzeel app on the App Store and Google Play.
- **Why it matters:** Apple's organization enrollment requires a live, working website on the organization's own domain, so mitori.mn must be reachable over HTTPS with real content.
- **Type:** plain static site. No build step, no server code, no database. Serve the repository root as-is.
- **Repo:** `https://github.com/fastroc/mitori` (branch `main`). Local copy on Bataa's PC: `C:\Code\mitori`.

## Facts you can rely on

| Item | Value |
| --- | --- |
| Server | `202.131.237.167` (same box as ozzeel.mn; it answers with `nginx/1.24.0 (Ubuntu)`) |
| Domains | `mitori.mn` (canonical) and `www.mitori.mn` |
| DNS | Managed at Datacom (manage.datacom.mn). A records `@` and `www` → `202.131.237.167`, TTL 7200. Set 2026-10-19; Google DNS already resolves `mitori.mn` to this IP |
| Email (do not touch) | MX `mx.mail.mn` (5) and `mx2.mail.mn` (10), TXT `v=spf1 include:spf.mail.mn ~all`, A `autoconfig` and `autodiscover` → `69.172.250.13`. This is how info@mitori.mn works |
| Site entry | `index.html` at repo root; `404.html` for unknown paths |

## Files in the repo

```
index.html               the site (Mongolian by default, English via the MN/EN toggle or ?lang=en)
404.html                 not-found page
favicon.svg, apple-touch-icon.png, og-image.png
robots.txt, sitemap.xml
assets/mitori-logo-horizontal.svg
assets/fonts/*.woff2     self-hosted Rubik + Golos Text (latin, cyrillic, cyrillic-ext)
README.md, DEPLOY-HANDOFF.md, .gitignore   NOT to be served publicly
```

All asset paths in `index.html` are relative, so the site works from any web root.

## Ground rules

1. **Do not break ozzeel.mn.** Read its nginx config and certificate setup first and mirror its conventions (web root location, file ownership, TLS tool, security headers). Do not edit ozzeel's server block or certificates.
2. Back up any file before you change it. Run `sudo nginx -t` before every `sudo systemctl reload nginx`. Reload, never restart.
3. Do not touch DNS, email or anything at Datacom. If DNS looks wrong, stop and tell Bataa.
4. If the GitHub repo is private and the server cannot clone it, use the same method that ozzeel uses to pull from GitHub (deploy key, gh auth, etc.). If none exists, stop and ask Bataa rather than adding credentials yourself.

## Steps

### 0. Look before you touch

```
dig +short mitori.mn
dig +short www.mitori.mn           # both should print 202.131.237.167
ls -la /etc/nginx/sites-available /etc/nginx/sites-enabled
sudo cat /etc/nginx/sites-available/<ozzeel config>
sudo certbot certificates           # if certbot is what ozzeel uses
```

Note ozzeel's web root pattern (for example `/var/www/ozzeel.mn`) and use the same pattern for mitori.

### 1. Get the code

```
sudo git clone https://github.com/fastroc/mitori.git /var/www/mitori.mn
sudo chown -R <same owner as ozzeel's web root> /var/www/mitori.mn
```

### 2. nginx server block

Create `/etc/nginx/sites-available/mitori.mn`, starting with HTTP only so the certificate can be issued:

```
server {
    listen 80;
    listen [::]:80;
    server_name mitori.mn www.mitori.mn;
    root /var/www/mitori.mn;
    index index.html;
    error_page 404 /404.html;

    location / { try_files $uri $uri/ =404; }
    location ~ /\.            { deny all; }   # .git, .gitignore
    location ~* \.md$          { deny all; }   # README.md, DEPLOY-HANDOFF.md
    location /assets/fonts/   { add_header Cache-Control "public, max-age=31536000, immutable"; }
}
```

Enable and reload:

```
sudo ln -s /etc/nginx/sites-available/mitori.mn /etc/nginx/sites-enabled/mitori.mn
sudo nginx -t && sudo systemctl reload nginx
```

### 3. HTTPS

Use the same tool ozzeel uses. With certbot:

```
sudo certbot --nginx -d mitori.mn -d www.mitori.mn --redirect
```

Then make the final config match these behaviours (certbot covers most of it; add the rest by hand):

- `http://` (both hosts) → 301 to `https://mitori.mn$request_uri`
- `https://www.mitori.mn` → 301 to `https://mitori.mn$request_uri` (canonical URL is `https://mitori.mn/`)
- On the HTTPS block for `mitori.mn`, the same security headers ozzeel sends today: `Strict-Transport-Security: max-age=31536000`, `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: strict-origin-when-cross-origin`, `Content-Security-Policy: frame-ancestors 'none'`
- HTML served with `Cache-Control: no-cache` (as ozzeel does) so updates show immediately

`sudo nginx -t && sudo systemctl reload nginx` after each change.

### 4. Verify (all must pass)

```
curl -sI http://mitori.mn                 | head -3   # 301 → https://mitori.mn/
curl -sI https://www.mitori.mn            | head -3   # 301 → https://mitori.mn/
curl -sI https://mitori.mn                | head -20  # 200, security headers present
curl -s  https://mitori.mn | grep -c "Митори ХХК"     # > 0
curl -sI https://mitori.mn/assets/fonts/rubik-latin-600-normal.woff2 | grep -i cache-control   # immutable
curl -sI https://mitori.mn/og-image.png   | head -1   # 200
curl -sI https://mitori.mn/README.md      | head -1   # 403 or 404
curl -sI https://mitori.mn/.git/config    | head -1   # 403 or 404
curl -sI https://mitori.mn/no-such-page   | head -1   # 404, body is 404.html
curl -sI https://ozzeel.mn                | head -1   # still 200
sudo certbot certificates | grep -A3 mitori           # expiry date
```

### 5. Future updates

```
cd /var/www/mitori.mn && sudo git pull
```

No reload is needed for content changes.

## Report back to Bataa

- Config file path and web root used
- Certificate tool, expiry date and whether auto-renewal is active (`systemctl list-timers | grep certbot` or equivalent)
- The output of the verification commands in step 4
- Anything you changed outside `/etc/nginx/sites-available/mitori.mn`, `/etc/nginx/sites-enabled/mitori.mn` and `/var/www/mitori.mn` (ideally nothing)
