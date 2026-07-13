# Aku EMS – RGC3P

**Energy Management System pro akumulační nádrž s 3-fázovým SSR relé Carlo Gavazzi RGC3P60V20EDP**

## Popis

Systém reguluje výkon topné spirály akumulační nádrže (bojleru) pomocí 3-fázového SSR relé s fázovým řízením. Řídící jednotka ESP32 (DIN module SB) komunikuje s Home Assistant a čte data z Victron Cerbo GX přes sfstar/hass-victron integraci.

## Hardware

| Komponenta | Model | Stav |
|---|---|---|
| Řídící jednotka | DIN module SB (ESP32) | ✅ funkční |
| 3-fázové SSR | Carlo Gavazzi RGC3P60V20EDP | ✅ funkční |
| DAC převodník | DFRobot GP8101S (DFR1036) | ⚠️ poškozený – náhrada na cestě |
| Napájení | Meanwell 24V DC | ✅ funkční |
| Baterie/invertor | Victron Cerbo GX | ✅ online |
| Teplotní senzor | MQTT sensor (Kotelna P AKU) | ✅ funkční |

## Aktuální stav

- **GPIO12** (SSR pin RJ45) – **vadný**, nepoužíván
- **GPIO13** (Stykač1 pin RJ45) – funkční, přemapován jako SSR výstup
- **GP8101S** – pravděpodobně poškozen hroty multimetru při testovacím měření (modul byl před měřením funkční), náhrada objednána
- Firmware **v3.0** nasazen, OTA přes ethernet

## Signálový řetězec

```
ESP32 GPIO13 (LEDC 1kHz)
  → pin 4 RJ45 (modrá žíla T568B)
  → odpor 2k7 (dělič z 24V na ~5V)
  → GP8101S SIG pin (modrý)
  → GP8101S OUT+ (0–10V DC)
  → RGC3P A4 (řídicí vstup 0–10V)
  → RGC3P A1 (GND reference)
  → 3× fáze L1/L2/L3 → spirála AKU nádrže
```

## RJ45 konektor (T568B) – využité piny

| Pin | Barva T568B | Funkce | Využití |
|---|---|---|---|
| 5 | Bílo-modrá | 24V DC | GP8101S VCC (stejný pár jako pin 4) |
| 4 | Modrá | Stykač1 / GPIO13 | **SSR výstup (přemapováno)** |
| 6 | Zelená | SSR / GPIO12 | ❌ vadný GPIO |

## RGC3P60V20EDP – zapojení řídicích svorek

| Svorka | Připojeno | Funkce |
|---|---|---|
| A1 | GP8101S OUT– / GND | GND reference 0–10V |
| A4 | GP8101S OUT+ | 0–10V řídicí signál |
| Us+ | Meanwell 24V+ | Napájení modulu |
| Us– | Meanwell GND | Napájení GND |

## LED indikace (DIN module SB)

Od USB nahoru:

| LED | Barva | Funkce | Stav |
|---|---|---|---|
| 1 | Zelená | Napájení | Svítí = OK |
| 2 | Červená | Pojistka | Nesvítí = OK |
| 3 | Červená | SSR/GPIO12 | Svítí trvale (GPIO12 vadný) |
| 4 | Červená | Stykač1/GPIO13 | Svítí = idle, Bliká = regulace, Zhasnuta = plný výkon |
| 5 | Červená | Stykač2/GPIO14 | Nevyužito |

## Victron integrace

- Integrace: [sfstar/hass-victron](https://github.com/sfstar/hass-victron)
- SOC entita: `sensor.victron_system_battery_soc_2`
- Teplota AKU: `sensor.teplota_kotelna_p_aku`

## Regulační logika

### SOC regulace (solární přebytek)
1. Baterie dosáhne 100 % → nastaví `full_soc_reached = true`
2. Lineární regulace výkonu v pásmu `SOC_KEEP ± SOC_SPREAD`
3. SOC klesne pod `SOC_THRESHOLD` → regulace se deaktivuje

### PID termostat
- Cílová teplota nastavitelná přes HA / web UI
- PID s autotuningem (spustit po instalaci)
- Koeficienty po autotuning aktualizovat v YAML

### Kombinace
Výstup = `max(PID_výstup, SOC_výstup)` – vyhrává vyšší požadavek.

## Webové rozhraní

```
http://[IP-ESP32]
```

## OTA aktualizace

```
http://[IP-ESP32]/update
```

## TODO

- [ ] Nahradit poškozený GP8101S novou jednotkou
- [ ] Po výměně GP8101S: přepojit modrý vodič na pin 4 RJ45
- [ ] Spustit PID autotuning po stabilizaci systému
- [ ] Aktualizovat PID koeficienty po autotuning

## Verze firmware

| Verze | Datum | Změny |
|---|---|---|
| 1.0 | 2025-07 | Základní struktura, Modbus TCP |
| 2.0 | 2025-07 | Přepracování na HA entity, GPIO13, LEDC |
| 3.0 | 2025-07-12 | Logování, verzování, zobrazení verze na webu |
