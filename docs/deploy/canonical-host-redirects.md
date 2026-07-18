# Canonical-host redirects (`www` → apex)

**Rule:** every BAS app with a custom domain serves its content on **exactly one** host.
Alternate hosts (`www.`, legacy domains) return a **301** to the canonical apex — they never
serve the app directly.

**Why it's a correctness rule, not cosmetics.** Sessions and CSRF cookies are host-scoped. If
`www.example.org` and `example.org` both serve the app, a user who lands on `www` gets a
*separate* session: signing in on one host leaves them signed out on the other, and a CSRF token
minted on one host is rejected by the other. It also splits OIDC state — the Keycloak client
registers callbacks on the apex only, so a login begun on `www` dead-ends.

Applied to both custom-domain apps on **2026-07-18**; before that, both served `www` directly
(verified by `curl` returning 200 with no `Location`).

## Benefits Navigator — DigitalOcean App Platform ingress rule

App Platform ingress rules support host matching, so this needs **no application code**. The
redirect rule must come **before** the catch-all rule that routes to the `web` component
(rules are evaluated in order):

```yaml
ingress:
  rules:
  - match:
      authority:
        exact: www.vabenefitsnavigator.org
      path:
        prefix: /
    redirect:
      authority: vabenefitsnavigator.org
      redirect_code: 301
  - component:
      name: web
    match:
      path:
        prefix: /
```

Apply with `doctl apps update <app-id> --spec <file>`. Note this lives in the **app spec**, not
in the repo — `app-spec.yaml.template` in the BN repo has no ingress section, so there is nothing
to keep in sync there. Leave `www` in `ALLOWED_HOSTS`; it costs nothing and avoids a 400 if the
rule is ever removed.

Validate a spec edit against the live app before applying — `doctl apps propose --spec <file>
--app <app-id> --output json` echoes back the **parsed** ingress, which is how you confirm the
`authority` match survived rather than being silently dropped.

## KindredAccess — nginx server block on the droplet

⚠️ **The live config is not the repo mirror.** `kindredaccess_files/deploy/nginx-kindredaccess.conf`
documents an idealized layout; the droplet runs certbot's rewrite of it, with different socket
paths and `if ($host = …)` redirect blocks. Read the live file (`grep -n` its block structure)
before patching, or you will clobber working TLS config.

Shape of the change:

1. Drop `www.kindredaccess.org` from the app's `443` `server_name` (leave the apex + bare IP).
2. Add a dedicated `443` block for `www` that reuses the certbot cert (it already covers `www`)
   and does `return 301 https://kindredaccess.org$request_uri;`.
3. Point certbot's port-80 redirects at the apex directly (`https://kindredaccess.org$request_uri`
   instead of `https://$host$request_uri`) so `http://www` collapses in one hop.

Back up first, then gate the reload: `nginx -t && systemctl reload nginx`.

**Access:** the usual SSH key is rejected on this droplet. Use the DO web console — navigate
directly to `cloud.digitalocean.com/droplets/537567342/terminal/ui/?os_user=root` (the dashboard's
console *button* opens a popup that browser automation can't reach). Console sessions drop every
few minutes, so keep each command atomic.

## Verifying

```sh
curl -sI https://www.<domain>/path?q=1 | grep -iE '^(HTTP|location)'
```

Expect `301` and a `Location` on the apex with **path and query preserved**. Then confirm the apex
still serves: `curl -s -o /dev/null -w '%{url_effective} %{http_code}\n' -L https://www.<domain>/accounts/login/`
should land on the apex login page with `200`.

**Known residual:** `http://www.vabenefitsnavigator.org` takes two hops (Cloudflare's http→https,
then the app's www→apex). Collapsing it to one would need a redirect rule in the Cloudflare zone.
Harmless; documented so it isn't re-investigated.

**Do not** remove the `www` entries from the iOS wrappers' Capacitor `allowNavigation` configs —
they are harmless belt-and-braces if a redirect is ever lost.
