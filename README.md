# Batocera Home Assistant MQTT Bridge (`mqtt_events`)

A native Batocera Linux background service that bridges **EmulationStation** events, **scraped game artwork** (Box Art & Marquees), **system telemetry**, and **remote power controls** directly into **Home Assistant** using MQTT Auto-Discovery—with zero YAML configuration required in Home Assistant.

*(Originally based on the script by `kit` from the Batocera Discord, enhanced with improved artwork discovery, `gamelist.xml` full-title resolution on launch, Favorites support, and Raspberry Pi `cpu_thermal` support.)*

---

## Features

* **Zero-Config Home Assistant MQTT Auto-Discovery:** Automatically registers a device named after your Batocera hostname (default: `retrogaming`) with all sensors, images, and buttons.
* **Live Gameplay Tracking:**
  * Reports whether a game is actively running (`binary_sensor.<host>_playing`).
  * Reports the active emulator/system (`sensor.<host>_system`) and full game title (`sensor.<host>_now_playing`).
  * Resolves the full `<name>` from `gamelist.xml` on `game-start` (so MAME/FBNeo games display `"Teenage Mutant Ninja Turtles (World 4 Players, version X)"` instead of the raw ROM slug `"tmnt"`), even when launched from the **Favorites** collection.
* **Live Box Art & Marquee Image Entities:**
  * Publishes raw image bytes to `image.<host>_box_art` and `image.<host>_marquee`.
  * Searches `-thumb`, `-box`, `-image`, `-marquee`, `-wheel`, `-logo`, `media/images/`, `downloaded_images/`, and parses `<thumbnail>`, `<image>`, and `<marquee>` paths from `gamelist.xml`.
  * Automatically converts `.webp` artwork to `.png` on the fly if a converter (`dwebp`, `magick`, `convert`, or `ffmpeg`) is present.
* **System Telemetry (every 5 seconds):**
  * **CPU Temperature** (`sensor.<host>_cpu_temp`) — Supports both x86 (`coretemp`) and Raspberry Pi / ARM (`cpu_thermal`) hardware monitors with median sampling.
  * **CPU Load** (`sensor.<host>_cpu_load`) — 1-minute load average.
  * **RAM Used** (`sensor.<host>_ram_used`) — Memory used in `MB`.
  * **Disk Used** (`sensor.<host>_disk_used`) — `/userdata` partition usage in `%`.
  * **Uptime** (`sensor.<host>_uptime`) — System uptime in seconds.
* **Remote Controls:**
  * **Reboot** (`button.<host>_reboot_<host>`) and **Shutdown** (`button.<host>_shutdown_<host>`) buttons in Home Assistant.

---

## Home Assistant Entities Created

Assuming your Batocera hostname is `retrogaming`, the script automatically creates the following entities under a single MQTT device:

| Entity ID | Type | Description |
| :--- | :--- | :--- |
| `binary_sensor.retrogaming_playing` | Binary Sensor | `on` (`Playing`) when a game is running; `off` (`Menu`) when in EmulationStation |
| `sensor.retrogaming_now_playing` | Sensor | Name of the currently selected or running game |
| `sensor.retrogaming_system` | Sensor | Short name of the current system/emulator (e.g., `fbneo`, `snes`, `psx`) |
| `image.retrogaming_box_art` | Image | Live Box Art / Thumbnail / Flyer for the selected or running game |
| `image.retrogaming_marquee` | Image | Live Marquee / Wheel Logo / Composite image for the game |
| `sensor.retrogaming_cpu_temp` | Sensor | CPU temperature in `°C` (with companion `cpu_temp_source` and `cpu_temp_status` sensors) |
| `sensor.retrogaming_cpu_load` | Sensor | 1-minute CPU load (`%`) |
| `sensor.retrogaming_ram_used` | Sensor | RAM used (`MB`) |
| `sensor.retrogaming_disk_used` | Sensor | `/userdata` disk usage (`%`) |
| `sensor.retrogaming_uptime` | Sensor | System uptime (`s`) |
| `button.retrogaming_reboot_retrogaming` | Button | Remotely reboots the Batocera machine |
| `button.retrogaming_shutdown_retrogaming` | Button | Remotely shuts down the Batocera machine |

---

## Configuration

Before starting the service, open `mqtt_events` in a text editor and update the **Broker / topics** section at the top of the file (lines 14–30):

```bash
######################################
# Broker / topics
######################################
MQTT_HOST="YOUR_MQTT_BROKER_IP"       # Home Assistant / Mosquitto Broker IP (e.g., "192.168.1.100")
MQTT_PORT="1883"                      # Broker Port (usually 1883)
MQTT_USER="YOUR_MQTT_USERNAME"        # MQTT Broker Username
MQTT_PASSWORD="YOUR_MQTT_PASSWORD"    # MQTT Broker Password

DISCOVERY_PREFIX="homeassistant"      # Home Assistant MQTT discovery prefix (default: "homeassistant")
HOSTNAME="$(hostname)"                # Uses your Batocera hostname (e.g., "retrogaming")
MQTT_BASE_TOPIC="batocera/${HOSTNAME}"

# Telemetry
TELEMETRY_ENABLE=1                    # 1 = enable CPU/RAM/Disk/Uptime reporting, 0 = disable
TELEMETRY_INTERVAL=5                  # Seconds between telemetry updates

# Media behavior
RETAIN_MEDIA_BYTES=1                  # Retain last published image bytes on broker
PUBLISH_IMAGE_B64=1                   # Also publish base64 image string to *_b64 topic
```

### What You Need to Fill In:
1. **`MQTT_HOST`**: The IP address or hostname of your MQTT broker (usually your Home Assistant IP address if using the Mosquitto Broker add-on).
2. **`MQTT_PORT`**: Your MQTT port (`1883` by default for unencrypted MQTT).
3. **`MQTT_USER`**: The username configured in your MQTT broker / Home Assistant user.
4. **`MQTT_PASSWORD`**: The password for that MQTT user.

---

## Prerequisites on Batocera

Batocera requires the `mosquitto` client tools (`mosquitto_pub` and `mosquitto_sub`) to communicate with your MQTT broker.

Check if `mosquitto_pub` is already installed via SSH:

```bash
which mosquitto_pub
```

If it is not installed, you can install it via Batocera's `pacman` package manager (if supported on your Batocera build) or check your Batocera version's packages:

```bash
pacman -Sy mosquitto
```

---

## Installation Instructions

### Step 1: Copy `mqtt_events` to Batocera
1. Enable SSH on your Batocera machine (usually enabled by default; default user is `root` and default password is `linux`).
2. Create the `/userdata/system/services` directory if it doesn't already exist:

```bash
mkdir -p /userdata/system/services
```

3. Copy `mqtt_events` to `/userdata/system/services/mqtt_events` (using `scp`, WinSCP, Cyberduck, or the `\\BATOCERA\share\system\services\` SMB network share):

```bash
scp mqtt_events root@<BATOCERA_IP>:/userdata/system/services/mqtt_events
```

*(Note: If you edited the file on Windows, make sure line endings are LF (`Unix`) and not CRLF (`Windows`). You can run `dos2unix /userdata/system/services/mqtt_events` or `sed -i 's/\r$//' /userdata/system/services/mqtt_events` on Batocera just in case.)*

### Step 2: Make the Script Executable
SSH into Batocera and set executable permissions:

```bash
chmod +x /userdata/system/services/mqtt_events
```

### Step 3: Enable and Start the Service
You can enable and start the service either from the **command line** or through the **Batocera UI**:

**Option A — Via SSH Command Line:**

```bash
batocera-services enable mqtt_events
batocera-services start mqtt_events
```

**Option B — Via Batocera EmulationStation UI:**
1. Press **START** on your controller to open the **Main Menu**.
2. Go to **System Settings** -> **Services**.
3. Toggle **`MQTT_EVENTS`** to **ON**.

---

## How It Works Under the Hood

1. **Hook Installation (`install()`):**
   When `mqtt_events start` runs, it automatically creates lightweight hook scripts inside:
   * `/userdata/system/configs/emulationstation/scripts/<event>/mqtt_events.sh` (for EmulationStation events like `game-selected`, `system-selected`, `game-start`, `game-end`, `screensaver-start`, `reboot`, `shutdown`, etc.)
   * `/userdata/system/scripts/mqtt_events.sh` (for Batocera's `gameStart` / `gameStop` lifecycle hooks).
2. **MQTT Discovery (`ha_discover()`):**
   On startup, the script publishes Home Assistant MQTT Discovery JSON payloads to `homeassistant/<domain>/batocera_<hostname>_*/config` and marks `batocera/<hostname>/availability` as `online`.
3. **Background Daemons:**
   * `listen_commands` subscribes to `batocera/<hostname>/command` to listen for `reboot` or `shutdown` button presses from Home Assistant.
   * `telemetry_loop` publishes CPU temp, CPU load, RAM usage, disk usage, and uptime every 5 seconds.
4. **Artwork Resolution (`_find_game_image` & `_find_marquee_image`):**
   Whenever a game is selected or launched, the script extracts the real system folder from the ROM path (`/userdata/roms/<system>/<rom>`) so even games launched from **Favorites** find their home system's artwork and `gamelist.xml` metadata.

---

## Troubleshooting

* **Service isn't starting / entities are `unavailable`:**
  Test running the script manually over SSH to check for errors:

```bash
/userdata/system/services/mqtt_events stop
/userdata/system/services/mqtt_events start
ps aux | grep mqtt_events
```

* **Debugging Artwork Lookup:**
  You can subscribe to the debug MQTT topic from Home Assistant (**Settings** -> **Devices & Services** -> **MQTT** -> **Configure** -> **Listen to a topic**):

```text
batocera/+/media_debug
```

  It will log messages like `NO_IMAGE`, `NO_MARQUEE`, or `CONVERTED_WEBP` as you scroll through games.

* **Uninstalling Hooks:**
  To remove all auto-generated EmulationStation hook scripts cleanly:

```bash
/userdata/system/services/mqtt_events stop
/userdata/system/services/mqtt_events uninstall
batocera-services disable mqtt_events
```
