# Architektur-Regeln: ArchUnit-Konzept & Spring Modulith

Dieses Dokument erklärt **was** geprüft werden soll und **warum**. Für Regel 1–3 gibt es mit Spring Modulith eine einfachere Alternative zu selbst geschriebenen ArchUnit-Tests; Regel 4–6 bleiben eigene ArchUnit-Regeln.

## Wann sich die Einrichtung überhaupt lohnt

**Nicht schon beim Projektstart mit nur einer Komponente.** Solange ein Projekt nur eine einzige Komponente hat (Prototyp-/Frühphase), gibt es noch keine Komponentengrenze zu schützen — der in `SKILL.md` beschriebene Sonderfall (direkter Repository-Zugriff aus der External Facade, Repository fungiert faktisch als Service) ist in dieser Phase bewusst erlaubt, und ArchUnit/Modulith aufzusetzen würde nur Regeln gegen eine noch gar nicht existierende Grenze prüfen.

**Sinnvoller Zeitpunkt: sobald eine zweite Komponente zur ersten hinzukommt.** Ab da existiert überhaupt eine Grenze zwischen zwei Komponenten, die verletzt werden kann — das ist der richtige Moment, um ArchUnit-Regeln oder den Spring-Modulith-Test einzurichten und gleichzeitig den Prototyp-Sonderfall aus `SKILL.md` aufzulösen (Repository-Zugriff aus der External Facade zurück in die interne Fassade verschieben, sofern die zweite Komponente betroffen ist).

## Spring Modulith: automatische Prüfung + Visualisierung

Statt Schichtzugriff, Zyklenfreiheit und internen Zugriff selbst mit `layeredArchitecture()`/`slices()` zu formulieren (ArchUnit-Regel 1–2 unten), prüft Spring Modulith das automatisch anhand der Package-Struktur — jedes Package unter der Hauptklasse gilt als eigenes Modul (= Komponente in diesem Skill).

**Beispiel, wie man das in einem Projekt einrichtet** (Klassenname und Package sind projektspezifisch anzupassen):

```java
package org.sterl.componentarchitecture;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.modulith.core.ApplicationModules;
import org.springframework.modulith.docs.Documenter;

@SpringBootTest
class ComponentArchitectureApplicationTests {

    @Test
    void contextLoads() {
        var modules = ApplicationModules.of(ComponentArchitectureApplication.class);

        new Documenter(modules).writeModulesAsPlantUml().writeIndividualModulesAsPlantUml();

        modules.verify(); // selbst wenn das nicht aktiv ist bringt das was!!
    }
}
```

- **`modules.verify()`** wirft einen Testfehler bei Zyklen zwischen Komponenten oder wenn eine Komponente auf eine interne (nicht-`model`/nicht-Facade) Klasse einer anderen Komponente zugreift — deckt die Wechselseitige-Abhängigkeiten-Regel aus `SKILL.md` direkt ab, ohne dass man ArchUnit-Regel 1 und 2 selbst schreiben muss.
- **`Documenter(...).writeModulesAsPlantUml()`** generiert PlantUML-Diagramme der Komponentenstruktur — das ist wertvoll, **auch wenn `verify()` nicht aktiv/bestehend ist oder bewusst nicht scharf gestellt wird**. Das UML-Diagramm ist ein Dokumentations-Artefakt, in das ein Agent direkt hineinschauen kann, um strukturelle Probleme (unerwartete Pfeile zwischen Komponenten, fehlende Trennung) visuell zu erkennen — reines Rendering ohne Validierung liefert also schon Nutzen für Docs und für die KI-gestützte Architektur-Analyse.
- Dieser Test läuft einmal auf Anwendungsebene (nicht pro Komponente) und ersetzt die separaten ArchUnit-Regeln 1–2 unten, sobald er im Projekt vorhanden ist.

## ArchUnit-Regeln (falls kein Spring Modulith verwendet wird, oder für Regel 4–6 ergänzend)

ArchUnit ist eine Java-Bibliothek, die Architektur-Constraints als JUnit-Tests ausführt: sie liest den kompilierten Bytecode ein und prüft Paket-/Klassen-/Methoden-Beziehungen gegen deklarativ formulierte Regeln [web:19][web:32].

### Regel 1 — Schicht-Zugriff (layeredArchitecture)

*Durch Spring Modulith `modules.verify()` abgedeckt, falls verwendet.* Prüft, dass jede Schicht nur von den erlaubten Nachbarn aufgerufen wird:

- External Facade: darf von außen erreicht werden, darf selbst **nicht** von anderen internen Schichten aufgerufen werden.
- Internal Facade: darf **nur** von der External Facade derselben Komponente aufgerufen werden.
- Functions/Repository/Connector: dürfen **nur** von der Internal Facade derselben Komponente aufgerufen werden, nie direkt von außen.

*Gilt erst, sobald diese Regel überhaupt eingerichtet wird (siehe "Wann sich die Einrichtung lohnt" oben) — der Prototyp-Sonderfall mit nur einer Komponente ist davor kein Verstoß, weil die Regel schlicht noch nicht aktiv ist.*

### Regel 2 — Keine Zyklen zwischen Komponenten

*Durch Spring Modulith `modules.verify()` abgedeckt, falls verwendet.* Prüft mit `slices()`, dass `basket`, `catalog`, `payment` … sich nicht gegenseitig abhängen. Ein Zyklus würde einen späteren Split in separate Module/Microservices unmöglich machen.

### Regel 3 — Namenskonvention passt zum Package

Prüft, dass z. B. jede Klasse mit Suffix `*Repository`/`*DAO` auch tatsächlich im Package `..repository..` liegt, und umgekehrt kein `*Controller` im `service`-Package landet [web:26]. Damit wird die Tabelle aus `references/naming-conventions.md` nicht nur Dokumentation, sondern Build-Regel.

### Regel 4 — `@Transactional` nur an der Internal Facade

Prüft, dass die `@Transactional`-Annotation ausschließlich auf Methoden/Klassen liegt, die zur Internal-Facade-Namenskonvention (`*BM`/`*Service`/`*Manager`) gehören, und dass Functions/Components stattdessen `Propagation.MANDATORY` verwenden (nie eine eigene neue Transaktion öffnen). Verhindert exakt den Fehler aus `references/transactions.md`: Transaktionsgrenze am Controller oder Connector.

**Zusätzlich prüfen:** Jede `@Transactional`-Annotation hat `readOnly` explizit gesetzt (nicht auf den Default verlassen) — siehe `references/transactions.md`, Abschnitt "`readOnly` ist Pflichtangabe". Ergänzend, falls `@TransactionalEventListener` verwendet wird: `fallbackExecution` sollte explizit `true` sein, außer bewusst anders entschieden (siehe `references/transactions.md`, Abschnitt "Events und Transaktionen") — der Default `false` führt sonst dazu, dass der Listener außerhalb einer Transaktion stillschweigend nie läuft.

### Regel 5 — Entities nur über die eigene Komponente

Prüft, dass eine `@Entity`-Klasse nur von Klassen aus demselben Komponenten-Package gelesen/verändert wird — keine fremde Komponente greift direkt auf ein Entity zu, sondern immer über dessen Internal Facade. Ausnahme: eine bewusst gewählte, einseitige Entity-Beziehung zwischen zwei Komponenten (siehe "Wechselseitige Abhängigkeiten auflösen" in `SKILL.md`) — hier prüft die Regel stattdessen, dass **keine zweite** Komponente ebenfalls auf dieselbe fremde Entity referenziert (sonst Zyklusgefahr).

**Zusätzliche Ausnahme:** Ein `*Event`, das eine Komponente publiziert und eine andere per `@EventListener`/`@TransactionalEventListener` konsumiert, ist **keine** Entity-Referenz und fällt nicht unter diese Regel — das ist die vorgesehene Lösung für die Rückrichtung wechselseitiger Abhängigkeiten, keine Umgehung, die geprüft/verhindert werden müsste.

### Regel 6 — `.shared`-Package und `*Utils`

Zwei gegenläufige Prüfungen für die Ausnahme-Regel:

- Jede Klasse mit Suffix `*Utils` muss im Package `..shared..` liegen.
- Das Package `..shared..` darf **keine** Abhängigkeit zu einem fachlichen Komponenten-Package haben.

## Nächster Schritt

Sobald die Konzepte hier bestätigt sind, werden daraus konkrete `@ArchTest`-Klassen für Regel 3–6 (ein Test pro Regel) [web:18][web:26][web:29] — Regel 1–2 sind mit dem Spring-Modulith-Test oben bereits abgedeckt, sobald er ins Projekt aufgenommen wird und mindestens zwei Komponenten existieren.
