# esphome-common

**Biblioteca de plantillas ESPHome** reutilizables (WiFi, API cifrada, OTA,
diagnósticos, entradas, control). Fuente única para no repetir ni divergir: se mejora
en un solo sitio y todas las placas se benefician.

Este repo es la **fuente de verdad**. En el NAS existe una **copia local** en
`$dkco/esphome/config/` (un clon de este repo) que se sincroniza con `git pull`/`push`.
Cada dispositivo puede consumir las plantillas por **dos caminos** (ver abajo).

> Repo hermano de [`ESPHome_ESP-NOW`](https://github.com/ydiaz1699/ESPHome_ESP-NOW)
> (ese es para nodos ESP-NOW autónomos sin WiFi/HA). **Este** es para placas con WiFi
> + API nativa de Home Assistant.

---

## Catálogo de plantillas

> **Esta tabla es el catálogo oficial** (fuente única). Al añadir o mejorar una
> plantilla, actualízala aquí. El meta-prompt de `nas-dotfiles` apunta a este catálogo,
> no lo copia.

| Carpeta | Archivo | Qué aporta | Mapas únicos | Substitutions requeridas |
|---|---|---|---|---|
| `common/` | `base.yaml` | `esphome` (+`min_version`), `logger`, `api` cifrada, `ota` | `esphome, logger, api, ota` | `device_name, friendly_name, device_comment, log_level, api_encryption_key, ota_password` |
| `common/` | `wifi.yaml` | WiFi + IP fija + portal de recuperación | `wifi, captive_portal` | `wifi_ssid, wifi_password, wifi_fallback, static_ip, gateway, friendly_name` |
| `common/` | `diagnostics.yaml` | debug, señal/uptime/memoria, IP/SSID/versión, estado online, botones restart/safe_mode | `debug` | `friendly_name, wifi_update_interval, uptime_update_interval, debug_update_interval` |
| `boards/` | `esp32.yaml` | Placa ESP32 genérica, 4MB, ESP-IDF | `esp32` | — |
| `modules/` | `entradas.yaml` | Pulsador GPIO (→Sonoff vía HA) + TTP223 (uso futuro) | — (listas) | `boton_name, boton_pin, debounce_time, sonoff_entity, ttp_name, ttp_pin` |
| `modules/` | `touch_switch.yaml` | Switch template + indicador de un Sonoff (fuente de verdad) | — (listas) | `sonoff_entity, switch_name` |

**Convención de carpetas:** `common/` = igual en todas las placas · `boards/` =
hardware · `modules/` = bloques por responsabilidad reutilizables. Cada **mapa único**
vive en **un solo** archivo; los `modules/` aportan solo **listas** que ESPHome
concatena.

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
`openssl rand -base64 32`. El `secrets.yaml` vive en tu instalación, **nunca** aquí.

---

## Dos caminos de consumo (local y GitHub)

Las plantillas son idénticas en ambos caminos; solo cambia **cómo** las trae el device.

### Camino A — Local (`!include`)

El `common/`/`boards/`/`modules/` están clonados en `$dkco/esphome/config/`. El device
los incluye como archivos locales. **Ventaja:** edición inmediata, sin depender de la
red. **Para:** desarrollo, iterar rápido.

```yaml
packages:
  base: !include common/base.yaml
  placa: !include boards/esp32.yaml
  wifi: !include common/wifi.yaml
  diagnostics: !include common/diagnostics.yaml
  entradas: !include modules/entradas.yaml
  touch_switch: !include modules/touch_switch.yaml
```

### Camino B — GitHub (remote packages)

El device descarga las plantillas de este repo por `github://`, fijando un **tag**.
**Ventaja:** versionado y reutilizable entre instalaciones. **Para:** producción, varias
placas, compartir.

```yaml
packages:
  remoto:
    url: https://github.com/ydiaz1699/esphome-common
    ref: v1.0.0          # tag fijo, NO main (controlas cuándo actualizas)
    refresh: 1d
    files:
      - common/base.yaml
      - boards/esp32.yaml
      - common/wifi.yaml
      - common/diagnostics.yaml
      - modules/entradas.yaml
      - modules/touch_switch.yaml
```

> **Mismo contenido, dos caminos.** Como la copia local es un clon de este repo, lo que
> ves en `!include common/base.yaml` y en `github://…/common/base.yaml@<ref actual>` es
> lo mismo. El Camino A usa el estado del working tree local; el B, el del tag.
> Funciona con **GitHub, GitLab y Codeberg**.

### Device completo de ejemplo (sea cual sea el camino)

```yaml
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
  # …Camino A o Camino B (ver arriba)…
```

Validar antes de flashear:

```bash
esphome config <dispositivo>.yaml
# en el NAS:  docker exec esphome esphome config /config/<dispositivo>.yaml
```

---

## Flujo de mejora continua (para el LLM y el usuario)

Esta biblioteca **crece y mejora**; no se repiten plantillas. Cuando se cree una
plantilla nueva, o el usuario comparta/pida una (p. ej. "aquí tienes un YAML de un
sensor DHT"), seguir **siempre** este flujo:

1. **Mirar antes de crear.** Revisar el catálogo de arriba. Si ya existe algo que
   cubre el caso → **mejorarlo**, no crear un duplicado.
2. **Fusión no-destructiva** (regla de oro del meta-prompt de `nas-dotfiles`): si dos
   versiones hacen lo mismo, comparar y quedarse con la mejor; si cada una aporta algo
   único, fusionar lo mejor de ambas. Nada se borra en silencio.
3. **Parametrizar.** Todo valor configurable → `substitution`. **Nunca `!secret`** en
   estos archivos (regla de remote packages); los secretos los pone el device.
4. **Clasificar bien.** Mapa único → su archivo dueño en `common/` o `boards/`.
   Bloque por responsabilidad → `modules/`. Listas (`sensor`, `binary_sensor`…) con
   `id` únicos.
5. **Actualizar el catálogo** (la tabla de arriba) con la plantilla nueva/mejorada y
   sus substitutions requeridas.
6. **Validar** con `esphome config` sobre un device de prueba antes de publicar.
7. **Versionar:** commit + **tag nuevo** (`v1.1.0`, …). Las placas siguen en su `ref`
   hasta que decidan subirlo. Documentar el cambio en los tags/releases.

> **Dónde ejecuta cada paso:** un LLM en Kiro Web / CLI puede hacer 1–6 y pushear aquí
> (es GitHub). La validación en hardware real y la actualización del `ref` en el NAS
> las hace el usuario por SSH (la LAN del NAS no es accesible desde fuera).

---

## Versionado

Fija siempre `ref:` a un **tag** (`v1.0.0`), no a `main`: así un cambio en este repo no
entra en tu próxima compilación sin que tú lo decidas (subiendo el `ref`). `refresh`
solo controla la caché del mismo `ref`.

- `v1.0.0` — plantillas base iniciales (base, wifi, diagnostics, esp32, entradas,
  touch_switch).

## Licencia

MIT.
