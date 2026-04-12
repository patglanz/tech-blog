---
title: "Bosch Brauchwasserwärmepumpe Compress 5000 DW in Home Assistant einbinden"
date: 2026-04-12T19:00:00+02:00
draft: false # Solange das auf "true" steht, geht es nicht online!
tags: ["Home Assistant", "Bosch", "Wärmepumpe", "Smarthome"]
categories: ["Tutorials"]
cover:
    image: "images/bosch_compress_5000dw.jpg" # Pfad relativ zu 'static'
    alt: "Die Bosch Wärmepumpe 5000DW" # Für Barrierefreiheit
    caption: "Nicht so smarte Geräte Home Assistant ready machen. Einen Gabelschlüssel braucht ihr dazu nicht." # Text unter dem Bild
    relative: false # Falls das Bild nicht im selben Ordner wie der Post liegt
---
Seit kurzem bin ich Besitzer einer Warmwasserwärmepumpe Bosch Compress 5000 DW, die mir von einem SHK-Fachbetrieb an unseren Brauchwasserkreislauf in unserem Haus angeschlossen wurde.

Ich war nach Einbau des Gerätes und Sichtung der Anleitung überrascht, dass Die Compress 5000 DW keinerlei digitale Schnittstellen nach “außen” hat, weder eine WLAN-Anbindung für lokale/cloud based Services oder eine anders geartete digitale Schnittstelle, mit der man programmatisch das Verhalten und das Timing des Aufheizens oder der wöchentlichen Legionellenbehandlung des Gerätes beeinflussen könnte.
Es gibt allerdings einen “Ein/Aus-Kontakt” in der Elektronikbox der Warmwasserwärmepumpe, der in der Anleitung (https://bosch-de-de.boschhc-documents.com/download/file/file/6721842158.pdf) als “​​EIN/AUS-Kontakt für PV-Wechselrichter” bezeichnet wird.
Leider ist die Dokumentation der Wärmepumpe unzureichend und irreführend, sodass mir eigentlich nur der experimentelle Weg blieb, um die Möglichkeiten des Kontakts auszuloten.

Ab Werk hat dieser Kontakt in der Elektronikbox der Wärmepumpe den Status “geschlossen”, einfach zu erkennen an der werksseitig gelegten Kabelbrücke:
{{<figure src="/images/pv_switch_pw5000.jpg" width="600px" title="Der Photovoltaik-Schaltkontakt, werksseitig gebrückt">}}

Ist der Kontakt unterbrochen, wird bei aktiviertem Display “P7” angezeigt:
{{<figure src="/images/p7_pw5000.jpg" width="600px" title="Displayanzeige, wenn der Photovoltaik-Kontakt offen ist">}}

Bei unterbrochenem Kontakt heizt die Pumpe das Wasser auch bei einem Temperaturabfall von mehr als 5 Grad (Werkseinstellung) unter der Solltemperatur nicht mehr auf. 
Der Legionellenschutz ist hiervon nicht betroffen. Einmal wöchentlich (ab dem Tag der Einschaltung der Pumpe) wird die Wassertemperatur mittels des Zuheizers auf 65 Grad gebracht (Werkseinstellung)
Bei geschlossenem Kontakt wird ebenso nicht aufgeheizt, wenn die momentane Zeit außerhalb der am Display eingestellten Timerzeitspanne liegt!

Damit Bosch-Warmwasserwärmepumpe anfängt, das Wasser aufzuheizen wird, müssen 3 Dinge erfüllt sein: 

1. Der Photovoltaikkontakt muss geschlossen sein (anstatt P7 wird die aktuelle Wassertemperatur angezeigt)
2. Der Zeitpunkt muss innerhalb der an der Wärmepumpe eingestellten Timerzeitspanne liegen.
3. Die Warmwassertemperatur muss mehr als 5 Grad unterhalb der eingestellten Solltemperatur liegen (Werkseinstellung)

Der einzige extern beeinflussbare Faktor zum Einschalten des Aufheizvorgangs ist also Punkt 1, der Photovoltaikkontakt.
Damit das Aufheizen der Warmwasserwärmepumpe erst dann geschieht, wenn die Solarbatterie voll aufgeladen ist, war meine Idee, den Kontakt potentialfrei über einen in Home Assistant eingebundenen Shelly 1-Schalter anzusteuern:
{{<figure src="/images/circuit_diagram_pw5000.jpg" width="800px" title="Shelly 1: PV-Eingang potentialfrei schalten">}}

Sicherlich wäre es auch möglich, den Shelly-Schalter auch innerhalb der Elektronikbox der Wärmepumpe unterzubringen und die 220V Versorgungsspannung dort abzugreifen. Aus Garantiegründen und da die Verkabelung innerhalb der Wärmepumpe bereits werkseitig sehr unsauber (keine geraden Kabelführungen, offenbar musste an Kabellängen gespart werden) ausgeführt ist, habe ich mich dazu entschlossen, den Shelly-Schalter außerhalb der Wärmepumpe in einer Verteilerdose unterzubringen. Wahrscheinlich ist das auch besser für den WLAN-Empfang des Shelly-Schalters:
{{<figure src="/images/shelly_pw5000.jpg" width="600px" title="Der Shelly in einer Verteilerdose">}}

(Das schwarze Kabel, das an der rechten Seite der Verteilerdose abgeht, ist das Schaltkabel, das mit dem Photovoltaik Ein/Aus-Port innerhalb der Elektronikbox verbunden ist.)

Zur Steuerung des Shelly 1-Schalters und somit der Warmwasserwärmepumpe habe ich insgesamt 3 Automatisierungen in Home Assistant angelegt.

Automatisierung 1:
{{<figure src="/images/pw5000_automation1.jpg" width="600px" title="Solarbatterie über 98%, loslegen mit Laden! ">}}

Zweck: Wenn der Ladezustand der Photovoltaikbatterie über 98% ansteigt und für 5 Minuten darüber bleibt und es gleichzeitig nach 9:00 morgens ist, dann soll der Shelly Switch (hier als Water heat pump booster switch bezeichnet) angeschaltet werden.

Automatisierung 2:
{{<figure src="/images/pw5000_automation2.jpg" width="600px" title="Nach 16:00 müssen wir mit Warmwasserbereitung loslegen, so oder so, ob die Batterie voll ist oder nicht.">}}

Zweck: Wenn bis 16:00 Nachmittags der Batterieladezustand immer noch nicht über 98% ist, brauchen wir ja trotzdem warmes Wasser. Deswegen schalten wir den Shelly-Switch jetzt an.

Automatisierung 3:
{{<figure src="/images/pw5000_automation3.jpg" width="600px" title="Nach 18:00 wird nichts mehr aufgeheizt.">}}

Zweck: Nach 18:00 schalten wir den Shelly-Schalter aus, um zu verhindern, dass die Warmwasserwärmepumpe Abends noch mal nachheizt. Dieser Zeitpunkt muss im Winter gegebenenfalls etwas nach hinten verschoben werden, da bei kälteren Temperaturen 2 Stunden Aufheizzeit wahrscheinlich nicht genügen.

Bislang funktionieren diese Automatisierungen gut. Es wird dann aufgeheizt, wenn die Batterie voll ist und die Einspeisung der PV-Anlage beginnt.
Die Solltemperatur habe ich von werksseitig 55 Grad auf 50 Grad abgesenkt. Bei täglich drei Duschvorgängen ist das Wasser am nächsten Tag auf ca. 43 Grad heruntergekühlt und wird, bei im Moment sommerlich warmen Außentemperaturen (August 2025) in zirka 2 Stunden bei insgesamt 1-1,5 KW/h Stromverbrauch auf 50 Grad aufgeheizt.

Da man aus der Ferne keinerlei Informationen von der Wärmepumpe über ihren Betriebszustand erhalten kann, ohne einen Blick auf das Display des Gerätes zu werfen, habe ich mich entschlossen, den Warmwasserabgang der Wärmepumpe mit einem in Home Assistant eingebundenen ESP8266 Board mit BME280-Temperatursensor auszustatten. Der BME280-Sensor kann zwar mehr als nötig (Temperatur, Luftdruck und relative Luftfeuchte), war aber gerade vorhanden.

Hier das verwendete Board:
https://www.berrybase.de/nodemcu-v3-esp8266-development-board-ch340g

Hier der Sensor:
https://www.berrybase.de/gy-bme280-breakout-board-3in1-sensor-fuer-temperatur-luftfeuchtigkeit-und-luftdruck

Dieses Gehäuse habe ich für das Board gedruckt:
https://www.thingiverse.com/thing:5873177

So habe ich das Board und den Sensor angebracht. Rot umkreist sieht man den BME280-Sensor, angebracht am Warmwasserauslass der Warmwasserwärmepumpe:
{{<figure src="/images/pw5000_esp_bme280.jpg" width="600px" title="Der ESP-Temperatursensor">}}

Der Sensor am Auslass der Wärmepumpe kann natürlich nicht die tatsächliche Wassertemperatur im Wasserspeicher messen, aber man kann sehr gut Zapf-und Aufheizvorgänge in Homeassistant sehen:

{{<figure src="/images/pw5000_water_temp.jpg" width="600px" title="Der ESP-Temperatursensor">}}




