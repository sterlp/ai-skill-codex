# Namenskonvention

Der Skill verwendet **einen kanonischen Namenssatz** (Spring-Sprachgebrauch: `*Resource`, `*Service`, `*Component`, `*Connector`, `*Repository`, `*Entity`, …). Das ist bewusst *ein* Vorschlag statt vieler Alternativen: Benutzt ein Projekt andere Namen, genügt **ein** Mapping von diesen kanonischen Namen auf die Projektnamen (Legacy-Spalte unten), statt im ganzen Skill Varianten mitzuführen. Grund für die Suffixe: API-Klasse und Domain-Modell heißen fachlich oft gleich (beide "Person"), sind aber unterschiedlich aufgebaut. Der Suffix sitzt am **Domain-Modell** (`PersonEntity`), damit die API-Klasse `Person` ohne Suffix kollisionsfrei bleibt.

| Schicht / Konzept | Kanonischer Name | Spring-Annotation | Legacy (JEE) |
|---|---|---|---|
| External Facade | `*Resource` | `@RestController` / `@Controller` | `*BF` / `*BusinessFacade` — *Remote Interface* |
| Internal Facade | `*Service` | `@Service` | `*BM` / `*BusinessManager` / `*Manager` — *Local Interface* |
| Function / Component | `*Component` | `@Component` | `*BA` / `*BusinessActivity` / `*Activity` |
| Connector | `*Connector` | `@Component` / `@Connector` | `*BC` / `*BusinessConnector` / `*ESI` |
| Repository | `*Repository` | `@Repository` | `*DAO` |
| Converter | `*Converter` | `@Component` | – |
| Entity (Domain-Modell) | `*Entity` | `@Entity` | `*BE` / `*BusinessEntity` |
| API-Klasse (externes Modell) | kein Suffix; versioniert `…V1` / `…V2` | – | – |
| Event | `*Event` | (kein Bean; siehe Abschnitt "Events" unten) | – |
| Shared Utility | `*Utils` | – (kein Bean, außer zwingend nötig) | – |
| Stub-Endpunkt | `*Stub` (z. B. `PaymentProviderStub`) | siehe [testing.md](testing.md) | – |
| Ende-zu-Ende-Test | `*IT` (z. B. `OrderPlacementIT`) | – | – |
| Component-Test | `<Name>ComponentTest` | siehe `SKILL.md`, Abschnitt "Testbarkeit" | – |

`*Converter` liegt bei der `*Resource` im Package `api/converter/` und wandelt `*Entity` ↔ API-Klasse (siehe `SKILL.md`, Abschnitt "Modell-Grenze").

## Package-Zuordnung: internes vs. externes Modell

Suffix **und** Package unterscheiden die beiden Modellfamilien (siehe `SKILL.md`, Abschnitt "Modell-Grenze: wo internes und externes Modell umgewandelt werden"):

| Familie | Suffix | Package | Lebt in |
|---|---|---|---|
| Entity (internes Modell) | `*Entity` | `model/` | Internal Facade, Functions/Components, Repository |
| API-Klasse (externes Modell) | **kein Suffix** — der Suffix sitzt am Domain-Modell (`*Entity`), die API-Klasse braucht darum keinen. Nur bei Versionierung `V1`/`_V1` an Resource **und** API-Klasse (`PersonResourceV2`/`PersonV2`) | `api.<modul>.model` | ab External Facade nach außen |

- Gewandelt wird ausschließlich an der External Facade — per Converter im Package `api/converter/` oder per Spring-Data-Projektion. Eine Internal Facade, die eine API-Klasse annimmt oder zurückgibt, ist ein Fehler.
- **Ausnahme:** Version-stabile Value-Typen — Enums und Strong-IDs (z. B. `record PersonId(String id)`) — dürfen zwischen beiden Familien geteilt und aus dem `api`-Layer im Repository/Service/Component verwendet werden; ein Mapping lohnt erst mit API-Versionierung. Vollwertige DTOs (`PersonV2`) bleiben an der External Facade.
- Am Connector liegt dieselbe Grenze spiegelverkehrt: Er wandelt das interne `*Entity`-Modell in die externen API-Klassen des Fremdsystems, die er befüttert. Dieses Fremdmodell ist weder Entity noch eine eigene `api.<modul>.model`-Klasse und gehört dem Connector — es fällt nicht unter diese Zuordnung.

## Legacy-Namen (JEE) und Remote/Local-Interface

Die Legacy-Spalte oben führt die klassische JEE-Terminologie. Zwei Punkte lohnt es zu kennen, weil sie etwas erklären, das die Spring-Namen verstecken:

- **External Facade = Remote Interface, Internal Facade = Local Interface.** Diese Paarung begründet die Zwei-Fassaden-Trennung: Die External Facade (`*Resource`) ist die *entfernte* Schnittstelle über die Prozessgrenze, die Internal Facade (`*Service`) die *lokale*, in-process aufgerufene Schnittstelle. Wer nur "Resource/Service" liest, sieht diesen Grund nicht mehr — deshalb hier bewusst dokumentiert.
- `*Manager` (JEE) und `*Service` (Spring) bezeichnen **dieselbe Schicht** — die Internal Facade. Welcher Name im Projekt gilt, ist reine Team-Konvention, keine Architektur-Entscheidung.

## Events

Klassen enden auf `*Event` (z. B. `OrderEvent`) und werden von `ApplicationEventPublisher` genutzt, um die "Wechselseitige Abhängigkeiten auflösen"-Regel aus `SKILL.md` umzusetzen (Rückrichtung zwischen zwei Komponenten ohne direkte Referenz).

- **Standard-Package: `model`** — bei wenigen Events reicht es, sie direkt neben den Entities im `model`-Package der publizierenden Komponente abzulegen.
- **Ausnahme `event`-Package:** Erst wenn eine Komponente viele Events hat, wird ein eigenes `event`-Subpackage sinnvoll — das ist die Ausnahme, kein Standardfall, und lohnt sich erst ab einer gewissen Anzahl, nicht schon beim ersten oder zweiten Event.
- **Gemeinsames Marker-Interface empfehlenswert** (z. B. `DomainEvent`), das alle `*Event`-Klassen implementieren. Damit lassen sich alle Events eines Projekts über eine einzige Typ-Suche auffinden, statt sie nur über die Namenskonvention zu erkennen.

## Empfehlung

- Versioniert wird **nur** die API-Seite (`PersonV1`/`PersonV2` samt `PersonResourceV2`); das Domain-Modell behält immer `*Entity` und bleibt unversioniert — der Namenskonflikt löst sich, weil die API-Klasse ohnehin anders heißt als `PersonEntity`.
- Eigene Annotationen (z. B. eine Meta-Annotation, die `@Transactional` + Timing/Metrics bündelt) statt generischer Spring-Annotationen erleichtern spätere Cross-Cutting-Änderungen an einer zentralen Stelle.
- Functions sollten idealerweise nur eine öffentliche Einstiegsmethode haben (`execute()`/`call()`), private Hilfsmethoden sind erlaubt — das erleichtert Testbarkeit und Wiederverwendung (vgl. Netflix Hystrix Command Pattern).
