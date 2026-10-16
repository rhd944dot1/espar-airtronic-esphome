# Remote start through a Teltonika tracker (BLE, no wiring)

Start and stop the heater from anywhere through a Teltonika GPS tracker that is
already in the vehicle, with feedback whether the command arrived and the heater
runs. Tracker and ESP32 talk over BLE only — no wire between them.

Tested with an **FMC920**, firmware 04.00.00, a **Traccar** server and Home
Assistant. Other Teltonika FMB/FMC models with the same BLE features should work;
parameter IDs may differ between models (check them in the Teltonika
Configurator).

Files:

| File | What |
|---|---|
| `esphome/espar-heater-teltonika.yaml` | `espar-heater-full.yaml` plus remote start (presets, BLE command, BLE feedback, LAN action) |
| `esphome/home-assistant/teltonika-remote-start.yaml` | Home Assistant package: Traccar command, sensors decoding the feedback, start script, notification |

This is an example on top of the component, not part of it. Nothing in
`components/espar_can` changes.

## How it works

```
 Home Assistant ──REST──► Traccar ──GPRS──► tracker ──BLE (service UUID)──► ESP32 ──CAN──► heater
 Home Assistant ◄──────── Traccar ◄──GPRS── tracker ◄──BLE (manufacturer data)── ESP32
 Home Assistant ─────────────── ESPHome API (LAN, when the ESP is in WiFi) ──► ESP32
```

**Command (tracker → ESP).** The tracker's parameter **1111 "BLE broadcasting
service ID"** is advertised as a 16-bit service UUID. A change through
`setparam 1111:XXXX` is advertised at once — no reboot of the tracker (measured
< 40 s from sending the command on the server until the ESP sees it). Only
4 hex digits are accepted.

```
 bits 15–8  0x53            fixed marker
 bit  7     T               toggle bit — flipped for every command, so every command is a change
 bits 6–4   0
 bits 3–0   C               0 = off, 1–8 = preset
```

The ESP reacts only to a **change** of the value. The first value it sees after
boot is only the baseline — a standing "on" command does not restart the heater
after an ESP reboot. The ESP takes over T from that baseline, so the next command
(with T flipped) is always a change.

**Presets.** 8 presets on the ESP, each mode (heat / fan), setpoint and duration,
editable in Home Assistant ("Espar Preset n …"). The ESP ends the run itself when
the duration is over — no network needed for that. *Max Run Time* still applies
on top.

**Feedback (ESP → tracker).** The ESP advertises manufacturer data
`FF FF 53 48` + 22 bytes (page 1) and `FF FF 53 49` + 22 bytes (page 2),
alternating every 2 s. The device name goes into the scan response, so the
advertising packet is exactly 31 bytes. Byte layout: comment "BLE feedback" in
the YAML.

The tracker reads both pages with its **"Advanced" BLE sensor** table and sends
them as AVL IO elements:

| AVL ID (FMC920) | Content | Priority / operand | Sent |
|---|---|---|---|
| 467 | status: state + result of the last command + acknowledge bit | High / On Change | immediately on change |
| 468 | fault | High / On Change | immediately on change |
| 332 | page 1, 22 bytes (Traccar shows it as hex text) | Low / Monitoring | with every record |
| 333 | page 2, 22 bytes | Low / Monitoring | with every record |

So a start gives within seconds: "command arrived" (acknowledge bit flips), then
"heating", or a fault. Temperatures and counters come with the regular records.

**LAN path.** When the ESP is online in Home Assistant, the script calls the ESP
action `remote_preset` directly (≈ 1 s). It runs the same code on the ESP as the
BLE path. If the climate entity does not react within 15 s, the script falls
back to the tracker path.

## Measured

| Test | Path | Result |
|---|---|---|
| Fan preset on / off | tracker | acknowledge + state 19 s after the command; off → fan runs on ~76 s, then idle |
| Heat preset 20 °C, 30 min | tracker | acknowledge + "heating" after 36 s, flame after 91 s, switched itself off after 30 min |
| Fan preset on / off | LAN | script start → notification 0.65 s |

## Setup

### 1. ESP32

Use `esphome/espar-heater-teltonika.yaml`. Set `tracker_mac` to the tracker's
BLE MAC (scan with nRF Connect or `bluetoothctl scan le` while the tracker
advertises — see step 2, the tracker shows up with its "Local name" + `_LE` and
the service UUID once 1111 is set).

`reboot_timeout: 0s` on `api` and `wifi` is important: out of WiFi range the
ESP would otherwise reboot every 15 min — and an ESP reboot ends a run (the
component always starts with the heater off).

### 2. Tracker

All settings can be sent as SMS/GPRS commands, e.g. from Traccar
("Send command" → custom). `<ESP_MAC>` is the ESP's BLE MAC without colons.
These are FMC920 IDs; verify them for your model in the Configurator
(hovering a field shows its ID).

```
setparam 70076:3                         Sensor configuration = Custom sensors
setparam 1250:3;1252:<ESP_MAC>           BLE connection #2: mode Advanced, MAC of the ESP
setparam 14200:FF;14210:2;14220:2;14230:0;14240:0;14250:5348
                                          #2 row 1: EIR type FF, offset 2, size 2, action Match, data 5348
setparam 14201:FF;14211:4;14221:22;14231:1;14241:4
                                          #2 row 2: offset 4, size 22, action Save → Custom 1 (page 1)
setparam 14202:FF;14212:4;14222:1;14232:1;14242:8
                                          #2 row 3: offset 4, size 1 → Custom 2 (status)
setparam 14203:FF;14213:5;14223:1;14233:1;14243:9
                                          #2 row 4: offset 5, size 1 → Custom 3 (fault)
setparam 51200:1;51900:2;51901:5;51910:2;51911:5
                                          priorities: Custom 1 Low; Custom 2/3 High + On Change
setparam 1300:3;1302:<ESP_MAC>           BLE connection #3: mode Advanced, MAC of the ESP
setparam 14400:FF;14410:2;14420:2;14430:0;14440:0;14450:5349
setparam 14401:FF;14411:4;14421:22;14431:1;14441:4
                                          #3: match 5349, page 2 → Custom 1
setparam 51210:1;51211:3                  priority Custom 1 of #3: Low, Monitoring
```

Table columns (connection #2, rows 1–10): type 14200–14209, offset
14210–14219, size 14220–14229, action 14230–14239 (0 = Match, 1 = Save), IO
14240–14249 (4 = Custom 1, 8 = Custom 2, 9 = Custom 3, 10 = Custom 4,
11 = Custom 5), match data 14250–14259. Connection #3 starts at 14400, #4 at
14600. Other useful settings: BLE non-stop scan on (1115 = 1). Bluetooth must be
enabled (800). BLE connection control (1112) can stay at its default — the
sensor table keeps receiving while the tracker advertises the command.

The command is then: `setparam 1111:5381` (preset 1, T = 1), next command e.g.
`setparam 1111:5300` (off, T = 0).

Findings on the way:

- `getparam` with a `;` list silently drops IDs that do not exist.
- `readio` shows `N/A` for elements with priority None — no proof of "no data".
- Custom 1 has a variable size (up to 128); Custom 2–5 are fixed.
- Byte order of the 22-byte block is kept as sent (row endianness 0).
- A command response becomes a Traccar position with only a few attributes,
  so the "latest position" briefly lacks all IO values. The HA sensors below
  are trigger-based and keep their last value.

### 3. Traccar + Home Assistant

1. Traccar integration (*Traccar Server*): add `io332`, `io333`, `io467`,
   `io468` to the custom attributes.
2. Copy `esphome/home-assistant/teltonika-remote-start.yaml` into your packages
   (or its blocks into `configuration.yaml`) and adjust the entity IDs marked
   `# ← adjust`.
3. `secrets.yaml`: `traccar_auth: "Bearer <token>"` (Traccar → user → token)
   and `traccar_command_url: "http://<traccar-host>:8082/api/commands/send"`.
4. Start with the script `script.heater_remote_start` (field *preset* 0–8).

## Safety notes

- The heater limits its run time only through the ESP (preset duration and
  Max Run Time). Both run on the ESP without network.
- An ESP reboot ends a run and the ESP does **not** restart it. The package
  sends a notification when the ESP rebooted or the heater reports a fault.
- The command is a fixed 16-bit value from a fixed MAC. Someone in BLE range who
  knows this can imitate it and start the heater (limited by the durations).
  The lock switch "Espar Remote Start" disables the remote start completely.
- Never let two controllers transmit on the CAN bus (fault P000342).
