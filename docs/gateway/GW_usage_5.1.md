# GW — BLE to MQTT gateway

<!-- One manual for every platform of the gateway. The html comments are for
the tools, see tools/md_platform.py and tools/gen_app_json: a block between
"platform: nrf52" or "platform: esp32" and "platform: end", a table row with
"nrf52" or "esp32" at its end, or "hidden: esp32" in a row, tell the settings
description of the phone application what applies to which platform. The
text itself names the platforms where they differ. -->

## What the firmware does

GW is a gateway between Bluetooth Low Energy devices and an MQTT backend. It
exists on two hardware platforms, running the same application:

- **The nRF52 gateway**, built on the Nordic nRF52840 with an LTE-M / NB-IoT
  modem (Sony Altair Adrastea) as its network connection, or optionally WiFi
  through an attached ESP32-C3 module. It is battery powered and designed
  for low power: the modem is off whenever it is not needed, and everything
  happens in short, timed bursts.
- **The ESP32 gateway**, an ESP32-S3 module with WiFi as its network
  connection, meant to be USB powered.

Towards the backend the two are the same device: the same MQTT topics, the
same report format, the same command set and the same signed-command scheme,
so a fleet can mix them. This manual covers both and says so where they
differ.

In operation the gateway repeats a cycle driven by its timers:

1. **Scan** for BLE advertisements, on the normal or the long range (coded)
   PHY, and collect the manufacturer data of the devices that match the
   configured patterns — optionally only from a whitelist of devices.
2. **Report** what was seen to the MQTT broker, over TLS, as one line per scan
   in the report format described below; optionally after collecting several
   scans first, so that the network is used less often.
3. **Listen** for commands on the MQTT command topic for a configurable time,
   and answer them on the response topic.
4. **Poll the backend** for a queued batch of commands, execute them and
   acknowledge the results; report the whitelist status.
5. **Synchronise the whitelist** devices now and then: set their clocks and
   timers by connecting to each one over BLE, so that their advertising
   windows line up with the gateway's scans.

In between, the gateway **advertises** itself periodically so that a BLE
client — the phone app or a service tool — can connect and send commands over
the Nordic UART Service (NUS). Commands can also be given on the serial
console.

The gateway is a BLE **central** as well: it connects to the devices, sends
them commands (signed by the backend when they require it) and updates their
firmware over the air through the Nordic secure DFU bootloader. Its own
firmware is updated from the backend as well, with signed packages.

Beyond that the nRF52 hardware gives it a **position** (GNSS in the modem)
which can be reported on movement, detected by the accelerometer; **NFC**,
used to activate a new unit from the phone app; and, on the Acconeer XM122
board variant, a **radar** sensor whose measurements are reported like the
data of an external device. The ESP32 gateway has none of these, and the
settings and commands belonging to them are kept for compatibility only.

Everything is stored in a file system in flash memory, external on the
nRF52, the module's own on the ESP32: settings, the log, the whitelist,
downloaded firmware and certificates.

### Hardware

**nRF52 gateway**

Two boards run this firmware: the `connectivity_nrf52_lte` board made for
the gateway by iLabs (Invector Labs), and the Acconeer XM122 radar module
with a miThings modem add-on board, the `xm122` variant. Both carry the same
modem, external flash and accelerometer; the XM122 adds the radar and has no
NFC, LED or WiFi option.

| | |
|---|---|
| MCU | Nordic nRF52840, BLE 5 with normal (1M) and long range (coded) PHY |
| Network | LTE-M or NB-IoT through the Würth Adrastea-I modem, with u.FL antenna connectors for LTE and GNSS. On the connectivity board optionally WiFi through an iLabs BSerial module, an ESP32-C3 running ESP-AT, attached to the board's BConnect FFC connector |
| Position | GNSS receiver in the modem |
| Sensors | MC3419 accelerometer for movement detection; the Acconeer A111 radar on the XM122 variant |
| NFC | Activation and a configurable broadcast url, connectivity board only |
| Storage | 8 MB QSPI flash (W25Q64) holding the file system |
| Console | USB serial, 115200 baud |
| Power | LiPo battery, charged over USB on the connectivity board; the voltage is reported with every scan |
| LED | Green status LED on the connectivity board, used only during start-up |

**ESP32 gateway**

| | |
|---|---|
| Module | ESP32-S3-WROOM-1 (N8R2) or Seeed XIAO ESP32-S3, 8 MB flash |
| Network | WiFi, 2.4 GHz |
| BLE | 5.0, normal (1M) and long range (coded) PHY, both in the same firmware |
| Console | USB serial, 115200 baud |
| Button | the BOOT button starts an advertisement |
| Power | USB; the battery voltage in the reports is always 0.00 |

### Status LED (nRF52)

The green LED of the nRF52 gateway shows how far the start-up has come and is
off once the gateway is running:

| Blink | Meaning |
|---|---|
| Slow, 1 s on / 1 s off | Waiting for the USB console to be connected; only with `usb_log` and `usb_wait` set |
| Fast, 0.5 s on / 0.5 s off | Starting: sensors and modem are initiated, then the network is attached to get the time and to sync the whitelist |
| Off | Start-up finished, normal operation |

A gateway that keeps blinking slowly is waiting for a terminal; one that keeps
blinking fast cannot get on the network, or is still attaching.

The bootloader uses the same LED while it updates the firmware, after
`loc_dfu` or `dfu`:

| Blink | Meaning |
|---|---|
| 1 s on / 1 s off | The bootloader is active: unpacking and writing a package from the file system, or waiting for one over BLE |
| 0.25 s on / 0.25 s off | A package is being received over BLE |
| 0.1 s on / 0.1 s off | The update failed or was aborted |

A successful update ends with the normal start-up blinking of the new
firmware.

## The console

The command console is on the USB serial port at 115200 baud. Lines end with
LF; the ESP32 gateway also accepts CR and CRLF. Backspace edits the line, and
an empty line prints the help. The gateway does not echo what is typed; use a
terminal with local echo (the development manual describes the tooling).

Every log line is also sent to a connected NUS client, which is how a BLE
client receives the answers to its commands. The log is written to the file
`log.txt` in the file system too, rotated into `log.bck` at half the size
given by the `log_file` setting, so that it can be read afterwards with
`cat_log` or uploaded with `upload_log`.

Log lines have the form

    2026-09-03 14:22:37.921|I|Reporting...

with the level letter `E` error, `W` warning, `I` info, `D` debug or `T`
trace. Until the clock has been set from the network the time is the time
since boot.

On the nRF52 gateway, commands that need the network take a while: the modem
is normally off to save power and has to attach first.

## Command syntax

A command consists of the command name, then either a space or `:`, then the
parameters separated by commas, or the query character `?`. A command that
takes parameters is a query when none are given.

    mqtt_data mqtt.example.org,$backend,,miThings/BLE_GW,ISRG_Root_X1,8883,0
    mqtt_data?
    backend:https://example.org,1,1

A parameter may be left empty, in which case the current value is kept. To
set a parameter to an empty value give a single space. A parameter that
contains commas is written between double quotes.

Every command ends with a response line whose first character says whether it
succeeded, `=`, or failed, `*`:

    =version?4.12.0 (3.2.1-15.3.0) 2026-09-02 14:35:10
    =version?4.12.0 (idf 5.5.5, wroom) 2026-09-07 14:08:28
    *rem_command:*Failed to connect

The first two are the version answers of an nRF52 and of an ESP32 gateway.
Some commands print more before the response line; the response line is always
the last one.

### Signed commands

Once a public key has been installed with the `key` command, commands arriving
over BLE, MQTT or from the backend must carry a signature: the command text,
`#`, and the ECDSA signature as hex. The signature is made by the backend,
which holds the private key. Commands from the console are trusted and need no
signature. `mac` never needs one.

    version#973438a45979c63d...

### First start of an ESP32 gateway

A freshly flashed ESP32 gateway knows nothing about its surroundings. On the
console, in this order:

1. `wifi ssid,password` — the network to use. The gateway cannot reach
   anything before this, so it has to be done on the console.
2. `backend https://...` — the backend url, if not the default.
3. `key ...` — the public key issued by the backend for this gateway. From
   now on commands over BLE, MQTT and from the backend must be signed, and
   the gateway can log in to the backend.
4. `mqtt_data ...` — the broker, if not the default.
5. `activate 1` — start the timers.

Everything is remembered across restarts. The WiFi credentials are kept by
the WiFi driver itself rather than in the settings file, so they survive a
`factory` reset; give the `wifi` command again to change networks.

An nRF52 gateway gets on the network by itself through its modem and is
activated as described next.

### Activation

A newly produced gateway is inactive: no scanning and no periodic advertising,
whatever the settings say, though there is always a 20 second advertising
period after power on. The nRF52 gateway leaves that state through NFC
activation from the phone app — the NFC touch starts an advertisement, the app
connects and activates — or with the `activate` command on the console; the
ESP32 gateway with the `activate` command over the console or BLE. Activation
is remembered in the file system.

**activate 1|0|?** — activate, deactivate, or show the state.
<!-- app-command "Activate" action="activate:1" warning="The gateway will start consuming battery power" -->
<!-- app-command "Deactivate" action="activate:0" warning="This will put the gateway in hibernation, touch NFC and connect with the app to get it back" -->
<!-- app-command "Activate" action="activate:1" warning="The gateway starts advertising, scanning and reporting" -->
<!-- app-command "Deactivate" action="activate:0" warning="The gateway stops reporting until activated again" -->

## Settings

Each settings group is a command of its own. Setting values also answers with
the resulting values, and saves them. The settings survive a restart.

When the gateway is used with the SWEiot backend, the MQTT settings are to
be left at their defaults: the backend supplies the broker credentials, which
is what `$backend` as `user_id` means. Change them only for a broker of your
own.

**mqtt_data broker,user_id,password,topic_header,cert_name,port_no**

How scan reports are published.

| Parameter | Default | Range | Description |
|---|---|---|---|
| broker | mqtt.sweiot.mithings.org | | Address of the MQTT broker |
| user_id | $backend | | Leave unchanged with the SWEiot backend, `$backend` takes the credentials from it; otherwise the broker user id |
| password | | | Leave unchanged with the SWEiot backend; otherwise the broker password |
| topic_header | miThings/BLE_GW | | First part of the topic; the gateway's MAC address and `report` are appended, e.g. `miThings/BLE_GW/FE:14:13:CF:85:B1/report` |
| cert_name | ISRG_Root_X1 | | Leave unchanged with the SWEiot backend; otherwise the certificate of the broker, see certificate handling. Giving one selects TLS |
| port_no | 8883 | 0–65532 | Leave unchanged with the SWEiot backend; otherwise the broker port, when not the standard one |

**mqtt_cmd broker,user_id,password,topic_header,cert_name,port_no,timeout_s**

Where the gateway listens for commands after a report. The topic gets `cmd`
appended for incoming commands and `cmd_resp` for the answers. Retained
messages are the way to queue a command, since the gateway only listens for
`timeout_s` seconds after each report and keeping that short saves power;
with `timeout_s` 0 it does not listen at all. After a command has been
received the retained message is cleared.

| Parameter | Default | Range | Description |
|---|---|---|---|
| broker |  | | Address of the MQTT broker |
| user_id |  | | Leave unchanged with the SWEiot backend, `$backend` takes the credentials from it; otherwise the broker user id |
| password | | | Leave unchanged with the SWEiot backend; otherwise the broker password |
| topic_header |  | | First part of the topic; the gateway's MAC address and `cmd` are appended, e.g. `miThings/BLE_GW/FE:14:13:CF:85:B1/cmd` |
| cert_name |  | | Leave unchanged with the SWEiot backend; otherwise the certificate of the broker, see certificate handling. Giving one selects TLS |
| port_no | 0 | 0–65532 | Leave unchanged with the SWEiot backend; otherwise the broker port, when not the standard one |
| timeout_s | 0 | 0–300 | Seconds to listen for commands after each report, 0 never |

**mqtt_pos broker,user_id,password,topic_header,cert_name,port_no**

For reporting the position of an nRF52 gateway. The topic gets `position`
appended. The ESP32 gateway has no position source and never publishes here;
the group is kept for compatibility.

| Parameter | Default | Range | Description |
|---|---|---|---|
| broker |  | | Address of the MQTT broker <!-- hidden: esp32 --> |
| user_id |  | | Leave unchanged with the SWEiot backend, `$backend` takes the credentials from it; otherwise the broker user id <!-- hidden: esp32 --> |
| password | | | Leave unchanged with the SWEiot backend; otherwise the broker password <!-- hidden: esp32 --> |
| topic_header |  | | First part of the topic; the gateway's MAC address and `position` are appended, e.g. `miThings/BLE_GW/FE:14:13:CF:85:B1/position` <!-- hidden: esp32 --> |
| cert_name |  | | Leave unchanged with the SWEiot backend; otherwise the certificate of the broker, see certificate handling. Giving one selects TLS <!-- hidden: esp32 --> |
| port_no | 0 | 0–65532 | Leave unchanged with the SWEiot backend; otherwise the broker port, when not the standard one <!-- hidden: esp32 --> |

The published position has the form

    fix_type,time,date,latitude,longitude,altitude,utc,accuracy,speed,eph_type

**ble scan_timeout_s,adv_interval_ms,adv_timeout_s,tx_power,adv_type,connect_timeout_s,scan_type,adv_data_hex**

| Parameter | Default | Range | Description |
|---|---|---|---|
| scan_timeout_s | 10 | 1–600 | Length of a scan in seconds |
| adv_interval_ms | 100 | 25–10000 | Advertising interval in milliseconds, a system parameter <!-- hidden --> |
| adv_timeout_s | 20 | 1–600 | Length of an advertising period in seconds, a system parameter <!-- hidden --> |
| tx_power | 8 | -40–8 | Transmit power in dBm, a system parameter <!-- hidden --> |
| adv_type | 0 | 0–2 | 0 normal, 1 alternating, 2 long range <!-- warning 2: Long range only: the phone will no longer see the gateway --> |
| connect_timeout_s | 150 | 10–500 | Timeout in seconds when connecting to a device |
| scan_type | 0 | 0–2 | 0 normal, 1 alternating, 2 long range |
| adv_data_hex | 001120 | | Manufacturer data of the gateway's own advertisement, as hex; the first two bytes are the company id. This is the default of the nRF52 gateway |
| adv_data_hex | 001130 | | Manufacturer data of the gateway's own advertisement, as hex; the first two bytes are the company id. This is the default of the ESP32 gateway, so that the two can be told apart |

Alternating means half of the period on the normal PHY and the other half on
the long range one. `scan_type` also decides how the gateway connects to
devices: long range only when it is 2.

The scan windows of the gateway must overlap the advertising windows of the
devices. With a whitelist this is arranged automatically by the whitelist
synchronisation (`per_wlist_sync_h` below, and once at power on), which lets
`scan_timeout_s` be kept short to save power. Make the scan and advertising
periods equal, or one a multiple of the other.

**timers per_scan_s,per_adv_s,per_wlist_sync_h**

| Parameter | Default | Range | Description |
|---|---|---|---|
| per_scan_s | 120 | 0–86400 | Seconds between scans, 0 never scans |
| per_adv_s | 60 | 0–86400 | Seconds between advertising periods, 0 never advertises |
| per_wlist_sync_h | 48 | 0–168 | Hours between whitelist synchronisations, 0 never |

Changing the timers restarts them and, with a whitelist defined, triggers a
synchronisation. The gateway keeps the phase of its scan grid across the
change: the next scan comes where a grid with the new period, laid out from
the last scan, has it. A device that could not be synchronised and keeps its
old period therefore stays in step with the gateway, as long as the old and
the new period are multiples of each other, and is synchronised at the next
opportunity instead of dropping out of the reports.

A synchronisation connects to each whitelisted device in turn, with up to
three attempts within `connect_timeout_s` each. A device the gateway has not
heard from in the last three scan periods (at least ten minutes) gets one
attempt only, so an absent device does not hold the others up. With
`scan_type` 1 a failed attempt on the normal PHY is followed by one on the
long range PHY. A device that could not be synchronised is tried again after
each of the next two reports, then at the next periodic synchronisation.

**report frequency[,match,start_pos,length]+**

What is reported, and how often. Up to four match patterns. The example is
the default: one scan per report, the miThings manufacturer id `0011` with
everything after it reported, and iBeacon frames of the given UUID reporting
the 10 hex characters from position 38.

| Parameter | Default | Range | Description |
|---|---|---|---|
| frequency | 1 | 1–1000 | Scans per report; with more than 1 the scans are collected in a file and sent together |
| match | | | Hex string the manufacturer data of a device must start with |
| start_pos | | 0–63 | Position in the hex string from which data is reported; may be beyond the match to skip further parts |
| length | | 0–65 | Number of hex characters reported, 0 for everything after start_pos |

Example:

    report:1,0011,4,0,4C000215BD79CA110BFA41F4A334BF3AB66BB8,38,10

The report published for each scan has the form

    version;time;battery_volts:unit,unit,...

where `version` is the report format version, currently 3, `time` is
`YYYYMMDDhhmmss`, `battery_volts` is the battery voltage of an nRF52 gateway
and always `0.00` from an ESP32 gateway, and each unit is the device MAC
address, the matched data as configured, and the RSSI as one hex byte:

    3;20260722162536;4.16:FB26E31F97A31101F0199F06DE0000000000D2,E6972E1CB58D4466701199B8

**sys log_level,usb_log,usb_cmd,usb_wait,nfc_url,send_pos_move_m,radar_method,radar_change_only,radar_and_scan**

The last five parameters belong to hardware only the nRF52 gateway has; the
ESP32 gateway keeps them for compatibility and ignores them.

| Parameter | Default | Range | Description |
|---|---|---|---|
| log_level | 0 | 0–2 | 0 normal, 1 debug, 2 more debug |
| usb_log | 1 | 0–1 | 1 to log on the console, 0 for silence; applied at boot <!-- hidden --> |
| usb_cmd | 1 | 0–1 | 1 to accept commands on the console. With 0 the console is locked and can only be unlocked over BLE, MQTT or the backend <!-- hidden --> |
| usb_wait | 0 | 0–1 | 1 to wait for the console to be connected before starting, nRF52 only <!-- hidden --> |
| nfc_url | mithings.se | | Url broadcast over NFC, nRF52 only <!-- hidden: esp32 --> |
| send_pos_move_m | 0 | 0–60 | Above 0, the position is reported every this many minutes while the gateway is moving, nRF52 only <!-- hidden --> |
| radar_method | 0 | 0–7 | Measurement method on the XM122 board, as in the r-iot firmware <!-- hidden --> |
| radar_change_only | 0 | 0–1 | 1 to report only when the method detects a change, XM122 only <!-- hidden --> |
| radar_and_scan | 0 | 0–2 | 0 measure and report the radar only, 1 report both radar and scanned devices, 2 scanned devices only; XM122 only <!-- hidden --> |

**gnss fix_timeout_s,fix_hold_s**

The position fix of the nRF52 gateway; kept for compatibility on the ESP32.

| Parameter | Default | Range | Description |
|---|---|---|---|
| fix_timeout_s | 300 | 5–300 | Longest wait for a position fix, in seconds <!-- hidden: esp32 --> |
| fix_hold_s | 0 | 0–20 | Time after the first fix before the position is reported, to let it settle <!-- hidden: esp32 --> |

**network rat**

| Parameter | Default | Range | Description |
|---|---|---|---|
| rat | CATM | | The radio access technology of the nRF52 gateway: `CATM` (LTE-M) or `NBIOT` for the modem, `WIFI` on boards with the WiFi module |
| rat | WIFI | | The radio access technology; the ESP32 gateway accepts `WIFI` only <!-- hidden --> |

**backend backend_url,poll_cmd,report_wl_status**

| Parameter | Default | Range | Description |
|---|---|---|---|
| backend_url | https://sweiot.mithings.org | | Url of the SweIoT backend; the gateway logs in with its MAC address as user name and the installed key as password |
| poll_cmd | 1 | 0–1 | 1 to poll the backend for queued commands after each report |
| report_wl_status | 1 | 0–1 | 1 to report the whitelist status to the backend after each synchronisation, the content of `wlist_show` |

**log_file size_kb**

| Parameter | Default | Range | Description |
|---|---|---|---|
| size_kb | 50 | 50–500 | Total size of the log on the file system in KB (1024 bytes); up to 500 on the nRF52 gateway |
| size_kb | 50 | 50–250 | Total size of the log on the file system in KB (1024 bytes); up to 250 on the ESP32 gateway |

The log is written to `log.txt`, which is rotated into `log.bck` at half the
size, so the two files together stay within it. Reading or uploading the full
log makes a temporary copy of both files, so count twice the size in file
system space.

### WiFi

**wifi ssid,password** — store the WiFi credentials: on the ESP32 gateway,
and on nRF52 boards with the WiFi module. The gateway joins the network at
once and sets its clock from it. On the ESP32 they are kept by the
WiFi driver itself, not in the settings, so they are unaffected by `factory`
and only need to be given once — on the console, since a fresh gateway has no
other way in. The password is never shown; `network_info` shows the SSID.

**wifi_clr** — forget the WiFi credentials, for a gateway that changes
hands or site. The gateway has no network afterwards until `wifi` is given
again, so sent over MQTT or from the backend it cuts the gateway off.

## File system commands

The file system holds the settings, the log, the whitelist, downloaded files
and certificates. Files are referred to by their name in it; directories are
allowed.

| | |
|---|---|
| **ls** | List the file system |
| **cat file_name** | Print a file to the log |
| **rm file_name** | Delete a file |
| **mkdir dir_name** | Create a directory |
| **rmdir dir_name** | Remove a directory with everything in it |
| **unzip zip_file_name[,dest_dir]** | Extract an archive, into `dest_dir` if given |
| **cat_log** | Print the log file, oldest lines first |
| **clr_log** | Clear the log file |
| **download source_url,destination_file_name[,certificate]** | Download a file, http or https. The certificate is only needed for a server that the default one does not cover. A source without any `/` is taken as a firmware version to fetch from the backend |
| **upload file_name,url[,certificate]** | Upload a file with http PUT |
| **upload_log** | Upload the gateway log to the backend |
| **upload_rem_log mac_address** | Fetch the log of a remote device over BLE and upload it to the backend |
| **put_file / get_file** | Binary file transfer over the console, used by the development tooling rather than typed by hand |

## System commands

| | |
|---|---|
| **settings** | Print all settings |
| **factory** | Restore the factory settings <!-- app-command "Reset all settings to factory defaults" action="factory?" warning="This will erase the custom settings" --> |
| **boot_cnt** | Number of boots |
| **version** | Firmware version and build time; on the ESP32 also the ESP-IDF version and the board |
| **mac** | The gateway's BLE MAC address, which is its identity towards the backend |
| **restart** | Restart |
| **dfu** | nRF52: enter the bootloader's DFU mode, normally not needed since the gateway supports buttonless DFU. Not used on the ESP32, which has no separate DFU mode |
| **loc_dfu version\|file_name\|url[,certificate]** | Update the gateway's own firmware, see below |
| **set_clock [time]** | Set the clock, from the network when no time is given, otherwise seconds since 1 January 1970 |
| **get_clock** | Show the clock |
| **vbatt** | The battery voltage; always 0.00 on the ESP32 |
| **idf_log [level]** | ESP32 only: verbosity of the ESP-IDF system messages on the console, 0 none, 1 error, 2 warning (the default), 3 info, 4 debug, 5 verbose. These messages are not part of the log |

### Firmware update

**loc_dfu source[,certificate]** updates the gateway's firmware. The source
is a version known to the backend, a file already in the file system, or a
url; a certificate is only needed for a server the built in one does not
cover.

On the nRF52 gateway the firmware is a signed DFU zip package. It is
unpacked, and the bootloader performs the update at the restart that follows;
the bootloader refuses anything not signed with the project's key.

    loc_dfu GW_4.12.0

On the ESP32 gateway the firmware is a plain, signed application image. It is
streamed straight into the inactive application slot, nothing is written to
the file system, and an unsigned or tampered image is refused. After a
restart the new image runs, and confirms itself once its file system,
settings and network are up; an image that fails to get that far is rolled
back to the previous one by the bootloader. The file name form of the
command, `loc_dfu source,name,certificate`, is accepted and the name ignored.

    loc_dfu gw_esp32_4.12.0
    loc_dfu http://192.168.1.10:8000/gw_esp32_wroom_4.12.0.bin

## Certificate handling

TLS connections are verified against a certificate. The ISRG Root X1
certificate is built in and used by default; others can be added when a
server requires it. A certificate is referred to by name in the settings and
commands that take one.

| | |
|---|---|
| **add_cert cert_url[,certificate]** | Download a certificate; the file name of the url becomes its name |
| **ls_cert** | List the certificates, built in and downloaded |
| **rm_cert cert_name** | Remove a downloaded certificate |

## BLE commands

**rem_command mac_address,string[,no_sign]**

Connect to a device and send it a command over the Nordic UART Service. The
response of the device is the response of this command.

| | |
|---|---|
| mac_address | The device address. A public address is written with a `P` prefix, `PFB:26:E3:1F:97:A3`; without the prefix the address is random static. A wrong kind costs one failed connection attempt before the other kind is tried |
| string | The command. Several commands can be sent in one connection separated by `&`. A string with commas is written between quotes |
| no_sign | If present the command is sent without backend signing |

The commands `set_clk` and `sync` may be given without parameters; the gateway
fills in the current time and the time since its own timer period started, so
that the device's clock and timers line up with the gateway's.

**rem_dfu mac_address,version|file_name|url[,certificate]**

Update the firmware of a device over BLE with the Nordic secure DFU. The
firmware is a DFU zip, given as a version known to the backend, a file already
in the file system, or a url. The gateway enters the device's bootloader
through its buttonless DFU service, or with the `dfu` command over NUS when
the device has no such service.

    rem_dfu C9:62:20:E5:07:10,xm122_4.0.0

### Whitelist

A whitelist restricts the scan to the listed devices; with an empty whitelist
every device matching the report patterns is reported. Up to 8 devices.

| | |
|---|---|
| **wlist_add mac_address[,mac_address]** | Add devices |
| **wlist_rm mac_address[,mac_address]** | Remove devices |
| **wlist_clr** | Empty the whitelist |
| **wlist_show** | The devices with their state: `mac;sync_ok;last_seen` per device, `last_seen` as `YYYYMMDDhhmmss` or `*Never` |
| **wlist_command command_string[,no_sign]** | Send a command to every device, as `rem_command` does |
| **wlist_sync [connect_timeout_s]** | Send `set_clk` and `sync` to every device, with an optional connection timeout, default 120 s |
| **wlist_dfu version\|file_name\|url[,certificate]** | `rem_dfu` for every device; the response lists `OK` or `*FAIL` per device |

Example responses:

    =wlist_show?C0FEA61EAE10;1;20260425122132,C96220E50710;1;20260425122214,D8BFCFB49BA8;1;20260425122201
    =wlist_sync:C0FEA61EAE10=sync:OK,C96220E50710=sync:OK,D8BFCFB49BA8=sync:OK
    *wlist_sync:C0FEA61EAE10=sync:OK,C96220E50710=sync:OK,D8BFCFB49BA8*Failed to connect

For the synchronisation to be meaningful the timer values of the devices must
be a multiple of the gateway's.

## Miscellaneous commands

| | |
|---|---|
| **key key_in_hex** | Install the public key, 64 bytes as hex, for validation of signed commands. Its hex form is also the password towards the backend. After this, signed commands are required |
| **scan** | One scan and report now, regardless of the timers. Fails if a scan is in progress |
| **send_pos** | nRF52: report the position now according to the `mqtt_pos` settings, whether or not movement reporting is enabled. Not used on the ESP32 |
| **network_rssi** | The modem or WiFi signal strength |
| **network_info** | Network details; on the ESP32 the address, SSID, signal and certificates |
| **back_cmd** | Poll the backend for queued commands now and execute them; otherwise done after each report <!-- app-command "Connect to server" action="back_cmd?" warning="This may require you to disconnect the app from the gateway for it to be able to connect to the sensors" --> |
| **AT command** | nRF52: send an AT command to the modem and print the answer |

## Radar method configuration (XM122)

On the XM122 board of the nRF52 gateway the parameters of a measurement
method are set, as in the r-iot firmware, with a command whose name is the
method number, followed by the parameters:

    3:300,20,0.500,3.000,3.00,0.650,0.200,0.090,0,0.000,1,200,1000,2,0

A method number alone queries its parameters.
