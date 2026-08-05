# Agent Build Loop

Drei Phasen:
- **A - Bootstrap** (einmalig, User da, Rückfragen erlaubt)
- **B - Loop** (autonom, keine Rückfragen)
- **C - Abschluss**

Langzeitgedächtnis `peon-plan/agent-loop-memory.md` überlebt jeden Compact. Der Plan
(`peon-plan/overview.md` + `task-NN-*.md`) ist das **Ziel**; das Memory hält nur
Projektspezifika, Overall-Status und die Lessons-Learned-Tabelle.

## Phase A - Bootstrap (nur wenn `agent-loop-memory.md` fehlt)

1. **Plan finden** in dieser Reihenfolge:
   a) `peon-plan/overview.md` bzw. `peon-plan/*.md` enthält einen Plan → **as-is nutzen**.
   b) Plan-Referenz/Pfad im User-Command oder Plan-Text im Chat → lesen und selbst
      `peon-plan/overview.md` (+ `task-NN-*.md`) schreiben.
   c) Nichts gefunden → **User fragen** (Plan oder Pfad zum Plan). **NICHT raten!**

   **Vollständigkeit:** Der Overall-Plan (`overview.md`) muss **alle in den `docs/` beschriebenen
   Features abdecken** – Ziel ist, ohne User alle Features umzusetzen. Fehlt etwas, ergänzen.
   `overview.md` hält das Ziel im Fokus, diese Struktur:

   ```markdown
   # Overall Plan

   ## Goal
   <1–2 Sätze: was am Ende funktionierend + testbar existiert>

   ## Work-Status
   <eine Zeile: erledigt / offen / blockiert>

   ## Tasks
   | NN | Titel | Modul/Service | Status (offen/fertig) | Datum + 1-Zeilen-Ergebnis |
   |----|-------|---------------|-----------------------|---------------------------|
   ```
2. **Projektspezifika ermitteln** - Projekt erkunden (Build-Files, Test-Setup, Struktur),
   Kommandos/Pfade **und Teststrategie** ableiten (z. B. E2E gegen die REST-API vs. Unit für
   reine Algorithmik, Test-Framework, „sauberer Start"). Unsicheres **jetzt beim User erfragen**
   (danach nie mehr).
3. **`peon-plan/agent-loop-memory.md` schreiben** (überschreiben falls vorhanden), Struktur:

   ```markdown
   # Agent Loop Memory

   ## Ziel
   <1-2 Sätze; verweist auf peon-plan/overview.md>

   ## Projektspezifika
   - Build-Kommando: <cmd>
   - Test-Kommando: <cmd>   (»grün« = <Definition: alle Tests grün, Kontext startet, …>)
   - Teststrategie: <z. B. BDD/E2E gegen REST-API, Unit nur für reine Algorithmik>
   - Code-Pfade: <wo Code hin darf>
   - Doku-Pfad: <wo Docs hin dürfen>
   - Harte Regeln: <z. B. nie committen; nichts löschen; Schema-Ort; …>

   ## Overall-Status
   <eine Zeile: was steht, was fehlt>

   ## Agent memory
   <deine AI Notizen / Langzeitgedächtnis>
   
   ## Lessons Learned
   | # | Problem | Lösung / Merke |
   |---|---------|----------------|
   ```
4. **`compactSession`** mit Preserve-Text (Bootstrap fertig → in den Loop):
   »Bootstrap fertig. Lies `peon-plan/agent-loop-memory.md` und `peon-plan/overview.md`
   und arbeite den nächsten offenen Task ab (Command: agent-build-loop).«

## Phase B - Arbeitsschleife (autonom, keine Rückfragen)

User **nicht verfügbar** - nach bestem Wissen entscheiden. Punkte, die eine User-Entscheidung
bräuchten: **überspringen** → `docs/offene-punkte.md`. Ganz ausgelassenes Feature →
`docs/todo-<thema>.md` mit Begründung.

Pro Task:

1. `agent-loop-memory.md` + `overview.md` lesen → nächster Task `offen`.
   **Lessons-Learned-Tabelle zuerst lesen** - bekannte Fallen nicht neu erfinden.
2. Task-Datei `peon-plan/task-NN-*.md` lesen (keine Tasks vorhanden → Plan selbst als Task nehmen).
   **Zu groß? → aufteilen, `overview.md` aktualisieren, kleinsten zuerst.**
   Jede Task ist ein in sich geschlossenes Increment mit lauffähigem, testbarem Ergebnis,
   möglichst auf **ein** Modul/Service beschränkt, und hat: ein GOAL, 1..n Regeln, je Regel
   2..n BDD-Use-Cases (GIVEN / WHEN / THEN) für die Testumsetzung.
3. **Implementieren** - Code nur in den Memory-Code-Pfaden, Docs nur im Doku-Pfad, testdriven wenn möglich.
4. **Review-Gate (Pflicht, Task-Ende):** Task-Text gegen Code prüfen; Build ohne Fehler;
   Tests **alle grün** gemäß »grün«-Definition; sauberer Start.
   Rot ⇒ fixen, nicht weitergehen und **nie Assertions weichklopfen**.
5. Regeln/Doku (falls vorhanden) ❌ → ✅ flippen - **nur bei grün**.
6. `overview.md`: Task-Status auf `fertig`, Datum + 1 Zeile Ergebnis. Die Task auch in der task-NN-*.md als **done** markieren.
7. **Lessons Learned pflegen:** neue Tabellenzeile im Memory (was funktionierte / welche Falle /
   was der nächste Task wissen muss) - gefundene Probleme samt Lösung (Tool-Calls, Shell, …)
   festhalten, das Rad nicht neu erfinden. Größere Anleitungen als eigene `lessons-learned-NN.md`
   (wie ein SKILL) schreiben und im Memory mit einem Satz referenzieren.
   Overall-Status im Memory in einer Zeile aktualisieren.
8. **Dann erst** `compactSession` mit Preserve-Text:
   »Lies `peon-plan/agent-loop-memory.md` und `peon-plan/overview.md` (falls nicht schon gelesen)
   und arbeite den nächsten offenen Task ab (Command: agent-build-loop).«
   Falls das Tooling es zulässt, beide Dateien direkt danach wieder einlesen (ein tool batch aufruf) - sonst reicht der Preserve-Text.

Kein `offen`-Task mehr → **weiter zu Phase C** (kein compactSession mehr).

## Phase C - Abschluss

- **nur jetzt einmal** das tool `planImplemented` aufrufen - markiert den Plan als "done" mit timestamp
- Timestamp per Shell holen, oder aus dem neuen plan namen (`date +%Y-%m-%d-%H-%M-%S`) - **nicht schätzen**.
- `peon-plan/agent-loop-memory.md` umbenennen auf `agent-loop-memory-done-<timestamp>.md` - `overview-done-<timestamp>.md` falls tool `planImplemented`nicht vorhanden.
- Lessons Learned zusammenfassen (inkl. Verweise auf die `lessons-learned-NN.md`).
- Kurzen Report geben: erledigte Tasks, `offene-punkte.md`/`todo-*.md` falls vorhanden, Verifikationsstatus.

## Rules

- **Nach JEDEM Task lauffähig** - Tests grün, kein Zwischenzustand über Task-Grenzen hinweg.
- **Konkrete Kommandos/Pfade/Regeln stehen NUR im Memory** - der Command bleibt generisch.
  Bei Unklarheit: Memory lesen, nicht raten; Annahme in `offene-punkte.md` dokumentieren.
- Use-Cases/BDD werden Tests; Testname = der in der Doku genannte, falls vorhanden.
- Bestandscode wird migriert, nicht gelöscht (sofern Memory/Plan nichts anderes sagt).
- TDD/SOLID anwenden, gemeinsamen Util-Code wiederverwenden statt duplizieren.
- Harte Regeln aus dem Memory haben Vorrang (z. B. »nie committen«).
