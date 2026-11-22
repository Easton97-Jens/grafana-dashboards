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

Für die Systemprotokolle richte anschließend [Syslog mit Alloy und Loki](#5-systemmeldungen-mit-alloy-an-loki-senden) ein. [Grafana-Import](https://grafana.com/docs/grafana/latest/visualizations/dashboards/build-dashboards/import-dashboards/)

## 5. Systemmeldungen mit Alloy an Loki senden

Das [Alloy-Beispiel](../examples/alloy/tp-link-sx3008f.alloy) empfängt RFC3164-Syslog über UDP und wurde mit Alloy 1.22.0 geprüft. Alloy verwendet die Empfangszeit. Es erkennt typische Meldungen für Linkwechsel und fehlgeschlagene Anmeldung. Andere Meldungen bleiben als Systemereignis mit ihrem ursprünglichen Nachrichtentext sichtbar.

1. Setze für den Alloy-Dienst `TPLINK_LOKI_PUSH_URL` auf deinen vollständigen Loki-Schreibendpunkt, zum Beispiel `https://loki.example.invalid/loki/api/v1/push`. Für HTTP Basic Auth über HTTPS aktiviere den auskommentierten Block im Beispiel. Setze `TPLINK_LOKI_USER` und `TPLINK_LOKI_PASSWORD_FILE` lokal. Die Passwortdatei muss für den Alloy-Dienst lesbar sein. Bei einem vorhandenen `loki.write`-Ziel kannst du stattdessen `forward_to` im Block `loki.process` auf dessen Empfänger umstellen und den Beispielblock `loki.write "tplink"` weglassen. Sende und lies bei Mehrmandantenbetrieb im selben Loki-Mandanten.
2. Übernimm die Beispielblöcke in die Alloy-Konfiguration. Prüfe die vollständige Datei mit `alloy validate /etc/alloy/config.alloy` und lade den Dienst mit seiner lokalen Umgebung neu. Für den Test lauscht das Beispiel standardmäßig auf `127.0.0.1:5514`. Diese Einstellung ist vom Switch aus nicht erreichbar.
3. Setze für den Empfang vom Switch `TPLINK_SYSLOG_LISTEN` auf die lokale Empfangsadresse und den benötigten UDP-Port. Begrenze den Zugang in der Firewall auf die Quelladresse des Switches. Aktiviere unter **Maintenance → Logs → Remote Logs** die Weiterleitung an den Alloy-Rechner und wähle die gewünschte Mindestpriorität. TP-Link verwendet standardmäßig UDP 514. Muss der Switch an diesen Port senden, verwende eine auf den Switch begrenzte Weiterleitung zum unprivilegierten Alloy-Port oder gib dem dedizierten Linux-Dienst gezielt `CAP_NET_BIND_SERVICE`, falls das System es für Port 514 verlangt. [TP-Link Remote Logs](https://www.tp-link.com/us/configuration-guides/configuring_system_logs/)

Für einen lokalen Test bei aktivem Loopback-Listener sende eine synthetische Warnung. Verwende Netcat auf dem Alloy-Rechner.

```sh
printf '<132>%s switch-example tp-link: Port 1 link down (Test)\n' \
  "$(LC_ALL=C date '+%b %e %T')" | nc -u -w 1 127.0.0.1 5514
```

Wähle danach in Grafana unter **Explore** die beim Dashboard-Import zugeordnete Loki-Datenquelle und frage die letzten fünf Minuten ab.

```logql
{host="tplink-switch",source="tplink_syslog",category="interfaces"}
  | json syslog_priority="priority", event_type="event_type"
  | syslog_priority="warning"
  | event_type="link_down"
```

Die Meldung erscheint unter **Systemmeldungen** und **Fehler und Warnungen · einzelne Ereignisse**. Die Stream-Labels `host`, `source` und `category` sind auf wenige Werte begrenzt. Die JSON-Meldung enthält `category`, `event_type`, `priority` und `message`. Der ursprüngliche Nachrichtentext kann im eigenen Betrieb interne Angaben enthalten. Beschränke die Leserechte für diese Protokolle entsprechend. [Alloy Syslog-Empfang](https://grafana.com/docs/alloy/latest/reference/components/loki/loki.source.syslog/) · [Loki-Schreibziel](https://grafana.com/docs/alloy/latest/reference/components/loki/loki.write/)
