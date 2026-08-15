# Namenskonvention

Es ist egal, welches Schema gewählt wird — wichtig ist, dass der gesamte Code derselben Struktur folgt. Ein Grund für Suffixe: API-Klasse und Entity heißen fachlich oft gleich (z. B. beide "Order"), sind aber unterschiedlich aufgebaut. Der Suffix vermeidet Namenskollisionen ohne lange Package-Importpfade.

| Schicht | Kurz | Lang | Alternative 1 | Alternative 2 | Spring-Annotation |
|---|---|---|---|---|---|
| API-Klasse | `*-` | `*_V1` / `*_V2` | `*` | `*DTO` | – |
| External Facade | `*BF` | `*BusinessFacade` | `*Resource` | `*Controller` | `@RestController`, `@Controller` |
| Interne Facade | `*BM` | `*BusinessManager` | `*Service` | `*Manager` / `*Management` | `@Service`, `@Manager` |
| API Adapter | `*BC` | `*BusinessConnector` | `*Connector` | `*ESI` | `@Component`, `@Connector` |
| Functions | `*BA` | `*BusinessActivity` | `*Component` | `*Command` / `*Action` | `@Component`, `@Activity` |
| Repository | `*DAO` | – | `*Repository` | – | – |
| Entity | `*BE` | `*BusinessEntity` | `*Entity` | – | `@Entity` |
| Event | `*Event` | – | – | – | (kein Bean; siehe Abschnitt "Events" unten) |
| Shared Utility | – | – | `*Utils` | – | – (kein Bean, außer zwingend nötig) |
| Stub-Endpunkt | – | – | `*Stub` (z. B. `PaymentProviderStub`) | – | siehe [testing.md](testing.md) |
| Ende-zu-Ende-Test | – | – | `*IT` (z. B. `OrderPlacementIT`) | – | – |
| Component-Test | – | – | `<Name>ComponentTest` | – | siehe `SKILL.md`, Abschnitt "Testbarkeit" |

## Manager vs. Service

`*Manager` (klassische EJB-Terminologie) und `*Service` (moderner Spring-Sprachgebrauch) bezeichnen **dieselbe Schicht** — die Interne Facade. Welcher Begriff verwendet wird, ist Projekt-/Team-Konvention, keine architektonische Entscheidung.

## Events

Klassen enden auf `*Event` (z. B. `OrderEvent`) und werden von `ApplicationEventPublisher` genutzt, um die "Wechselseitige Abhängigkeiten auflösen"-Regel aus `SKILL.md` umzusetzen (Rückrichtung zwischen zwei Komponenten ohne direkte Referenz).

- **Standard-Package: `model`** — bei wenigen Events reicht es, sie direkt neben den Entities im `model`-Package der publizierenden Komponente abzulegen.
- **Ausnahme `event`-Package:** Erst wenn eine Komponente viele Events hat, wird ein eigenes `event`-Subpackage sinnvoll — das ist die Ausnahme, kein Standardfall, und lohnt sich erst ab einer gewissen Anzahl, nicht schon beim ersten oder zweiten Event.
- **Gemeinsames Marker-Interface empfehlenswert** (z. B. `DomainEvent`), das alle `*Event`-Klassen implementieren. Damit lassen sich alle Events eines Projekts über eine einzige Typ-Suche auffinden, statt sie nur über die Namenskonvention zu erkennen.

## Empfehlung

- Bei API-Versionierung (`_V1`, `_V2`) braucht die Entity keinen zusätzlichen Suffix mehr — der Namenskonflikt verschwindet dadurch bereits.
- Eigene Annotationen (z. B. eine Meta-Annotation, die `@Transactional` + Timing/Metrics bündelt) statt generischer Spring-Annotationen erleichtern spätere Cross-Cutting-Änderungen an einer zentralen Stelle.
- Functions sollten idealerweise nur eine öffentliche Einstiegsmethode haben (`execute()`/`call()`), private Hilfsmethoden sind erlaubt — das erleichtert Testbarkeit und Wiederverwendung (vgl. Netflix Hystrix Command Pattern).
