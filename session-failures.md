# Session Failure Log

Append-only record of things that went wrong, so patterns become visible across sessions.

---

## Session: 2026-07-18

**Project:** bas-platform (DO App Platform secrets/DB hardening on benefits-navigator-staging)

### Failures

- **Wrong-platform assumption (Keycloak):** proposed "apply the bindable-DB pattern to the Keycloak
  app" and only discovered on inspection that Keycloak is Docker Compose on a droplet, not an App
  Platform app — bindable refs don't exist there → had to walk the recommendation back to the user.
  Lesson: confirm *where* a service runs before recommending a platform-specific pattern; `doctl
  apps list` was a 2-second check.
- **`doctl apps update --format ActiveDeployment.Phase`:** `Error: unknown column` → the app *was*
  updated before the format error; re-queried deployment state with `doctl apps list-deployments`.
  Lesson: an output-format error can mask that the mutation already succeeded — check state, don't
  assume the whole command failed and retry it.
- **`doctl apps spec validate` on a live spec:** rejected with "secret env value must not be
  encrypted before app is created" because the spec contained `EV[...]` blobs → validated a scrubbed
  copy instead and applied the real spec via `update`. Now documented in the
  `do-app-platform-debug` skill.
- **zsh parameter substitution with escaped slashes:** `${URI/\/defaultdb/\/benefits_navigator}`
  produced a mangled URI (`invalid integer value "25060\"` for port) → dropped the backslashes
  (`${URI/defaultdb/benefits_navigator}`). Cost one debugging round-trip inspecting the URI.
- **BSD vs GNU tool assumptions (twice):** `cat -A` → "illegal option" (used `cat -vet`);
  `timeout 120 ...` → "command not found" on macOS. Lesson: this is a Mac — no `timeout`, and
  `cat`/`sed` are BSD variants.
- **Classifier blocks on infra commands (3×):** `doctl databases firewalls append`,
  and `git push origin main` were denied by the auto-mode permission classifier. The firewall one
  the user ran manually; the push is left pending. Recurring friction — worth a permission rule.
- **Connection-attribution probe returned nothing:** attempted to catch in-flight app DB
  connections by user via `pg_stat_activity` while curling the app; the pooled connections closed
  too fast to sample → verified the new DB user a better way (migrate job log + the fact that the
  binding is the app's only `DATABASE_URL` source).

---
