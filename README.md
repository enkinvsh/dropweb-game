# dropweb-game

Remote descriptor for the **«Игровой» (Gaming)** routing mode of the **dropweb** VPN app.

This repo hosts a single file — [`game.yml`](./game.yml) — that the dropweb app
fetches and applies to turn on a gaming-optimized routing mode: **games +
gaming-adjacent services → VPN (via a low-latency Hysteria2 tunnel), everything
else → DIRECT**. Behaviour is changed by editing this file, **without an app
release**.

---

## How it works (header-driven, zero hardcode)

```
Panel (Remnawave)                 dropweb app
─────────────────                 ───────────
Subscription response header  ──▶ reads `dropweb-game` header
  dropweb-game: <raw game.yml>      │
                                    ├─ header present? → show "Игровой" mode
                                    ├─ fetch game.yml (this repo)
                                    └─ apply patch to the Mihomo config:
                                         • inject Hysteria2 proxies (password = vlessUuid)
                                         • create the "🎮 Gaming" group
                                         • replace rules: games → group, rest → DIRECT
```

The provider controls rollout entirely from the panel: **add the header → the
mode appears; remove it → the mode disappears.** Per-subscription or global.

---

## Panel setup (the header)

Remnawave panel → **Subscription Settings → Custom Response Headers** → add:

```json
{
  "dropweb-game": "https://raw.githubusercontent.com/enkinvsh/dropweb-game/main/game.yml"
}
```

Same mechanism as the existing `fallback-url` header. The app auto-collects any
`dropweb-*` header (no whitelist), so no app change is needed to wire it.

To **disable** the gaming mode for everyone: remove the header. To gate it to a
subset of users: use the panel's per-squad / response-rules targeting.

---

## `game.yml` schema (version 1)

| Field | Meaning |
|---|---|
| `version` | Schema version. App rejects unknown majors. |
| `mode.id` | Internal id (`gaming`). |
| `mode.name` | UI label (`"Игровой"`). |
| `mode.icon` | Icon hint; app maps it to its icon set. |
| `mode.minAppVersion` | App hides the mode if older (forward-safe rollout). |
| `hysteria[]` | Hy2 endpoint templates the app injects as proxies. **No `password`** — see below. Fields: `name`, `server`, `port`, `sni`, `alpn`, `skip-cert-verify`. |
| `group` | The proxy-group gaming traffic routes to. Members = injected Hy2 proxies. `url-test` picks the lowest-latency node. |
| `rule-providers` | Remote rule-set(s) the app registers (e.g. game-domain lists). |
| `rules[]` | Routing applied **only** in gaming mode. App **replaces** the active rules with these. Targets resolve to `group.name`. Must end with `MATCH,DIRECT`. |

### Per-user password (important)

Hy2 endpoints carry **no password** in this file. The app fills each proxy's
`password` from the **user's `vlessUuid`** — already present in the
subscription's `vless` proxies. On Remnawave **2.7.4** the Hysteria2 client auth
**equals the user's vlessUuid** (verified end-to-end). So injection needs zero
extra credential plumbing.

### Routing gotcha (for the app implementer)

The rules are applied as **rules** (`RULE-SET … → 🎮 Gaming`). Do **NOT** use
mihomo `mode: global` to force traffic through the group — `global` ignores
custom proxy-groups and falls back to `DIRECT`. Use `mode: rule` + the rules
here (or the app's equivalent additive patch path), strictly gated to the
gaming WorkMode so other modes are untouched.

---

## Operations

- **Add / swap Hy2 nodes** (e.g. ones near game datacenters): edit `hysteria[]`,
  push. All clients pick it up on next subscription/config refresh.
- **Tune the game list**: edit `rules[]` / `rule-providers` (game domains come
  from [legiz-ru/mihomo-rule-sets](https://github.com/legiz-ru/mihomo-rule-sets)
  `other/games-direct.yaml` + mihomo `GEOSITE` categories).
- **Multipath (future)**: switch `group.type` to `load-balance` or add several
  Hy2 nodes for redundancy/lower jitter (the ExitLag-style play).
- **Rollback**: remove the `dropweb-game` header in the panel — the mode vanishes
  instantly, no app release.

## Security

- The app **pins the source** to `raw.githubusercontent.com/enkinvsh/dropweb-game`
  (this repo). It must reject `dropweb-game` headers pointing elsewhere.
- `game.yml` may add **provider-owned** Hy2 proxies + rules/groups — never
  arbitrary proxies from unpinned sources (would be a MITM vector).
- Schema is validated; malformed files fall back to last-good cache.

---

## Server side

The Hysteria2 inbound that these endpoints connect to is provisioned on the
nodes by the webpanel Ansible repo. See its spec:
`webpanel/docs/plans/2026-06-25-hysteria2-patch-playbook.md`
(adds a Hy2 inbound coexisting with Reality on an existing exit node).

App side: `dropweb-app/docs/plans/2026-06-25-gaming-mode.md`.
