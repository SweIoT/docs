# GW — BLE to MQTT gateway on nRF52

## What the firmware does

GW is a gateway between Bluetooth Low Energy devices and an MQTT backend,
built on the Nordic nRF52840 with an LTE-M / NB-IoT modem (Sony Altair
Adrastea) as its network connection, or optionally WiFi through an attached
ESP32-C3 module. It is battery powered and designed for low power: the modem
is off whenever it is not needed, and everything happens in short, timed
bursts.

In operation the gateway repeats a cycle driven by its timers:

1. **Scan** for BLE advertisements, on the normal or the long range (coded)
   PHY, and collect the manufacturer data of the devices that match the
   configured patterns — optionally only from a whitelist of devices.
2. **Report** what was seen to the MQTT broker, over TLS, as one line per scan
   in the report format described below; optionally after collecting several
   scans first, so that the modem is powered up less often.
3. **Listen** for commands on the MQTT command topic for a configurable time,
   and answer them on the response topic.
4. **Poll the backend** for a queued batch of commands, execute them and
   acknowledge the results; report the whitelist status.
5. **Synchronise the whitelist** devices now and then: set their clocks and
   timers by connecting to each one over BLE, so that their advertising
   windows line up with the gateway's scans.

In between, the gateway **advertises** itself periodically so that a BLE
client — the phone app or a service tool — can connect and send commands over
the Nordic UART Service (NUS). Commands can also be given on the USB serial
console.

The gateway is a BLE **central** as well: it connects to the devices, sends
them commands (signed by the backend when they require it) and updates their
firmware over the air through the Nordic secure DFU bootloader. Its own
firmware is updated the same way, with signed DFU packages.

Beyond that the hardware gives it a **position** (GNSS in the modem) which
can be reported on movement, detected by the accelerometer; **NFC**, used to
activate a new unit from the phone app; and, on the Acconeer XM122 board
variant, a **radar** sensor whose measurements are reported like the data of
an external device.

Everything is stored in a file system in an external flash memory: settings,
the log, the whitelist, downloaded firmware and certificates.

### Hardware

| | |
|---|---|
| MCU | Nordic nRF52840, BLE 5 with normal (1M) and long range (coded) PHY |
| Network | LTE-M or NB-IoT through the Adrastea modem; WiFi through an ESP32-C3 module on boards that have one |
| Position | GNSS receiver in the modem |
| Sensors | MC34X9 accelerometer for movement detection; Acconeer radar on the XM122 variant |
| NFC | Activation and a configurable broadcast url |
| Console | USB serial, 115200 baud |
| Power | Battery, voltage reported with every scan |
| LED | Green status LED, used only during start-up |

### Status LED

The green LED shows how far the start-up has come and is off once the gateway
is running:

| Blink | Meaning |
|---|---|
| Slow, 1 s on / 1 s off | Waiting for the USB console to be connected; only with `usb_log` and `usb_wait` set |
| Fast, 0.5 s on / 0.5 s off | Starting: sensors and modem are initiated, then the network is attached to get the time and to sync the whitelist |
| Off | Start-up finished, normal operation |

A gateway that keeps blinking slowly is waiting for a terminal; one that keeps
blinking fast cannot get on the network, or is still attaching.

## The console

The command console is on the USB serial port at 115200 baud. Lines end with
LF; a CR before it is ignored. Backspace edits the line, and an empty line
prints the help. The gateway does not echo what is typed; use a terminal with
local echo (the development manual describes the tooling).

Every log line is also sent to a connected NUS client, which is how a BLE
client receives the answers to its commands. The log is written to the file
`log.txt` in the file system too (rotated at 25 kB into `log.bck`), so that
it can be read afterwards with `cat_log` or uploaded with `upload_log`.

Log lines have the form

    2026-09-03 14:22:37.921|I|Reporting...

with the level letter `E` error, `W` warning, `I` info, `D` debug or `T`
trace. Until the clock has been set from the network the time is the time
since boot.

Commands that need the network take a while: the modem is normally off to
save power and has to attach first.

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

    =version?4.9.1 (3.2.1-15.3.0) 2026-09-02 14:35:10
    *rem_command:*Failed to connect

Some commands print more before the response line; the response line is always
the last one.

### Signed commands

Once a public key has been installed with the `key` command, commands arriving
over BLE, MQTT or from the backend must carry a signature: the command text,
`#`, and the ECDSA signature as hex. The signature is made by the backend,
which holds the private key. Commands from the console are trusted and need no
signature. `mac` never needs one.

    version#973438a45979c63d...

### Activation

A newly produced gateway is inactive: no scanning and no periodic advertising,
whatever the settings say. It leaves that state through NFC activation from
the phone app — the NFC touch starts an advertisement, the app connects and
activates — or with the `activate` command on the console. Activation is
remembered in the file system.

**activate 1|0|?** — activate, deactivate, or show the state.

There is always a 20 second advertising period after power on, active or not.

## Settings

Each settings group is a command of its own. Setting values also answers with
the resulting values, and saves them. The settings survive a restart.

In the MQTT settings the `user_id` may be `$backend`, in which case the
gateway's backend credentials are used as user id and password.

**mqtt_data broker,user_id,password,topic_header,cert_name,port_no**

How scan reports are published.

| | |
|---|---|
| broker | Address of the MQTT broker |
| user_id | Optional user id, or `$backend` |
| password | Optional password |
| topic_header | First part of the topic; the gateway's MAC address and `report` are appended, e.g. `miThings/BLE_GW/FE:14:13:CF:85:B1/report` |
| cert_name | Certificate of the broker, downloaded beforehand with `add_cert`. Giving one selects TLS |
| port_no | Broker port, when not the standard one |

**mqtt_cmd broker,user_id,password,topic_header,cert_name,port_no,timeout_s**

Where the gateway listens for commands after a report. The first six
parameters are as for `mqtt_data`; the topic gets `cmd` appended for incoming
commands and `cmd_resp` for the answers, e.g. `miThings/BLE_GW/FE:14:13:CF:85:B1/cmd`.
Retained messages are the way to queue a command, since the gateway only
listens for `timeout_s` seconds after each report and keeping that short
saves power; with `timeout_s` 0 it does not listen at all. After a command
has been received the retained message is cleared.

**mqtt_pos broker,user_id,password,topic_header,cert_name,port_no**

The same parameters, for reporting the gateway's position. The topic gets
`position` appended. The published position has the form

    fix_type,time,date,latitude,longitude,altitude,utc,accuracy,speed,eph_type

**ble scan_timeout_s,adv_interval_ms,adv_timeout_s,tx_power,adv_type,connect_timeout_s,scan_type,adv_data_hex**

| | |
|---|---|
| scan_timeout_s | Length of a scan in seconds |
| adv_interval_ms | Advertising interval in milliseconds |
| adv_timeout_s | Length of an advertising period in seconds |
| tx_power | Transmit power in dBm |
| adv_type | 0 normal, 1 alternating, 2 long range |
| connect_timeout_s | Timeout when connecting to a device |
| scan_type | 0 normal, 1 alternating, 2 long range |
| adv_data_hex | Manufacturer data of the gateway's own advertisement, as hex; the first two bytes are the company id |

Alternating means half of the period on the normal PHY and the other half on
the long range one. `scan_type` also decides how the gateway connects to
devices: long range only when it is 2.

The scan windows of the gateway must overlap the advertising windows of the
devices. With a whitelist this is arranged automatically by the whitelist
synchronisation (`per_wlist_sync_h` below, and once at power on), which lets
`scan_timeout_s` be kept short to save power. Make the scan and advertising
periods equal, or one a multiple of the other.

**timers per_scan_s,per_adv_s,per_wlist_sync_h**

| | |
|---|---|
| per_scan_s | Seconds between scans, 0 never scans |
| per_adv_s | Seconds between advertising periods, 0 never advertises |
| per_wlist_sync_h | Hours between whitelist synchronisations, 0 never |

Changing `per_scan_s` with a whitelist defined triggers a synchronisation.

**report frequency[,match,start_pos,length]+**

What is reported, and how often. Several match patterns can be given. The
example is the default: one scan per report, the miThings manufacturer id
`0011` with everything after it reported, and iBeacon frames of the given
UUID reporting the 10 hex characters from position 38.

| | |
|---|---|
| frequency | Scans per report; with more than 1 the scans are collected in a file and sent together |
| match | Hex string the manufacturer data of a device must start with |
| start_pos | Position in the hex string from which data is reported; may be beyond the match to skip further parts |
| length | Number of hex characters reported, 0 for everything after start_pos |

Example:

    report:1,0011,4,0,4C000215BD79CA110BFA41F4A334BF3AB66BB8,38,10

The report published for each scan has the form

    version;time;battery_volts:unit,unit,...

where `version` is the report format version, currently 3, `time` is
`YYYYMMDDhhmmss`, and each unit is the device MAC address, the matched data as
configured, and the RSSI as one hex byte:

    3;20260722162536;4.16:FB26E31F97A31101F0199F06DE0000000000D2,E6972E1CB58D4466701199B8

**sys log_level,usb_log,usb_cmd,usb_wait,nfc_url,send_pos_move_m,radar_method,radar_change_only,radar_and_scan**

| | |
|---|---|
| log_level | 0 normal, 1 debug, 2 more debug |
| usb_log | 1 to log on the console, 0 for silence |
| usb_cmd | 1 to accept commands on the console. With 0 the console is locked and can only be unlocked over BLE, MQTT or the backend |
| usb_wait | 1 to wait for the console to be connected before starting |
| nfc_url | Url broadcast over NFC |
| send_pos_move_m | Above 0, the position is reported every this many minutes while the gateway is moving |
| radar_method | Measurement method on the XM122 board, as in the r-iot firmware |
| radar_change_only | 1 to report only when the method detects a change |
| radar_and_scan | 0 measure and report the radar only, 1 report both radar and scanned devices, 2 scanned devices only |

The `radar_*` parameters apply to the Acconeer XM122 board only.

**gnss fix_timeout_s,fix_hold_s**

| | |
|---|---|
| fix_timeout_s | Longest wait for a position fix, in seconds |
| fix_hold_s | Time after the first fix before the position is reported, to let it settle |

**network rat**

The radio access technology: `CATM` (LTE-M) or `NBIOT` for the modem, `WIFI`
on boards with the WiFi module.

**backend backend_url,poll_cmd,report_wl_status**

| | |
|---|---|
| backend_url | Url of the SweIoT backend; the gateway logs in with its MAC address as user name and the installed key as password |
| poll_cmd | 1 to poll the backend for queued commands after each report |
| report_wl_status | 1 to report the whitelist status to the backend after each synchronisation, the content of `wlist_show` |

### WiFi

**wifi ssid,password** — the WiFi credentials, on boards with the WiFi module.

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
| **factory** | Restore the factory settings |
| **boot_cnt** | Number of boots |
| **version** | Firmware version and build time |
| **mac** | The gateway's BLE MAC address, which is its identity towards the backend |
| **restart** | Restart |
| **dfu** | Enter the bootloader's DFU mode. Normally not needed, the gateway supports buttonless DFU |
| **loc_dfu version\|file_name\|url[,certificate]** | Update the gateway's own firmware, see below |
| **set_clock [time]** | Set the clock, from the network when no time is given, otherwise seconds since 1 January 1970 |
| **get_clock** | Show the clock |
| **vbatt** | The battery voltage |

### Firmware update

**loc_dfu source[,certificate]** updates the gateway's firmware from a DFU
zip package. The source is a version known to the backend, a file already in
the file system, or a url. The package is unpacked, and the bootloader
performs the update at the restart that follows. Packages are signed; the
bootloader refuses anything not signed with the project's key.

    loc_dfu GW_3.1.0

## Certificate handling

TLS connections are verified against a certificate. The ISRG Root X1
certificate is available by default; others can be downloaded and used when a
server requires it. A certificate is referred to by name in the settings and
commands that take one.

| | |
|---|---|
| **add_cert cert_url[,certificate]** | Download a certificate; the file name of the url becomes its name |
| **ls_cert** | List the certificates, default and downloaded |
| **rm_cert cert_name** | Remove a downloaded certificate |

## BLE commands

**rem_command mac_address,string[,no_sign]**

Connect to a device and send it a command over the Nordic UART Service. The
response of the device is the response of this command.

| | |
|---|---|
| mac_address | The device address. A public address is written with a `P` prefix, `PFB:26:E3:1F:97:A3`; without the prefix the address is random static |
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

    rem_dfu C9:62:20:E5:07:10,xm122_2.18.0

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
| **send_pos** | Report the position now according to the `mqtt_pos` settings, whether or not movement reporting is enabled |
| **network_rssi** | The modem or WiFi signal strength |
| **network_info** | Network details |
| **back_cmd** | Poll the backend for queued commands now and execute them; otherwise done after each report |
| **AT command** | Send an AT command to the modem and print the answer |

## Radar method configuration

On the XM122 board the parameters of a measurement method are set, as in the
r-iot firmware, with a command whose name is the method number, followed by
the parameters:

    3:300,20,0.500,3.000,3.00,0.650,0.200,0.090,0,0.000,1,200,1000,2,0

A method number alone queries its parameters.
