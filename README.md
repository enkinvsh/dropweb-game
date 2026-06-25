# dropweb-game

Remote descriptor for the **«Игровой» (Gaming)** routing mode of the **dropweb** VPN app.

[`game.yml`](./game.yml) is a **generic, infra-free** routing descriptor: it
defines the gaming mode, the `🎮 Gaming` group, the games rule-set and the
routing rules — **games + gaming-adjacent services → VPN (low-latency
Hysteria2), everything else → DIRECT**. The actual **node pool lives only in a
panel header**, never in this repo — so `game.yml` is safe to keep public and
exposes nothing about the infrastructure.

---

## Two headers (infra stays server-side)

The panel sets **two** subscription response headers (auto-collected by the app's
existing `dropweb-*` sweep):

| Header | Holds | Where it lives |
|---|---|---|
| `dropweb-game` | URL of this `game.yml` (generic rules/mode) | public (this repo) |
| `dropweb-game-nodes` | the Hy2 pool, comma-separated region domains | **panel only** (never committed) |

So nothing about your nodes is in any repo. Change the pool → edit the header in
the panel. Change the rules/game-list → edit `game.yml`. Neither needs an app
release.

```
Panel (Remnawave)                          dropweb app
─────────────────                          ───────────
dropweb-game:       <url to game.yml>  ──▶ header present? → show "Игровой"
dropweb-game-nodes: pl.meybz.asia,...      ├─ fetch game.yml (generic rules/mode)
                                           ├─ build Hy2 proxies, one per node
                                           │    in dropweb-game-nodes
                                           │    (password = user's vlessUuid)
                                           ├─ create "🎮 Gaming" group (members = those)
                                           └─ apply rules: games → group, rest → DIRECT
```

---

## Panel setup

Remnawave panel → **Subscription Settings → Custom Response Headers** → add:

```json
{
  "dropweb-game":       "https://raw.githubusercontent.com/enkinvsh/dropweb-game/main/game.yml",
  "dropweb-game-nodes": "pl.meybz.asia,de.meybz.asia"
}
```

- `dropweb-game-nodes` = comma-separated **region POOL** domains (`{cc}.meybz.asia`).
  Each becomes one Hy2 proxy. Add/remove pools here, not in `game.yml`.
- **Disable** the mode: remove the headers. **Gate** to a subset: per-squad /
  response-rules targeting in the panel.

---

## `game.yml` schema (version 1)

| Field | Meaning |
|---|---|
| `version` | Schema version. App rejects unknown majors. |
| `mode.id` / `mode.name` / `mode.icon` | Mode id (`gaming`), UI label (`"Игровой"`), icon hint. |
| `mode.minAppVersion` | App hides the mode if older (forward-safe rollout). |
| `hysteria.template` | **GENERIC** Hy2 proxy params only (`port`, `alpn`, `skip-cert-verify`). **No node domains.** The app builds one `hysteria2` proxy per domain from the **`dropweb-game-nodes`** header. |
| `group` | The proxy-group gaming traffic routes to. Members = the built Hy2 proxies. `url-test` picks the lowest-latency node. |
| `rule-providers` | Remote rule-set(s) the app registers (game-domain lists). |
| `rules[]` | Routing applied **only** in gaming mode. App **replaces** the active rules with these. Targets resolve to `group.name`. Must end with `MATCH,DIRECT`. |

### Per-user password

The built Hy2 proxies carry **no password** from this file or the header. The app
fills each proxy's `password` from the **user's `vlessUuid`** — already present in
the subscription's `vless` proxies. On Remnawave **2.7.4** the Hysteria2 client
auth **equals the user's vlessUuid** (verified e2e). Zero extra credential
plumbing — and no secret ever lives in `game.yml` or the header.

### Pool domains (node = cattle)

`dropweb-game-nodes` entries must be **region pool** domains (`{cc}.meybz.asia`,
e.g. `pl.meybz.asia`) — never a single node (`pl-001.meybz.asia`). A node is
disposable: when its IP dies the node domain points at a corpse; the pool
resolves across all live region nodes and drops dead ones automatically. Adding a
node to a region's pool is transparent — no header change needed.

Hy2 uses a **real LE cert** (unlike Reality, which borrows via `serverNames`), so
each node's cert must SAN its pool domain — the webpanel `certbot` role already
does (verified: `pl-001` SAN = `pl-001.meybz.asia, pl.meybz.asia`). `sni` =
`server` = the pool domain validates out of the box.

### Routing gotcha (for the app implementer)

Apply the rules as **rules** (`RULE-SET … → 🎮 Gaming`). Do **NOT** use mihomo
`mode: global` to force traffic through the group — `global` ignores custom
proxy-groups and falls back to `DIRECT`. Use `mode: rule` + these rules (or the
app's equivalent additive patch path), strictly gated to the gaming WorkMode so
other modes are untouched.

---

## Operations

- **Add / swap Hy2 region pools** (e.g. ones near game datacenters): edit the
  `dropweb-game-nodes` **panel header** (not `game.yml`). Instant, per-sub or global.
- **Tune the game list / routing**: edit `rules[]` / `rule-providers` here (game
  domains from [legiz-ru/mihomo-rule-sets](https://github.com/legiz-ru/mihomo-rule-sets)
  `other/games-direct.yaml` + mihomo `GEOSITE` categories), push.
- **Multipath (future)**: switch `group.type` to `load-balance` or list several
  pools in `dropweb-game-nodes` for redundancy/lower jitter (the ExitLag-style play).
- **Rollback**: remove the headers in the panel — the mode vanishes instantly,
  no app release.

## Security

- **No secrets anywhere.** `game.yml` is generic (public-safe); the node pool is
  in the panel header (server-side); the per-user password = vlessUuid the app
  already holds. Nothing sensitive is committed or transmitted in the clear.
- The app **pins the `game.yml` source** to
  `raw.githubusercontent.com/enkinvsh/dropweb-game` and rejects `dropweb-game`
  pointing elsewhere.
- Node domains come from the **authed panel** (the operator's own, trusted)
  header — never from the public `game.yml`.
- Schema validated; malformed `game.yml` falls back to last-good cache.

---

## Related

- Server side (Hy2 inbound on nodes): `webpanel/docs/plans/2026-06-25-hysteria2-patch-playbook.md`.
- App side (WorkMode.gaming, header parsing, injection): `dropweb-app/docs/plans/2026-06-25-gaming-mode.md`.
