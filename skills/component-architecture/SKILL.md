---
name: komponenten-architektur
description: Definiert und prüft die Backend-Komponenten-Architektur (externe Fassade, interne Fassade, Functions, Repository, Connector, shared Utilities) für Spring-Boot-Backend-Projekte. Use when neuer Code strukturiert wird, Package-/Modul-Grenzen, Transaktionsgrenzen, Namenskonventionen, Testbarkeit oder ArchUnit-/Modulith-Regeln geprüft werden, oder wenn "Komponenten Architektur", "Business Facade", "Transaktionsmanagement", "Paketstruktur", "shared" oder "Connector" erwähnt werden.
---

# Komponenten-Architektur

Muster, um Spring-Boot-Backend-Code fachlich statt technisch zu schneiden. Basiert auf den Videos ["Code effektiv strukturieren Teil 1"](https://www.youtube.com/watch?v=luaDzMKyF0g) und ["Teil 2"](https://www.youtube.com/watch?v=41coSMipVaA).

## Wann anwenden

- Neues Backend-Modul/Microservice wird aufgesetzt oder ein Monolith in fachliche Pakete zerlegt.
- Es soll entschieden werden, in welcher Schicht Logik, Transaktionen, Caching oder Autorisierung liegen.
- Ein Reviewer prüft, ob Packages/Klassen einer konsistenten Namensstruktur folgen.

## Begriffsklärung: Komponente ≠ Maven-Modul

**Eine Komponente ist ein fachliches Package** (`basket`, `catalog`, `payment`, …) — das ist die Grundeinheit dieses Skills. Ein **Maven-Modul** ist eine Build-/Deployment-Einheit. Beide sind normalerweise **nicht dasselbe**: die meisten Komponenten leben als Package in einem gemeinsamen Maven-Modul, ohne physisch getrennt zu sein.

Nur in Ausnahmefällen wird eine Komponente auch zu einem eigenen Maven-Modul erhoben (siehe unten) — das ist eine bewusste, seltene Entscheidung, keine Grundregel. Extremform am anderen Ende: in OSGi- oder JEE/EJB-Projekten kann es sinnvoll sein, praktisch jede Komponente als eigenes (Bundle-/EJB-)Modul mit eigenem `docs`-Verzeichnis zu führen — das ist dort aber eine Ermessensentscheidung, die davon abhängt, wie das jeweilige Framework mit Modulen umgeht (z. B. ob es Hot-Swap einzelner Module zur Laufzeit unterstützt), nicht ein Standardfall für ein normales Spring-Boot-Projekt.

## Fundament: Dependency Rule

Diese Architektur lehnt sich an Uncle Bobs *Dependency Rule* aus der Clean Architecture an: "Source-Code-Abhängigkeiten dürfen nur nach innen zeigen — nichts in einem inneren Kreis darf etwas über einen äußeren Kreis wissen." [web:6]

Die konkreten Schicht-Regeln in diesem Skill sind eine **eigenständige Ausformulierung** für die Komponenten-Architektur — im Kern deckungsgleich mit dem Original-Prinzip, in Schichtzuschnitt, Benennung und Detailregeln aber eigenständig und für dieses Projekt zu verwenden, nicht wortgleich mit Uncle Bobs Originaltext zu behandeln.

**Kein Widerspruch zur Dependency Rule:** Der Aufruf-Fluss External Facade → Internal Facade → Functions → Repository/Connector zeigt nach außen, das ist beabsichtigt und unproblematisch — Repository und Connector sind Adapter zur Außenwelt (DB, externe Systeme). Damit der Source-Code trotzdem nur nach innen zeigt, programmieren Functions/Internal Facade gegen eine **Repository-/Connector-Schnittstelle** (Dependency Inversion), nicht gegen deren konkrete Implementierung. Aufruf-Richtung und Abhängigkeits-Richtung sind zwei unterschiedliche Dinge — nur letztere muss nach innen zeigen.

## Design-Prinzipien

- **Law of Demeter:** Eine Klasse kennt nur ihre direkten Abhängigkeiten. Begründet die Zugriffsregel unten — eine Komponente greift nie durch die interne Fassade einer anderen Komponente hindurch auf deren Functions/Repository/Entities zu.
- **Open/Closed via Strategy:** Gilt für Functions/Components, Connectors **und** — soweit sinnvoll einsetzbar — auch für Services (Internal Facade). Eine neue Variante wird als **neue Klasse hinter derselben Schnittstelle** ergänzt, nie durch `if`/`switch` in bestehendem Code [web:47]: ein neuer Payment-Provider wird als neuer **Connector** hinter derselben Connector-Schnittstelle ergänzt, eine neue Rabattregel als neue **Function** hinter derselben Function-Schnittstelle. Bei der Internal Facade gilt das dort, wo sie selbst eine austauschbare Teil-Strategie enthält (z. B. eine von mehreren möglichen Workflow-Varianten hinter einer gemeinsamen Schnittstelle) — der reine Kontrollfluss/Workflow-Kern bleibt davon ausgenommen, weil er sich zwangsläufig mit dem fachlichen Prozess ändert. Das ist kein Open/Closed-Verstoß, sondern die Aufgabe der Internal Facade; den gesamten Kontrollfluss zu abstrahieren, nur um ihn "closed" zu halten, wäre unnötige Komplexität.
- **Methoden-Größe:** Eine Methode passt auf einen Bildschirm — maximal ca. 60 Zeilen. Über dieser Grenze wird zunächst ausgelagert, nicht toleriert. Diese Grenze und die vier Diagnosefragen unten gelten für **jede Methode in jeder Schicht** — nicht nur für die Internal Facade, sondern genauso für Functions/Components, Repository-Methoden und Connector-Methoden.

  Bevor die Grenze aufgeweicht oder die Methode aufgeteilt wird, folgende Fragen prüfen — jede "Ja"-Antwort hat eine konkrete Ziel-Schicht:

  ```
  - [ ] Gehört wirklich alles hierher, oder gehört ein Teil in eine andere Function/Component?
        (-> als eigene Function/Component extrahieren, damit sie auch von anderen Workflows wiederverwendet werden kann)
  - [ ] Stimmt das Abstraktionslevel, oder sind hier Low-Level-Details in den Service gerutscht?
        (-> Details in eine Function auslagern, Internal Facade bleibt auf Workflow-Ebene)
  - [ ] Ist Orchestrierung aus einem anderen Service hier hineingerutscht?
        (-> zurück in die zuständige Internal Facade der jeweiligen Komponente verschieben, Aufruf über deren Schnittstelle statt Vermischung)
  - [ ] Verbirgt sich in der Orchestrierung ein eigener, noch nicht benannter Service?
        (-> als eigenen Internal-Facade-Baustein extrahieren und benennen, damit er als wiederverwendbarer Baustein von mehreren Workflows genutzt werden kann)
  ```

  Ziel jeder Auslagerung ist immer eine der bestehenden Schichten (Function/Component, Repository, Connector, eigene Internal Facade) — nicht eine Kopie des Codes, sondern ein wiederverwendbarer Baustein, den auch andere Workflows aufrufen können.

  Sind alle vier Fragen mit "Nein" beantwortet, ist eine Überschreitung der 60 Zeilen eine seltene, bewusste Ausnahme — sie muss aber bei **jeder weiteren Änderung** an dieser Methode erneut beobachtet werden, ob sie noch gerechtfertigt ist oder ob sich doch einer der vier Fälle eingeschlichen hat.

## Testbarkeit

**Grundsatz: Ende-zu-Ende-Tests zuerst bauen (Test-Zwiebel).** Statt zuerst viele isolierte Unit-Tests zu schreiben, wird bevorzugt der gesamte Weg durch den Stack getestet: External Facade (Resource) → Internal Facade (Service) → Functions/Components → Repository → In-Memory-DB, wobei externe Systeme durch einen **Stub-Endpunkt** ersetzt werden — ein echter, laufender Test-Server, der wie das reale System auf HTTP-Ebene antwortet (aufgezeichnete Antworten, Introspektion, Fehler-Injection). Der Connector spricht so wirklich über HTTP mit dem Stub, statt nur gegen einen In-Process-Mock der Java-Schnittstelle zu laufen.

**Varianten/Edge-Cases werden direkt an der Function/Component getestet, nicht über den ganzen Stack.** Das ist einer der Hauptgründe, warum Logik überhaupt in eine eigene Function/Component-Klasse extrahiert wird, statt sie in der Internal Facade zu belassen:

1. **Wartbarkeit** — kleinere, überschaubare Klasse (siehe Methoden-Größe oben).
2. **Encapsulation and Abstraction** — Details bleiben in der Component gekapselt, die Internal Facade sieht nur die Schnittstelle.
3. **Testability** — jede Variante/jeder Edge-Case lässt sich isoliert testen, ohne den kompletten Stack (Facade, DB, Connector) mit aufzubauen.

Für eine extrahierte Function/Component wird eine eigene Testklasse `<Name>ComponentTest` angelegt, die ausschließlich diese eine Function testet. Mockito ist dabei willkommen, aber nicht zwingend: Läuft für den Test ohnehin ein Spring-In-Memory-Setup, wird das genutzt; ist Mockito für eine bestimmte Variante oder für Mock-Daten einfacher, wird Mockito verwendet.

Läuft der `<Name>ComponentTest` mit echtem Spring-Context (nicht reinem Mockito-POJO-Test) gegen eine Function mit `Propagation.MANDATORY`, muss der Test selbst eine Transaktion bereitstellen — sonst schlägt der Aufruf sofort mit `IllegalTransactionStateException` fehl. Das ist kein Fehler im Test, sondern zeigt, dass die Function korrekt nur innerhalb einer Transaktion aufrufbar ist.

**Konventionen zu Connector-Stub-Endpunkten, TransactionTemplate, stabilen Testsuiten und gezieltem Datenreset:** siehe [references/testing.md](references/testing.md)

## Kernprinzip: gerichteter, azyklischer Graph

Jede Komponente (`basket`, `catalog`, `payment`, …) ist ein eigenständiges fachliches Package, das theoretisch als eigenes Maven-Modul oder eigener Microservice exportierbar sein muss — ob das tatsächlich passiert, ist eine separate Entscheidung (siehe Begriffsklärung oben).

- Abhängigkeiten zeigen **nur in eine Richtung**, nie zyklisch — sonst würde ein Modul-Split nicht mehr kompilieren.
- Struktur geht **von grob nach fein**: Workflow → einzelne Funktion. Kein Rücksprung von einer Funktion zurück in die Fassade (siehe Dependency Rule oben).
- Stabilität vs. Komplexität: je mehr eingehende Abhängigkeiten eine Komponente hat, desto teurer und riskanter ist eine Änderung — das gehört in Aufwandsschätzungen.
- Jede Entität gehört genau einer Komponente. Eine fremde Entität wird nie direkt verändert, sondern immer über die interne Fassade der besitzenden Komponente angefragt.

### Bei wachsendem Projekt: echtes Maven-Modul in Betracht ziehen

**Konkreter Anhaltspunkt:** Hat das Projekt bereits **≥ 10 Packages/Komponenten**, wird bei jedem neuen Package geprüft, ob eine Extraktion einzelner Komponenten in ein eigenes Maven-Modul (mit eigenem `docs`-Verzeichnis, z. B. für die von Spring Modulith generierten PlantUML-Diagramme) sinnvoller ist, statt einfach ein weiteres Package im bestehenden Modul zu ergänzen. Dabei nicht nur die neue Komponente isoliert betrachten, sondern prüfen, ob verwandte, bereits bestehende Packages mit in das neue Modul gehören.

Drei Vorgehensweisen, mit Kriterium, wann welche greift:

```
1. Direkt neues Modul: wenn die neue Funktion fachlich klar von allen bestehenden Packages
   trennbar ist und die >= 10-Package-Schwelle bereits erreicht ist.
   -> Neues Modul anlegen, für den neuen Code von Anfang an entscheiden, was hineingehört.

2. Erst aufräumen, dann erweitern: wenn das bestehende Modul selbst schon unübersichtlich
   ist, die neue Funktion aber noch nicht zwingend ein eigenes Modul braucht.
   -> Etwas anderes aus dem bestehenden Modul herauslösen (eigenes Increment/eigene Aufgabe),
      danach die neue Funktion im aufgeräumten Zustand beginnen.

3. Beides kombiniert: wenn sowohl das bestehende Modul aufgeräumt werden muss als auch die
   neue Funktion von Anfang an separiert gehört.
   -> Erst 2 (aufräumen/auslagern), danach 1 (neues Modul für die neue Funktion).
```

Der Compiler erzwingt die Abhängigkeitsrichtung bei einem echten Modul physisch, nicht nur durch ArchUnit-/Modulith-Tests — und ein Agent kann pro Modul mit deutlich kleinerem Kontext arbeiten. **Diese Entscheidung trifft der Agent nie eigenständig — immer erst mit dem Team/User absprechen**, da ein Modul-Split Build- und Deployment-Struktur des gesamten Projekts verändert; das obige Kriterium dient dem Agenten dabei nur als begründeter Vorschlag für dieses Gespräch, nicht als Freibrief zur eigenständigen Umsetzung.

## Aufbau einer Komponente

| Schicht | Zweck | Annotation (Spring Boot) |
|---|---|---|
| External Facade | Übersetzt externe Sprache (REST/DTO) in interne Sprache, delegiert an interne Fassade, macht Versionierung/Caching/AuthN/AuthZ | `@RestController` / `@Controller` |
| Internal Facade | Implementiert den Workflow, High-Level-Logik, Cross-Cutting Concerns (**Transaktionsmanagement**, Autorisierung/Policies, Caching) | `@Service` (früher EJB: `*Manager` — beide Namen bezeichnen dieselbe Schicht) |
| Functions/Components | Ein Use-Case-Schritt, low-level, einzeln testbar, wie eine private Methode des Service | `@Component` |
| Repository | Persistenz-Abstraktion, Queries | `@Repository` |
| Connector | Übersetzt BL-Objekte nach außen (Mapping, Retries, Normalisierung, Workarounds) — spiegelverkehrt zur External Facade, die umgekehrt von außen nach innen übersetzt | `@Component` / eigene `@Connector` |
| Entity | Fachliches Datenmodell, liegt im `model`-Package | `@Entity` |

Paketkonvention: `de.<company>.<app>.[api|bl].<xyz>` — pro Komponente typischerweise `api/`, `model/` (Entities), `repository/`, die Facade-Klasse selbst liegt direkt auf oberster Ebene der Komponente (kein zusätzliches `facade`/`boundary`-Package nötig, das schafft keine weitere Übersichtlichkeit).

### Connector-Details: Sichtbarkeit und Translation-Rolle

- **Translation, spiegelverkehrt zur External Facade.** Die External Facade übersetzt von außen nach innen (externe DTOs → interne BL-Objekte). Der Connector übersetzt genau umgekehrt von innen nach außen (interne BL-Objekte → externes API-Format) — dieselbe Aufgabe, nur in die Gegenrichtung.
- **Ein Connector gehört in der Regel genau einer Komponente** und ist modul-privat (package-private) — andere Komponenten sehen ihn nicht.
- **Sehr seltener Sonderfall: mehrere Komponenten brauchen denselben Connector.** Das ist zuerst ein **Diagnose-Anlass, kein Extraktions-Automatismus** — unbedingt prüfen, warum das so ist, bevor irgendetwas verschoben wird. Häufig zeigt sich dabei, dass eigentlich ein "shared Service" in einer der bestehenden Komponenten versteckt war, der nie richtig herausgezogen wurde — der doppelte Connector-Bedarf ist dann nur ein Symptom, nicht die eigentliche Ursache.
- **Erst wenn die Prüfung einen echten, eigenständigen fachlichen Bedarf bestätigt** (kein versteckter shared Service als eigentliche Ursache), wird der Connector in eine **eigene Komponente** gezogen — typischerweise zusammen mit einem eigenen Service, damit der zugehörige gemeinsame Code mitzieht, nicht nur der Connector isoliert. Das ist zunächst eine neue Komponente (fachliches Package); ob daraus zusätzlich ein eigenes Maven-Modul wird, folgt separat den Regeln aus "Bei wachsendem Projekt" oben (≥10-Package-Schwelle, Team/User-Absprache) — die beiden Entscheidungen sind unabhängig voneinander.
- **Vorgehen bei bestätigtem Bedarf:** Wie beim Modul-Split wird das mit dem Team/User besprochen und als eigene, kleine **Vorbereitungs-Iteration** vorgeschlagen — die neue Komponente (Connector + zugehöriger Service) wird zuerst fertig gebaut, das eigentliche neue Feature, das sie wiederverwenden soll, kommt erst danach. Kein Vermischen von Extraktion und neuem Feature in einem Schritt.
- **Timeout- und Transaktionsregeln für Outbound-Connectoren:** siehe [references/transactions.md](references/transactions.md), Abschnitt "Connector-Aufrufe und Transaktionen".

### Reifegrad der Internal Facade: Orchestrierung

Eine Internal Facade beginnt oft mit eigener Logik direkt in der Methode. Wächst sie, sollte diese Logik konsequent in Functions/Components ausgelagert werden — die Internal Facade behält im Idealzustand nur noch **Kontrollfluss** (if/loop, Aufruf-Reihenfolge, Transaktionsgrenze), keine fachliche Logik mehr selbst. Es entsteht dabei keine neue Schicht, sondern die Internal Facade nähert sich diesem Orchestrierungs-Idealzustand an, je größer der Workflow wird.

Auch im Orchestrierungs-Idealzustand bleibt die Internal Facade der **einzige Ort für die Transaktionsgrenze** — Details dazu in [references/transactions.md](references/transactions.md).

## Zugriffsregeln zwischen Komponenten

- Zugriff auf eine andere Komponente erfolgt über deren **interne Fassade** (nicht über deren Functions/Repository/Entities) — siehe Law of Demeter oben.
- Nur wenn eine Komponente potenziell als separater Microservice deployt werden könnte, wird stattdessen über einen Connector auf die externe Fassade der anderen Komponente zugegriffen — das erspart doppeltes Mapping, solange beide im selben Deployment laufen.
- Einzige Ausnahme: Klassen im `shared`-Package (siehe unten) dürfen von jeder Komponente direkt importiert werden.
- **Sonderfall Prototyp-/Frühphase (nur eine Komponente im Projekt):** Direkter Repository-Zugriff aus der External Facade für Lesezugriffe ist hier okay — das Repository fungiert faktisch als Service, solange es noch keine zweite Komponente und damit noch keine echte Komponentengrenze zu schützen gibt. Es lohnt sich in dieser Phase noch nicht, ArchUnit/Spring Modulith einzurichten (siehe [references/archunit-rules.md](references/archunit-rules.md)). **Sobald eine zweite Komponente hinzukommt**, gilt dieser Sonderfall nicht mehr — ab da immer über die interne Fassade, und die Architektur-Prüfung wird eingerichtet.
- Erzwinge das mit Architekturtests (ArchUnit oder Spring Modulith), nicht nur per Konvention — siehe [references/archunit-rules.md](references/archunit-rules.md).

### Wechselseitige Abhängigkeiten auflösen

Brauchen zwei Komponenten sich fachlich gegenseitig (z. B. Order kauft, Payment bezahlt), meldet Spring Modulith das sofort als "cycle detected", sobald beide Seiten sich referenzieren.

- **Eine Integrationsrichtung wählen und beibehalten.** Nur die besitzende Komponente hält die Beziehung zur fremden Entity (z. B. `Order` hat `@ManyToOne(fetch = FetchType.LAZY) private Payment payment;`), die andere Komponente kennt die erste nicht.
- **Erstellen/Ändern läuft immer über die Facade der fremden Komponente** (`paymentFacade.createPayment(...)`), nie über Cascade oder direkten Entity-Zugriff — kein `cascade = CascadeType.ALL` auf fremde Entities, keine öffentlichen Setter für Beziehungen, die die andere Komponente selbst verwaltet.
- **Imports beim Review prüfen:** Ein Import einer **Model/Entity-Klasse** aus einer fremden Komponente ist okay — das ist die gewählte Integrationsrichtung. Jeder andere Import aus einer fremden Komponente (API-Klassen, Repository, Command/Function, interne Fassade-Interna) ist ein Verstoß und muss genau geprüft werden.
- **Umgekehrte Richtung per Event lösen.** Braucht die nachgelagerte Komponente (z. B. Shipping) Infos von der vorgelagerten (Order), soll aber Order Shipping nicht kennen: Order publiziert ein `*Event` (z. B. `OrderEvent`, im `model`-Package, öffentlich sichtbar) über `ApplicationEventPublisher`. Shipping hört per `@EventListener` darauf. Beide Seiten referenzieren sich nicht direkt — kein echter Zyklus, auch wenn es sich wie einer anfühlt (Pseudo-Zyklus).
- **ID statt Entity-Referenz für den Rückkanal.** Muss die vorgelagerte Komponente nach dem Event etwas vom Ergebnis wissen (z. B. Tracking-ID nach dem Versand), läuft das über die Facade der vorgelagerten Komponente mit einer ID (`orderFacade.setTrackingId(orderId, trackingId)`), nicht über eine Entity-Beziehung zurück zur nachgelagerten Komponente.

## Shared / Utility-Klassen

Rein technisches Handwerkszeug ohne Fachlogik (String-, Date-, Mapping-Helfer) gehört nicht in eine fachliche Komponente, sondern in ein eigenes Package.

- Liegen im Package `<basispackage>.shared`.
- Klassenname endet auf `*Utils` (z. B. `DateUtils`, `MappingUtils`).
- Zustandslos, statische Methoden, kein `@Component`/`@Service`, außer es wird als Bean zwingend benötigt.
- Dürfen von **jeder** Komponente importiert werden — das ist die einzige erlaubte Ausnahme zur Regel "nur über interne Fassade zugreifen".
- `shared` selbst darf **keine** Abhängigkeit zu einer fachlichen Komponente haben (sonst entsteht über die Hintertür ein Zyklus).
- Fachliche Regeln, DB-Zugriffe oder Aufrufe anderer Komponenten gehören **nicht** nach `shared` — dafür ist immer eine echte Komponente zu wählen.

## Transaktionsmanagement, Autorisierung, Caching

Diese Cross-Cutting Concerns gehören **ausschließlich in die interne Fassade**, nie in Controller oder Connector.

**Details, ACID-Bezug, Timeout-/Client-Timeout-Regeln und Spring-Boot-Regeln:** siehe [references/transactions.md](references/transactions.md)

## Namenskonvention

Beliebiges Schema ist erlaubt — wichtig ist Konsistenz im ganzen Projekt.

**Vollständige Tabelle mit Kurz-/Lang-/Alternativnamen:** siehe [references/naming-conventions.md](references/naming-conventions.md)

## Architektur-Prüfung mit ArchUnit / Spring Modulith

Alle Regeln oben (Schichtzugriff, keine Zyklen, Namenskonvention, Transaktionsgrenze, `shared`-Ausnahme) lassen sich als automatisierte Tests erzwingen statt nur zu dokumentieren. Spring Modulith prüft Zyklen und unerlaubten Zugriff auf interne Klassen anderer Komponenten automatisch und kann zusätzlich UML-Diagramme der Komponentenstruktur generieren.

**Konzept der Regeln und wann sich die Einrichtung überhaupt lohnt:** siehe [references/archunit-rules.md](references/archunit-rules.md)

## Workflow: neue Komponente anlegen

```
- [ ] Fachpaket anlegen (Name = Fachbegriff aus Gespräch mit Fachabteilung)
- [ ] Prüfen: hat das Projekt bereits >= 10 Packages? Falls ja, Modul-Extraktion statt neues Package prüfen (siehe Kernprinzip-Abschnitt)
- [ ] Entity(s) modellieren, nur mit "is-a"/Kompositions-Beziehung in dieser Komponente belassen
- [ ] Repository für Persistenz-Abstraktion anlegen
- [ ] Functions/Components als einzeln testbare Use-Case-Schritte implementieren, inkl. eigener <Name>ComponentTest pro Function
- [ ] Interne Fassade schreiben: Workflow, Transaktionsgrenze, Policies/Autorisierung
- [ ] Externe Fassade schreiben: DTO-Mapping, Versionierung, AuthN, Caching, Timeout in OpenAPI-Doku angeben
- [ ] Connector nur anlegen, wenn ein externes System angebunden wird; modul-privat, außer eine Prüfung bestätigt echten Bedarf mehrerer Komponenten
- [ ] Stub-Endpunkt für den Connector bereitstellen (aufgezeichnete Antworten, Introspektion, Fehler-Injection)
- [ ] Rein technische Helfer ohne Fachlogik nach shared.*Utils auslagern, nicht in die Komponente
- [ ] Bei wechselseitigem Bedarf zwischen zwei Komponenten: eine Integrationsrichtung wählen, Rückrichtung per Event lösen
- [ ] Abhängigkeitsgraph prüfen: keine Zyklen, kein Rücksprung von Function zu Fassade
- [ ] Ende-zu-Ende-Test über den gesamten Stack ergänzen, nicht nur isolierte Unit-Tests
- [ ] Sobald eine zweite Komponente hinzukommt: ArchUnit-/Modulith-Regeln einrichten und Prototyp-Sonderfall (direkter Repository-Zugriff) auflösen
- [ ] Bei Modul-Split- oder Connector-Extraktions-Bedarf: mit Team/User absprechen und als eigene Vorbereitungs-Iteration vorschlagen, nicht mit dem neuen Feature vermischen
```

## Häufige Fehler

- Transaktionsgrenze am Controller oder im Connector statt an der internen Fassade.
- Ein externer/langsamer Call (Connector) wird **innerhalb** einer offenen Transaktion ausgeführt, ohne dass dies eine bewusste, dokumentierte Rollback-Entscheidung ist.
- Fremde Entität wird direkt aus einer anderen Komponente heraus verändert statt über deren Fassade.
- Zyklische Abhängigkeit zwischen zwei Komponenten (verhindert späteren Modul-Split) — z. B. beide Seiten halten eine Entity-Beziehung zueinander.
- Cascade (`CascadeType.ALL` o. ä.) auf eine Entity-Beziehung zu einer fremden Komponente statt Erstellen/Ändern über deren Facade.
- Zu viele Entities in einer Komponente — meist ein Zeichen, dass die Komponente zu grob geschnitten ist.
- Fachlogik landet in `shared.*Utils`, weil es "praktisch" ist — das macht `shared` zu einer versteckten Komponente ohne Fassade und Tests.
- Internal Facade wächst mit eigener fachlicher Logik statt sie an Functions/Components zu delegieren.
- Eine neue Variante wird per `if`/`switch` in eine bestehende Function/Connector eingebaut statt als neue Klasse hinter der Schnittstelle.
- Eine Methode überschreitet die 60-Zeilen-Grenze, ohne die vier Diagnosefragen (Zugehörigkeit, Abstraktionslevel, fremde Orchestrierung, versteckter Service) durchzugehen — unabhängig davon, in welcher Schicht die Methode liegt.
- Code wird beim Auslagern kopiert statt als wiederverwendbarer Baustein (Function/Component/eigene Internal Facade) extrahiert.
- Ein neues Package wird ins bestehende Modul gehängt, obwohl das Projekt bereits ≥ 10 Packages hat, ohne eine Modul-Extraktion zu erwägen.
- Ein Agent führt einen Maven-Modul-Split eigenständig durch, ohne das vorher mit dem Team/User abzustimmen.
- Komponente und Maven-Modul werden gleichgesetzt, als wäre jede Komponente automatisch ein eigenes physisches Modul.
- Der Prototyp-Sonderfall (Repository-Zugriff direkt aus External Facade) bleibt bestehen, obwohl längst eine zweite Komponente existiert und die Regel nicht mehr gilt.
- Varianten/Edge-Cases werden nur über einen Ende-zu-Ende-Test durchgetestet, statt zusätzlich isoliert an der jeweiligen Function/Component in einer eigenen `<Name>ComponentTest`.
- `@Transactional` auf der Testmethode statt `TransactionTemplate`, wodurch unklar bleibt, ob ein Speichervorgang wirklich funktioniert (Rollback verschleiert das).
- Ein globaler `TestDataHelper.reset()` wird als Standardlösung genutzt statt gezielter Reset im Given-Teil des jeweiligen Tests.
- Ein von mehreren Komponenten benötigter Connector wird sofort in eine eigene Komponente extrahiert, ohne vorher zu prüfen, ob eigentlich ein versteckter shared Service die wahre Ursache ist.
- Client-Timeout ist kürzer oder gleich dem Transaktions-Timeout, statt mit Puffer (z. B. +1s) darüber zu liegen.
- Eine bestätigte Connector-/Service-Extraktion wird direkt mit dem neuen Feature vermischt, statt zuerst als eigene Vorbereitungs-Iteration abzuschließen.
- Ende-zu-Ende-Tests laufen gegen ein echtes Sandbox-System der Gegenseite statt gegen einen Stub-Endpunkt, wodurch die Testsuite instabil, langsam oder kostenpflichtig wird.
