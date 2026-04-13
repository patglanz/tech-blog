---
title: "Einen präzisen Temperatur- und Feuchtesensor für Home Assistant unter 20 Euro sebst bauen"
date: 2026-04-13T17:30:00+02:00
draft: false
tags: ["Home Assistant", "ESPHome", "DIY", "Hardware", "Sensoren"]
categories: ["Tutorials"]
cover:
    image: "images/esp32_bme280_title.jpg" # Pfad relativ zu 'static'
    alt: "ESP32 mit BME280" # Für Barrierefreiheit
    caption: "Einen präziseren Temperatur- und Feuchtesensor für Home Assistant bekommst Du nicht!" # Text $
    relative: false # Falls das Bild nicht im selben Ordner wie der Post liegt
---
---

Wer kennt es nicht: Kommerzielle Sensoren weichen oft 2-3 Grad von der Realtemperatur ab und haben eine Auflösung von 0,5 Grad, also umgangssprachlich wahre Schätzeisen, die man nicht wirklich für Automatisierungen mit Home Assistant nutzen kann.

 Heute bauen wir uns für unter 20 Euro einen Sensor, der fast auf Industriestandard misst – basierend auf einem ESP32 und einem BME280-Sensor.
 Auch wenn es sehr verlockend ist, empfehle ich, die Teile nicht bei AL*Exzess zu bestellen. Leider wird gerade der BME280 oft gefälscht oder ein BMP mit eingeschränktem Funktionsumfang geliefert. Das ist dann sehr ärgerlich, wenn untenstehende Anleitung einfach nicht mehr passt und Du frustriert aufgeben musst.
Qualitativ sehr gute Boards und Sensoren habe ich bislang immer von https://www.berrybase.de/ erhalten, meine 5 Cents.

## Die Software
Selbstredend solltest Home Assistant am Start haben. Falls Du es noch nicht getan hast, installiere die ESP Homebuilder-Extension: https://esphome.io/guides/getting_started_hassio/

## Die Hardware-Liste
Um dieses Projekt nachzubauen, brauchst du ein paar Dinge:
* **ESP32 Dev Board** (ca. 5-7 €).
* **BME280 Sensor Modul** (hohe Präzision, ca. 3-7 €).
* **Jumper Wire Kabel** (Bei der Firma Amazing oder auf der Bucht Nach "Dupont-Kabel" suchen, weiblich-weiblich).
* **USB (C)-Netzteil**, idealerweise mehr als 1A.
* **3D-Drucker**, für ein schickes Gehäuse.
* **Lötkolben**, um die Stiftleiste in das BME280 Modul einzulöten.


## Vorbereiten und initiales Flashen des ESP32

Bevor Du den ESP32 mit dem BME280-Sensor verkabelst, solltest Du ihn zuerst mit der ESPhome-Firmware flashen.
Das Flashen mit ESPhome Firmware ist recht einfach: Verbinde das ESP32-Board mit einem  geeigneten USB-C-Kabel mit Deinem Rechner (nicht dem Home Assistand-Computer!!). Benutze **Chrome** als Web Browser und gehe zu https://web.esphome.io/. Klicke auf "Connect":
{{<figure src="/images/esphome_connect.jpg" width="700px" title="Computer mit ESP32 verbinden">}}

Du solltest nun ein Auswahlfenster sehen, in dem Du eine USB(serielle) Verbindung auswählen kannst. Siehst Du diese Option nicht, ist Dein USB-Kabel in 95% aller Fälle nicht für dieses Projekt geeignet. Probiere ein anderes, für Datenübertragung geeigneteres Kabel aus.
{{<figure src="/images/esphome_connect2.jpg" width="500px" title="Serielle USB-Verbindung auswählen">}}

Dann mit Klicken auf "Prepare for first use" das Modul für die Erstbenutzung vorbereiten (flashen):
{{<figure src="/images/esphome_first_use.jpg" width="500px" title="ESP-Modul für die Erstbenutzung vorbereiten">}}

Wenn das Modul "geflasht" ist, unterbreche die Verbindung zum Rechner nicht direkt, sondern konfiguriere direkt im Anschluss die Wireless-Verbindung: 

{{<figure src="/images/esphome_wifi_config.jpg" width="500px" title="Wifi für das ESP32-Modul konfigurieren.">}}


## Verkabelung (I2C)
Jetzt kannst Du den BME280 Sensor anschließen. Die Verkabelung ist denkbar einfach, da wir den I2C-Bus nutzen:
* **VCC** an 3.3V
* **GND** an GND
* **SCL (SCK)** an GPIO 22 (D22)
* **SDA (SDI)** an GPIO 21 (D21)

{{<figure src="/images/ESP32-BME280-wiring.jpg" width="400px" title="So wird der BME280 mit dem ESP32 verdrahtet">}}


## Die Software: ESPHome Code
Hier ist die Konfiguration, die du direkt in ESPHome auf Dein frisch erkanntes ESP32-Device im ESPHome-GUI von Home Assistant einfügen kannst und "over the air" auf dem ESP32-Modul mit angeschlossenem BME280-Modul installieren kannst. Denke nur daran, WiFi SSID und Passwort anzuzupassen, und den Device-Namen.

```yaml
esphome:
  name: esphome-something
  friendly_name: precise_temp_sensor
  min_version: 2025.9.0
  name_add_mac_suffix: false

esp32:
  variant: esp32
  framework:
    type: esp-idf

# Enable logging
logger:

# Enable Home Assistant API
api:

# Allow Over-The-Air updates
ota:
- platform: esphome

wifi:
  ssid: "secret wifi_ssid"
  password: "secret wifi_password" 

i2c:
    sda: GPIO22 # Connected to SDA on the BME280
    scl: GPIO21 # Connected to SCL on the BME280
    scan: true  # Scans for I2C devices on startup, helpful for debugging

# Sensor configuration
sensor:
  - platform: bme280_i2c
    # The I2C address is typically 0x76 or 0x77.
    # If 0x76 doesn't work, try 0x77. The 'scan: true' above will show the correct address in the logs.
    address: 0x76
    update_interval: 60s
    temperature:
      name: "BME280 Temperature"
      # Optional: Add an offset if your sensor reading is slightly off
      # offset: -1.2
    pressure:
      name: "BME280 Pressure"
    humidity:
      name: "BME280 Humidity"
```

Nachdem Du den YAML-Code auf Dein ESP32-Board gepusht hast und alles glatt gelaufen ist, solltest Du im Home Assistant Dashboard nun unter Settings/Entites Deinen neunen Sensor finden. Einfach in der Suchzeile mal "ESP..." eingeben und den gefundenen Sensor einem Dasboard hinzufügen. Die Präzision dieses Setups ESP32 mit BME280 Sensor begeistert mich immer wieder:

{{<figure src="/images/temperature_flow.jpg" width="800px" title="Ein sehr präziser Temperaturverlauf, hier mein Außenthermometer">}}