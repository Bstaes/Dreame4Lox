# Dreame4Lox — LoxBerry Plugin

Control and monitor your Dreame robot vacuums from Loxone — directly via the
Dreame Home cloud. No Home Assistant, no Mi Home app, no local token extraction.

Tested with: **X50 Ultra**, **L40 Ultra AE**

---

## How It Works

```
Dreame Robot
    ↕  Dreame Home cloud (Alibaba IoT)
LoxBerry (Raspberry Pi)
  ├─ dreame_cloud.py     → authenticates + polls status every 30s
  ├─ dreame_bridge.py    → bridge daemon (runs as background service)
  ├─ dreame4lox_daemon.sh→ start/stop/status wrapper
  ├─ HTTP server :7778   ← receives commands from Loxone Virtual Outputs
  └─ UDP sender          → pushes state to Loxone Virtual UDP Inputs
```

Status polling is fully reliable. Commands go via the cloud relay and work
when the robot's cloud session is active (cleaning, recently used, or awake).

---

## Installation

1. Download the latest `dreame4lox_vX.X.X.zip`
2. In LoxBerry → **Plugin Administration** → **Install Plugin** → upload ZIP
3. The installer automatically runs `pip3 install requests paho-mqtt`
4. Open the **Dreame4Lox** plugin page

---

## First Setup

### 1. Enter Dreame Home credentials

Go to the **Account** tab:
- **Email** — your Dreame Home app login email
- **Password** — your Dreame Home app password
- **Region** — select **Europe (eu)** for Belgium, Netherlands, Germany, etc.

Click **💾 Save credentials**.

### 2. Discover your robots

Click **🔍 Discover my robots**. The plugin logs in and lists all your vacuums
with their Device IDs. Click **+ Add to config** for each robot.

### 3. Start the daemon

Go to the **Robots** tab → **💾 Save** → click **▶ Start**.

The log should show:
```
INFO  Cloud login OK for you@email.com (eu)
INFO  [X50_Ultra] state=charging battery=100%
INFO  MQTT keepalive active — cloud relay ready for commands
```

---

## Loxone Config Setup

### Receiving status (Virtual UDP Input)

Create a **Virtual UDP Input** on port **7777**.

Add a **Virtual UDP Input Command** for each value you want:

| Command Recognition String   | Value description                          |
|------------------------------|--------------------------------------------|
| `DREAME\i/STATE=\v`          | 0=idle 1=cleaning 2=returning 3=charging 4=paused 5=error |
| `DREAME\i/BATTERY=\v`        | Battery %                                  |
| `DREAME\i/CLEANING=\v`       | 1=currently cleaning, 0=not               |
| `DREAME\i/CHARGING=\v`       | 1=charging/docked, 0=not                  |
| `DREAME\i/ERROR=\v`          | 1=error condition, 0=ok                   |
| `DREAME\i/AREA=\v`           | Cleaned area in m²                        |
| `DREAME\i/TIME=\v`           | Cleaning time in minutes                  |
| `DREAME\i/MAINBRUSH=\v`      | Main brush life % (–1 if unavailable)     |
| `DREAME\i/SIDEBRUSH=\v`      | Side brush life %                         |
| `DREAME\i/FILTER=\v`         | Filter life %                             |

Replace `\i` with your robot's name as configured in the plugin,
e.g. `X50_Ultra_(Gelijkvloers)`. Spaces in the name become underscores.

### Sending commands (Virtual HTTP Output)

Create a **Virtual HTTP Output** with address:
```
http://<loxberry-ip>:7778
```

Add **Virtual HTTP Output Commands** (Method: GET, Digital Input: ✓):

| URL suffix (ON command)                              | Action              |
|------------------------------------------------------|---------------------|
| `/command?robot=X50_Ultra&cmd=start`                 | Start full clean    |
| `/command?robot=X50_Ultra&cmd=dock`                  | Return to dock      |
| `/command?robot=X50_Ultra&cmd=pause`                 | Pause cleaning      |
| `/command?robot=X50_Ultra&cmd=stop`                  | Stop cleaning       |
| `/command?robot=X50_Ultra&cmd=locate`                | Locate (beep)       |
| `/command?robot=X50_Ultra&cmd=start_room&segments=[3,5]` | Clean rooms 3 and 5 |

Replace `X50_Ultra` with your robot's configured name (URL-encoded if it
contains special characters).

---

## Command Reliability

Commands are sent via the Dreame Home cloud relay. The relay forwards
commands to the robot over its Alibaba IoT MQTT session.

**Commands work reliably when:**
- The robot is actively cleaning
- The robot was recently active (within ~30 minutes)
- The robot's dock has recently performed a wash/dry cycle

**Commands may be delayed or fail when:**
- The robot has been sitting idle in its dock for several hours
  (cloud session goes into deep sleep)
- The Dreame Home app has not been opened recently

**Automatic retry:** The plugin retries failed commands up to 3 times with
increasing delays. The log shows exactly what is happening.

**Best practice for Loxone automations:**
- "Leave home" trigger → start cleaning (session is fresh)
- "Return home" trigger → dock (robot is cleaning → session active)
- Avoid scheduling cleans many hours after the last activity without
  first waking the robot via the Dreame app or a manual start

> **Note:** This is a cloud relay limitation of the Dreame Home firmware.
> The robots do not expose any local API (port 54321 and 8123 are disabled
> in Dreame Home firmware). Mi Home app models support local control but the
> X50 Ultra and L40 Ultra are Dreame-Home-only devices.

---

## MQTT (Optional)

If you have the LoxBerry MQTT Gateway plugin installed, enable MQTT in the
plugin settings. Topics published:

```
dreame4lox/<robot_name>/state          → idle/cleaning/charging/etc.
dreame4lox/<robot_name>/battery        → 0-100
dreame4lox/<robot_name>/cleaning       → 0 or 1
dreame4lox/<robot_name>/charging       → 0 or 1
dreame4lox/<robot_name>/state_json     → full status as JSON
```

Send commands by publishing to:
```
dreame4lox/<robot_name>/command
```
Payload: `start`, `stop`, `dock`, `pause`, `locate`
or JSON: `{"command": "start_room", "params": {"segments": [3, 5]}}`

---

## SSH / Manual Commands

```bash
# Daemon control
bash /opt/loxberry/bin/plugins/dreame4lox/dreame4lox_daemon.sh start
bash /opt/loxberry/bin/plugins/dreame4lox/dreame4lox_daemon.sh stop
bash /opt/loxberry/bin/plugins/dreame4lox/dreame4lox_daemon.sh status

# Run in foreground with debug output (great for troubleshooting)
bash /opt/loxberry/bin/plugins/dreame4lox/dreame4lox_daemon.sh debug

# Watch live log
tail -f /opt/loxberry/log/plugins/dreame4lox/dreame4lox_daemon.log

# Test command via HTTP
curl "http://localhost:7778/command?robot=X50_Ultra&cmd=locate"

# Get current status as JSON
curl "http://localhost:7778/status"
```

---

## File Locations

| Path | Description |
|------|-------------|
| `/opt/loxberry/bin/plugins/dreame4lox/` | Python scripts + daemon |
| `/opt/loxberry/config/plugins/dreame4lox/dreame4lox.json` | Configuration |
| `/opt/loxberry/log/plugins/dreame4lox/dreame4lox_daemon.log` | Daemon log |

---

## Supported Models

All models registered in the **Dreame Home** app (dreamehome), including:
- Dreame X50 Ultra / X50 Ultra Complete
- Dreame L40 Ultra / L40 Ultra AE
- Dreame X40 Ultra
- Dreame L10s Ultra
- Any model with `dreame.vacuum.*` in its model string

Models that use the **Xiaomi / Mi Home** app are not the target of this
plugin (those support local miio control via python-miio directly).

---

## Troubleshooting

**Daemon won't start**
```bash
bash /opt/loxberry/bin/plugins/dreame4lox/dreame4lox_daemon.sh debug
```

**"No robots configured"**
Go to Account tab → enter credentials → Discover → add robots → Save.

**Commands not working / 80001 error**
The robot's cloud session is inactive. Open the Dreame Home app on your
phone, start a clean manually, then try the command again. Once the robot
has been active the session stays alive for ~30 minutes.

**Duplicate log lines**
Two daemon instances are running. Fix:
```bash
pkill -f dreame_bridge.py
bash /opt/loxberry/bin/plugins/dreame4lox/dreame4lox_daemon.sh start
```

**Auth failed**
Check your Dreame Home app email/password and region setting.
Note: Mi Home / Xiaomi credentials will not work.

---

## License

MIT — based on reverse-engineered Dreame Home protocol from
[Tasshack/dreame-vacuum](https://github.com/Tasshack/dreame-vacuum) (MIT).

## Contributing

Issues and pull requests welcome.
Discuss in the LoxForum: https://www.loxforum.com
