# Transaktionsmanagement

Cross-Cutting Concern der **internen Fassade** (`@Service`). Nie am Controller (External Facade), nie im Connector, nur ausnahmsweise am Repository (Bulk-Updates, native Queries).

## Warum genau dort

Eine Transaktion repräsentiert einen fachlichen Use-Case ("Order aufgeben"), nicht einen einzelnen DB-Call. Die interne Fassade orchestriert mehrere Repositories/Functions einer Komponente — genau dort muss Alles-oder-Nichts gelten.

```java
@Service
public class OrderBusinessManager {
    @Transactional
    public void placeOrder(OrderRequest request) {
        orderRepository.save(order);
        inventoryComponent.reduceStock(order);
        // kein Connector-/externer Call hier drin!
    }
}
```

## Bezug zu ACID

`@Transactional` garantiert die vier ACID-Eigenschaften auf DB-Ebene:

| Eigenschaft | Bedeutung | Spring-Mechanismus |
|---|---|---|
| Atomicity | Alles-oder-Nichts | Rollback bei Exception am Transaktionsende |
| Consistency | DB bleibt in gültigem Zustand | Constraints + Rollback-Regeln |
| Isolation | Sichtbarkeit paralleler Transaktionen | `@Transactional(isolation = ...)` |
| Durability | Commit ist dauerhaft | vom Datenbanktreiber garantiert |

### Isolation Levels (`Isolation` Enum)

| Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| READ_UNCOMMITTED | möglich | möglich | möglich |
| READ_COMMITTED | verhindert | möglich | möglich |
| REPEATABLE_READ | verhindert | verhindert | möglich |
| SERIALIZABLE | verhindert | verhindert | verhindert |

Default: Isolation-Level der Datenbank übernehmen, nur bei konkretem Anomalie-Risiko strenger stellen (Kosten: mehr Locking).

## Spring-Boot-Best-Practices

- **Nur auf öffentlichen Methoden der internen Fassade** annotieren — Proxy-basiertes AOP greift sonst nicht (Self-Invocation-Falle: `this.andereMethode()` innerhalb derselben Klasse umgeht die Transaktion).
- **`readOnly = true`** für reine Lesezugriffe setzen (Optimierungshinweis für Hibernate/JDBC-Treiber).
- **Transaktionsgrenzen kurz halten**: keine externen/Connector-Calls, kein Netzwerk-I/O, keine langsamen Berechnungen innerhalb der Transaktion — das hält Locks unnötig lange offen.
- **Rollback-Regeln explizit setzen**: Standardmäßig lösen nur `RuntimeException`/`Error` ein Rollback aus. Für Checked Exceptions `rollbackFor` angeben.
- **Exceptions nicht schlucken**: Wer eine Exception innerhalb einer `@Transactional`-Methode fängt und nicht weiterwirft, verhindert das Rollback ungewollt.
- **Propagation bewusst wählen**: `REQUIRED` (Default, hängt sich an vorhandene Transaktion an) vs. `REQUIRES_NEW` (eigene, unabhängige Transaktion — z. B. für Audit-Logs, die auch bei Rollback des Hauptvorgangs bestehen bleiben sollen).
- **Microservice-Grenze beachten**: `@Transactional` wirkt nur lokal in einem Service/einer DB. Für fachliche Konsistenz über mehrere Komponenten/Microservices hinweg braucht es Sagas/Kompensation statt einer verteilten Transaktion.

## Checkliste beim Review

```
- [ ] @Transactional sitzt nur an der internen Fassade, nicht am Controller/Connector
- [ ] Kein externer/Connector-Call innerhalb der Transaktion
- [ ] readOnly=true bei reinen Lesemethoden
- [ ] Rollback-Regeln passen zur Fehlerbehandlung (keine verschluckten Exceptions)
- [ ] Propagation (REQUIRED vs. REQUIRES_NEW) bewusst gewählt
- [ ] Bei Aufrufen über Komponentengrenzen: keine verteilte Transaktion angenommen
```
