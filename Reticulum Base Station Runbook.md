# Reticulum Base Station for Project NOMAD

A repeatable guide for adding a Reticulum node to a Project NOMAD box: a two-container setup that provides LoRa radio connectivity, message forwarding/relay for field nodes, and a browser-based GUI — alongside (not replacing) existing Meshtastic/MeshCore radios.

## Architecture

```
                        ┌──────────────────────────────────────┐
                        │            NOMAD BOX                 │
  Phones/PCs on LAN ───▶│  ┌────────────┐      ┌─────────────┐ │
  (Sideband, MeshChat   │  │  MeshChat  │ TCP  │  NomadNet   │ │
   browser clients)     │  │  GUI :8000─┼────▶ │  daemon     │ │
                        │  └────────────┘8602  │  :4242      │ │
                        │                      │      │      │ │
                        │                      │  RNodeInterface│
                        │                      └──────┼──────┘ │
                        └─────────────────────────────┼────────┘
                                                      │ TCP/WiFi
                                               ┌──────┴──────┐
   Remote/roaming nodes  ◀──── LoRa ──────────▶│    RNode    │
   (Sideband in the field)                     │  (Heltec V3)│
                                               └─────────────┘
```

**Division of labor**

| Component | Role |
|---|---|
| NomadNet container (router) | Owns the RNode, runs with `enable_transport = Yes` to **relay traffic between nodes** (the "in the middle" function), serves NomadNet pages, acts as LXMF propagation node |
| MeshChat container (GUI) | Browser dashboard for LXMF messaging, announces, attachments, browsing NomadNet pages. Has **no radio** — connects through the router over TCP |
| RNode (Heltec V3 + RNode firmware) | Pure LoRa modem, joined to WiFi, reached by IP |

**Why TCP instead of USB:** the Supply Depot's custom-app form has no device-passthrough field, and RNode firmware (RNS 1.0.3+) accepts connections over TCP. Flashing RNode firmware onto a WiFi-capable board and joining it to the LAN sidesteps USB entirely.

**Important constraint:** one process owns the RNode. MeshChat must *never* have an RNode interface configured — only a TCP Client pointing at the router. Two processes connecting to the same RNode corrupt each other's radio sessions.

---

## Prerequisites

1. Project NOMAD installed and running (Command Center on port 8080).
2. One Heltec V3 (or other supported board) flashed with **RNode firmware** — this replaces any Meshtastic/MeshCore firmware on that board, so dedicate a board to this role.
3. Both the NOMAD box and the RNode have **static DHCP reservations** on the router. Note both IPs — they are the only moving parts in this setup. This guide uses these placeholders:

   | Placeholder | Meaning | Example from reference build |
   |---|---|---|
   | `<BOX_IP>` | NOMAD box LAN IP | `192.168.4.209` |
   | `<RNODE_IP>` | RNode LAN IP | e.g. `192.168.4.50` |
   | `<BOX_HOSTNAME>` | Box mDNS hostname | e.g. `nomad5.local` |
   | `<RNODE_HOSTNAME>` | RNode mDNS hostname | e.g. `rnodef3b9.local` |
4. Region-correct LoRa parameters (915 MHz US, 868 MHz EU — match what the RNode was provisioned with).

### Flashing the RNode

**Web flasher (recommended, no install):** open **https://rmap.world/flasher** in Chrome/Edge (Web Serial) — pick your board (Heltec LoRa32 v3/v4 are listed explicitly), pick the frequency/model for your region, plug the board in over USB, Connect, Flash. An alternative web flasher exists in Liam Cottle's tooling, linked from the MeshChat project. Firmware is fetched on demand and discarded after flashing; nothing installs on your computer.

CLI alternative:

- `pip install rns --upgrade` then `rnodeconf --autoinstall` from any computer.

Flasher settings that matter:

- Pick the board model **and the correct frequency variant** for your region.
- Leave the device in default **Normal (host-controlled)** mode after flashing.
- Sanity check: status LED blinks briefly every ~2 seconds; the onboard display shows device state.

Then join the RNode to the WiFi network (its own captive configuration, same as any ESP32 device) and reserve its DHCP lease.

---

## Container 1: NomadNet Router

**Add Custom App →**

| Field | Value |
|---|---|
| Docker Image | `ghcr.io/markqvist/nomadnet:master` |
| Display Name | `Reticulum Router (NomadNet)` |
| Category | Custom |
| Port Mapping | container `4242` → host `8602` |
| Volume 1 | `/opt/project-nomad/storage/reticulum/rns` → `/root/.reticulum` |
| Volume 2 | `/opt/project-nomad/storage/reticulum/nomadnet` → `/root/.nomadnetwork` |
| Environment Variables | none |
| Resource Limits | 512 MB / 1 CPU |

Notes:
- Keep host volume paths **under `/opt/project-nomad/storage/`** — NOMAD warns about bind mounts outside its managed storage root.
- Volumes are mandatory: they hold the Reticulum identity, config, and message store. Without them every restart generates a fresh identity.
- The official image runs in daemon mode by default when started detached — no startup arguments needed.
- Docker auto-creates the host directories on first boot; no pre-generation required.

### Configure the router

1. Install the app and let it boot **once** to generate config files (Reticulum cannot be configured via environment variables — INI files only).
2. Edit `/opt/project-nomad/storage/reticulum/rns/config`:
   - Under `[reticulum]`: set `enable_transport = Yes` (this is the packet-forwarding switch — the default has it commented out).
   - Delete the generated `[[Default Interface]]` (AutoInterface) block — it cannot find peers inside the container's bridge network and only adds noise.
   - Replace `[interfaces]` with:

```ini
[interfaces]

  [[LoRa RNode]]
    type = RNodeInterface
    enabled = Yes
    port = tcp://<RNODE_IP>

    frequency = 915000000
    bandwidth = 125000
    spreadingfactor = 8
    codingrate = 5

  [[LAN TCP Server]]
    type = TCPServerInterface
    enabled = Yes
    listen_ip = 0.0.0.0
    listen_port = 4242
```

   Frequency/spreading factor must match the RNode's provisioning and the rest of your network — mismatched nodes talk past each other.
3. Edit `/opt/project-nomad/storage/reticulum/nomadnet/config`: set `enable_node = Yes` and a `node_name`. The generated config file is well-commented and also documents enabling the node as an **LXMF propagation node** — recommended once basics work, so field devices can retrieve queued messages when they come into range.
4. Restart the app from the Command Center.
5. Verify: container logs show the RNode interface come up (prints detected board/firmware) and the TCP server listening. No restart-looping.

---

## Container 2: MeshChat GUI

**Add Custom App →**

| Field | Value |
|---|---|
| Docker Image | `ghcr.io/liamcottle/reticulum-meshchat:latest` |
| Display Name | `Reticulum MeshChat GUI` |
| Category | Custom |
| Port Mapping | container `8000` → host `8603` (next free 8600+ port) |
| Volume | `/opt/project-nomad/storage/reticulum-meshchat` → `/config` |
| Environment Variables | none |
| Resource Limits | 512 MB / 1 CPU |

Notes:
- The `/config` container path is confirmed from the project's `docker-compose.yml` (`meshchat-config:/config`).
- The router container must be running first, or MeshChat has nothing to connect to.

### Configure MeshChat (web UI)

1. Open `http://<BOX_IP>:8603` → **Interfaces**.
2. A built-in **Default Interface** (AutoInterface) will be present and cannot be deleted — **disable it** via its power/menu control and leave it under Disabled Interfaces. Inside the container's bridge network it will never find peers.
3. **+ Add Interface** → **TCP Client** type (**not** TCP Server — this is the #1 misconfiguration):
   - Target host: `<BOX_IP>`
   - Target port: `8602` (the *host-mapped* port of the router container — not 4242, which is only reachable inside the router's own container)
   - A client interface has no listen/bind address fields. If the form asks for one, you picked the wrong type.
4. Restart MeshChat when prompted (interface changes only load at startup).
5. Verify: the interface card shows **Connected** with nonzero TX bytes.

---

## Verification — full chain

1. **Router up:** `rnstatus` (from any machine with RNS installed) or router container logs show RNode interface *Up* with airtime stats, and the TCPServerInterface listening.
2. **GUI connected:** MeshChat interfaces page shows the TCP client Connected.
3. **Announce flow:** within a few minutes, announces appear in MeshChat → Messages (at minimum the NomadNet node's own announce).
4. **LoRa integration:** take a Sideband phone just into/out of LoRa range and announce. It should appear in MeshChat — proving LoRa → RNode → router → TCP → GUI.
5. **Relay ("middle") behavior:** with `enable_transport = Yes`, the router forwards traffic between its interfaces automatically. A third node positioned between the base station and a remote peer will have its traffic relayed; announces arriving on one interface and departing another confirm it.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `OSError: [Errno 99] Cannot assign requested address` in MeshChat logs, interface fails to create | A TCP **Server** interface was added (bind to an IP that doesn't exist inside the container) | Delete it; add a TCP **Client** with `target_host`/`target_port` instead. Via UI, or edit the Reticulum config under `/opt/project-nomad/storage/reticulum-meshchat/` |
| MeshChat "can't launch" after restart | Usually the above; container retries rather than dying | Check `sudo docker logs --tail 50 nomad_custom_reticulum_meshchat_gui` |
| TCP client shows Disconnected | Router container down, wrong `<BOX_IP>`, or firewall blocking 8602 | Verify router logs show TCPServerInterface listening; confirm port mapping 4242→8602 in router app form |
| NOMAD storage-root warning on volume fields | Bind mount outside `/opt/project-nomad/storage/` | Move host paths under the managed root (ideally before first boot; otherwise `mv` the existing dirs to preserve identities) |
| RNode interface errors in router logs | Wrong `<RNODE_IP>`, lost DHCP lease, or mismatched radio parameters | Static reservation on the router; confirm params match `rnodeconf` provisioning |
| Identity/config lost after update | Volume mounts missing or pointed at the wrong container path | Confirm both/all volume rows; container paths are `/root/.reticulum`, `/root/.nomadnetwork`, `/config` |
| Two devices fight over the radio | RNode interface configured in more than one container/process | Exactly one owner (the router). Everything else connects as TCP clients to it |
| Everything breaks after the box moved to a different network | Hard-coded LAN IPs changed with the new network's DHCP | Apply the "names over numbers" pattern from Extending the Base Station below; update only the affected addresses |

## Maintenance

- Both apps update via the Supply Depot like any other; identities, configs, and message stores persist in `/opt/project-nomad/storage/reticulum*/`.
- The only hand-edited files are the Reticulum configs — and only when IPs or radio parameters change.
- If a Reticulum/RNode firmware update ever breaks the radio link, the usual fix is re-running the RNode firmware installer so versions match — not config changes.
- Occasionally check router logs for interface flaps; the RNode's airtime stats there are a good health indicator.

---

## Extending the Base Station: Other Mediums

RNS forwards traffic between *any* interfaces in one config — adding a medium is just adding `[interfaces]` blocks. But evaluate each one honestly first:

### WiFi — already covered

The `TCPServerInterface` **is** the WiFi/Ethernet interface: any Sideband, Ratspeak, or MeshChat client on the LAN connects through it. AutoInterface (the other WiFi mode — zero-config multicast peer discovery) cannot work inside the container's bridge network; it only becomes viable with host networking, which sacrifices the Supply Depot's port management. Not recommended. TCP server covers WiFi comprehensively.

### Bluetooth — deliberately skipped on the base station

Bluetooth in Reticulum serves two *client-side* roles: connecting a phone to its own RNode over BLE, and phone-to-phone "Bluetooth Peer" links. It depends on the host's Bluetooth stack (BlueZ/DBus), which a headless container cannot reach through the Supply Depot form (no device or DBus passthrough). It's also unnecessary here: a phone with Sideband *is* the Bluetooth node. Bluetooth is how field devices talk to their own radios and each other — not a channel the base station needs to offer.

### Meshtastic / MeshCore bridging — not possible yet

There is no native RNS interface for either protocol. Bridging requires a protocol translator, and cross-project interop (e.g. the MeshCore Reticulum-backhaul RFC) is at the experimental-discussion stage, not shipped. The three networks — Meshtastic, MeshCore, Reticulum — run in parallel off the same box, sharing nothing but the shelf. Revisit this section if one of the interop efforts matures into usable software.

---

## Resilience Against IP Changes ("names over numbers")

DHCP reassignment is the weakest point in the setup. Mitigate it with stable names instead of raw addresses wherever possible:

### Interior links (MeshChat → router, router → RNode)

- **MeshChat's TCP Client target:** use `172.17.0.1` (Docker's stable host-gateway address) instead of `<BOX_IP>`. It never touches DHCP and survives any LAN reconfiguration. Tradeoff: it's less self-explanatory to a future reader and only works for containers on the same box.
- **Router's RNode target:** RNS supports hostnames in the port field — `port = tcp://<RNODE_HOSTNAME>` (e.g. `rnodef3b9.local`) is a documented form. Requires mDNS resolution on the network (most home routers/networks support it). With this, the RNode can move networks and keep working.

### Inbound links (clients finding the base station)

In escalating order of robustness:

1. **Static DHCP reservation** for `<BOX_IP>` — fine for a single stable network.
2. **mDNS hostname** — the box is reachable as `<BOX_HOSTNAME>` (e.g. `nomad5.local`); clients can target that instead of an IP, and it survives address changes *within* a network.
3. **Reticulum interface discovery** (RNS 1.5.x+) — on the router's `TCPServerInterface`, set:

   ```ini
   discoverable = Yes
   reachable_on = <BOX_IP>
   ```

   The router then periodically broadcasts its own connection details through the Reticulum network, so peers find it even after readdressing. RNS automatically stamps the interface mode appropriately (TCP servers become `gateway`). Note: this advertises the endpoint to every peer that can hear the announce — review whether that's acceptable for your deployment, and confirm the exact syntax against the RNS manual version in use.

### Full relocation (moving the base station to a new network)

RNS has interface modes for this (`roaming`, `border_roam`), but practically: with mDNS hostnames in place, relocating usually requires touching nothing; without them, update the RNode's IP (or use its `.local` name), the box's reservation, and MeshChat's target once at the new site. Assume one short config-touch per move, then verify with the announce-flow test above.

---

## What this setup does not do

- MeshChat **cannot host** a NomadNet node or act as an LXMF propagation node (planned upstream feature) — that's the router container's job via NomadNet. MeshChat is the client/GUI only.
- RNode firmware replaces Meshtastic/MeshCore on the flashed board; it is not a dual-boot. Use dedicated hardware for each network.
- Meshtastic/MeshCore and Reticulum remain separate networks running in parallel off the same box; there is no native cross-network bridging (community interop efforts exist but nothing standard).
- No Bluetooth interface on the base station (see Extending the Base Station — it belongs on field devices, and containers can't reach the host's Bluetooth stack).

## Reference

- RNode Web Flasher: https://rmap.world/flasher (browser-based, Web Serial; Heltec v3/v4 listed)
- Reticulum: https://reticulum.network — manual, interfaces documentation (including interface discovery)
- NomadNet: https://github.com/markqvist/NomadNet — image `ghcr.io/markqvist/nomadnet`
- RNode Firmware / supported boards: https://github.com/markqvist/RNode_Firmware
- Reticulum MeshChat: https://github.com/liamcottle/reticulum-meshchat — image `ghcr.io/liamcottle/reticulum-meshchat`
- Sideband (mobile client): connects to the base station via TCP at `<BOX_IP>:8602`
