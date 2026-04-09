---
title: "Hauswasserverbrauch zuverlässig erfassen mit Home Assistant"
date: 2026-04-08
draft: false
tags: ["Home Assistant", "Shelly", "IoT", "Tutorial"]
summary: "Von scheiternden KI-Kameras zu einer soliden Impulszählung am Wasserzähler."
cover:
    image: "images/wasserzaehler.jpg" # Pfad relativ zu 'static'
    alt: "Impulswasserzähler mit Shelly 1 Gen4" # Für Barrierefreiheit
    caption: "Wasserverbrauch zuverlässig messen" # Text unter dem Bild
    relative: false # Falls das Bild nicht im selben Ordner wie der Post liegt
---
## Die erste Etappe: AI on the edge (Der Fehlversuch)
Unseren Hauswasserverbrauch präzise zu messen, vor Leckagen gewarnt zu werden und generell einen besseren Überblick über den Frischwasserkonsum in unserem Haushalt zu erhalten, war schon seit längerem eines meiner Integrationsziele für meine Home Assistant-Instanz.

Das Thema “AI on the edge”, eine Kameraauswertung des analogen Wasserzählers mit hilfe eines recht günstigen Setups, bestehend aus einem ESP32-Kameramodul ESP32-CAM (https://www.berrybase.de/berrybase-esp32-cam-dev-board-ov2640-kamera-usb-zu-seriell-modul-wifi-bt-usb)  und ein paar 3d-gedruckten Komponenten, war preislich recht verlockend (10 Euro plus versand für das ESP-Kamera-Modul sind echt ein Schnapper).
 Die Setup-Dokumentation ist auch recht gut: https://www.home-assistant.io/docs/energy/water/ und https://github.com/jomjol/AI-on-the-edge-device und genügend 3D-Modelle, um das Ganze auf eine analoge Wasseruhr draufzubasteln gibt es auch, z.B. hier:
https://makerworld.com/en/search/models?keyword=AI+on+the+edge

## Mein Scheitern auf der ersten Etappe

Was ich nicht wusste: Das ESP32-CAM-Modul mit dem On Board-Wifimodul verlangt ohne externe Antenne einen 2.4 Ghz-Access Point in einem Radius von geschätzt 5-6 Metern. Du brauchst also zwingend eine externe Antenne, z.B. die hier: https://www.amazon.de/dp/B0CJY47TTJ Ansonsten bleibt die WiFi-Verbindung zu Deiner “AI on the edge”-Instanz dunkel.
Auch mit der externen Antenne musste ich, um eine stabile Verbindung zum “AI on the Edge”-Setup zu erreichen, einen zusätzlichen WiFi-Access-Point im Keller installieren.

Bei den ESP32-CAMs gibt es wohl auch Qualitätsunterschiede, insbesondere beim Objektiv. Ich habe an meinem Setup stundenlang herumjustiert (Brennweite, Beleuchtung), leider ohne Erfolg. Nur der Rollenzähler wurde abgelesen, somit hatte ich bei meiner amtlichen Wasseruhr nur eine Genauigkeit von 1000 Litern.

Mein erster Prototyp sah so aus:
{{<figure src="/images/espprototype.jpg" width="300px" title="der erste Wurf">}}
An der Stelle habe ich beschlossen, das Projekt “AI on the edge” mit der ESP32-CAM zu beenden. Das war zu viel Aufwand und zu wenig Ergebnis für mich, leider.

## Plan B: Ein zweiter Wasserzähler
Die Idee: Einen Impulswasserzähler einzusetzen. Impulswasserzähler werden mit einem Impulsausgang geliefert: Ein zweiadriges Kabel, über das via Reedkontakt nach Durchfluss einer bestimmten Wassermenge (1,10,100 Liter) ein Schalter geschlossen wird.
Ich habe mir dann einen “Diehl AQUARIUS P mit Impulsausgang (Reedschalter), Kabellänge 2 m, Kontaktbelastung 24 V ~ 0.2 A” zugelegt. Die Idee: Die Impulse am Kabel des Wasserzählers mit einem Shelly 1 Gen4-Modul zu zählen. 
(Den Einbau eines solchen Zählers in die Hauptwasserleitung sollte man unbedingt vom Fachmann vornehmen lassen.)
    {{< figure src="/images/aquariusp.jpg" width="200px" title="der Impulszähler" >}}
    {{< figure src="/images/shelly1gen4.jpg" width="200px" title="Das Shelly 1-Modul" >}}

## Wasserverbrauch mit Impulszähler, Shelly 1 und Home Assistant erfassen
Nachdem der Wasserzähler vom Fachmann eingebaut wurde😉, kann die Verkabelung mit dem Shelly wie folgt angelegt werden:
{{< figure src="/images/blueprint.jpg" width="300px" title="Wiring" >}}

Das Netzteil ist hier etwas übertrieben dargestellt. Ein einfaches Steckernetzteil mit 12V/1A reicht vollkommen aus. 
### ⚠️ ACHTUNG: SICHERHEITSHINWEIS
AUF KEINEN FALL DEN SHELLY IN DIESEM SETUP MIT 220V BETREIBEN! Die Klemme SW (Switch) am Shelly hat keine galvanische Trennung zur Stromversorgung. Wenn der Shelly an 230V angeschlossen ist, liegt auf dem dünnen Kabel, das von der SW-Klemme zum Wasserzähler führt, die volle Netzspannung an und wird den Reedkontakt zerstören!! Zur Stromversorgung des Shelly Moduls in diesem Setup immer nur ein 12V Steckernetzteil benutzen.

Der eingebaute Zähler inklusive Shelly 1 Gen4 im im 3-D-Gedrucktem Gehäuse sieht bei mir so aus:
    {{< figure src="/images/insitu.jpg" width="700px" title="der Impulszähler, mit Shelly-Modul" >}}
Das weiße Gehäuse über dem oberen Absperrschieber ist ein 3D-gedrucktes Gehäuse für das Shelly-Modul. Das schützt das Shelly 1 vor Feuchtigkeit und kann hier downgeloadet werden: https://makerworld.com/en/models/470912-shelly-1-plus-wallmount

Wichtig: Nach Einbindung des Shelly 1 in das Netzwerk sollte der Schalterport auf “detached” gestellt werden, um ein Mitlaufen des Relais zu verhindern, wenn der Zähler einen Impuls sendet:
{{< figure src="/images/shellywebui.jpg" width="600px" title="Shelly 1 Web Interface" >}}

Den Shelly in Home Assistant einbinden: https://www.home-assistant.io/integrations/shelly/

## Home Assistant: Die Impulse zählen
Nachdem der Shelly in Home Assistant eingebunden wurde, hat man nun eine Entität für den Shelly (meist ein Schalter oder Binärsensor), der bei jedem Impuls kurz von Aus auf An springt.\
Um diese Impulse zu sammeln, legt man einen Zähler an:\
Man geht  in Home Assistant auf Einstellungen -> Geräte & Dienste -> Helfer.\
Unten rechts auf Helfer erstellen gehen und Zähler wählen (Counter).\
 Z.B. Wasser Impulse (der Startwert ist 0) nennen und auf Erstellen klicken.\
Nun zu Einstellungen -> Automatisierungen gehen und eine neue Automatisierung erstellen:\
Auslöser (Trigger): Zustand. Entität ist der Shelly-Schalter/Eingang. Von Aus (off) zu An (on).\
 In meinem Fall habe ich den Shelly 1 “watermeter” genannt. Entsprechend hieß der sensor/Schalter des Shelly:  “binary_sensor.watermeter_input_0”\
Aktion: Dienst ausführen.\ Zähler: Erhöhen (counter.increment) wählen und den neuen Zähler "Wasser Impulse" als Ziel auswählen.\ Automatisierung speichern. Jetzt zählt Home Assistant jeden Impuls (1, 2, 3...) hoch.

hier der YAML-Code meiner Automatisierung, um mit dem “Puls” des Shelly-Binärsensors den Zähler (Die Helpervariable) hochzusetzen (binary_sensor.watermeter_input_0 ist der Sensor des Shellys, counter.water_pulse ist die Helper-Variable.)

```yaml
alias: "Water consuption: Count the pulse"
description: Increases the helper counter for every pulse received.
triggers:
  - entity_id: binary_sensor.watermeter_input_0
    from: "off"
    to: "on"
    trigger: state
actions:
  - action: counter.increment
    target:
      entity_id: counter.water_pulse
mode: single
```

Mit diesem Setup hat man nun einen für alle Ewigkeit kumulierenden Zähler, der jedesmal, wenn die Wasseruhr einen “Puls” an den Shelly sendet, “1” weiterzählt.\
In einem nächsten Schritt wird eine weitere Helfervariable angelegt, die einen “Puls” als 10 Liter darstellt und zählt. Hierzu müssen die Daten des im vorherigen Schrittes angelegten Pulszählers in einer neu angelegten Helfervariable umgerechnet werden:\
Gehe in Home Assistant in der linken Seitenleiste auf: **Einstellungen -> Geräte & Dienste: Reiter Helfer**.\
Klicke unten rechts auf Helfer erstellen.\
Wähle in der Liste Template und danach Template für einen Sensor erstellen aus.\
Fülle die Felder in dem neuen Fenster exakt so aus:\
Name: Wasserverbrauch Gesamt (oder ein Name deiner Wahl)\
Zustandstemplate: Kopiere den folgenden Code genau so hinein:\
Code-Snippet
```{{ states('counter.water_pulse') | float(0) * 10 }}```


Maßeinheit: L (Wichtig: Großes L für Liter)\
Geräteklasse: Water (Wasser)\
Zustandsklasse: Total increasing (Total zunehmend)
{{<figure src="/images/templatesensor.jpg" width="500px" title="Template für den Sensor">}}

Klicke auf Absenden.\
Damit ist die Umrechnung fertig! Home Assistant erstellt nun automatisch die Entität sensor.wasserverbrauch_gesamt.\
Dieser Sensor nimmt ab sofort die reine Impuls-Zahl aus deinem Counter (counter.water_pulse), rechnet immer sofort eine Null hinten dran (mal 10) und meldet dem System das Ergebnis als echte Liter. Genau diesen neuen Sensor kannst du jetzt im Energie-Dashboard unter "Wasserverbrauch" eintragen. Dann solltest Du in kürzester Zeit auch schon Deinen Wasserverbrauch im Energiedashboard sehen:
{{<figure src="/images/waterconsumption.jpg" width="700px" title="Der Wasserverbrauch im Energiedashboard">}}

