---
title: "Water-Quality Monitoring with ESP-NOW Sensor Nodes and a Shared LTE-M Uplink"
date: "2026-11-20"
summary: "How we built Hydrostat, a monitoring system for ponds and lakes without Wi-Fi: ESP32-C6 sensor nodes measure dissolved oxygen, a three-depth temperature profile and air pressure, send averaged readings over ESP-NOW to an ESP32-C6 gateway, and an nRF9161 forwards them over LTE-M to a web dashboard. The article covers the packet format, automatic gateway discovery, the sensor pitfalls we hit and a barometric way to measure water level from a floating node."
authors:
  - "kubascik-michal"
  - "tupy-andrej"
tags:
  - ESP32-C6
  - ESP-NOW
  - Sensors
  - Environmental monitoring
  - community contribution
---

Ponds, small reservoirs and lakes are exactly the places where you want
continuous measurements and exactly the places where there is no Wi-Fi. Oxygen
in a pond can drop to dangerous levels within one warm night, and the bottom
layer of a lake can run out of oxygen while the surface still looks fine. We
wanted a system that measures dissolved oxygen (DO), water temperature at three
depths and water level, runs on solar power and needs no infrastructure at the
site.

The result is Hydrostat. Sensor nodes built on the
[ESP32-C6](https://www.espressif.com/en/products/socs/esp32-c6) talk to a nearby
gateway over [ESP-NOW](https://docs.espressif.com/projects/esp-idf/en/stable/esp32c6/api-reference/network/esp_now.html),
and the gateway forwards the data over LTE-M to a server and a web dashboard at
[hydrostat.kubascik.eu](https://hydrostat.kubascik.eu). Only the gateway needs a
SIM card, however many nodes there are.

The system has been running in the laboratory for about a month, and the lake
deployment starts in January 2027. The firmware uses ESP-IDF v6.0.1. This
article walks through the parts that are specific to ESP32 and ESP-NOW, and the
mistakes we made along the way.

## Architecture

{{< figure src="img/architecture.webp" alt="Fig. 1 - The layers of the system, from the sensor node to the web dashboard" >}}

- **Sensor node** (ESP32-C6): reads the sensors every 30 s, averages eight
  samples and sends one 78-byte packet every 4 minutes over ESP-NOW.
- **Gateway** (ESP32-C6): receives the packets, validates them and writes each
  one as a text line to a UART at 115200 Bd.
- **Cellular modem** (Nordic nRF9161): keeps the newest reading of each node and
  uploads the pending ones every 5 minutes over HTTPS.
- **Server**: a Node.js/Express API with PostgreSQL, and a React dashboard.

Splitting the gateway into two chips was deliberate. The ESP32-C6 keeps
listening for ESP-NOW frames while the modem attaches to the network, does a TLS
handshake or waits for a slow server, so no local data is lost during cellular
operations.

## The sensor node

{{< figure src="img/sensor-node.webp" alt="Fig. 2 - Sensors connected to the ESP32-C6 node" >}}

The node measures:

| Quantity | Sensor | Interface |
| --- | --- | --- |
| Water temperature at the surface, middle and bottom | 3x DS18B20 | 1-Wire, GPIO23/22/21 |
| Dissolved oxygen | DFRobot SEN0237 galvanic probe | ADC1 channel 3 (GPIO3) |
| Air pressure and temperature | Bosch BMP585 | I²C |
| Supply | Li-ion cell, BMS, solar panel | — |

### One GPIO per thermometer

The DS18B20 can share one 1-Wire bus, but each of our probes has its own GPIO
and its own 4.7 kΩ pull-up. The probes hang in the water on long cables, and a
damaged cable on a shared bus takes all three readings down. On separate lines,
the other two keep working.

The 1-Wire protocol is bit-banged. Two details were worth adding after the
first tests:

```c
/* A data line stuck low reads as all zeros, which passes the CRC */
uint8_t any_bits = 0;
for (int i = 0; i < 9; i++) {
    any_bits |= scratchpad[i];
}
if (any_bits == 0) {
    return ESP_FAIL;
}

if (ds18b20_crc8(scratchpad, 8) != scratchpad[8]) {
    return ESP_ERR_INVALID_CRC;
}
```

The CRC-8 of nine zero bytes is zero, so a shorted or disconnected line produces
a "valid" scratchpad of 0 °C. The second trap is the power-on value of the
temperature register, 85 °C, which you read if the conversion didn't finish. The
main loop drops it:

```c
if (err == ESP_OK && temp_c == 85.0f) {
    /* DS18B20 power-on value: the conversion didn't finish */
    printf("GPIO%d: conversion not ready (85 C)\n", ds18b20_pins[i]);
}
```

Both checks reject a real 0 °C or 85 °C as well. For water that is still liquid
and below 85 °C, that is an acceptable price.

### Dissolved oxygen on the ADC

The DO probe's conditioning board outputs 0–3.3 V for 0–20 mg/L. The ADC is
read in one-shot mode with 12 dB attenuation and corrected with the chip's
calibration scheme. ESP32-C6 supports curve fitting, and the code falls back to
line fitting on chips that only have that:

```c
#if ADC_CALI_SCHEME_CURVE_FITTING_SUPPORTED
    adc_cali_curve_fitting_config_t cali_config = {
        .unit_id = do_adc_unit,
        .chan = do_adc_channel,
        .atten = ADC_ATTEN_DB_12,
        .bitwidth = ADC_BITWIDTH_12,
    };
    err = adc_cali_create_scheme_curve_fitting(&cali_config, &do_adc_cali_handle);
#elif ADC_CALI_SCHEME_LINE_FITTING_SUPPORTED
    /* ... line fitting ... */
#endif
```

The conversion to mg/L is currently linear. Galvanic probes are
temperature-dependent, so the node also sends the raw millivolts in the packet;
a temperature-compensated conversion can be added at the gateway later without
touching the nodes.

## Averaging and the validity mask

Each 30 s the node adds every successful reading to a running mean, and every
eighth period it sends the means. The running means use `double`, because adding
up eight pressures of around 1000 hPa in `float` loses exactly the small changes
the water-level method needs (more on that below):

```c
/* Double, so that small pressure changes survive being added up */
typedef struct {
    double sum;
    uint32_t count;
} running_mean_t;
```

The most useful change compared to our first prototype was the validity mask.
The first version sent `0` for a sensor that failed, and the dashboard showed
water at 0 °C. Now each reading has a bit, and a reading is valid if at least one
of the eight samples succeeded:

```c
if (means->do_mv.count > 0) {
    packet.do_mv = (int32_t)lround(means->do_mv.sum / means->do_mv.count);
    packet.do_mg_l = do_mg_l_from_voltage_mv(packet.do_mv);
    packet.valid_mask |= VALID_OXYGEN;
}
```

The packet itself is a packed struct shared by the node and the gateway. A
static assertion in both projects catches any change that would break the
format:

```c
typedef struct __attribute__((packed)) {
    uint8_t version;
    char snr[TELEMETRY_SNR_LEN]; /* Always NUL-terminated */
    uint32_t sequence;
    uint32_t uptime_ms;
    int32_t hydro_mv;    /* Unused: always -1 */
    int32_t do_mv;
    float pressure_mpa;  /* Unused: always 0 */
    float water_level_m; /* Set by the central unit, not here */
    float do_mg_l;
    float ds18b20_temp_c[3];
    float air_temp_c;
    float humidity_pct;  /* Unused: no humidity sensor */
    float air_pressure_hpa;
    uint8_t valid_mask;
} telemetry_packet_t;

_Static_assert(sizeof(telemetry_packet_t) == 78, "must match esp32-hydrostat-cu");
```

The `snr` field identifies the node on the server. It is `"SN"` followed by the
factory base MAC address, so no node needs to be configured by hand. The
`version` field lets the gateway reject packets from incompatible firmware
instead of misreading them.

## Finding the gateway automatically

Nodes don't have the gateway's MAC address compiled in. Until a node is
registered, it broadcasts the text `PING-CENTRAL-NODE` every 30 s. The gateway
adds the sender as a peer and answers with a unicast `REGISTER`, and the node
stores the sender of that answer as its destination:

```c
static void espnow_recv_cb(const esp_now_recv_info_t *recv_info, const uint8_t *data, int data_len)
{
    size_t register_len = sizeof(espnow_register_response) - 1;

    if (recv_info == NULL || data == NULL || recv_info->src_addr == NULL) {
        return;
    }
    if (data_len != (int)register_len || memcmp(data, espnow_register_response, register_len) != 0) {
        return;
    }
    /* The main loop reads the MAC while registered, so only change it while not */
    if (espnow_central_registered) {
        return;
    }
    if (espnow_add_peer_if_needed(recv_info->src_addr) != ESP_OK) {
        return;
    }
    memcpy(espnow_central_mac, recv_info->src_addr, ESP_NOW_ETH_ALEN);
    espnow_send_failures = 0;
    espnow_central_registered = true;
}
```

If the gateway restarts or is replaced, the node has to notice. The send
callback counts failed unicast transmissions, and after two in a row (about 8
minutes without an acknowledgement) the node forgets the gateway and starts
pinging again:

```c
static void espnow_send_cb(const wifi_tx_info_t *tx_info, esp_now_send_status_t status)
{
    /* Broadcast pings always report success, so only unicast sends can count up */
    if (status == ESP_NOW_SEND_SUCCESS) {
        espnow_send_failures = 0;
    } else if (++espnow_send_failures >= ESPNOW_MAX_SEND_FAILURES && espnow_central_registered) {
        espnow_central_registered = false;
        espnow_send_failures = 0;
    }
}
```

The comment in that callback is the lesson: ESP-NOW broadcasts are never
acknowledged, so their send status is always success and tells you nothing about
whether anybody is listening. Only unicast failures are meaningful.

This scheme has a known weakness: the first device that answers the broadcast
becomes the gateway, and the frames are not encrypted. For a lake it is
acceptable; for anything that controls equipment, use encrypted peers with a
pre-shared key.

## The gateway

The receiving ESP32-C6 must not miss frames that arrive at random times, so its
Wi-Fi power saving is switched off, and both sides use a fixed channel:

```c
ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));
ESP_ERROR_CHECK(esp_wifi_start());
ESP_ERROR_CHECK(esp_wifi_set_ps(WIFI_PS_NONE));

/* Sender and receiver must share the same channel. */
ESP_ERROR_CHECK(esp_wifi_set_channel(1, WIFI_SECOND_CHAN_NONE));
```

The receive callback answers pings, and for everything else checks the length,
the protocol version and that the serial number is a terminated alphanumeric
string. Valid packets go into a 16-entry FreeRTOS queue. A separate task turns
them into text lines for the modem. The ESP-NOW callback runs in the Wi-Fi task,
so it should never block on the UART.

The UART line is plain comma-separated text, with an empty field for every
reading whose validity bit is cleared:

```text
SNR,sequence,water_level,do,t_surface,t_middle,t_bottom,air_temp,humidity,pressure
SN206EF1171CF0,1287,,4.31,,,,28.70,,965.20
```

Text costs a few bytes more than binary, but you can debug the link with any
serial terminal, and the modem firmware doesn't need to know the C struct.
Every 10 s the gateway also writes a `PING` line with its own serial number,
which the modem uses as a heartbeat.

## The cellular side, briefly

The nRF9161 side is outside the scope of an Espressif article, but three
decisions there shaped the whole system:

- The modem keeps only the **newest** reading of each node and uploads it every
  5 minutes. A batch of old readings is less useful than a fresh one, and it
  costs cellular data.
- All requests of one cycle share **one TLS connection**, because each TLS
  handshake costs several kilobytes of mobile data.
- A reading is cleared only after the server's final answer, and it keeps its
  original timestamp. The database has a unique index on node and timestamp, so
  a resend after a timeout can never create a duplicate.

The data flow from the node to the browser looks like this:

{{< figure src="img/data-flow.webp" alt="Fig. 3 - Sequence of messages from the sensor node to the web browser" >}}

## Water level from a floating node

Our first prototype measured water level with a submerged pressure sensor. That
needs a fixed mounting point and an atmospheric reference, and the sensor gets
covered in algae. In the current design the node floats, so its own height is
the water level. The node's BMP585 measures the air pressure at the water
surface, the gateway on the shore has a second barometer, and the nRF9161
measures the gateway's altitude once at start-up with its GNSS receiver.

{{< figure src="img/water-level.webp" alt="Fig. 4 - The floating node measures pressure at the water surface; the height difference to the shore station follows from the pressure ratio" >}}

The height difference between the two barometers follows from the hypsometric
equation, Δh = 29.27 m/K × (T + 273.15) × ln(p_shore / p_node). Weather changes
both pressures equally, so they cancel in the ratio.

The catch is sensitivity. At 15 °C and 980 hPa, **1 Pa of pressure difference
corresponds to 8.6 cm**. The two barometers must therefore agree to within a few
pascals, which takes a common offset calibration with both sensors at the same
height. This method is meant for water bodies where the level changes by
decimetres or more, not for millimetre precision.

The laboratory test showed why the second barometer is needed at all. The
gateway's barometer is still being added, and indoors the GNSS receiver got no
fix, so the firmware fell back to the standard-atmosphere formula on the node's
pressure alone. The dashboard showed a water level of 407 m, and over 24 hours it
"rose" from about 290 m to 410 m while the water in the tank didn't move. That
is a pressure drop of about 14 hPa from the weather, read as altitude. With two
barometers, this drift cancels.

## The dashboard

{{< figure src="img/dashboard-detail.webp" alt="Fig. 5 - The detail view of a sensor node during the laboratory test, 24-hour interval" >}}

The screenshot is from the laboratory test. Three things in it are results of
the decisions above:

- **The temperature cards are empty**, not 0 °C. The probes were disconnected
  during this interval, and the validity mask carried that all the way to the
  browser.
- **Dissolved oxygen** stayed between 4.24 and 4.40 mg/L in the still tank,
  which shows the short-term stability of the averaged readings.
- **The water level climbs by 120 m in a day**, which is the single-barometer
  drift described above.

The server derives a node's status from when its last message arrived, not from
the timestamp inside it: a node turns to "warning" after 15 minutes without data
and to "offline" after an hour. A node with a wrong clock can't look alive that
way. Gaps of up to 6 hours are drawn as dashed interpolated lines and never
stored; longer gaps stay empty.

## Power: what we would change

Honest numbers first: the current node firmware never enters deep sleep. It
keeps the radio initialised so it can receive the `REGISTER` answer at any time,
and runs on a solar panel. With an assumed average of 30 mA, a 3000 mAh cell
alone would last about four days.

The node's work fits deep sleep well, though. Our estimates for the same 30 s
sampling and 4-minute reporting, using the 7 µA deep-sleep current from the
ESP32-C6 datasheet and assumed currents for the active phases:

| Variant | Active time per 30 s | Average current | 3000 mAh lasts |
| --- | --- | --- | --- |
| Current firmware, radio always on | — | ~30 mA (assumed) | ~4 days |
| Deep sleep, DS18B20 conversions one after another | ~2.5 s | ~2.1 mA | ~2 months |
| Deep sleep, all three conversions started together | ~0.9 s | ~0.8 mA | ~5 months |

The radio is not the expensive part: one ESP-NOW packet every 4 minutes is
negligible. What dominates a sleeping node is the 750 ms DS18B20 conversion,
done three times in a row. Starting all three at once is the cheapest
improvement, and registration can move to the moment after wake-up.

The gateway is different. Its ESP32-C6 must listen all the time, which costs
tens of milliamperes, so it is meant for a site with mains or solar power.
Synchronising the nodes' transmissions to time slots would let the receiver
sleep between them.

## Lessons learned

- **Send "no value", not zero.** A validity bit per reading saved us from a
  dashboard full of plausible but false numbers.
- **Check what a CRC can't catch.** An all-zero DS18B20 scratchpad has a valid
  CRC.
- **ESP-NOW broadcast status means nothing.** Count unicast failures to detect a
  lost peer.
- **Put a version and a size check on every binary packet,** and a
  `_Static_assert` on its size in both projects.
- **Use `double` for running sums of large values** when the small differences
  matter.
- **Barometric altitude moves with the weather.** Measure it differentially, or
  not at all.

## Conclusion

ESP-NOW turned out to be a good fit for a site without infrastructure: no
access point, no association, automatic pairing in a few lines of code, and one
cellular modem for the whole site. The parts that took the most time were not
the radio but the edge cases around it: failing sensors, restarted gateways and
weather-driven pressure changes.

Hydrostat was presented at the SEIA 2026 conference in Granada. The next step is
the lake deployment in January 2027, with deep sleep on the nodes and the second
barometer on the gateway.
