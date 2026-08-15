# Testbarkeit: Konventionen

## Connector-Tests: Stub-Endpunkt statt reiner In-Process-Mock

Für Ende-zu-Ende-Tests wird ein externes System, das ein Connector anspricht, durch einen **Stub-Endpunkt** ersetzt — ein echter, laufender Test-Server, der wie das reale System auf HTTP-Ebene antwortet. Der Connector spricht also wirklich über HTTP mit diesem Stub (kein reiner In-Process-Mock einer Java-Schnittstelle), wodurch auch Serialisierung/Deserialisierung und die tatsächliche Connector-Logik (Retries, Fehler-Mapping, Timeouts) mitgetestet werden — nicht nur die Business-Logik dahinter.

**Namenskonvention:** `*Stub` (z. B. `PaymentProviderStub`) — siehe [naming-conventions.md](naming-conventions.md).

Der Stub kann mehr als nur Antworten zurückgeben:

- **Aufgezeichnete/konfigurierbare Antworten** — im einfachsten Fall liefert er feste, vorher aufgezeichnete Antworten zurück.
- **Introspektion** — der Test kann abfragen, was der Stub empfangen hat und wie oft er aufgerufen wurde (z. B. um zu prüfen, dass ein Retry tatsächlich genau zweimal erfolgt ist).
- **Fehler-Injection** — der Stub lässt sich gezielt so konfigurieren, dass er Fehlerantworten oder Timeouts liefert, um die Fehlerbehandlung des Connectors (Retry-Logik, Fallback, Exception-Mapping) tatsächlich zu testen, nicht nur den Erfolgsfall.

Ein echtes Sandbox-System der Gegenseite (also der tatsächliche externe Anbieter) ist für die reguläre Testsuite **nicht** die bevorzugte Lösung — Rate-Limits, Instabilität oder Kosten pro Call würden die Testsuite unzuverlässig machen (siehe "Stabile Tests" unten). Der Stub-Endpunkt ersetzt das für den Regelfall vollständig.

### Stubs implementieren `Resetable` — Reset ist hier der Standard, nicht die Ausnahme

Stubs implementieren in der Regel dasselbe `Resetable`-Marker-Interface (mit einer `reset()`-Methode) wie die `TestDataHelper` (siehe unten) — hier ist der automatische Reset aber **der Standardfall, nicht die Ausnahme**: Ein Stub setzt sich per `.reset()` **vor jedem Test** (`@BeforeEach`) zurück, damit konfigurierte Antworten, Fehlerzustände und aufgezeichnete Aufrufe (Empfang, Aufrufzähler) nicht zwischen Tests durchsickern. Ohne diesen Reset könnte ein Test fälschlich einen Aufruf aus einem vorherigen Test zählen oder auf eine im vorherigen Test konfigurierte Fehlerantwort treffen.

Der Unterschied zur `TestDataHelper`-Regel unten ist bewusst: Bei fachlichen Testdaten soll der Reset **gezielt** im Given-Teil erfolgen, weil pauschales Zurücksetzen verschleiert, welche Daten ein Test wirklich braucht. Bei einem Stub gibt es diesen fachlichen Bezug nicht — sein gesamter Zustand (Antworten, Zähler, Fehlerkonfiguration) ist reiner Test-Support ohne eigene fachliche Bedeutung, ein vollständiger Reset vor jedem Test ist hier unproblematisch und Standard.

## Transaktion im Spring-Context-Test: `TransactionTemplate` statt `@Transactional`

Läuft ein `<Name>ComponentTest` mit echtem Spring-Context gegen eine Function/Component mit `Propagation.MANDATORY` (siehe [transactions.md](transactions.md)), braucht der Test selbst eine Transaktion. Bevorzugt wird dafür `TransactionTemplate`, nicht die `@Transactional`-Annotation auf der Testmethode:

- `@Transactional` auf einer Testmethode führt standardmäßig zu einem **Rollback statt Commit** am Ende des Tests — das verschleiert, ob tatsächlich gespeichert wurde: der Test läuft "grün", auch wenn das Speichern real fehlschlagen würde, weil ohnehin nie committet wird.
- `TransactionTemplate` gibt explizite Kontrolle: der Test entscheidet selbst, ob und wann committet wird, und kann damit wirklich prüfen, ob die Function unter einer echten Transaktion funktioniert — nicht nur unter einer, die sowieso verworfen wird.

```java
@Autowired
TransactionTemplate trx;

@Test
void reduceStock_shouldPersist() {
    trx.execute(status -> {
        reduceStockActivity.reduceStock(order);
        return null;
    });
    // Assertions außerhalb der Transaktion, gegen tatsächlich committete Daten
}
```

Ein reiner Mockito-Test (POJO direkt instanziiert, kein Spring-Proxy) ruft die Methode ohne AOP-Interception auf — `MANDATORY` greift hier gar nicht, weil kein Proxy dazwischenliegt. Das ist kein Widerspruch, sondern der Grund, warum reine Unit-Tests ohne Spring-Context für Varianten/Edge-Cases oft einfacher sind (siehe `SKILL.md`, Abschnitt "Testbarkeit").

## Stabile Tests: Random-Testdaten, ein einziger Spring-Context

- Testdaten werden **zufällig generiert** (Random-IDs, Random-Werte für Felder ohne fachliche Bedeutung für den Test), nicht hartcodiert — verhindert, dass sich Tests gegenseitig über feste IDs/Werte stören.
- Der Spring-Context wird für die gesamte Testsuite **genau einmal** gestartet. Startet er mehrfach (z. B. weil verschiedene Testklassen unterschiedliche `@ActiveProfiles`-/`@MockBean`-Kombinationen nutzen und Spring deshalb einen neuen Context cached statt den bestehenden wiederzuverwenden), ist das ein **Bug in der Testkonfiguration**, kein Normalzustand — es verlangsamt die Testsuite drastisch und deutet auf inkonsistente Test-Setups zwischen Testklassen hin.

## Reset von Testdaten: gezielt, nicht global

Ein `TestDataHelper` mit gemeinsamem `Resetable`-Marker-Interface und einer `reset()`-Methode, die eine `AbstractSpringTest` über eine Liste aller registrierten Helper aufruft, ist möglich, um fachliche Daten/Zustand zurückzusetzen — **das ist aber die Ausnahme, nicht der Standardfall**, und sollte möglichst vermieden werden.

**Bevorzugt:** Reset gezielt im "Given"-Teil des jeweiligen Tests, nur für die Daten, die dieser Test tatsächlich benötigt — kein globaler Reset aller Testdaten für jeden Test. Ein globaler Reset über alle `TestDataHelper` verschleiert, welche Daten ein Test wirklich braucht, koppelt Tests unnötig aneinander und macht die Suite langsamer als nötig.

*(Der Kontrast zu Stubs oben ist bewusst: Dasselbe `Resetable`-Interface, aber unterschiedliche Standard-Politik — bei Stubs ist automatischer Reset vor jedem Test die Regel, bei fachlichen Testdaten die Ausnahme.)*

## Checkliste beim Review

```
- [ ] Externe Systeme werden im E2E-Test durch einen Stub-Endpunkt (*Stub) ersetzt, nicht durch einen echten Sandbox-Call
- [ ] Der Stub erlaubt Introspektion (Empfang, Anzahl Aufrufe) und Fehler-Injection (Fehlerantworten, Timeouts)
- [ ] Stubs implementieren Resetable und setzen sich per reset() vor jedem Test zurück
- [ ] Spring-Context-Tests gegen MANDATORY-Functions nutzen TransactionTemplate, nicht @Transactional auf der Testmethode
- [ ] Testdaten sind randomisiert, nicht hartcodiert
- [ ] Der Spring-Context startet genau einmal für die gesamte Suite
- [ ] Datenreset für TestDataHelper erfolgt gezielt im Given-Teil des jeweiligen Tests, nicht global über alle Helper
- [ ] Ein globaler TestDataHelper.reset() wird nur als bewusste Ausnahme eingesetzt, nicht als Standardlösung
```
