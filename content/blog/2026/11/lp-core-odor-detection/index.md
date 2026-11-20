---
title: "Odor Detection on the ESP32-C6 LP Core with Integer Classifiers"
date: "2026-11-20"
summary: "An electronic nose that runs entirely on the low-power RISC-V core of the ESP32-C6: the LP core drives a BME690 gas sensor over LP I²C, computes integer features against an adaptive baseline, classifies them with a nearest-centroid, k-NN or random-forest model, and wakes the main core only to send an ESP-NOW message. The article covers the LP-core firmware, a training toolchain that reproduces the firmware bit for bit, and where the energy actually goes."
authors:
  - "kubascik-michal"
  - "tupy-andrej"
tags:
  - ESP32-C6
  - LP Core
  - ULP
  - ESP-NOW
  - Machine Learning
  - Sensors
  - community contribution
---

Bathroom fans are usually switched by the light, a timer or a humidity
threshold, none of which knows whether the room actually smells. We wanted a
small battery-powered sensor that recognises an unpleasant odor and tells a fan
controller about it, and we wanted the main processor to stay asleep while it
does so.

The ESP32-C6 has a second, low-power RISC-V core (the LP core) that keeps running
while the high-performance (HP) core is in deep sleep. It can read I²C devices
on its own bus and wake the HP core. That is enough to run a complete electronic
nose: read a gas sensor, compute features, classify them, and wake the HP core
only when there is something to report.

The firmware is in [LP-enose](https://github.com/kubascikmichal/LP-enose), and
the training toolchain is in
[integer-based-odor-classification](https://github.com/kubascikmichal/integer-based-odor-classification).
Both use ESP-IDF v6.0.1.

## The system

{{< figure src="img/system.webp" alt="Fig. 1 - The BME690 sits on the LP I²C bus; the LP core classifies and wakes the HP core, which sends ESP-NOW messages to a receiver" >}}

- A **Bosch BME690** metal-oxide gas sensor (with temperature, humidity and
  pressure) is connected to the LP I²C pins.
- The **LP core** scans the sensor every 5.75 s, classifies the scan and counts
  consecutive positive decisions.
- After three positives in a row, the LP core wakes the **HP core**, which sends
  the string `odor` over ESP-NOW and goes back to deep sleep.
- Every 5 minutes, a timer wakes the HP core to send `PING`, so the receiver
  knows the sensor is alive.

The receiver is meant to drive an exhaust fan. The fan controller isn't built
yet; so far the receiver only logs the messages.

The firmware has two build modes. **Logging** mode keeps the HP core awake,
reads a label switch and prints every scan over USB, which is how training data
is collected. **Production** mode is the one described above.

## Talking to the BME690 from the LP core

The HP core sets up the LP I²C peripheral and loads the LP program once, at
the first boot:

```c
void init_lp_i2c(void)
{
    const lp_core_i2c_cfg_t i2c_cfg = LP_CORE_I2C_DEFAULT_CONFIG();
    ESP_ERROR_CHECK(lp_core_i2c_master_init(LP_I2C_NUM_0, &i2c_cfg));
}

void init_lp_core(void)
{
    ulp_lp_core_cfg_t cfg = {
        .wakeup_source = ULP_LP_CORE_WAKEUP_SOURCE_LP_TIMER,
        .lp_timer_sleep_duration_us = 10000,
    };
    ESP_ERROR_CHECK(ulp_lp_core_load_binary(lp_core_main_bin_start,
                                            (lp_core_main_bin_end - lp_core_main_bin_start)));
    ESP_ERROR_CHECK(ulp_lp_core_run(&cfg));
}
```

The LP core has no sensor driver library, so it talks to the BME690 at register
level. For each of three heater steps (200, 300 and 400 °C, 100 ms each), it
selects the heater profile, triggers a forced measurement and reads the 17
result bytes:

```c
uint8_t gas1[2] = {0x71, (uint8_t)(0x20 | step)};
lp_core_i2c_master_write_to_device(LP_I2C_NUM_0, BME690_I2C_ADDR, gas1, 2,
                                   LP_I2C_TRANS_TIMEOUT_CYCLES);

uint8_t cmd[2] = {0x74, 0xB5 | 0x01};   // forced mode
lp_core_i2c_master_write_to_device(LP_I2C_NUM_0, BME690_I2C_ADDR, cmd, 2,
                                   LP_I2C_TRANS_TIMEOUT_CYCLES);

// TPH at 16x oversampling (~40ms) + 100ms heater/gas conversion; add margin.
ulp_lp_core_delay_us(250000);

uint8_t reg = BME690_REG_DATA;
uint8_t data_rd[17];
esp_err_t ret = lp_core_i2c_master_write_read_device(LP_I2C_NUM_0, BME690_I2C_ADDR,
                                                     &reg, 1, data_rd, sizeof(data_rd),
                                                     LP_I2C_TRANS_TIMEOUT_CYCLES);
```

The compensation formulas for temperature, pressure, humidity and gas
resistance are the integer versions from the BME690 datasheet, so the LP core
never touches floating point. One detail matters for data quality: the sensor
reports whether the gas measurement completed (`gas_valid`) and whether the
heater reached its target in time (`heat_stab`). A resistance is used only if
both bits are set; otherwise it is stored as 0.

## Sharing variables with the HP core

Global variables of the LP program are visible to the HP core with an `ulp_`
prefix, which is how the HP core reads the last scan and the decision after it
wakes up. One trap cost us an afternoon: **an LP-core global smaller than 4
bytes is exported as a 32-bit word, and only its low byte is real**. A `uint8_t`
label read from the HP core came back with garbage in the upper bytes. All shared
variables are now 32-bit:

```c
// uint32_t rather than uint8_t: a scalar sub-4-byte LP-core global exports as
// a 32-bit word with garbage upper bytes (only the low byte is real)
volatile uint32_t last_label;                       // 0 = fresh air, 1 = toilet
volatile int32_t  last_features[BME690_NUM_FEATURES];
```

## Integer features and an adaptive baseline

Metal-oxide sensors drift, and two sensors of the same type never read the same
absolute resistance. The firmware therefore uses each heater step's resistance
relative to a clean-air baseline, as an integer percentage. Odors lower the
resistance, so an odor reads below 100. Temperature, humidity and pressure are
added as integers, which gives six features:

```c
for (int i = 0; i < BME690_NUM_HEATER_STEPS; i++) {
    if (!baseline_initialized[i]) {
        // Bootstrap assumption: the very first reading is fresh air.
        baseline[i] = (int32_t)gas_resistance[i];
        baseline_initialized[i] = true;
    }
    features[i] = (baseline[i] > 0)
        ? (int32_t)(((int64_t)gas_resistance[i] * 100) / baseline[i])
        : 100;
}
features[BME690_NUM_HEATER_STEPS + 0] = (int32_t)temperature; // °C *100
features[BME690_NUM_HEATER_STEPS + 1] = (int32_t)humidity;    // %RH *100
features[BME690_NUM_HEATER_STEPS + 2] = (int32_t)pressure;    // Pa
```

The baseline follows slow drift with a moving average (weight 1/8), but only in
cycles classified as clean air. Otherwise a long odor episode would slowly
become the new "normal". The same branch counts positives and decides when to
wake the HP core:

```c
if (label == 1) { // toilet
    if (debounce_count < 0xFF) debounce_count++;
} else { // fresh air: reset debounce, re-arm, and adapt the baseline
    debounce_count = 0;
    armed = true;
    for (int i = 0; i < BME690_NUM_HEATER_STEPS; i++) {
        baseline[i] += (int32_t)(((int64_t)gas_resistance[i] - baseline[i]) / BME690_BASELINE_EMA_DIVISOR);
    }
}

if (armed && debounce_count >= BME690_DEBOUNCE_CYCLES) {
    armed = false; // don't fire again until fresh air is seen
    ulp_lp_core_wakeup_main_processor();
}
```

Three consecutive positives suppress single false alarms, and the `armed` flag
means one wake-up per odor episode rather than one per scan. The price is
latency: at least three cycles, i.e. about 17 s, which is nothing compared with
how long a fan runs.

The baseline weight is a trade-off too. With 1/8, half of an initial baseline
error is gone after about 5 clean cycles (30 s) and 90 % after 17 cycles
(1.6 minutes). That tracks temperature and humidity changes quickly, but an
odor that builds up very slowly could be partly absorbed before it is detected.

{{< figure src="img/timeline.webp" alt="Fig. 2 - One scan every 5.75 s on the LP core; the HP core wakes only after the third consecutive positive" >}}

## Three classifiers in a few hundred bytes

Any of three classifiers can be compiled into the LP program; only the selected
one is linked. All are integer-only.

The nearest-centroid and k-NN classifiers first scale each feature to 0–1000
with the minimum and maximum seen in training. Without that, pressure (around
97 500 Pa) would dominate every distance and the gas ratios (around 100) would
not matter at all:

```c
static int32_t bme690_normalize(int feature_idx, int32_t raw)
{
    int32_t lo = bme690_feature_range[feature_idx].min;
    int32_t hi = bme690_feature_range[feature_idx].max;
    if (raw < lo) raw = lo;
    if (raw > hi) raw = hi;
    if (hi == lo) return 0;
    return (int32_t)(((int64_t)(raw - lo) * 1000) / (hi - lo));
}

static uint8_t classify(const int32_t features[BME690_NUM_FEATURES])
{
    int64_t d_fresh = 0, d_toilet = 0;
    for (int i = 0; i < BME690_NUM_FEATURES; i++) {
        int32_t n = bme690_normalize(i, features[i]);
        int32_t df = n - bme690_centroid_fresh[i];
        int32_t dt = n - bme690_centroid_toilet[i];
        d_fresh += (int64_t)df * df;
        d_toilet += (int64_t)dt * dt;
    }
    return (d_toilet < d_fresh) ? 1 : 0;
}
```

A tie goes to clean air, so an untrained device with all-zero centroids never
wakes the HP core. The k-NN classifier keeps up to 16 labelled examples and votes
among the 3 nearest. The random forest has up to five trees of 32 nodes, each
comparing one raw feature with an integer threshold.

Model data stored in LP RAM (16 KB in total on the ESP32-C6, shared with the LP
program):

| Classifier | Model data | Work per decision |
| --- | --- | --- |
| Feature ranges (centroid, k-NN) | 48 B | 6 divisions |
| Nearest centroid | 48 B | 12 squared differences |
| k-NN, 16 examples, K = 3 | 448 B | 96 squared differences + sort |
| Random forest, 5 trees × 32 nodes | 1920 B | up to 5 tree walks |

All three fit easily, and all three were tested on the device. The
classification takes a negligible fraction of the 0.75 s that the sensor scan
itself takes.

## Reporting over ESP-NOW

In production mode, the HP core runs only when something wakes it. The version
in the repository loads the LP program on the first boot and prints the decision
after a wake-up from the LP core. In our current build the wake-up handling looks
like this (simplified; Wi-Fi and ESP-NOW initialisation and error handling
omitted):

```c
void app_main(void)
{
    esp_sleep_wakeup_cause_t cause = esp_sleep_get_wakeup_cause();

    if (cause == ESP_SLEEP_WAKEUP_ULP) {
        espnow_send_text("odor");            // the LP core confirmed an event
    } else if (cause == ESP_SLEEP_WAKEUP_TIMER) {
        espnow_send_text("PING");            // 5-minute heartbeat
    } else {
        init_lp_core();                      // first boot
        init_lp_i2c();
    }

    ESP_ERROR_CHECK(esp_sleep_enable_ulp_wakeup());
    ESP_ERROR_CHECK(esp_sleep_enable_timer_wakeup(5 * 60 * 1000000ULL));
    esp_deep_sleep_start();
}
```

{{< figure src="img/messages.webp" alt="Fig. 3 - The two ESP-NOW messages: a heartbeat every 5 minutes and an odor event" >}}

The messages are plain strings. A binary event doesn't need a structured
payload, the sender is identified by the MAC address in the frame, and a string
is trivial to read on any receiver.

The heartbeat gives the fan controller a simple failure model: if no `PING`
arrives for two periods, the sensor is dead or its battery is empty, and the
controller can fall back to a timer or the light switch. A sensor fault then
never leaves the room unventilated.

On air, one ESP-NOW frame takes about **3 ms** at around 100 mA. Even ten
frames take only 30 ms. The transmission itself is not where the energy goes,
as the next section shows.

## The training toolchain

The models come from a Python package, `odorclf`, built on scikit-learn. Its
main goal is that the accuracy measured on the PC is exactly the accuracy the
device will have.

{{< figure src="img/toolchain.webp" alt="Fig. 4 - From logging-mode captures to a verified model header" >}}

```bash
# 1. Logs from LP-enose in logging mode -> one CSV row per scan
odorclf parse-logs data/raw/lp_enose_logs/*.log -o data/processed/lp_enose_dataset.csv

# 2. Train and compare 11 model families (5-fold cross-validation)
odorclf train --config configs/lp_enose.toml --data data/processed/lp_enose_dataset.csv

# 3. Export model_params.h straight into the firmware checkout
odorclf export --config configs/lp_enose.toml --lp-enose-dir ~/src/LP-enose
```

Three things make the host and the device agree:

- **The log parser emulates the firmware.** It recomputes the features from the
  raw resistances with the same start-up baseline, the same clean-air-only
  moving average and C's truncating integer division, not Python's floor
  division. The baseline restarts for every log session, as it does after a
  reset.
- **The firmware classifiers are scikit-learn estimators.** Their `predict()` is
  a NumPy port of the C `classify()`, including the tie-breaking and voting, so
  cross-validation scores are device scores.
- **The export is checked with a C compiler.** The exporter compiles the
  generated `model_params.h` together with the unmodified `classify()` source
  and compares its output with Python on every validation and test sample. If
  the firmware's `classify()` changes, the check fails instead of shipping a
  mismatched model.

We have tested the whole pipeline so far only on synthetic logs generated by the
toolchain itself (3600 scans in 6 sessions). On those, the compiled C
classifiers matched Python on all 1440 validation and test samples for all three
back-ends. Real recordings in a bathroom are the next step, so we don't quote
accuracy figures here: synthetic data says the pipeline works, not how well the
nose smells.

## Where the energy goes

Using datasheet currents for the BME690 and the ESP32-C6 and the measured 3 ms
ESP-NOW airtime, these are our estimates for a 2000 mAh battery:

{{< figure src="img/energy.webp" alt="Fig. 5 - Estimated average current against the fraction of cycles that wake the HP core" >}}

| Configuration | Average current | 2000 mAh lasts |
| --- | --- | --- |
| HP core wakes every scan, uploads over Wi-Fi | ~22 mA | ~4 days |
| HP core wakes every scan, sends one ESP-NOW frame | 1.31 mA | ~64 days |
| LP core screens, as in the repository | 0.98 mA | ~85 days |
| LP core screens and sleeps between scans | 0.54 mA | ~154 days |
| Extra cost of the 5-minute `PING` | +0.016 mA | — |

Three conclusions came out of this:

1. **Wake-ups cost, transmissions don't.** Waking the HP core from deep sleep
   and starting the radio takes far more charge than the 3 ms frame. Wi-Fi is far
   worse still, because every upload means associating with the access point and
   a TCP or UDP exchange on top. Keeping the HP core asleep is what matters.
2. **Our LP core is busier than it needs to be.** The LP program in the
   repository is one infinite loop, and its 5-second pause is
   `ulp_lp_core_delay_us(5000000)`, a busy-wait. The LP core is therefore always
   running. Restructuring `main()` to do one scan per call and letting the LP
   timer wake it every 5.75 s would roughly halve the average current. The loop's
   local variables (`debounce_count`, `armed`) then have to become globals.
3. **After that, the heater dominates.** With the LP core sleeping, the three
   100 ms heater pulses per scan are almost 90 % of the budget. Further savings
   have to come from longer scan intervals, fewer heater steps or shorter
   heating.

## Conclusion

The LP core of the ESP32-C6 is capable enough to run a complete small
classifier: sensor driver, integer features, adaptive baseline and model in a
few kilobytes, with the HP core and radio asleep almost all the time. The parts
that needed the most care were not the classifiers but the edges: 32-bit shared
variables, baseline adaptation that doesn't learn the odor, and a toolchain that
guarantees that the PC and the device compute the same thing.

The work was presented at the SEIA 2026 conference in Granada. Next on the list
are LP-timer sleep between scans, real recordings with a grouped evaluation, a
measured current profile, and the fan controller itself.
