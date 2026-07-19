# Withy maintenance flag

This directory is mounted read-only into `proxy-caddy` at `/srv/maintenance`.

When **`withy-maintenance.json`** exists here, Caddy short-circuits
`/withy/api/health` to serve that JSON (a `{"maintenance":true,...}` blob) instead
of proxying to `withy-spring`. That makes the app show the 점검 화면 even while the
Spring server / Postgres are intentionally down.

- **Presence of the file = maintenance ON.** Absence = normal.
- Toggle it with `tossa/scripts/withy-maintenance.sh` (`on` / `off` / `status`).
- No Caddy reload is needed — the `file` matcher re-checks on every request.

Do **not** commit `withy-maintenance.json` — it's a runtime toggle. See
`tossa/docs/maintenance.md` for the full runbook.
