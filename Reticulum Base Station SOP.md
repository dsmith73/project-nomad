# Reticulum Base Station for Project NOMAD — Standard Operating Procedure

A repeatable, operator-independent guide for building a Reticulum base station on a Project NOMAD box: LoRa radio connectivity for field nodes, packet forwarding/relay, LXMF messaging with a browser GUI, and a NomadNet node home page — running alongside (not replacing) existing Meshtastic/MeshCore radios.

This SOP assumes no prior state on the box beyond Project NOMAD itself. Every hand-edited file, command, and verification step is included so two different operators following it independently will converge on the same working build.

**Placeholders used throughout** (record actual values in the deployment sheet for each box):

| Placeholder | Meaning | Reference build |
|---|---|---|
| `<BOX_IP>` | NOMAD box LAN IP | `192.168.4.209` |
| `<BOX_HOSTNAME>` | Box mDNS hostname | e.g. `nomad5.local` |
| `<NODE_NAME>` | NomadNet node display name | e.g. `esmith5957` |
| `<REGION_FREQ>` | Region LoRa frequency | `915000000` (US), `868000000` (EU) |
| `<RNODE_IP>` / `<RNODE_NAME>` | RNode address on WiFi / RNode Bluetooth name | e.g. `192.168.4.50` / `RNode xxxx` |
| `<API_TOKEN>` | Token for the structured Node API (Section 9) | random 32+ hex chars |

---

## 1. Iron rules

Violating any of these causes most failures. They are short enough to memorize.

1. **One owner per RNode board.** Exactly one process — the host `rnsd` service — ever opens a given radio, and **only one transport link is active per board at any instant** (firmware enforces this: RNode 1.x allows a single host connection; a Bluetooth pairing *steals* the modem from USB). The base station as a *whole* may own **multiple boards simultaneously** — see "Multi-radio operation" in Section 4.
2. **Radio parameters are network constants.** Frequency, bandwidth, spreading factor, and coding rate are identical on every node and never change when switching connection profiles — a profile changes only *how the modem reaches rnsd*, never what goes on the air.
3. **INI files only.** Reticulum cannot be configured via environment variables or web UIs. All config is hand-written INI.
4. **Back up before every config edit.** `sudo cp <file> <file>.bak` — a corrupted or wiped config costs far more time than the backup does.
5. **Names over numbers.** Anything that can be made independent of DHCP-assigned IPs, should be.

---

## 2. Architecture

The LoRa modem(s) are owned by a **dedicated Reticulum service running on the host** (`rnsd`), never by a container. Both containers — NomadNet (router/node) and MeshChat (GUI) — connect to that service as **TCP clients** through Docker's stable host-gateway address.

Because the radio is owned by the host service, the box can reach its RNode over **USB, WiFi/TCP, or Bluetooth**, and can switch between them with a one-line config change (Section 4). This is deliberate: the box travels, and the radio link must survive the loss of any single component — including the travel router.

```
                                NOMAD BOX (host)
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  rnsd (systemd service, dedicated user "reticulum")            │
│    ├─ RNodeInterface(s) ← per-board link: USB | WiFi | BLE     │
│    └─ TCPServerInterface  ← 0.0.0.0:4243                       │
│           ↑                                                    │
│     TCP    │                                                   │
│  ┌─────────┴──────────┐     ┌──────────────────────┐           │
│  │ NomadNet container │     │ MeshChat container   │           │
│  │ router/node :4242  │←TCP→│ GUI :8000            │           │
│  │ host port :8602    │     │ host port :8603      │           │
│  └────────────────────┘     └──────────────────────┘           │
│        ↑ 172.17.0.1:4243 (Docker host-gateway)                 │
└────────┼───────────────────────────────────────────────────────┘
         │
  LAN browsers → http://<BOX_IP>:8603 (MeshChat GUI)
  structured API clients → http://<BOX_IP>:8602/page/<API_PAGE>.mu
         │
         │ LoRa RF
  Field nodes (Sideband/Ratspeak phone + field RNode over BLE/USB)
```

| Component | Role |
|---|---|
| `rnsd` on the host | **Sole owner of the RNode board(s)** over their active links. Runs with `enable_transport = Yes` → relays traffic between interfaces (the "in the middle" function). Shares the radio network out over TCP port 4243. |
| NomadNet container | NomadNet node: serves the home page, a structured API page, and provides an LXMF identity/propagation-node option. Has **no radio interface** — TCP client to the bridge. |
| MeshChat container | Browser GUI for messaging, announces, node browsing. Has **no radio interface** — TCP client to the NomadNet container. |
| RNode BASE board(s) | LoRa modem(s), attached to the box by any profile. One minimum; a second is optional true redundancy (Section 4). |
| FIELD board | Validation + field-kit radio; driven by a phone (BLE) or laptop (USB), never by the box. |

**Key property:** the containers never know or care how rnsd reaches the radio. Whether a modem hangs off USB, WiFi, or Bluetooth, `172.17.0.1:4243` is their entire world. Switching radio transports therefore touches exactly one file — the host rnsd config — and restarts one service.

---

## 3. Prerequisites

1. Project NOMAD installed and running (Command Center on port 8080).
2. RNode-capable boards (Heltec V3 recommended — SX1262, 850–950 MHz, OLED):
   - **Minimum:** one BASE + one FIELD board.
   - **Recommended for travel:** a second BASE board (enables multi-radio redundancy, Section 4).
   Flashing replaces Meshtastic/MeshCore firmware — dedicate boards to the role.
3. Static DHCP reservation for the box (`<BOX_IP>`), noted in the deployment sheet.
4. A physically stable USB port free on the box (avoid hubs; note which port — boards re-enumerate when moved). Used for the default USB profile and for recovery.
5. Region-correct LoRa parameters known in advance: `<REGION_FREQ>`, 125 kHz bandwidth, SF8, CR5, TX power per regional law. Identical values on **all** radios in the network.
6. For the WiFi profile: the travel router/SSID the box will normally ride (e.g. **RigNet**). For the Bluetooth profile: a Bluetooth adapter present on the box (built-in or USB dongle) with BlueZ available.

### Part 1 — Flash the RNode boards

**Web flasher (recommended, no install):** open **https://rmap.world/flasher** in Chrome/Edge (Web Serial). Repeat for **each** board:

1. **Flash firmware** — pick the exact board model and the correct **frequency variant** for your region. Progress bars, hashes verified.
2. **Provision EEPROM** — writes device info/checksum/signature. Skipping leaves "Hardware Failure: You need to provision the eeprom".
3. **Set firmware hash** — skipping leaves "Firmware Corrupt: You need to set the firmware hash".

Success signature: `EEPROM provisioned ✓`, `Firmware hash set ✓`, `=== DONE! Device ready as RNode ===`.

Notes:
- A "device signature validation failed / NOT verifiable" warning from the web flasher is a known community-signature quirk — **functionally harmless**; the board works. For a fully verifiable state, re-run `rnodeconf --autoinstall` from a computer with the RNS tools installed.
- Leave both boards in default **Normal (host-controlled)** mode.
- **Heltec V3 LED quirk:** the amber LED is the *battery-charge indicator* and pulses when no battery is attached — not a fault. The status LED blips briefly ~every 2 s when healthy. Trust `rnodeconf -i`, not LEDs.
- Verify each board (from any computer with RNS installed: `pip install rns`): `rnodeconf -i /dev/ttyUSB0` (or `ttyACM0`) → firmware version, `EEPROM checksum correct`, frequency range, `Device mode : Normal (host-controlled)`.
- Label the boards **BASE**, **BASE-2** (if used), and **FIELD**. Test batteries/charging now, not in the field.

### Part 2 — Deploy the containers

Boot **both apps once** before configuring (first boot generates the INI configs edited in Section 5; Reticulum cannot be configured via env vars — INI files only). Volumes are mandatory — they hold identities, configs, and message stores; without them every restart generates a fresh identity. Keep all host paths **under `/opt/project-nomad/storage/`** (NOMAD warns on binds outside its managed root).

**Container 1 — NomadNet Router. Add Custom App →**

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

**Container 2 — MeshChat GUI. Add Custom App →**

| Field | Value |
|---|---|
| Docker Image | `ghcr.io/liamcottle/reticulum-meshchat:latest` |
| Display Name | `Reticulum MeshChat GUI` |
| Category | Custom |
| Port Mapping | container `8000` → host `8603` |
| Volume | `/opt/project-nomad/storage/reticulum-meshchat` → `/config` |
| Environment Variables | none |
| Resource Limits | 512 MB / 1 CPU |

Notes:
- NomadNet's official image runs in daemon mode by default — no startup arguments.
- The `/config` container path for MeshChat is confirmed from the project's `docker-compose.yml`.
- NomadNet's own UI is terminal/LXMF only — **MeshChat is the web GUI**, not NomadNet.

### Part 3 — Build the host bridge (rnsd)

Run on the box as an admin (sudo) user. Steps 3.1–3.3 and 3.5 are identical for **all** radio connection profiles — only the `[LoRa RNode]` block inside the config differs (Section 4).

**3.1 — Dedicated service user** (isolates the daemon and its identity from admin users):

```bash
sudo useradd -r -m -d /home/reticulum -s /usr/sbin/nologin reticulum
```

**3.2 — Serial-port access** (needed for the default USB profile and for any `rnodeconf` maintenance):

```bash
ls -l /dev/ttyUSB* /dev/ttyACM* 2>/dev/null
lsusb                                  # expect 10c4:ea60 (CP210x) for Heltec V3
stat -c '%G' /dev/ttyUSB0              # usually "dialout"; Arch: "uucp"
sudo usermod -aG dialout reticulum
```

Group membership applies at the next service (re)start — no reboot needed.

**3.3 — Install RNS system-wide via pipx** (PEP 668 blocks system pip):

```bash
sudo env PIPX_HOME=/opt/pipx PIPX_BIN_DIR=/usr/local/bin pipx install rns
/usr/local/bin/rnsd --version
```

**3.4 — Write the service user's Reticulum config** with the default **USB profile** active (switching profiles is Section 4). This is the *host* config — entirely separate from any container's. Back up if a file already exists:

```bash
sudo mkdir -p /home/reticulum/.reticulum
sudo tee /home/reticulum/.reticulum/config > /dev/null <<'EOF'
[reticulum]
  enable_transport = Yes
  share_instance = Yes
  shared_instance_port = 37428
  instance_control_port = 37429
  panic_on_interface_error = No

[logging]
  loglevel = 4

[interfaces]
  [[LoRa RNode]]
    type = RNodeInterface
    interface_enabled = Yes
    port = /dev/ttyUSB0
    frequency = 915000000
    bandwidth = 125000
    txpower = 8
    spreadingfactor = 8
    codingrate = 5

  [[Container Server]]
    type = TCPServerInterface
    interface_enabled = Yes
    listen_ip = 0.0.0.0
    listen_port = 4243
EOF
sudo chown -R reticulum:reticulum /home/reticulum/.reticulum
```

Adjust `frequency` to `<REGION_FREQ>` and `txpower` to regional law. Match on all network radios. The radio-link parameters (frequency/bandwidth/SF/CR) are identical in every profile — only the `port` line changes between profiles.

**3.5 — systemd unit:**

```bash
sudo tee /etc/systemd/system/rnsd.service > /dev/null <<'EOF'
[Unit]
Description=Reticulum Network Stack daemon (RNode host bridge)
After=network.target

[Service]
User=reticulum
ExecStart=/usr/local/bin/rnsd --service
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now rnsd
```

**3.6 — Verify the bridge.** Always inspect *as the service user*:

```bash
sudo -u reticulum -H rnstatus
sudo tail -n 20 /home/reticulum/.reticulum/logfile
```

Expected: `RNodeInterface[LoRa RNode] Status : Up` (noise floor, CPU temp) **and** `TCPServerInterface[Container Server/0.0.0.0:4243] Status : Up`.

Operational facts that prevent later confusion:
- In `--service` mode rnsd logs to `/home/reticulum/.reticulum/logfile`, **not** to journald. `journalctl` shows only generic start/stop lines; the logfile holds the real errors.
- `loglevel = 4` logs errors only — routine traffic is invisible. Read the rnstatus traffic/airtime counters instead. Temporarily raise to `loglevel = 7` for packet-level tracing, then **set it back** (level 7 churns storage).
- A different user's `rnstatus` cannot attach to this shared instance ("digest sent was rejected") — that is the per-user identity design, not a fault. Containers are unaffected (plain TCP).

---

## 4. Radio Connection Profiles

The box reaches a BASE board over **USB (A)**, **WiFi/TCP (B)**, or **Bluetooth (C)**. Default: **A**. Commission A first (it is also the fallback of last resort), then prepare B and C as availability allows. The exclusivity constraint is **per board** — each radio has exactly one active link at any instant (rule 1). Different boards may use different profiles simultaneously.

| | A — USB | B — WiFi/TCP | C — Bluetooth |
|---|---|---|---|
| Radio location | Inside/at the box | Remote (window, mast, better antenna) | Same room as box (BLE range) |
| Needs travel router | No | **Yes** | No |
| Survives router failure | Yes | No | Yes |
| Complexity after setup | Lowest | Medium (DHCP/mDNS) | Highest (pairing per host) |
| Best for | Default, failover of last resort | Antenna placement, permanent camp sites | Router-dead contingency, quick swaps |

### Profile A — USB (default)

Already configured in Part 3.4: `port = /dev/ttyUSB0`. Physically secure the cable (strain relief; designated port — the board re-enumerates if moved, see troubleshooting). If the port ever comes up as `ttyACM0`, fix the `port` line.

### Profile B — WiFi/TCP

Put the RNode on the WiFi network and let rnsd reach it by address. Commission while setting up RigNet (or the current camp network):

1. Join the board to WiFi (2.4 GHz only):

   ```bash
   # stop rnsd first — it owns the port over USB
   sudo systemctl stop rnsd
   rnodeconf /dev/ttyUSB0 -w station --ssid "RigNet" --psk "<pass>"
   # return the board to USB ownership for now, or leave it on WiFi if B is active
   sudo systemctl start rnsd
   ```

2. Reserve the board's DHCP lease (`<RNODE_IP>`); note its mDNS name (`<RNODE_NAME>.local`) if the network runs mDNS.
3. Switch the config (with rnsd stopped):

   ```ini
     [[LoRa RNode]]
       type = RNodeInterface
       interface_enabled = Yes
       port = tcp://<RNODE_IP>
       frequency = 915000000
       ...same radio parameters as always...
   ```

   Prefer `tcp://<RNODE_HOSTNAME>.local` over the raw IP when mDNS is available — it survives readdressing (names over numbers). Documented form is the bare `tcp://` address with no port suffix.
4. Restart rnsd and verify with `sudo -u reticulum -H rnstatus` (interface Up, noise floor reported).
5. **Disable the board's other profile** (comment it out) while B is active for that board — never two links to one board at once.

Trade-offs: requires a working network (that's its weakness — this is the profile that *dies with the router*), but frees the radio to sit where the antenna should be rather than where the box is.

### Profile C — Bluetooth

Let rnsd on the host drive the RNode over Bluetooth LE. The host (unlike a container) has full BlueZ access. Requires a BT adapter on the box.

1. Enable Bluetooth on the RNode and put it in pairing mode (with the serial port free):

   ```bash
   sudo systemctl stop rnsd
   rnodeconf /dev/ttyUSB0 -b     # bluetooth on
   rnodeconf /dev/ttyUSB0 -p     # pairing mode (6-digit code on OLED)
   ```

   (On the Heltec itself: short-tap the boot button toggles BT; hold 5+ s for pairing mode.)

2. Pair and trust **the box** from its own Bluetooth stack (as root or via an admin in `bluetooth` group), entering the OLED code when prompted:

   ```bash
   bluetoothctl
   # inside bluetoothctl:
   #   scan on                  → find the device (name shown on the OLED, "RNode xxxx")
   #   pair XX:XX:XX:XX:XX:XX
   #   trust XX:XX:XX:XX:XX:XX  → allows automatic reconnection
   #   quit
   ```

   Linux-side pairing can be fiddly; expect a retry or two, and re-run pairing after firmware re-flashes.

3. Note the device name and switch the config (rnsd stopped):

   ```ini
     [[LoRa RNode]]
       type = RNodeInterface
       interface_enabled = Yes
       port = ble://<RNODE_NAME>
       ...same radio parameters as always...
   ```

4. Restart rnsd and verify as in 3.6. Disable the board's other profile while C is active.
5. **The firmware's one-host rule bites hardest here:** while the box owns the radio over BLE, no phone may pair with it, and vice versa. Never leave the board paired to a phone if it's a BASE board.

Trade-offs: no cables and no network dependency, but ~10 m range, an extra pairing surface, and the most fragile reconnection behavior of the three. Best treated as the contingency when USB is impractical and the router is dead.

### Failover playbook (profile switching)

Switching is always: stop rnsd → edit `port =` in `/home/reticulum/.reticulum/config` (leave exactly one active profile per board) → start rnsd → verify `sudo -u reticulum -H rnstatus`. Back up the config first (`sudo cp config config.bak`).

| Situation | Action |
|---|---|
| Normal operation | Profile A (USB), or A+B simultaneously if a second BASE board is commissioned |
| Radio needs better antenna placement / camp semi-permanent | Switch that board to B |
| Travel router down or network broken | Switch to A (cable) or C (BLE) — both router-independent |
| Second BASE board present | Run A+B concurrently; no switching needed when the router fails — the USB link is already live |
| USB port/board damaged in the field | Switch that board to C if BLE-reachable, else re-seat/replace cable for A |
| Rebuilding after total loss | Any profile works from the config template in Part 3.4 + Section 4 |

Record the active profile(s) and dormant profiles' settings in the deployment sheet so a new operator knows what's prepared.

### Multi-radio operation (optional redundancy)

RNS is a mesh router and handles multiple interfaces natively — a base station owning several boards is a supported design, not a hack. To add a second BASE board, add another interface block with a **unique interface name** to the host rnsd config:

```ini
  [[LoRa RNode 2]]
    type = RNodeInterface
    interface_enabled = Yes
    port = tcp://192.168.4.51        # e.g. BASE-2 on WiFi while BASE-1 stays on USB
    frequency = 915000000
    bandwidth = 125000
    txpower = 8
    spreadingfactor = 8
    codingrate = 5
```

Facts that make this safe and useful:

- **Announce deduplication is built in.** RNS Transport tracks seen packet hashes; when an announce arrives on one interface and is rebroadcast on the others, receivers that already heard it drop the duplicate. There is no announce storm or self-echo loop.
- **Hop-count growth is normal mesh behavior, not a defect.** Every transport node a packet traverses increments its hop count, whether those hops are radios, bridges, or shared-instance links — the reference single-radio build already behaves this way. RNS's hop ceiling is generous (default maximum 128 hops), so relay chains in any realistic deployment are nowhere near exhaustion. Traffic beyond the ceiling expires — that is the designed boundary.
- **Both/all boards must carry identical radio parameters** (rule 2) — two boards on the same channel parameters act as redundant receive paths and a redundant transmit path, which is precisely the point.
- **Per-board exclusivity still holds:** each board gets exactly one `[[...]]` block with exactly one active link. The multi-board win is that *different* boards can hold different profiles at once — e.g. BASE-1 on USB and BASE-2 on RigNet WiFi — giving true no-switch redundancy while traveling: the router dying just takes away one of two live paths.
- Minor cost: slightly higher channel utilization (each transport rebroadcast is a legal retransmission, same as any mesh relay) and one more USB port / power draw. On a base station this is an acceptable trade for link redundancy.

This is an optional extension, not required for a working deployment — single-radio boxes are fully supported by this SOP.

---

## 5. Wire the containers to the bridge

**5.1 — NomadNet container Reticulum config.** Edit `/opt/project-nomad/storage/reticulum/rns/config`. Under `[reticulum]` set `enable_transport = Yes`. Delete the generated `[[Default Interface]]` (AutoInterface — useless inside a container bridge) and any stale sample blocks. Replace the entire `[interfaces]` section:

```ini
[interfaces]

  [[Host Bridge]]
  type = TCPClientInterface
  interface_enabled = Yes
  target_host = 172.17.0.1
  target_port = 4243
```

`172.17.0.1` is Docker's stable host-gateway. **No `RNodeInterface` in any container, ever.** Restart the Router app.

**5.2 — NomadNet node config.** Edit `/opt/project-nomad/storage/reticulum/nomadnet/config`:
- `enable_node = Yes` and `node_name = <NODE_NAME>` (names the `nomadnetwork.node` announce).
- Recommended once basics work: enable the node as an **LXMF propagation node** (the generated config comments document the key) so field devices can retrieve queued messages in range; making the home node the propagation node also silences client warnings.
- Optional: under `[client]`, `announce_at_start = False` stops NomadNet announcing an unnamed LXMF peer that shows up as "Anonymous Peer" in every client. Leave it on if the base station should be message-reachable as a peer.
- Restart the Router app.

**5.3 — MeshChat GUI (browser).** Open `http://<BOX_IP>:8603` → **Interfaces**:
- The built-in **Default Interface** (AutoInterface) can't be deleted — **disable** it.
- **+ Add Interface → TCP Client** (never TCP Server — the classic `Errno 99` misconfiguration): target host `172.17.0.1` (or `<BOX_IP>`), target port `8602` — the router container's *host-mapped* port, not 4242.
- Restart MeshChat when prompted (interfaces only load at startup).

**5.4 — Bridge-side confirmation:**

```bash
sudo -u reticulum -H rnstatus      # Container Server: Clients: 1+
sudo tail -n 20 /home/reticulum/.reticulum/logfile   # TCP connect from 172.17.0.1
```

---

## 6. Node home page

Create `/opt/project-nomad/storage/reticulum/nomadnet/storage/pages/index.mu` (Micron markup) — a welcome/status page for `<NODE_NAME>`. Verify it renders when the node is opened from MeshChat.

**Micron authoring notes** (each of these has cost someone an hour):
- Micron headings use `# ` through `##### ` — there is no `==title==` heading form. Emphasis markup includes bold, italic, underline, strikethrough; code spans use backticks.
- Literal angle brackets **must be escaped** as `&lt;` and `&gt;`, or they begin a link. Raw HTML does not render — Micron escapes it as text.
- Curly braces are Micron's string-format tokens: to display a literal brace, double it (`{{`, `}}`).
- Dynamic executable `.mu` pages (shebang scripts, e.g. a live status page) are an optional later enhancement.

---

## 7. End-to-end validation

A deployment is not "done" until every line passes. Run the full sequence on the **default (USB)** profile; after each future profile switch, rerun steps 1 and 4:

1. **Host bridge:** `sudo -u reticulum -H rnstatus` → every enabled LoRa RNode Up, Container Server listening on 4243.
2. **Router container:** logs show the TCP client connected; no restart-looping.
3. **GUI:** MeshChat interface card **Connected** with nonzero traffic; announces appear (at minimum the node's own announce, named `<NODE_NAME>`).
4. **Radio TX proof:** note LoRa RNode `Traffic ↑` and `Airtime` in rnstatus, trigger announces (restart Router app or announce from MeshChat), re-check — TX bytes and airtime must climb. Strongest single-radio proof that rnsd keys the radio.
5. **Browser check:** the node home page renders from MeshChat.
6. **Over-the-air round trip (needs the FIELD board):** drive it from a laptop (`pip install rns`, minimal config with matching LoRa params) or pair to a phone running Sideband/Ratspeak over BLE. **BT is off by default** — on the Heltec, short-tap the boot button toggles BT, hold 5+ s enters pairing mode (code on OLED); or from a computer with the port free: `rnodeconf /dev/ttyUSB0 -b` then `-p`. Set the client's **Display Name first** (unnamed announces show as "Anonymous Peer"), then announce. It must appear in MeshChat with LoRa **RX** bytes/airtime rising on the base station. Reply from MeshChat to the named peer for the round trip. Start one room apart, then extend range.
7. **Relay behavior:** with `enable_transport = Yes`, a third node positioned between base and remote peer will have traffic forwarded; announces arriving on one interface and departing another confirm it. (With multi-radio, the base station itself demonstrates relay: announce heard on radio 1, rebroadcast on radio 2.)

**BLE warning:** an RNode allows one host connection — Bluetooth pairing *takes* the radio from USB. Pair BLE on the FIELD board only during tests; BASE boards' BLE (if commissioned) is reserved for the box.

---

## 8. Troubleshooting

Every row is a real incident from the reference build (or a documented hazard for the additional profiles).

| Symptom | Cause | Fix |
|---|---|---|
| `rnstatus`: "No shared RNS instance available"; service crash-looping (systemd exit 255/EXCEPTION) | rnsd died at startup; **the real error is in the logfile, not the journal** | `sudo cat /home/reticulum/.reticulum/logfile` |
| `ValueError: invalid literal for int() ... '4243[Unit]'` | Paste accident glued system-unit text onto a config line | Restore the line (`listen_port = 4243`); restore from `.bak`; adopt the backup-before-edit rule |
| "Could not load config file, creating default configuration file…" | RNS **replaced** a broken config with factory defaults, wiping all interfaces | Rewrite the config from Part 3.4 + Section 4; never patch a half-corrupted file |
| `[Errno 13] Permission denied` opening the serial port | Service user not in the serial-port group | `stat -c '%G' /dev/ttyUSB0` → `usermod -aG <group> reticulum` → restart rnsd |
| `[Errno 2]` port missing after it previously worked | Board re-enumerated to another port (cable/USB port moved; ttyACM0 ↔ ttyUSB0) | `ls -l /dev/ttyUSB* /dev/ttyACM*`; `lsusb` → `10c4:ea60`. Fix the `port =` line; designate one physically stable USB port per build |
| "port busy" in `rnodeconf` | Another process owns the port | Stop the owner first (`sudo systemctl stop rnsd`), query, restart |
| "Radio state mismatch … reported radio parameters did not match your configuration … Aborting RNode startup" | Config frequency/BW differs from the board's provisioned parameters | `rnodeconf -i` the board; align config to provisioned values (and regional law) |
| RNode unreachable on `tcp://` (Profile B) | Lost DHCP lease, wrong address, router down | `ping <RNODE_IP>` from the box; confirm reservation; fail over per Section 4 — this profile dies with the router by design |
| `ble://` device not found (Profile C) | Not paired/trusted with the box, BT off on the board, or name changed after re-flash | Re-run `rnodeconf -b/-p`, pair+trust via `bluetoothctl`, confirm the exact device name; verify with `bluetoothctl devices` |
| BLE link flaps or reconnects slowly (Profile C) | Weak signal/power management | Keep the board within a few meters; disable BT power saving on the host; Profile C is a contingency, not the default |
| `unable to open database file` in logfile | Root-owned files under the service user's home (from sudo-era edits) | `sudo chown -R reticulum:reticulum /home/reticulum/.reticulum` |
| `rnstatus` as admin user: "digest sent was rejected" | Expected — the shared instance only serves clients holding its own identity | Inspect as the service user: `sudo -u reticulum -H rnstatus` |
| `OSError: [Errno 99] Cannot assign requested address` in MeshChat | A TCP **Server** interface was added in the GUI | Delete it; add a TCP **Client** instead |
| MeshChat "can't launch" after restart | Usually the above; container retries rather than dying | `sudo docker logs --tail 50 nomad_custom_reticulum_meshchat_gui` |
| "Anonymous Peer" in announce lists | Some client announced without a display name (often NomadNet's internal LXMF peer) | Set Display Name in the client (Sideband: Preferences). NomadNet's peer: `announce_at_start = False` in its config, or name it locally in MeshChat |
| RNode never appears in network graphs | **Correct** — it's a modem, not an LXMF identity; it never announces | Validate via rnstatus interface stats and `rnodeconf -i`, not the graph |
| Bluetooth pairing killed the USB link | RNode 1.x allows ONE host connection per board; BT took it | Disconnect BT; restart rnsd. BASE boards' BLE is reserved for the box; phones pair only with FIELD |
| Amber LED pulsing on Heltec V3 | Battery-charge indicator (no battery attached) — not a fault | Ignore or attach a battery; status LED blips ~every 2 s when healthy |
| Web-flash signature "NOT verifiable" | Known web-flasher community-signature quirk | Harmless; `rnodeconf --autoinstall` for a fully verifiable state |
| Config/identity lost after container update | Volume mounts missing or wrong container paths | Verify all volume rows: `/root/.reticulum`, `/root/.nomadnetwork` (router), `/config` (MeshChat) |
| Everything breaks after a network move | Hard-coded LAN IPs changed | Profiles A/C are network-independent; only MeshChat browser access needs `<BOX_IP>` — re-reserve DHCP or use mDNS |
| `rnodeconf`/`rnsd` "command not found" after reboot | Shell PATH lost user-local bin | Full paths: `/usr/local/bin/rnsd`, `~/.local/bin/rnodeconf`; or `source ~/.bashrc` |

---

## 9. Structured Node API (optional, build later)

Prerequisite: Sections 3–6 complete, node announcing. Then give the node a machine-readable face that also works from a browser.

**Design:** a Micron page with a short shebang script (`/opt/project-nomad/storage/reticulum/nomadnet/storage/pages/api.mu`) that emits key-value lines. One caveat before committing: verify against the NomadNet version in use how executable pages behave in your image, and whether NomadNet offers a machine-readable (non-rendered) fetch mode for page data — do not build tooling that depends on undocumented behavior; test both the rendered and raw fetch paths before writing any consumer.

Starter page — a read-only status snapshot per hit; keep fields finite:

```micron
#!/usr/bin/env python3

import RNS
import time

reticulum = RNS.Reticulum()

uptime = int(time.time() - reticulum.transport_time_started) if reticulum.transport_enabled else 0

print("> Nomad Node API")
print("^ `version`: "+RNS.__version__)
print("^ `uptime`: "+str(uptime))

print("^ `node_hash`:")
print("`"+RNS.prettyhexrep(reticulum.identity.hash)+"`")
```

Publishing rules:

1. Save the page so the container can serve it:

   ```bash
   sudo chown reticulum:reticulum /opt/project-nomad/storage/reticulum/nomadnet/storage/pages/api.mu
   ```

2. Keep the shebang line `#!/usr/bin/env python3` as the **first line** of the file; the script prints Micron text to stdout, which NomadNet serves.
3. Restrict the script's filesystem access; the API should read state, not mutate it.
4. Expose to the LAN only (`:8602` on the host LAN interface). If remote access is ever needed, put an authenticated reverse proxy in front — don't hole-punch.
5. Auth for scripts: a per-deployment token passed as a query parameter (`api.mu?t=<API_TOKEN>`), checked in the script before emitting data; log access and rotate the token.
6. Validate: opening the page in a browser returns formatted text; a Sideband/Ratspeak client can browse it over LoRa too.

**Update cadence:** treat the API like a published contract — additive fields only, note changes in the deployment sheet.

---

## 10. Maintenance

- Containers update via the Supply Depot; identities, configs, and message stores persist under `/opt/project-nomad/storage/reticulum*/`.
- Hand-edited files: the host bridge config (the only file that changes between profiles), the router container's two configs, the systemd unit, and the `index.mu`/`api.mu` pages. Back up (`*.bak`) before every edit.
- After OS/package updates, confirm rnsd is enabled and each board still enumerates on its configured link (USB port present / WiFi lease held / BLE trusted).
- If a firmware/RNS update breaks the radio link, re-run the RNode installer so versions match — not config changes. A re-flash means redoing all **three** steps (flash → provision → hash), **and** re-doing profile preparation: the board forgets WiFi credentials and un-pairs Bluetooth on re-flash.

### Backups & handover

For a box to be reconstructable or handover-ready, keep a **deployment sheet** per box (below) plus copies of: the host bridge config (note which profile is active), `rnsd.service`, both router configs, and the pages. Note that `.reticulum` identity files and any PSKs/API tokens are **credentials** — store backups accordingly and hand them over personally. On handover, walk the new operator through `sudo -u reticulum -H rnstatus`, the logfile-vs-journal trap, and the failover playbook so they can self-diagnose and self-recover.

### Deployment sheet (fill per box)

| Item | Value |
|---|---|
| Build date / operator | |
| Box hostname / LAN IP | |
| Router app | 4242 → 8602 |
| MeshChat app | 8000 → 8603 |
| Active radio profile(s) (A/B/C per board) | |
| BASE board USB port & serial device at build time | e.g. left rear port, `/dev/ttyUSB0` |
| Second BASE board (if any): link + addressing | |
| Profile B: SSID / `<RNODE_IP>` / mDNS name / DHCP reserved | |
| Profile C: RNode BT device name / paired+trusted on box | |
| Firmware version / provisioned frequency | |
| Region LoRa params (freq/BW/SF/CR/TX) | |
| `<NODE_NAME>` | |
| NomadNet node destination hash (from logs) | |
| MeshChat identity hash | |
| Propagation node enabled? | |
| API token set? (record where, not the value) | |
| Travel router SSID (RigNet) / box reservation | |
| Backup location of configs | |
| Handover date / recipient | |

---

## 11. Resilience Against IP Changes

The container→bridge path uses the fixed Docker gateway `172.17.0.1` and never touches DHCP. Radio profiles A and C have no network dependency at all. The only address a relocation touches is `<BOX_IP>` (browser/API access) — and Profile B, if active, rides the travel router (fixed RigNet SSID).

1. **Static DHCP reservation** for the box — baseline.
2. **mDNS:** clients should prefer `<BOX_HOSTNAME>` over the IP.
3. **Reticulum interface discovery** (RNS 1.5.x+): on the bridge's `TCPServerInterface`, optionally set `discoverable = Yes` (plus `reachable_on`) so the node's connection details propagate through the mesh and peers find it after readdressing. Security note: this advertises the endpoint to every peer that hears the announce — review for your deployment, and confirm syntax against the RNS manual version in use.

Field nodes work with or without the base station: RNS is a mesh, not a star.

---

## 12. Mobility: RigNet travel profile

For boxes that travel (RV/mobile deployments), plan a dedicated travel router (e.g. GL.iNet) broadcasting a fixed SSID — **RigNet** — so the base station's network is constant wherever the vehicle parks: box and router get fixed addresses/reservations; field nodes work identically in or out of base-station range with no reconfiguration.

**Redundancy posture while traveling:** run Profile A (USB) as the daily driver; with a second BASE board, run A+B concurrently so router failure degrades nothing (Section 4, Multi-radio). Profile C (Bluetooth) is prepared as the cable-dead fallback. Because switching profiles is a one-line config edit, degraded-mode recovery is minutes, not a rebuild.

---

## 13. What this setup does not do

- MeshChat is a **client/GUI only** — it cannot host a NomadNet node or act as an LXMF propagation node; that's the router container's job.
- RNode firmware replaces Meshtastic/MeshCore on a flashed board — not a dual-boot. Dedicated hardware per network.
- Meshtastic/MeshCore and Reticulum run in parallel off the same box; no native cross-network bridging exists yet (community interop at experimental-discussion stage). Revisit if it matures.
- No Bluetooth interface exposed **to clients** by the base station: BASE boards' BLE (when Profile C is commissioned) is reserved for the box's own rnsd; containers still can't reach BlueZ. Field phones pair with their own FIELD radios.
- A single board carries exactly one link at a time (firmware rule) — redundancy comes from multiple boards or fast profile switching, not from one radio holding two links.
- The microns/file layout here is tuned to the NomadNet container image's daemon-mode behavior (default page under `/root/.nomadnetwork/storage/pages`); alternate NomadNet setups may store pages elsewhere.

## 14. Deferred enhancements

- Remote read-only API page from NomadNet over public carrier networks — parked pending a credible security review of NomadNet's page-serving stack and RNS transports; revisit if upstream hardening lands or a fronting proxy design matures
- ZIM export of this SOP for offline NomadNet library distribution (pandoc → zim-wiki/zim-tools conversion path)
- Meshtastic/MeshCore bridging (blocked on upstream interop maturing)
- Dynamic `.mu` status pages (rnstatus, interface health) as node pages

## 15. Glossary

| Term | Meaning |
|---|---|
| RNS | Reticulum Network Stack — the routing/encryption layer |
| LXMF | Messaging protocol on top of RNS (what carries chat) |
| Announce | Broadcast declaring an identity's presence + name + how to reach it |
| Transport mode | Node forwards traffic for other nodes (our `enable_transport = Yes`) |
| Propagation node | Holds encrypted LXMF messages for offline peers, delivers on reconnect |
| RNode | LoRa hardware running RNode firmware; a modem, never an identity |
| Shared instance | One rnsd process shared by clients on the same user account |
| Host bridge | This design: host rnsd owns the radio(s), containers reach it via TCP |
| Profile | The physical link between a BASE board and rnsd: USB, WiFi/TCP, or BLE |

## 16. Reference

- RNode Web Flasher: https://rmap.world/flasher
- Reticulum manual & interfaces docs: https://reticulum.network
- NomadNet: https://github.com/markqvist/NomadNet — image `ghcr.io/markqvist/nomadnet`
- RNode Firmware / supported boards: https://github.com/markqvist/RNode_Firmware
- Reticulum MeshChat: https://github.com/liamcottle/reticulum-meshchat — image `ghcr.io/liamcottle/reticulum-meshchat`
- Sideband (field client): LAN connects via TCP to `<BOX_IP>:8602`; over the air pairs with its own RNode via BLE (unsigned.io RNode Bluetooth guide)
- RNode CLI config utility (incl. `-w` WiFi, `-b`/`-B`/`-p` Bluetooth controls): https://unsigned.io/software/RNode_Configuration_Utility.html
