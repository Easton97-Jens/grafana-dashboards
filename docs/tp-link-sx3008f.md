# TP-Link SX3008F in Grafana einrichten

Das Dashboard zeigt für die acht Frontports Betriebs- und Verwaltungsstatus, Linkrate, Auslastung, Empfang und Senden sowie Paketfehler und verworfene Pakete. Telegraf liest die Portdaten per SNMPv3. Prometheus ruft die von Telegraf bereitgestellten Metriken ab und Grafana zeigt sie an.

## 1. SNMPv3 am Switch einrichten

Aktiviere SNMPv3, lege einen eigenen Monitoring-Benutzer mit Leserechten an und wähle die Sicherheitsstufe **AuthPriv**. Die Leseansicht muss `sysUpTime.0` sowie die verwendeten Spalten aus `ifTable` und `ifXTable` umfassen. Verwende zwei verschiedene Passphrasen. Die Beispielkonfiguration nutzt **SHA** (SHA-1) und **AES-128**. Falls deine Firmware eine stärkere SHA-Variante unterstützt, setze auf beiden Seiten denselben Algorithmus. Erlaube **UDP 161** zum Switch ausschließlich vom Telegraf-Rechner. [TP-Link-Anleitung](https://static.tp-link.com/configurationguide/configuring_snmp_rmon.pdf)

## 2. Telegraf konfigurieren

Das Beispiel verwendet einen Switch, Telegraf 1.40.x und Linux mit systemd ab 250. Prometheus und Telegraf laufen auf demselben Rechner. Kopiere die [Telegraf-Beispielkonfiguration](../examples/telegraf/tp-link-sx3008f.conf) in das Konfigurationsverzeichnis deines Dienstes, bei einer Paketinstallation typischerweise `/etc/telegraf/telegraf.d/`. Ersetze in der lokalen Kopie `switch.example.invalid` und `monitoring-readonly` durch die Adresse und den SNMPv3-Benutzernamen deines Switches. Wenn dein Telegraf-Dienst Port 9273 bereits für Prometheus verwendet, lass in der Beispielkonfiguration den Abschnitt `[[outputs.prometheus_client]]` samt `metric_types` weg. Prüfe, ob der vorhandene Ausgang die Switch-Metriken mit den unten gezeigten Namen bereitstellt.

Speichere die beiden Passphrasen als verschlüsselte systemd-Credentials auf dem Telegraf-Rechner. Die Abfragen lesen sie verdeckt ein.

```sh
sudo -v
sudo mkdir -p /etc/credstore.encrypted
sudo systemd-creds setup
systemd-ask-password -n 'SNMPv3-Authentifizierung' |
  sudo systemd-creds encrypt - /etc/credstore.encrypted/telegraf.tplink_auth
systemd-ask-password -n 'SNMPv3-Verschlüsselung' |
  sudo systemd-creds encrypt - /etc/credstore.encrypted/telegraf.tplink_priv
```

Öffne mit `sudo systemctl edit telegraf` die Ergänzung zum Dienst und hinterlege die beiden Credential-Dateien.

```ini
[Service]
LoadCredentialEncrypted=telegraf.tplink_auth:/etc/credstore.encrypted/telegraf.tplink_auth
LoadCredentialEncrypted=telegraf.tplink_priv:/etc/credstore.encrypted/telegraf.tplink_priv
```

Starte den Dienst mit `sudo systemctl restart telegraf` neu. Mit `sudo journalctl -u telegraf -n 30` kannst du Start- und SNMP-Fehler prüfen. Die Credentials werden über den systemd-Dienst bereitgestellt. Verwende zur Kontrolle daher den laufenden Dienst und seinen lokalen Endpunkt.

```sh
curl -fsS http://127.0.0.1:9273/metrics | grep '^tplink_sx3008f_'
```

Erwartet werden `tplink_sx3008f_poll_uptime_ticks` und Portreihen wie `tplink_sx3008f_port_oper_status` mit den Labels `device="sx3008f"` und `index`. Prüfe bei fehlenden Ports, welche IF-MIB-Indizes die Firmware für die Frontports liefert. Falls sie von `49153` bis `49160` abweichen, passe die Indexliste im Telegraf-Filter, die Grafana-Variable `port` und die Wertzuordnung der Spalte **Anschluss** im Tabellenpanel an. [Telegraf SNMP](https://docs.influxdata.com/telegraf/v1/input-plugins/snmp/) · [systemd-Credentials für Telegraf](https://docs.influxdata.com/telegraf/v1/secretstore-plugins/systemd/)

## 3. Prometheus anbinden

Ergänze den Eintrag unter der vorhandenen Liste `scrape_configs`. Prüfe die Konfiguration mit `promtool check config /etc/prometheus/prometheus.yml` und lade Prometheus neu. Passe den Dateipfad bei Bedarf an. Bei einem vorhandenen Job mit diesem Namen füge das Ziel in dessen Konfiguration ein. Die Labelwerte sind Namenskonventionen des Dashboards.

```yaml
  - job_name: telegraf-dashboard
    scrape_interval: 15s
    static_configs:
      - targets: ["127.0.0.1:9273"]
        labels:
          host: dashboard
```

Kontrolliere in Prometheus **Status → Target health** und frage `up{job="telegraf-dashboard",host="dashboard"}` ab. Der Wert sollte `1` sein. Prüfe danach `count(tplink_sx3008f_port_oper_status{job="telegraf-dashboard",host="dashboard",device="sx3008f"})`. Für einen Switch sollten es nach einigen Abfragen acht Reihen sein. Die Aktualitätsprüfung im Dashboard braucht mindestens zwei unterschiedliche Uptime-Messwerte. Läuft Prometheus auf einem anderen Rechner, binde den Endpunkt an eine für Prometheus erreichbare Adresse und beschränke den Zugriff auf den Prometheus-Rechner. [Prometheus-Konfiguration](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)

## 4. In Grafana importieren

1. Lade [mon-tplink-sx3008f.json](../dashboards/Monitoring/Netzwerkger%C3%A4te/mon-tplink-sx3008f.json) herunter.
2. Öffne **Dashboards → Neu → Importieren** und lade die JSON-Datei hoch.
3. Wähle den Zielordner und ordne **Prometheus · mon-prometheus** deiner Prometheus-Datenquelle und **Loki · mon-loki** deiner Loki-Datenquelle zu.
4. Klicke auf **Importieren** und öffne **Anschlussübersicht · Status und Verkehr**. Wähle im Portfilter einen Anschluss aus, um dessen Verlauf zu sehen. Die Tabelle zeigt die Übersicht aller erfassten Frontports.

Die Systemprotokolle benötigen zusätzlich einen Syslog-Empfänger mit Weiterleitung an Loki. Die Loki-Abfragen erwarten die Stream-Labels `host="tplink-switch"`, `source="tplink_syslog"` und `category`. Die Tabellen lesen `category`, `priority` und `event_type` aus der JSON-Meldung. Bis zu dieser Anbindung bleiben die Log-Panels ohne Meldungen. [Grafana-Import](https://grafana.com/docs/grafana/latest/visualizations/dashboards/build-dashboards/import-dashboards/)
