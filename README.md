# esphome-common

Plantillas base **reutilizables** de ESPHome (WiFi, API cifrada, OTA, diagnósticos,
entradas y control), pensadas para consumirse como **git/remote packages** desde el
YAML de cada dispositivo. Un dispositivo solo declara sus `substitutions` + los
`packages` que necesita; lo común vive aquí, versionado.

> Repo hermano de [`ESPHome_ESP-NOW`](https://github.com/ydiaz1699/ESPHome_ESP-NOW)
> (ese es para nodos ESP-NOW autónomos sin WiFi/HA). **Este** es para placas con WiFi
> + API nativa de Home Assistant.

---

## Estructura

```
esphome-common/
├── common/
│   ├── base.yaml          esphome + min_version, logger, api, ota
│   ├── wifi.yaml          wifi + manual_ip + captive_portal
│   └── diagnostics.yaml   debug, sensores/text_sensor, estado online, botones restart/safe_mode
├── boards/
│   └── esp32.yaml         esp32: variant/flash/framework (ESP-IDF)
└── modules/
    ├── entradas.yaml      pulsador GPIO + TTP223
    └── touch_switch.yaml  switch template + indicador de un Sonoff (fuente de verdad)
```

Cada mapa único vive **una sola vez**: `esphome/logger/api/ota` → `common/base.yaml`;
`esp32` → `boards/esp32.yaml`; `wifi/captive_portal` → `common/wifi.yaml`;
`debug` → `common/diagnostics.yaml`. Los `modules/` aportan solo **listas**
(`binary_sensor`, `switch`) que ESPHome concatena.

---

## ⚠️ Regla de secretos (importante)

Un **remote package NO puede usar `!secret`**. Por eso aquí los valores sensibles son
**substitutions** (`${wifi_ssid}`, `${api_encryption_key}`, …). Cada dispositivo las
rellena desde su **propio** `secrets.yaml` con el patrón *secret → substitution*:

```yaml
substitutions:
  api_encryption_key: !secret pueba_api_key   # !secret se asigna a una substitution
```

Usa claves **únicas por dispositivo** (API y OTA). Genera una clave API con
`openssl rand -base64 32`.

---

## Substitutions requeridas

| Package | Substitutions que necesita |
|---|---|
| `common/base.yaml` | `device_name`, `friendly_name`, `device_comment`, `log_level`, `api_encryption_key`, `ota_password` |
| `common/wifi.yaml` | `wifi_ssid`, `wifi_password`, `wifi_fallback`, `static_ip`, `gateway`, `friendly_name` |
| `common/diagnostics.yaml` | `friendly_name`, `wifi_update_interval`, `uptime_update_interval`, `debug_update_interval` |
| `boards/esp32.yaml` | — (ninguna) |
| `modules/entradas.yaml` | `boton_name`, `boton_pin`, `debounce_time`, `sonoff_entity`, `ttp_name`, `ttp_pin` |
| `modules/touch_switch.yaml` | `sonoff_entity`, `switch_name` |

---

## Cómo consumirlo desde un dispositivo

```yaml
# <dispositivo>.yaml  (en tu instalación ESPHome)
substitutions:
  device_name: pueba
  friendly_name: "Prueba"
  device_comment: "ESP32 con pulsador GPIO23 (Sonoff) + TTP223 GPIO22"
  log_level: "INFO"
  static_ip: "192.168.1.132"
  gateway: "192.168.1.1"
  sonoff_entity: "switch.minir4_sonoff_1002a2995b_1"
  boton_pin: "GPIO23"
  boton_name: "Pulsador GPIO23"
  ttp_pin: "GPIO22"
  ttp_name: "Touch TTP223"
  switch_name: "Interruptor Pulsador"
  debounce_time: "20ms"
  wifi_update_interval: 60s
  uptime_update_interval: 300s
  debug_update_interval: 60s
  # --- secretos: !secret -> substitution (NO van en este repo) ---
  wifi_ssid: !secret wifi_ssid
  wifi_password: !secret wifi_password
  wifi_fallback: !secret wifi_fallback
  api_encryption_key: !secret api_encryption_key
  ota_password: !secret esp32_pruebas_ota_password

packages:
  remoto:
    url: https://github.com/ydiaz1699/esphome-common
    ref: v1.0.0          # fijar a un tag, NO a main (control de cuándo actualizas)
    refresh: 1d
    files:
      - common/base.yaml
      - boards/esp32.yaml
      - common/wifi.yaml
      - common/diagnostics.yaml
      - modules/entradas.yaml
      - modules/touch_switch.yaml
```

> El `secrets.yaml` (con `wifi_ssid`, `api_encryption_key`, etc.) vive **en tu
> instalación**, nunca en este repo.

Validar antes de flashear:

```bash
esphome config <dispositivo>.yaml
```

---

## Versionado

Fija siempre `ref:` a un **tag** (`v1.0.0`), no a `main`: así un cambio en este repo
no entra en tu próxima compilación sin que tú lo decidas (subiendo el `ref`). Al
mejorar las plantillas, publica un tag nuevo (`v1.1.0`, …) y actualiza el `ref` en el
dispositivo cuando quieras adoptarlo.

## Licencia

MIT.
