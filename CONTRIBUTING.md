# Beiträge

Verbesserungen an Abfragen, Beschriftungen, Dokumentation und Darstellung sind über Pull Requests willkommen.

1. Den betroffenen Anwendungsfall und die erwartete Darstellung beschreiben.
2. Änderungen auf die benötigten Dashboard- und Dokumentationsdateien begrenzen. Bestehende Dashboard-/Panel-/Ordner-UIDs und Navigationslinks erhalten; notwendige Änderungen begründen.
3. JSON syntaktisch prüfen, Datenquellen über die bestehenden Import-Eingaben verwenden und neue Abhängigkeiten dokumentieren. Abfragen mit eigenen Daten prüfen; leere Daten und große übliche Zeiträume berücksichtigen.
4. Vor dem Commit alle betroffenen Dateien auf Zugangsdaten und private Konfiguration prüfen. Keine Logzeilen, Livewerte, Screenshots mit privaten Daten, privaten Adressen, MACs, Seriennummern, Benutzernamen, Hardwarekennungen oder gespeicherten privaten Variablenwerte einreichen. Für Beispiele neutrale Platzhalter verwenden.
5. Die beobachtete Wirkung und durchgeführte Prüfungen im Pull Request festhalten. Änderungen an Datenquellen-Verbindungen, Benutzerrechten oder globalen Abfragelimits gehören nicht in diese Dashboarddateien.

Die Exportdateien enthalten keine Sammler oder Dienstkonfigurationen. Vorschläge für solche Komponenten bitte als gesonderten Vorschlag beschreiben. Eine Lizenzwahl steht noch aus.
