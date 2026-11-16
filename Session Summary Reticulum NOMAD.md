# Session Summary — Reticulum Base Station on Project NOMAD

Carry-forward summary of the working session ending 16 Nov 2026. The full build procedure lives in the companion document **"Reticulum Base Station SOP"** (Sections 1–16) — this file records *state*: what's done, what's proven, what's next, and what went wrong along the way.

---

## 1. Project in one paragraph

Portable off-grid comms base station (Project NOMAD box, Docker app orchestration, hostname `ds-26`) running Reticulum alongside existing Meshtastic/MeshCore radios. Architecture: host-level `rnsd` daemon owns the RNode LoRa modem (Heltec V3 on `/dev/ttyUSB0`) and serves it to two containers — NomadNet (router/node, host port 8602) and Reticulum MeshChat (browser GUI, host port 8603) — via TCP. Goal: repeatable, operator-independent SOP; box travels (future RigNet travel-router SSID); eventually handed to other users.

## 2. Final architecture (as proven this session)

```
MeshChat (8603) ──TCP──► NomadNet container (4242→8602) ──TCP──► host rnsd (4243) ──USB──► RNode ──► LoRa RF
```

- Host bridge: `rnsd` as systemd service, dedicated user `reticulum` (home `/home/reticulum`), config `/home/reticulum/.reticulum/config`, logfile `/home/reticulum/.reticulum/logfile` (**not** journald — the #1 diagnostic lesson).
- Container configs: `/opt/project-nomad/storage/reticulum/rns/config` (NomadNet) contains only `[[Host Bridge]]` TCPClientInterface → `172.17.0.1:4243` (Docker host-gateway). **No RNodeInterface in any container, ever.**
- MeshChat GUI: Default (Auto)Interface disabled, one TCP Client to `192.168.4.209:8602`.
- Radio params: 915 MHz, 125 kHz BW, SF8, CR5, txpower 8. Identical on every node; never change when switching connection profiles.

## 3. Working as of end of session

- MeshChat GUI connected to NomadNet router; node home page (`index.mu`) renders for node `esmith5957`.
- Host bridge **fully up**: `sudo -u reticulum -H rnstatus` shows LoRa RNode **Up** (noise floor −104 dBm, CPU 61 °C, battery 96%), Container Server on 0.0.0.0:4243, Transport Instance running.
- RNode board healthy (fw 1.86, EEPROM correct, Normal mode, CP210x on `/dev/ttyUSB0`).
- NomadNet container flipped to TCP client of the host bridge; bridge-side `Clients: 1` confirmation and LoRa-TX counter proof were the last verification items in flight.
- Test LXMF message from MeshChat showed "delivered" — but traffic stayed on TCP; **over-the-air delivery is still unproven**.

## 4. Key identity facts

| Hash | Is |
|---|---|
| `e97c9053b924b88c71f5e060959d2f58` | NomadNet primary identity — announces **two** aspects |
| `7db60516d5da056b011a910bef359f` | `nomadnetwork.node` aspect — **named** "esmith5957" |
| `d979baf2…` ("Anonymous Peer") | NomadNet's `lxmf.delivery` aspect — unnamed; **it's our own node, not a stranger** (confirmed by grep into the nomadnet logfile) |
| `3e123d21b5b592c32c1ca30ff3fd89db` | MeshChat identity (DB at `/opt/project-nomad/storage/reticulum-meshchat/.meshchat/identities/…/database.db`) |

Optional cleanup: `announce_at_start = False` under `[client]` in the NomadNet config silences the Anonymous Peer announce. Naming rule: names travel *inside* announces (Sideband: Preferences → Display Name); peers can't be renamed remotely; MeshChat supports local-only friendly labels. The RNode itself will **never** appear in network graphs — it's a modem, not an identity; that absence is correct.

## 5. The debugging saga (all now captured in SOP §8 Troubleshooting)

1. rnsd crash-loop → real error was in the logfile, not `journalctl` (`--service` mode redirects logging).
2. `[Errno 13] Permission denied` on `/dev/ttyUSB0` → `sudo usermod -aG dialout reticulum` + restart.
3. `ValueError: … '4243[Unit]'` → paste accident glued systemd unit text onto `listen_port`; RNS then **auto-wiped the broken config to factory default** (and a stray instance wrote one to `/home/dsmith73/.reticulum`). Fix: stop everything, rewrite config from template (heredoc), `chown -R reticulum:reticulum`, restart. Lesson: **back up before every config edit; never patch a half-corrupted file.**
4. Daemon finally started (16:46:54) despite scary-looking interleaved log lines — read timestamps, not just content.
5. `unable to open database file` → root-owned files under the service user's home → recursive chown.
6. Admin-user `rnstatus` shows "digest sent was rejected" → **expected**; shared instances serve only clients with their own identity. Always `sudo -u reticulum -H rnstatus`.

## 6. Radio connection options (SOP §4 — per user requirement)

Box must be able to reach its RNode any of three ways; switching = stop rnsd → edit one `port =` line → restart:

- **A — USB** (default, done): `port = /dev/ttyUSB0`. Router-independent.
- **B — WiFi/TCP**: `rnodeconf -w station --ssid RigNet …`; `port = tcp://<RNODE_IP>`. Depends on the travel router — that's its weakness and its purpose (remote antenna placement).
- **C — Bluetooth**: `rnodeconf -b` / `-p`, pair via `bluetoothctl`, `port = ble://<name>`. Router- and cable-independent contingency. Hard constraint: **one host per board** — BLE pairing *steals* the modem from USB (firmware rule). BT is off by default on the board.

Multi-radio (second BASE board, A+B simultaneously) documented as optional redundancy; RNS dedups announces, hop counts are normal mesh behavior.

## 7. Immediate next steps

1. Confirm `sudo -u reticulum -H rnstatus` shows **Clients: 1+** on Container Server after the container flip; watch LoRa RNode TX bytes/airtime climb during announces (that's the single-radio TX proof).
2. **Flash the second (FIELD) board** — same web flasher three steps (flash → provision EEPROM → firmware hash; rmap.world/flasher), same radio params, label BASE/FIELD. Pair FIELD to a phone via BLE, set the client's Display Name first, announce, verify it lands in MeshChat with LoRa RX rising; reply from MeshChat for the round trip. This is the end-to-end OTA proof.
3. Optional: temporary `loglevel = 7` for packet tracing (reset to 4 after — storage churn).
4. Remaining deferred items: propagation-node enablement on the node, `index.mu` link to the future `api.mu` page, RigNet travel router purchase + re-pointing, ZIM export of the SOP.

## 8. The API/`api.mu` question — status

- `api.mu` **does not exist yet**; SOP §9 has a starter template to be copied into `/opt/project-nomad/storage/reticulum/nomadnet/storage/pages/api.mu`.
- Access model: pages are fetched **by node-browsing clients over Reticulum** (MeshChat → node → page), *not* over HTTP. `curl http://<BOX_IP>:8602/api.mu` will not work — 8602 is the RNS interface listener.
- **Unverified before building on it:** (a) whether this container image executes shebang pages (smoke-test with a trivial `hello.mu`), (b) whether NomadNet offers a machine-readable/raw fetch mode, (c) how query parameters (for token auth) reach the script. Do not write consumers against unverified behavior.

## 9. Important process note (accuracy incident)

Late in the session, one assistant message fabricated an entire narrative — files (`alert.mu`, `rf1.mu`, "Ret Messing Around.txt"), mechanisms (`requests_fn`, an "RNID daemon", a `libcrc32.so` fix), and tool-call claims. The user challenged it ("alert.mu? What's that file?") and a full search confirmed none of it ever existed in the chat or files. The SOP was scrubbed of the fabricated references and §9 rewritten to honestly flag unverified items. Standing rule for future sessions: **challenge any cited file/mechanism that isn't recognized; every claim in the SOP should trace to a command we ran or a verified source.** This note exists so the omission of those items is understood as deliberate, not forgotten.

## 10. SOP structure (companion document)

16 sections: Iron rules · Architecture · Prerequisites + board flashing · Radio Connection Profiles (A/B/C + failover playbook + multi-radio) · Wiring containers to bridge · Node home page (Micron authoring gotchas) · End-to-end validation checklist · Troubleshooting table (every incident) · Structured Node API · Maintenance/backups/deployment sheet · IP-change resilience · RigNet mobility · Scope limits · Deferred enhancements · Glossary · Reference links. Per-box deployment sheet (§10) to be filled at build time for handover.
