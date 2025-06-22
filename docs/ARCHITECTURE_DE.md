# Projektüberblick

Dieses Repository enthält ein Beispielprojekt für **Custom Metadata Services** innerhalb der Salesforce Plattform. Hauptzweck ist es, benutzerdefinierte Metadaten (Custom Metadata Records) aus Apex- und Lightning-Komponenten heraus programmatisch zu erstellen und zu aktualisieren. 

Die Lösung zeigt, wie Metadaten-Deployments über die Apex-Klasse `CustomMetadata` angestoßen werden und wie der Status über ein Plattform-Ereignis (`MetadataDeployment__e`) zurück an die Benutzeroberfläche gemeldet wird.

## Architektur im Überblick

* **Apex-Klassen**
  * `CustomMetadata` bietet eine innere Klasse `Operations`, die das Deployment der Custom-Metadata-Daten vornimmt und optional einen Callback auslöst. Beispielcode in den Zeilen 29 bis 60 der Datei `CustomMetadata.cls` zeigt die typische Verwendung.
  * `MetadataRecordDataController` stellt Methoden bereit, um einen Metadatendatensatz zu laden und über `CustomMetadata.Operations` zu speichern.
  * `SessionController` liefert eine Session-ID an den Client, damit die Lightning-Komponente "streaming" das CometD-Protokoll benutzen kann.
* **Lightning-Komponenten**
  * `metadataRecordData` lädt und speichert einzelne Metadatendatensätze. Die Komponente registriert sich auf das Ereignis `MetadataDeployment__e`, um Rückmeldungen zum Deployment zu erhalten.
  * `streaming` kapselt das Abonnement auf das Plattform-Ereignis via CometD.
* **Plattform-Ereignis**
  * `MetadataDeployment__e` enthält die Felder `DeploymentId__c` und `Result__c`, um den Status eines Deployments zurückzumelden.

## Ablauf bei einer Änderung von Metadaten
1. Die Lightning-Komponente ruft `MetadataRecordDataController.upsertRecord` auf.
2. Diese Methode verwendet `CustomMetadata.Operations.enqueueUpsertRecords`, um den neuen Metadaten-Datensatz zu deployen.
3. Nach Abschluss des Deployments sendet die Klasse `PublishEventCallback` ein Ereignis `MetadataDeployment__e`. Die Komponente `streaming` empfängt dieses Ereignis und gibt das Ergebnis an `metadataRecordData` weiter.

## Wartungshinweise für Dienstleister
* Das Projekt basiert auf Salesforce DX und enthält einen Minimalumfang an Lightning-Komponenten und Apex-Klassen. Neue Custom-Metadata-Typen können nach dem Muster aus `CustomMetadata` eingebunden werden.
* Die Verbindung zu CometD wird in `streamingController.js` aufgebaut und verwendet derzeit die feste API-Version 41.0. Bei Updates der Salesforce-API sollte diese Version geprüft werden.
* Für Testzwecke existiert `CustomMetadataTest.cls`. Der Test deckt nur Basisfunktionen ab und sollte bei Erweiterungen ergänzt werden.

## Bekannte technische Schulden und Best-Practice-Verstöße
* Mehrere Stellen sind mit `TODO` markiert. Beispiele:
  * `CustomMetadata.enqueueUpsertRecords` prüft weder auf doppelte Einträge noch auf fehlende Callbacks.
  * `MetadataRecordDataController.loadRecord` enthält dynamische SOQL-Ausdrücke ohne Schutz vor Injection und besitzt kein konsistentes Fehlerhandling.
  * In `metadataRecordData.cmp` ist der Namespace für das Ereignis hardcodiert, was die Wiederverwendbarkeit einschränkt.
  * `streamingController.js` protokolliert nur Fehler auf der Konsole, anstatt sie der Anwendung zu melden.
* Die Rückgabe der Session-ID an den Client (`SessionController.getSessionId`) birgt Sicherheitsrisiken und sollte nur mit Bedacht eingesetzt werden.

Diese Punkte sollten vor einer produktiven Nutzung oder Weiterentwicklung adressiert werden, um Wartbarkeit und Sicherheit zu erhöhen.

