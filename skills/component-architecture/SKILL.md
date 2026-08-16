---
name: komponenten-architektur
description: Definiert und prüft die Backend-Komponenten-Architektur (externe Fassade, interne Fassade, Functions, Repository, Connector, shared Utilities) für Spring-Boot-Backend-Projekte. Use when neuer Code strukturiert wird, Package-/Modul-Grenzen, Transaktionsgrenzen, Namenskonventionen, Testbarkeit oder ArchUnit-/Modulith-Regeln geprüft werden, oder wenn "Komponenten Architektur", "Business Facade", "Transaktionsmanagement", "Paketstruktur", "shared" oder "Connector" erwähnt werden.
---

# Komponenten-Architektur

Muster für Spring-Boot-Backends, um Code fachlich statt technisch zu schneiden. Basiert auf ["Teil 1"](https://www.youtube.com/watch?v=luaDzMKyF0g) und ["Teil 2"](https://www.youtube.com/watch?v=41coSMipVaA).

## Wann anwenden

- Neues Backend-Modul/Microservice wird aufgesetzt, oder ein Monolith wird in fachliche Pakete zerlegt.
- Es soll entschieden werden, in welcher Schicht Logik, Transaktionen, Caching oder Autorisierung liegen.
- Ein Reviewer prüft Namenskonsistenz oder Package-Struktur.
- Ein neues Feature package soll eingeführt werden.

## Begriffsklärung: Komponente ≠ Maven-Modul

Eine **Komponente** ist ein fachliches Package (`basket`, `payment`, …) — die Grundeinheit dieses Skills. Ein **Maven-Modul** ist eine Build-Einheit. Meist ist eine Komponente nur ein Package im gemeinsamen Modul, kein eigenes Modul. Nur selten wird eine Komponente zum eigenen Modul erhoben (siehe unten). Extremfall: OSGi/JEE-Projekte mit Hot-Swap-Fähigkeit, wo jede Komponente ein eigenes Bundle/EJB-Modul ist — dort Framework-Ermessen, hier kein Standardfall.

## Fundament: Dependency Rule

Angelehnt an Uncle Bobs *Dependency Rule*: "Source-Code-Abhängigkeiten zeigen nur nach innen." [web:6] Die konkreten Regeln hier sind eine **eigene Ausformulierung** dafür — im Kern deckungsgleich, im Detail eigenständig, nicht wortgleich mit Uncle Bobs Original zu behandeln.

Der Aufruf-Fluss External Facade → Internal Facade → Functions → Repository/Connector zeigt nach außen — das ist okay, weil Functions/Internal Facade gegen eine **Schnittstelle** von Repository/Connector programmieren (Dependency Inversion), nicht gegen die Implementierung. Aufruf-Richtung und Abhängigkeits-Richtung sind verschieden; nur Letztere muss nach innen zeigen.

## Design-Prinzipien

- **Law of Demeter:** Eine Klasse kennt nur direkte Abhängigkeiten — begründet, warum Komponenten sich nur über die interne Fassade der anderen ansprechen.
- **Open/Closed via Strategy:** Für Functions/Components, Connectors und — soweit sinnvoll — Services: neue Variante = neue Klasse hinter derselben Schnittstelle, nie `if`/`switch` in bestehendem Code [web:47]. Neuer Payment-Provider → neuer Connector; neue Rabattregel → neue Function. Die Internal Facade ist ausgenommen, wo sie reiner Workflow ist (der ändert sich zwangsläufig mit dem Prozess) — nur ihre austauschbaren Teil-Strategien folgen der Regel.
- **Methoden-Größe:** Max. ~60 Zeilen (ein Bildschirm), in jeder Schicht. Vor Aufteilen/Aufweichen prüfen:
  ```
  - [ ] Gehört alles hierher? -> als Function/Component extrahieren
  - [ ] Stimmt das Abstraktionslevel? -> Details in eine Function auslagern
  - [ ] Fremde Orchestrierung hier drin? -> zurück zur zuständigen Internal Facade
  - [ ] Versteckter, unbenannter Service? -> als eigenen Baustein extrahieren
  ```
  Nur wenn alle vier "Nein" sind, ist eine Überschreitung eine seltene Ausnahme — bei jeder weiteren Änderung neu prüfen.

## Testbarkeit

**Ende-zu-Ende zuerst (Test-Zwiebel):** Resource → Service → Component → Repository → In-Memory-DB, externe Systeme über einen **Stub-Endpunkt** (echter HTTP-Test-Server, aufgezeichnete Antworten, Introspektion, Fehler-Injection) statt In-Process-Mock oder echtem Sandbox-Call.

**Varianten/Edge-Cases isoliert an der Function/Component**, eigene `<Name>ComponentTest`-Klasse — Gründe: Wartbarkeit, Encapsulation, Testability. Mockito optional, nicht zwingend.

Läuft ein solcher Test mit echtem Spring-Context gegen eine `MANDATORY`-Function, braucht der Test selbst eine Transaktion, sonst `IllegalTransactionStateException` — kein Testfehler, sondern korrektes Verhalten.

**Details (Stub-Konventionen, TransactionTemplate, stabile Suites, Datenreset):** [references/testing.md](references/testing.md)

## Kernprinzip: gerichteter, azyklischer Graph

- Abhängigkeiten zeigen **nur in eine Richtung**, nie zyklisch.
- Struktur **grob → fein**: Workflow → Funktion, nie zurück zur Fassade.
- Mehr eingehende Abhängigkeiten = teurere, riskantere Änderungen.
- Jede Entität gehört genau einer Komponente; fremde Entitäten nur über deren Fassade anfragen.

### Bei wachsendem Projekt: echtes Maven-Modul erwägen

Ab **≥ 10 Packages** im Projekt: bei jedem neuen Package prüfen, ob Extraktion in ein eigenes Modul (mit `docs`-Verzeichnis für Modulith-PlantUML) sinnvoller ist — inklusive verwandter bestehender Packages.

```
1. Direkt neues Modul: neue Funktion fachlich klar trennbar, Schwelle erreicht.
2. Erst aufräumen: bestehendes Modul unübersichtlich, neue Funktion braucht (noch) kein eigenes Modul.
3. Beides: aufräumen, dann neues Modul.
```

Der Compiler erzwingt die Richtung dann physisch, ein Agent arbeitet pro Modul mit kleinerem Kontext. **Nie eigenständig entscheiden — immer erst mit Team/User absprechen**; das Kriterium ist nur ein begründeter Vorschlag für dieses Gespräch.

## Aufbau einer Komponente

| Schicht | Zweck | Annotation (Spring Boot) |
|---|---|---|
| External Facade | Extern→intern übersetzen, delegieren, Versionierung/Caching/AuthN/AuthZ | `@RestController` |
| Internal Facade | Workflow, Transaktionsmanagement, Autorisierung/Policies, Caching | `@Service` (= `*Manager`, EJB-Terminologie) |
| Functions/Components | Ein Use-Case-Schritt, einzeln testbar | `@Component` |
| Repository | Persistenz-Abstraktion | `@Repository` |
| Connector | Intern→extern übersetzen (spiegelverkehrt zur External Facade) | `@Component`/`@Connector` |
| Entity | Fachliches Datenmodell, im `model`-Package | `@Entity` |

Paketkonvention: `de.<company>.<app>.[api|bl].<xyz>`; pro Komponente `api/`, `model/`, `repository/`, Facade direkt auf oberster Ebene (kein `facade`-Package nötig).

### Connector-Details

- **Ein Connector gehört einer Komponente**, modul-privat.
- **Selten: mehrere Komponenten brauchen ihn** → zuerst Ursache prüfen (oft ein versteckter, nie extrahierter shared Service), kein Extraktions-Automatismus.
- Bestätigt sich echter Bedarf: eigene Komponente + Service, mit Team/User besprochen, als **eigene Vorbereitungs-Iteration** vor dem neuen Feature — nicht vermischen.
- **Transaktions-/Timeout-Regeln:** [references/transactions.md](references/transactions.md#connector-aufrufe-und-transaktionen)

### Reifegrad der Internal Facade: Orchestrierung

Wächst die Internal Facade, wird Logik an Functions/Components delegiert, bis nur noch **Kontrollfluss** (if/loop, Transaktionsgrenze) bleibt. Die Transaktionsgrenze bleibt dabei immer bei ihr — Details: [references/transactions.md](references/transactions.md).

## Zugriffsregeln zwischen Komponenten

- Zugriff nur über die **interne Fassade** der anderen Komponente.
- Nur bei potenzieller Microservice-Eigenständigkeit: Connector → deren externe Fassade statt interner Fassade.
- Ausnahme: `shared`-Package (siehe unten), von jeder Komponente importierbar.
- **Prototyp-Sonderfall (nur eine Komponente):** Repository direkt aus External Facade lesen ist okay, ArchUnit/Modulith lohnt sich noch nicht. **Endet, sobald eine zweite Komponente hinzukommt.**
- Erzwingen mit ArchUnit/Modulith, nicht nur Konvention: [references/archunit-rules.md](references/archunit-rules.md)

### Wechselseitige Abhängigkeiten auflösen

- **Eine Integrationsrichtung wählen:** nur die besitzende Komponente hält die Entity-Beziehung (`@ManyToOne(fetch=LAZY)`), nie beidseitig.
- **Ändern nur über die fremde Facade**, nie Cascade/direkter Zugriff.
- **Imports-Kriterium:** Model/Entity-Import einer fremden Komponente ist okay; alles andere (API, Repository, Function, Fassade-Interna) ist ein Verstoß.
- **Rückrichtung per `*Event`** (`ApplicationEventPublisher`/`@EventListener`, im `model`-Package) — kein echter Zyklus, da keine direkte Referenz.
- **Rückkanal per ID**, nicht per Entity-Referenz (z. B. `orderFacade.setTrackingId(...)`).

## Shared / Utility-Klassen

Technisches Handwerkszeug ohne Fachlogik im Package `<basispackage>.shared`, Klassen enden auf `*Utils`, zustandslos/statisch, kein `@Component` außer nötig. Von jeder Komponente importierbar (einzige Ausnahme zur Fassaden-Regel), darf selbst aber keine Komponente kennen. Fachliche Regeln oder DB-Zugriffe gehören nicht hierher.

## Transaktionsmanagement, Autorisierung, Caching

Ausschließlich in der internen Fassade, nie in Controller oder Connector.

**ACID, MANDATORY-Propagation, readOnly-Pflicht, Timeouts, `@TransactionalEventListener`:** [references/transactions.md](references/transactions.md)

## Namenskonvention

Beliebiges Schema erlaubt, Konsistenz ist entscheidend. **Vollständige Tabelle:** [references/naming-conventions.md](references/naming-conventions.md)

## Architektur-Prüfung mit ArchUnit / Spring Modulith

Erzwingt Schichtzugriff, Zyklenfreiheit, Namenskonvention, Transaktionsgrenze, `shared`-Ausnahme als Test statt Dokumentation. Spring Modulith prüft Zyklen/internen Zugriff automatisch, generiert UML. **Regeln und Zeitpunkt der Einrichtung:** [references/archunit-rules.md](references/archunit-rules.md)

## Workflow: neue Komponente anlegen

```
- [ ] Fachpaket anlegen (Name = Fachbegriff)
- [ ] Bei >= 10 Packages im Projekt: Modul-Extraktion statt neues Package prüfen
- [ ] Entity(s) modellieren, nur mit Bezug in dieser Komponente
- [ ] Repository, Functions/Components (inkl. <Name>ComponentTest), Interne Fassade, Externe Fassade
- [ ] Connector nur wenn nötig; modul-privat, außer bestätigter Mehrfachbedarf
- [ ] Stub-Endpunkt für den Connector bereitstellen
- [ ] Technische Helfer nach shared.*Utils, nicht in die Komponente
- [ ] Wechselseitiger Bedarf: eine Richtung wählen, Rückrichtung per Event
- [ ] Abhängigkeitsgraph prüfen: keine Zyklen, kein Rücksprung
- [ ] Ende-zu-Ende-Test über den Stack ergänzen
- [ ] Ab 2. Komponente: ArchUnit/Modulith einrichten, Prototyp-Sonderfall auflösen
- [ ] Modul-/Connector-Split: mit Team/User absprechen, als eigene Iteration
```

## Häufige Fehler

- Transaktionsgrenze am Controller/Connector statt an der internen Fassade.
- Connector-Call innerhalb einer Transaktion ohne bewussten Rollback-Grund.
- Fremde Entität direkt verändert oder per Cascade statt über deren Facade.
- Zyklische Abhängigkeit zwischen Komponenten (beidseitige Entity-Beziehung).
- Zu viele Entities in einer Komponente (zu grob geschnitten).
- Fachlogik in `shared.*Utils`, weil "praktisch".
- Internal Facade wächst mit eigener Logik statt zu delegieren.
- Neue Variante per `if`/`switch` statt neuer Klasse hinter der Schnittstelle.
- 60-Zeilen-Grenze überschritten ohne die vier Diagnosefragen zu prüfen.
- Code beim Auslagern kopiert statt als Baustein extrahiert.
- Neues Package trotz ≥10-Package-Schwelle ohne Modul-Extraktion zu erwägen.
- Modul-Split oder Connector-Extraktion eigenständig durchgeführt, ohne Team/User-Absprache oder eigene Vorbereitungs-Iteration — oder Komponente/Maven-Modul gleichgesetzt.
- Prototyp-Sonderfall bleibt bestehen, obwohl längst eine zweite Komponente existiert.
- Connector-Extraktion ohne vorherige Prüfung, ob ein versteckter shared Service die wahre Ursache ist.
- Varianten nur per Ende-zu-Ende-Test geprüft, keine isolierte `<Name>ComponentTest`.
- `@Transactional` statt `TransactionTemplate` im Spring-Context-Test (verschleiert Speicherfehler durch Rollback).
- Globaler `TestDataHelper.reset()` als Standard statt gezielter Reset im Given-Teil.
- E2E-Test gegen echtes Sandbox-System statt Stub-Endpunkt (instabil/teuer).
- Client-Timeout kürzer/gleich dem Transaktions-Timeout statt mit Puffer (z. B. +1s) darüber.
