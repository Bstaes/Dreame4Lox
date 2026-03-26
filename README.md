# Dreame4Lox — LoxBerry Plugin

Control your Dreame robot vacuums (X50 Ultra, L40 Ultra, and other miio-compatible models) directly from your Loxone Miniserver via LoxBerry — **no Home Assistant required**.

---

## How It Works

```
Dreame Robot (local LAN)
      ↕  miio protocol (UDP 54321)
LoxBerry (Raspberry Pi)
  ├─ dreame_bridge.py  ← polls robot state every N seconds
  ├─ HTTP server :7778 ← receives commands from Loxone
  ├─ MQTT broker       ← publishes state (optional)
  └─ UDP sender        → pushes state to Loxone Virtual UDP Inputs
```

---

## Installation

1. Download the plugin ZIP from GitHub Releases
2. In LoxBerry → Plugin Administration → Install Plugin → upload ZIP
3. After install, open the Dreame4Lox plugin page

---

## Getting Your Robot Token

The miio protocol requires a **32-character hex token** from your robot.

### Method 1 — Xiaomi Home app (easiest)
1. Add your robot to the **Xiaomi Home** (Mi Home) app (not Dreame app)
2. Install `python-miio`: `pip3 install miio`
3. Run: `miiocli cloud --username YOUR_EMAIL --password YOUR_PASS`
4. Find your device and copy the token

### Method 2 — Packet sniffing (Dreame app)
Use a tool like `miiocli` with `--debug` or proxy the app traffic.
See: https://python-miio.readthedocs.io/en/latest/legacy_token_extraction.html

### Method 3 — LAN packet capture
During robot setup, the token is sent in plaintext over UDP port 54321.
Use Wireshark to capture it.

---

## Plugin Configuration

### 1. Add Robots
In the plugin UI → **Robots** tab:
- Name: friendly name, no spaces (e.g. `X50_Ultra`)
- IP: robot's local IP (set a DHCP reservation!)
- Token: 32-char hex token

### 2. MQTT (optional but recommended)
Configure your MQTT broker (the LoxBerry MQTT plugin can run one).
The bridge publishes each robot's state to `dreame4lox/<name>/<property>`.

### 3. Loxone UDP
Enter your Miniserver IP and UDP port (default 7777).

---

## Loxone Config Setup

### Receiving status (Virtual UDP Input)

Create a **Virtual UDP Input** on port **7777**.
Add **Virtual UDP Input Commands** for each value:

| Command Recognition String | Analog Input |
|---------------------------|--------------|
| `DREAME\i/STATE=\v`       | Robot state (0–9) |
| `DREAME\i/BATTERY=\v`     | Battery % |
| `DREAME\i/CLEANING=\v`    | 1=cleaning, 0=not |
| `DREAME\i/CHARGING=\v`    | 1=charging, 0=not |
| `DREAME\i/ERROR=\v`       | 1=error, 0=ok |
| `DREAME\i/AREA=\v`        | Cleaned area m² |
| `DREAME\i/MAINBRUSH=\v`   | Main brush life % |
| `DREAME\i/SIDEBRUSH=\v`   | Side brush life % |
| `DREAME\i/FILTER=\v`      | Filter life % |

Replace `\i` with e.g. `X50_Ultra` (must match robot name exactly).

**State codes:**
- 0 = idle
- 1 = cleaning
- 2 = returning to dock
- 3 = charging
- 4 = paused
- 5 = error
- 9 = unreachable

### Sending commands (Virtual HTTP Output)

Create a **Virtual HTTP Output** with Address:
```
http://<loxberry-ip>:7778
```

Add **Virtual HTTP Output Commands**:

| Purpose | URL suffix (ON command) |
|---------|------------------------|
| Start cleaning | `/command?robot=X50_Ultra&cmd=start` |
| Dock / return home | `/command?robot=X50_Ultra&cmd=dock` |
| Pause | `/command?robot=X50_Ultra&cmd=pause` |
| Stop | `/command?robot=X50_Ultra&cmd=stop` |
| Locate (beep) | `/command?robot=X50_Ultra&cmd=locate` |
| Clean specific rooms | `/command?robot=X50_Ultra&cmd=start_room&segments=[3,5]` |

Set HTTP Method to **GET**, check **Digital Input**.

---

## Supported Models

All models supported by python-miio's DreameVacuum class, including:
- Dreame X50 Ultra
- Dreame L40 Ultra
- Dreame X40 Ultra
- Dreame L10s Ultra
- Dreame W10 / W10 Pro / W10s Pro
- Dreame S10 / S10 Pro
- And many more — see python-miio docs

---

## Troubleshooting

**Robot shows as "unreachable":**
- Verify IP address is correct and robot is on same subnet as LoxBerry
- Set a DHCP reservation for the robot so its IP doesn't change
- Test connectivity: `python3 -c "from miio import DreameVacuum; v=DreameVacuum('YOUR_IP','YOUR_TOKEN'); print(v.status())"`

**Token is wrong:**
- Token must be exactly 32 hex characters
- Some tools return it base64-encoded — convert if needed

**X50 Ultra / L40 Ultra use Dreame Home app:**
- These newer models may not work with Mi Home app token extraction
- Try Method 2 (packet sniffing) or use the Dreame app + proxy

---

## License

MIT — feel free to contribute, improve, and share!

## Contributing

Submit issues and pull requests on GitHub.
Discuss in the LoxForum: https://www.loxforum.com
