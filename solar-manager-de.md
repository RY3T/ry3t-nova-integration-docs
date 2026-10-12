# RY3T Nova im Solar Manager einbinden

Anleitung für Installateure und Kunden, die einen RY3T Nova mit dem
[Solar Manager](https://www.solarmanager.ch) betreiben.

## Einstellungen am RY3T Nova

Am RY3T Nova muss nichts konfiguriert werden.
Die Konfiguration der Software auf dem RY3T Nova erfolgt durch RY3T direkt.

> [!NOTE]
> Die IP-Adresse des Solar Managers muss im Router im DHCP-Server fix
> reserviert werden, da RY3T die IP-Adressen, mit denen der RY3T Nova
> kommunizieren darf, intern freigeben muss. Bitte teilen Sie RY3T diese
> IP-Adresse mit (support@ry3t.com).
>
> Reservieren Sie im Router ebenfalls die IP-Adresse des RY3T Nova, damit der
> Solar Manager ihn immer unter derselben Adresse erreicht.

## Einstellungen am Solar Manager

### Gerät hinzufügen

Fügen Sie ein neues "Gerät hinzufügen" und wählen Sie "Warmwasser".
Wählen Sie danach aus der Liste den "RY3T Nova".
Geben Sie die IP-Adresse des RY3T Nova ein.
Parametrieren Sie das Gerät wie gewünscht.
Speichern Sie danach die Einstellungen.

### Anzahl Miner

Ein RY3T Nova besteht aus einem bis drei Minern und erscheint im Solar Manager
als ein einziges Gerät. Die Anzahl Miner und den möglichen Leistungsbereich
meldet der RY3T Nova selbst. Eine Einstellung dafür ist nicht nötig.

### So regelt der RY3T Nova

Der Solar Manager gibt dem RY3T Nova eine Leistung vor, zum Beispiel den
aktuellen PV-Überschuss. Der RY3T Nova verteilt diese Leistung auf seine Miner.
Pro Miner sind ca. 500 W bis 3000 W möglich; ein RY3T Nova mit drei Minern
deckt also ca. 500 W bis 9000 W ab. Liegt die Vorgabe unter der
Minimalleistung, pausiert der RY3T Nova.

Die Miner fahren ihre Leistung schrittweise hoch und herunter. Bis eine neue
Vorgabe vollständig erreicht ist, kann es bis zu etwa zwei Minuten pro Miner
dauern.

### Steuerung nur über den Solar Manager

Der RY3T Nova sollte nur vom Solar Manager gesteuert werden. Er führt jeweils
die zuletzt erhaltene Vorgabe aus. Steuert ein weiteres System (z. B. Home
Assistant) gleichzeitig, überschreiben sich die Vorgaben gegenseitig.

---

Technische Details zur Schnittstelle: [Modbus TCP Integration Guide](modbus.md)
(Englisch). Support: support@ry3t.com
