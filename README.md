# Grafana · Monitoring-Dashboards

16 deutsche Dashboards für Hostüberwachung, Netzwerkverbindungen, Logs, Proxmox, DNS, Proxy, UniFi sowie Netzwerkgeräte von ASUS und TP-Link. Die Dashboards wurden aus Grafana 13.3.1 exportiert.

Die Dateien enthalten Dashboard-Konfigurationen. Livewerte, Logeinträge, Datenquellen-Verbindungen und Zugangsdaten gehören nicht zum Export. Gespeicherte Auswahlen und umgebungsspezifische Adressen beziehungsweise Hardwarekennungen wurden bereinigt.

## Import

1. Die benötigten Datenquellen im eigenen Grafana anlegen. Loki, Prometheus und InfluxDB werden nur dort eingerichtet; Zugangsdaten bleiben in Grafana.
2. Die sieben Ordner anhand von `folders.json` zuerst vom übergeordneten zum untergeordneten Ordner anlegen. Die Ressourcen verwenden die Grafana-Folder-API `folder.grafana.app/v1`; `metadata.name` ist die zu erhaltende Ordner-UID, `grafana.app/folder` die Eltern-UID.
3. Die gewünschten JSON-Dateien über **Dashboards → Neu → Importieren** importieren. Datenquellen den Import-Eingaben zuordnen und den jeweiligen Ordner aus `EXPORT-MANIFEST.json` wählen. Dashboard-UIDs beibehalten, damit die relativen Navigationslinks funktionieren.
4. Die unten genannten Platzhalter und die eigene Metrik-/Logstruktur abgleichen. Rechner- und Anschlussauswahl werden mit neutralen Startwerten importiert.

## Datenquellen

| Ursprüngliche Konfigurations-UID | Plugin | Import-Eingabe |
| --- | --- | --- |
| `mon-influx-pve` | influxdb | `${DS_MON_INFLUX_PVE}` |
| `mon-loki` | loki | `${DS_MON_LOKI}` |
| `mon-prometheus` | prometheus | `${DS_MON_PROMETHEUS}` |

Die UID-Spalte dient ausschließlich der Zuordnung. Jede JSON-Datei enthält passende `__inputs`; der Import bindet die ausgewählte eigene Datenquelle. Eingebaute Grafana-/Mixed-Datenquellen bleiben als solche erhalten.

## Abhängigkeiten und Anpassung

- Grafana 13.3.x mit den eingebauten Paneltypen; Ordnerhierarchie benötigt verschachtelte Ordner. Andere Versionen wurden mit diesem Export nicht geprüft.
- Prometheus: vorhandene Host-/Telegraf-Metriken, Monitoring-Regeln und UniFi-Poller-Metriken. Statische Job- und Rollenlabels sind Teil der Abfragekonfiguration.
- Loki liefert strukturierte Journal-, Sicherheits- und Verbindungsereignisse (`connection_snapshot`), UniFi-CEF (`unifi_cef`) und bereinigtes Syslog für ASUS und TP-Link. Parser und Logsammler sind eigene Komponenten.
- Die [Einrichtung für TP-Link SX3008F](docs/tp-link-sx3008f.md) beschreibt SNMPv3, die Telegraf-Beispielkonfiguration, Prometheus und den Grafana-Import.
- InfluxDB: die in den Abfragen verwendeten Messungen, Felder und Bucketnamen müssen im eigenen System vorhanden sein. Erforderliche Rechte der Datenquellen: Lesen.
- Netzwerkübersichten zählen teils wiederholte Sichtungen. Fehlende Daten sind keine gesunde Null. Beschreibungen der einzelnen Panels beachten.
- Grafana-Benutzer, Rechte, anonyme Freigaben, Zertifikate, Alarmregeln und Export-/Empfangsdienste werden nicht importiert. Der Zugriff wird im eigenen Grafana eingerichtet.

Benannte Anpassungen ohne ursprüngliche private Werte:

- `CONFIG_INTERFACE_1`: interface in der eigenen Umgebung konfigurieren.
- `CONFIG_SERVICE_URL_1`: service url in der eigenen Umgebung konfigurieren.

## Inhalt

- [ASUS RT-BE58U · Systemprotokoll](dashboards/Monitoring/Netzwerkgeräte/mon-asus-ap.json) · `mon-asus-ap`
- [Netzwerk · Verbindungen und Gegenstellen](dashboards/Monitoring/mon-connections.json) · `mon-connections`
- [Dashboard-PC · Ressourcen](dashboards/Monitoring/Dashboard-PC/mon-dashboard-host.json) · `mon-dashboard-host`
- [Dashboard-PC · Netz und Dienste](dashboards/Monitoring/Dashboard-PC/mon-dashboard-system.json) · `mon-dashboard-system`
- [DNS · Hostübersicht](dashboards/Monitoring/DNS/mon-dns.json) · `mon-dns`
- [Logs · Analyse und Sicherheit](dashboards/Monitoring/mon-logs.json) · `mon-logs`
- [Monitoring · Betriebsübersicht](dashboards/Monitoring/mon-operations.json) · `mon-operations`
- [Monitoring · Gesamtüberblick](dashboards/Monitoring/mon-overview.json) · `mon-overview`
- [Proxy · Hostübersicht](dashboards/Monitoring/Proxy/mon-proxy.json) · `mon-proxy`
- [Proxmox · Hostübersicht](dashboards/Monitoring/Proxmox/mon-pve.json) · `mon-pve`
- [TP-Link SX3008F · Anschlüsse und System](dashboards/Monitoring/Netzwerkgeräte/mon-tplink-sx3008f.json) · `mon-tplink-sx3008f` · [Anleitung](docs/tp-link-sx3008f.md)
- [UniFi · Netzwerk und WLAN](dashboards/Monitoring/UniFi/mon-unifi.json) · `mon-unifi`
- [UniFi · Geräte und System](dashboards/Monitoring/UniFi/mon-unifi-devices.json) · `mon-unifi-devices`
- [UniFi · Systemprotokoll und Blockierungen](dashboards/Monitoring/UniFi/mon-unifi-logs.json) · `mon-unifi-logs`
- [UniFi · Ports und Leistung](dashboards/Monitoring/UniFi/mon-unifi-ports.json) · `mon-unifi-ports`
- [UniFi · Internet und WAN](dashboards/Monitoring/UniFi/mon-unifi-wan.json) · `mon-unifi-wan`

`folders.json` beschreibt die Ordner, `EXPORT-MANIFEST.json` ordnet die Dashboards zu. `SHA256SUMS` enthält Prüfsummen aller übrigen Exportdateien. Die Konfigurationen wurden auf gespeicherte private Werte geprüft und anschließend einzeln mit dem Secrets-Scanner kontrolliert. Diese Prüfung ersetzt keine Abnahme in der eigenen Umgebung.

Beiträge bitte nach [CONTRIBUTING.md](CONTRIBUTING.md) einreichen. Eine Lizenzwahl steht noch aus.
