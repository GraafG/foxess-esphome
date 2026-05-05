# FoxESS ESPHome — Outstanding TODOs

Items here require hardware testing or significant architectural changes and were intentionally deferred.

---

## TODO 1: Drive `flow_control_pin` LOW immediately after `setup()` [1-B]

### Background

RS485 is a half-duplex bus: the driver direction is controlled by the DE (Driver Enable) and RE̅ (Receiver Enable) pins on the transceiver IC. When DE is HIGH the chip transmits; when DE is LOW (and RE̅ is LOW) the chip receives.

The Smart-Stuff P1 Modbus Dongle used in this project ties DE and RE̅ together on a single GPIO (GPIO4, our `flow_control_pin`). This is the standard "auto-direction" or "manual flow control" wiring used in virtually all ESPHome RS485 setups.

### The problem

In `FoxessSolar::setup()` the current code calls:

```cpp
if (this->flow_control_pin_ != nullptr) {
    this->flow_control_pin_->setup();
}
```

`setup()` configures the pin as an output, but does **not** drive it to a known level. On most ESP32-C3 boards the output register defaults to `0` (LOW) at power-on, which is the receive direction — so in practice it probably works. However:

- The GPIO output register state after `setup()` is **implementation-defined** in ESPHome's GPIO abstraction and can vary between boards and framework versions.
- If the pin starts HIGH (transmit), the transceiver holds the RS485 bus in driver mode, which actively suppresses any data the inverter tries to send. The ESP would receive nothing, time out after 300 s, and log the inverter as Offline — with no obvious error message indicating the cause.
- The issue is intermittent (only on cold boots where the GPIO register happens to start HIGH) and very hard to reproduce and diagnose.

### The fix

Add a single line immediately after `this->flow_control_pin_->setup()`:

```cpp
void FoxessSolar::setup() {
  ESP_LOGVV(TAG, "setup start");

  if (this->flow_control_pin_ != nullptr) {
    this->flow_control_pin_->setup();
    this->flow_control_pin_->digital_write(false);  // ← assert receive mode
  }

  this->millis_lastmessage_ = millis();
}
```

`digital_write(false)` drives DE LOW → RE̅ LOW → transceiver in receive mode, guaranteed, before the first `update()` poll fires.

### How to test

1. Flash the change.
2. Power-cycle the ESP (do **not** just OTA-reset, because OTA preserves the GPIO state from the previous run).
3. Observe that the inverter appears Online within a few seconds and no CRC errors appear in the log.
4. Swap A/B RS485 wires deliberately to confirm CRC errors appear (proves the line is live), then swap back.

### Risk

Zero functional risk — this is the correct default for a receive-only Modbus listener. The only scenario where this could cause a problem is if the upstream code ever needs to transmit a Modbus request before calling `setup()`, which it does not.

---

## TODO 2: Migrate from Arduino framework to ESP-IDF [4-C]

### Background

ESPHome supports two firmware frameworks for ESP32 variants:

| Framework | Default for ESP32-C3 | Flash size | RAM usage | UART driver |
|-----------|---------------------|------------|-----------|-------------|
| Arduino | No (legacy default) | Larger | Higher | Blocking, Arduino Serial |
| ESP-IDF | Yes (recommended) | ~15 % smaller | ~30 kB less | Native, async, DMA-capable |

The current `foxess-inverter.yaml` uses `framework: type: arduino`. This works, but ESP-IDF is Espressif's native SDK and is now the officially recommended framework in ESPHome for all ESP32-C3 boards. It offers better async UART, lower memory overhead, and longer-term support.

### How to change the config

In `foxess-inverter.yaml`, replace:

```yaml
esp32:
  board: esp32-c3-devkitm-1
  framework:
    type: arduino
```

with:

```yaml
esp32:
  board: esp32-c3-devkitm-1
  framework:
    type: esp-idf
```

That is the only YAML change required. ESPHome handles the rest.

### What to test before committing

1. **Clean build** — run `esphome compile foxess-inverter.yaml` from scratch after deleting `.esphome/build/` for the device. ESP-IDF compiles differently and the first build will take 5–10 minutes.
2. **OTA** — confirm OTA still works after flashing the ESP-IDF build over USB. (The partition layout changes; a full USB flash may be needed for the first ESP-IDF build.)
3. **UART behaviour** — verify the inverter appears Online within the normal timeout. Watch for:
   - Unexpected `header mismatch` or CRC errors (would indicate timing differences in byte delivery between Arduino Serial and the ESP-IDF UART driver).
   - Increased or decreased latency between `update()` calls.
4. **Logging** — confirm `logger: baud_rate: 0` (USB CDC disabled) still suppresses serial output on ESP-IDF. Some ESPHome versions use a different CDC implementation under ESP-IDF.
5. **Status LED** — confirm GPIO7 inverted active-low LED still behaves correctly.
6. **API encryption** — confirm Home Assistant can still connect and the API key is accepted.
7. **OTA on subsequent flashes** — flash a second time OTA to prove the workflow works end-to-end.

### Known caveats

- If you use the `web_server:` component (currently enabled), confirm the web interface loads correctly. The HTTP server implementation differs slightly between frameworks.
- `captive_portal:` should work identically.
- If any future external component uses Arduino-specific APIs (`WiFiClient`, `Preferences`, etc.), it will need to be ported or replaced.

### Rollback

If the ESP-IDF build causes regressions, revert to `type: arduino` and do a full USB reflash.

---
