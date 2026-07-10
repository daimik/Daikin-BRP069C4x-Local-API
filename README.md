# Daikin WiFi Module Local REST API Documentation

Unofficial documentation for the **Daikin BRP069C4x** WiFi adapter local REST API, reverse-engineered from a live device and cross-referenced with the [pydaikin](https://github.com/fredrike/pydaikin) library used by Home Assistant.

<img width="1263" height="958" alt="image" src="https://github.com/user-attachments/assets/c210d53e-5bfa-4341-9a5c-2391eaa9797f" />


## Tested Hardware

| Property | Value |
|----------|-------|
| WiFi Module | **4P359542-1H** (BRP069C4x series) |
| Firmware | **1_14_88** |
| Model Code | 0B66 (Type N) |
| Protocol Version | pv=2, cpv=2.00 |
| Region | EU |

> **⚠️ Firmware Warning:** Firmware **2.8.0+** uses a completely different API (`/dsiot/multireq` with JSON). This documentation covers the classic REST API available on firmware versions **below 2.8.0**. Do not update your firmware if you rely on local API access — the new firmware may remove these endpoints.

## Overview

The Daikin WiFi module exposes a local HTTP REST API on port 80 with no authentication required. All communication happens over plain HTTP on your LAN.

- **Protocol:** HTTP GET requests
- **Response format:** Comma-separated `key=value` pairs (e.g. `ret=OK,pow=1,mode=3,...`)
- **Authentication:** None (LAN access only)
- **Discovery:** UDP broadcast to port `30050` with payload `DAIKIN_UDP/common/basic_info`
- **Concurrency:** ~4 simultaneous connections
- **Response time:** 200–500ms typical

## Table of Contents

- [Available Endpoints](#available-endpoints)
- [Endpoint Details](#endpoint-details)
  - [common/basic_info](#commonbasic_info)
  - [aircon/get_control_info](#airconget_control_info)
  - [aircon/get_sensor_info](#airconget_sensor_info)
  - [aircon/get_model_info](#airconget_model_info)
  - [Power & Energy Endpoints](#power--energy-endpoints)
  - [Other Read Endpoints](#other-read-endpoints)
- [Write Endpoints](#write-endpoints)
  - [aircon/set_control_info](#airconset_control_info)
- [Value Reference](#value-reference)
- [Code Examples](#code-examples)
- [Unavailable Endpoints](#unavailable-endpoints)
- [Related Projects](#related-projects)
- [Contributing](#contributing)

## Available Endpoints

### Working Endpoints (19)

| Endpoint | Description |
|----------|-------------|
| `GET /common/basic_info` | Device identity, network, firmware info |
| `GET /common/get_remote_method` | Remote access configuration |
| `GET /common/get_notify` | Auto-off notification settings |
| `GET /common/get_datetime` | Device clock |
| `GET /common/get_holiday` | Holiday mode status |
| `GET /common/get_wifi_setting` | WiFi configuration |
| `GET /aircon/get_control_info` | Current AC control settings |
| `GET /aircon/get_sensor_info` | Live sensor readings |
| `GET /aircon/get_model_info` | Hardware capabilities |
| `GET /aircon/get_target` | Target settings |
| `GET /aircon/get_price` | Energy price config |
| `GET /aircon/get_scdltimer_info` | Schedule timer config |
| `GET /aircon/get_scdltimer_body` | Schedule timer data (needs params) |
| `GET /aircon/get_week_power` | Weekly power + today's runtime |
| `GET /aircon/get_week_power_ex` | Weekly power split heat/cool |
| `GET /aircon/get_day_power_ex` | Hourly power today + yesterday |
| `GET /aircon/get_year_power` | Yearly power consumption |
| `GET /aircon/get_year_power_ex` | Yearly power split heat/cool |
| `GET /aircon/get_month_power_ex` | Monthly power split heat/cool |
| `GET /aircon/get_monitordata` | Low-level diagnostic data (hex encoded) |

### Write Endpoints

| Endpoint | Description |
|----------|-------------|
| `GET /aircon/set_control_info` | Set AC mode, temp, fan, swing |
| `GET /common/set_holiday` | Enable/disable holiday mode |
| `GET /common/set_regioncode` | Change region code |
| `GET /common/set_notify` | Configure notifications |
| `GET /aircon/set_target` | Set target |

## Endpoint Details

### `common/basic_info`

Device identity, network info, and general status.

**Example response:**
```
ret=OK,type=aircon,reg=EU,dst=1,ver=1_14_88,rev=8A618141,pow=0,err=0,location=0,
name=%44%61%69%6b%69%6e,icon=0,method=home only,port=30050,id=,pw=,lpw_flag=0,
adp_kind=3,pv=2,cpv=2,cpv_minor=00,led=1,en_setzone=1,mac=XXXXXXXXXXXX,
adp_mode=run,en_hol=0,ssid1=MyNetwork,radio1=-51,ssid=DaikinAPxxxxx,grp_name=,en_grp=0
```

| Key | Type | Description |
|-----|------|-------------|
| `ret` | string | `OK` on success |
| `type` | string | Device type (`aircon`) |
| `reg` | string | Region code (`EU`, `US`, etc.) |
| `dst` | int | DST enabled (0/1) |
| `ver` | string | Firmware version |
| `rev` | string | Hardware revision |
| `pow` | int | Power state (0=OFF, 1=ON) |
| `err` | int | Error code (0=none) |
| `name` | string | URL-encoded device name |
| `method` | string | `home only` = LAN only |
| `port` | int | UDP discovery port (30050) |
| `id` | string | Cloud account ID (empty if not registered) |
| `pw` | string | Cloud password |
| `lpw_flag` | int | Local password flag |
| `adp_kind` | int | Adapter type |
| `pv` | int | Protocol version |
| `cpv` | int | Control protocol version |
| `led` | int | LED enabled (0/1) |
| `en_setzone` | int | Zone setting enabled |
| `mac` | string | MAC address (no separators) |
| `adp_mode` | string | Adapter mode (`run`) |
| `en_hol` | int | Holiday mode (0=OFF, 1=ON) |
| `ssid1` | string | Connected WiFi SSID |
| `radio1` | int | WiFi signal strength (dBm) |
| `ssid` | string | Device's own AP SSID |
| `en_grp` | int | Group enabled |

---

### `aircon/get_control_info`

Current AC control settings. This is the most important endpoint for reading and controlling the unit.

**Example response:**
```
ret=OK,pow=0,mode=3,adv=,stemp=25.0,shum=0,dt1=25.0,dt2=M,dt3=25.0,dt4=25.0,
dt5=25.0,dt7=25.0,dh1=AUTO,dh2=50,dh3=0,dh4=0,dh5=0,dh7=AUTO,dhh=50,b_mode=3,
b_stemp=25.0,b_shum=0,alert=255,f_rate=7,f_dir=1,b_f_rate=7,b_f_dir=1,
dfr1=5,dfr2=5,dfr3=7,dfr4=5,dfr5=5,dfr6=5,dfr7=5,dfrh=5,
dfd1=0,dfd2=0,dfd3=1,dfd4=0,dfd5=0,dfd6=0,dfd7=0,dfdh=0
```

#### Active Control Parameters

| Key | Type | Description | Values |
|-----|------|-------------|--------|
| `pow` | int | Power | `0`=OFF, `1`=ON |
| `mode` | int | Operating mode | `0`/`1`/`7`=Auto, `2`=Dry, `3`=Cool, `4`=Heat, `6`=Fan |
| `adv` | string | Advanced mode | empty=none, `13`=powerful, `12`=economy, `2`=streamer |
| `stemp` | float | Target temperature (°C) | `10.0`–`41.0`, `M`=no temp, `--`=N/A |
| `shum` | string | Target humidity | `0`=off, `AUTO`=auto, or percentage |
| `f_rate` | string | Fan rate | `A`=Auto, `B`=Silent, `3`–`7`=Level 1–5 |
| `f_dir` | int | Swing direction | `0`=Off, `1`=Vertical, `2`=Horizontal, `3`=Both |
| `alert` | int | Alert code | `255`=no alert |

#### Stored Default Values

The device stores last-used settings per mode. These are read-only — they update automatically when you change settings.

| Prefix | Description | Keys |
|--------|-------------|------|
| `dt*` | Default temperature per mode | `dt1`=Auto, `dt2`=Dry, `dt3`=Cool, `dt4`=Heat, `dt7`=Auto(alt) |
| `dh*` | Default humidity per mode | `dh1`=Auto, `dh2`=Dry, `dh3`=Cool, `dh4`=Heat, `dh7`=Auto(alt) |
| `dfr*` | Default fan rate per mode | `dfr1`=Auto, `dfr2`=Dry, `dfr3`=Cool, `dfr4`=Heat, `dfr6`=Fan, `dfr7`=Auto(alt) |
| `dfd*` | Default swing per mode | `dfd1`=Auto, `dfd2`=Dry, `dfd3`=Cool, `dfd4`=Heat, `dfd6`=Fan, `dfd7`=Auto(alt) |
| `b_*` | Backup settings | `b_mode`, `b_stemp`, `b_shum`, `b_f_rate`, `b_f_dir` |

---

### `aircon/get_sensor_info`

Live sensor readings from the indoor and outdoor units.

**Example response:**
```
ret=OK,htemp=24.0,hhum=-,otemp=15.0,err=0,cmpfreq=999
```

| Key | Type | Description | Notes |
|-----|------|-------------|-------|
| `htemp` | float | Indoor temperature (°C) | |
| `hhum` | float/string | Indoor humidity (%) | `-` if no sensor |
| `otemp` | float | Outdoor temperature (°C) | Some models only report when ON |
| `err` | int | Error code | 0=none |
| `cmpfreq` | int | Compressor frequency | `999`=idle/off, `0`=off |

---

### `aircon/get_model_info`

Hardware capabilities and feature flags.

**Example response:**
```
ret=OK,model=0B66,type=N,pv=2,cpv=2,cpv_minor=00,mid=NA,humd=0,s_humd=0,acled=0,
land=0,elec=0,temp=1,temp_rng=0,m_dtct=0,ac_dst=--,disp_dry=0,dmnd=0,en_scdltmr=1,
en_frate=1,en_fdir=1,s_fdir=1,en_rtemp_a=0,en_spmode=0,en_ipw_sep=0,en_mompow=0,
en_onofftmr=1
```

| Key | Description | Values |
|-----|-------------|--------|
| `model` | Model code | |
| `type` | Model type | |
| `humd` | Humidity control | 0=none, 1=supported |
| `s_humd` | Humidity sensor | 0=none, 1=present |
| `temp` | Temperature sensor | 0=none, 1=present |
| `temp_rng` | Temperature range type | 0=standard |
| `m_dtct` | Motion detection | 0=none |
| `dmnd` | Demand control | 0=none |
| `en_scdltmr` | Schedule timer | 0=no, 1=yes |
| `en_frate` | Fan rate control | 0=no, 1=yes |
| `en_fdir` | Fan direction control | 0=no, 1=yes |
| `s_fdir` | Swing direction | 0=no, 1=yes |
| `en_spmode` | Special mode | 0=no, 1=yes |
| `en_onofftmr` | On/Off timer | 0=no, 1=yes |
| `en_mompow` | Momentary power | 0=no, 1=yes |

---

### Power & Energy Endpoints

All energy values are in **100 Watt units** (divide by 10 for kWh).

#### `aircon/get_week_power`
```
ret=OK,today_runtime=146,datas=0/0/0/0/0/0/0
```
- `today_runtime` — minutes the unit ran today
- `datas` — last 7 days total consumption (slash-separated)

#### `aircon/get_week_power_ex`
```
ret=OK,s_dayw=4,week_heat=0/0/.../0,week_cool=0/0/.../0
```
- `s_dayw` — start day of week (0=Sun, 1=Mon, ... 6=Sat)
- `week_heat` / `week_cool` — 14 values (7 days × 2 per day)

#### `aircon/get_day_power_ex`
```
ret=OK,curr_day_heat=0/0/.../0,prev_1day_heat=0/0/.../0,
curr_day_cool=0/0/.../0,prev_1day_cool=0/0/.../0
```
- 24 values per field = hourly consumption for today and yesterday

#### `aircon/get_year_power` / `get_year_power_ex`
```
ret=OK,previous_year=0/0/0/0/0/0/0/0/0/0/0/0,this_year=0/0
```
- 12 values = monthly consumption

#### `aircon/get_month_power_ex`
```
ret=OK,curr_month_heat=0/0/.../0,prev_month_heat=0/0/.../0,
curr_month_cool=0/0/.../0,prev_month_cool=0/0/.../0
```
- Daily values for current and previous month

---

### Other Read Endpoints

#### `common/get_remote_method`
```
ret=OK,method=home only,notice_ip_int=3600,notice_sync_int=10
```

#### `common/get_datetime`
```
ret=OK,sta=2,cur=2026/2/26 17:46:5,reg=EU,dst=1,zone=0
```

#### `common/get_holiday`
```
ret=OK,en_hol=0
```

#### `common/get_wifi_setting`
```
ret=OK,ssid=MyNetwork,security=mixed,key=,link=1
```

#### `aircon/get_target`
```
ret=OK,target=0
```

#### `aircon/get_price`
```
ret=OK,price_int=27,price_dec=0
```

#### `aircon/get_scdltimer_info`
```
ret=OK,format=v1,f_detail=total#18;_en#1;_pow#1;_mode#1;_temp#4;_time#4;
_vol#1;_dir#1;_humi#3;_spmd#2,scdl_num=3,scdl_per_day=6,en_scdltimer=1,
active_no=1,scdl1_name=,scdl2_name=,scdl3_name=
```

#### `aircon/get_monitordata`

Returns hex-encoded diagnostic data. Values are ASCII hex (e.g. `mode=33` → ASCII `0x33` = character `3` = Cool mode).

```
ret=OK,mondata=pv=2,cpv=2,cpv_minor=00,mac=xxxxxxxxxxxx,fan=000000,tap=37,
mode=33,pow=30,rawrtmp=00000000,humid=ff,trtmp=00000000,fangl=00000000,
hetmp=00000000,itelc=3030303030303030,eepid=30423636,cmpfrq=393939,otmp=00000000
```

---

## Write Endpoints

### `aircon/set_control_info`

The main control endpoint. Uses HTTP GET with query parameters.

**⚠️ All 6 mandatory parameters must be included in every request**, even for values you are not changing. Always read current values first with `get_control_info`, then send them all back with your modifications.

```
GET /aircon/set_control_info?pow={pow}&mode={mode}&stemp={stemp}&shum={shum}&f_rate={f_rate}&f_dir={f_dir}
```

**Response:** `ret=OK` on success, `ret=PARAM NG` on invalid parameters.

#### Examples

| Action | URL |
|--------|-----|
| Cool, 22°C, auto fan | `?pow=1&mode=3&stemp=22.0&shum=0&f_rate=A&f_dir=0` |
| Heat, 24°C, level 3, vertical swing | `?pow=1&mode=4&stemp=24.0&shum=0&f_rate=5&f_dir=1` |
| Auto, 23°C, silent, both swing | `?pow=1&mode=1&stemp=23.0&shum=0&f_rate=B&f_dir=3` |
| Dry mode | `?pow=1&mode=2&stemp=M&shum=0&f_rate=A&f_dir=0` |
| Fan only, level 2, horizontal | `?pow=1&mode=6&stemp=--&shum=0&f_rate=4&f_dir=2` |
| Turn OFF | `?pow=0&mode=3&stemp=25.0&shum=0&f_rate=A&f_dir=0` |

### `common/set_holiday`

```
GET /common/set_holiday?en_hol=1    # Enable holiday mode
GET /common/set_holiday?en_hol=0    # Disable holiday mode
```

### `common/set_regioncode`

```
GET /common/set_regioncode?reg=eu
```

---

## Value Reference

### Mode (`mode`)

| Value | Mode |
|-------|------|
| `0`, `1`, `7` | Auto |
| `2` | Dry |
| `3` | Cool |
| `4` | Heat |
| `6` | Fan only |

### Fan Rate (`f_rate`)

| Value | Speed |
|-------|-------|
| `A` | Auto |
| `B` | Silent |
| `3` | Level 1 (lowest) |
| `4` | Level 2 |
| `5` | Level 3 |
| `6` | Level 4 |
| `7` | Level 5 (highest) |

### Swing Direction (`f_dir`)

| Value | Direction |
|-------|-----------|
| `0` | Off |
| `1` | Vertical |
| `2` | Horizontal |
| `3` | Both (3D) |

### Temperature (`stemp`)

| Value | Meaning |
|-------|---------|
| `10.0` – `41.0` | Target temperature in °C (0.5 step on some models) |
| `M` | No temperature control (used in Dry mode) |
| `--` | Not applicable (used in Fan mode) |

---

## Code Examples

### curl

```bash
DAIKIN_IP="192.168.1.100"

# Read status
curl -s "http://$DAIKIN_IP/aircon/get_control_info"
curl -s "http://$DAIKIN_IP/aircon/get_sensor_info"

# Turn on cool at 22°C
curl -s "http://$DAIKIN_IP/aircon/set_control_info?pow=1&mode=3&stemp=22.0&shum=0&f_rate=A&f_dir=0"

# Turn off
curl -s "http://$DAIKIN_IP/aircon/set_control_info?pow=0&mode=3&stemp=22.0&shum=0&f_rate=A&f_dir=0"
```

### PowerShell

```powershell
$DaikinIP = "192.168.1.100"

# Parse Daikin response into hashtable
function Parse-Daikin([string]$Response) {
    $r = [ordered]@{}
    $Response.Trim().Split(',') | ForEach-Object {
        $p = $_ -split '=', 2
        if ($p.Count -eq 2) { $r[$p[0]] = $p[1] }
    }
    return $r
}

# Read status
function Get-DaikinStatus {
    $ctrl = Parse-Daikin (Invoke-WebRequest "http://$DaikinIP/aircon/get_control_info" -UseBasicParsing).Content
    $sens = Parse-Daikin (Invoke-WebRequest "http://$DaikinIP/aircon/get_sensor_info" -UseBasicParsing).Content
    
    $modeMap = @{ '0'='Auto';'1'='Auto';'2'='Dry';'3'='Cool';'4'='Heat';'6'='Fan';'7'='Auto' }
    $fanMap  = @{ 'A'='Auto';'B'='Silent';'3'='Lv1';'4'='Lv2';'5'='Lv3';'6'='Lv4';'7'='Lv5' }
    $dirMap  = @{ '0'='Off';'1'='Vertical';'2'='Horizontal';'3'='Both' }
    
    [PSCustomObject]@{
        Power       = if($ctrl.pow -eq '1'){'ON'}else{'OFF'}
        Mode        = $modeMap[$ctrl.mode]
        TargetTemp  = "$($ctrl.stemp)°C"
        IndoorTemp  = "$($sens.htemp)°C"
        OutdoorTemp = "$($sens.otemp)°C"
        FanRate     = $fanMap[$ctrl.f_rate]
        Swing       = $dirMap[$ctrl.f_dir]
        Compressor  = $sens.cmpfreq
    }
}

# Set controls (only specify what you want to change)
function Set-DaikinControl {
    param(
        [ValidateSet('0','1')]$Power,
        [ValidateSet('0','1','2','3','4','6','7')]$Mode,
        [string]$Temp,
        [ValidateSet('A','B','3','4','5','6','7')]$FanRate,
        [ValidateSet('0','1','2','3')]$FanDir
    )
    $current = Parse-Daikin (Invoke-WebRequest "http://$DaikinIP/aircon/get_control_info" -UseBasicParsing).Content
    
    $p = if($Power)   { $Power }    else { $current.pow }
    $m = if($Mode)    { $Mode }     else { $current.mode }
    $t = if($Temp)    { $Temp }     else { $current.stemp }
    $f = if($FanRate) { $FanRate }  else { $current.f_rate }
    $d = if($FanDir)  { $FanDir }   else { $current.f_dir }
    $h = $current.shum
    
    $url = "http://$DaikinIP/aircon/set_control_info?pow=$p&mode=$m&stemp=$t&shum=$h&f_rate=$f&f_dir=$d"
    $result = Parse-Daikin (Invoke-WebRequest $url -UseBasicParsing).Content
    return $result.ret
}

# Usage
Get-DaikinStatus
Set-DaikinControl -Power 1 -Mode 3 -Temp 22.0 -FanRate A
Set-DaikinControl -Power 0
```

### Python

```python
import requests

DAIKIN_IP = "192.168.1.100"

def parse_response(text):
    return dict(item.split("=", 1) for item in text.strip().split(",") if "=" in item)

def get_control_info():
    r = requests.get(f"http://{DAIKIN_IP}/aircon/get_control_info")
    return parse_response(r.text)

def get_sensor_info():
    r = requests.get(f"http://{DAIKIN_IP}/aircon/get_sensor_info")
    return parse_response(r.text)

def set_control(pow=None, mode=None, stemp=None, shum=None, f_rate=None, f_dir=None):
    """Set AC controls. Reads current values first, applies changes."""
    current = get_control_info()
    params = {
        "pow":    pow    if pow    is not None else current["pow"],
        "mode":   mode   if mode   is not None else current["mode"],
        "stemp":  stemp  if stemp  is not None else current["stemp"],
        "shum":   shum   if shum   is not None else current["shum"],
        "f_rate": f_rate if f_rate is not None else current["f_rate"],
        "f_dir":  f_dir  if f_dir  is not None else current["f_dir"],
    }
    r = requests.get(f"http://{DAIKIN_IP}/aircon/set_control_info", params=params)
    return parse_response(r.text)

# Usage
info = get_control_info()
sensors = get_sensor_info()
print(f"Indoor: {sensors['htemp']}°C, Outdoor: {sensors['otemp']}°C")

set_control(pow="1", mode="3", stemp="22.0", f_rate="A")  # Cool 22°C
set_control(pow="0")  # Turn off
```

---

## Unavailable Endpoints

The following returned HTTP 404 on the tested hardware and are not supported by this model:

| Endpoint | Category |
|----------|----------|
| `/common/get_reg_info` | Registration info |
| `/aircon/get_timer` | Timer settings |
| `/aircon/get_program` | Program settings |
| `/aircon/get_scdltimer` | Schedule timer (use `get_scdltimer_info` instead) |
| `/aircon/get_notify` | Aircon notifications |
| `/aircon/get_zone_setting` | Zone control |
| `/aircon/get_zone_name` | Zone names |
| `/aircon/get_demand_info` | Demand control |
| `/airflow/*` | Airflow subsystem |
| `/cleaner/*` | Air cleaner subsystem |
| `/dewpoint/*` | Dew point subsystem |
| `/humidifier/*` | Humidifier subsystem |
| `/dsiot/multireq` | Firmware 2.8.0+ API |

## Related Projects

- [pydaikin](https://github.com/fredrike/pydaikin) — Python library used by Home Assistant (supports both old and 2.8.0 firmware)
- [daikin-controller](https://github.com/Apollon77/daikin-controller) — Node.js library for local control
- [daikin-controller-cloud](https://github.com/Apollon77/daikin-controller-cloud) — Node.js library for Onecta cloud API (BRP069C4x with cloud-only firmware)
- [daikin-control](https://github.com/ael-code/daikin-control) — Original unofficial API documentation (BRP069A/B series)
- [Home Assistant Daikin integration](https://www.home-assistant.io/integrations/daikin/) — Official HA integration
- [daikin_onecta](https://github.com/jwillemsen/daikin_onecta) — HA integration for Onecta cloud API
- [ESP32-Faikin](https://github.com/revk/ESP32-Faikin) — ESP32 replacement for WiFi module with MQTT local control

## Contributing

If you have a different Daikin WiFi module or firmware version, please open an issue or PR with your findings. Particularly useful:

- Endpoint responses from different models/firmware versions
- Additional endpoints not listed here
- Schedule timer (`set_scdltimer_body`) parameter documentation
- Advanced mode (`adv`) values for different models

## License

This documentation is provided as-is for educational and interoperability purposes. Daikin is a trademark of DAIKIN INDUSTRIES, LTD. This project is not affiliated with or endorsed by Daikin.
