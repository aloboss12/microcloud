# Antworten – Phase 4: Workflow verstehen

Workflow-Datei: `.github/workflows/microcloud-ci.yml` („DoD 2 · Microcloud-CI“)

---

## Teil A – Die Workflow-Datei lesen

### 1. Events – wann startet der Workflow?
Der Workflow startet bei drei Ereignissen (`on:`):
- `push`: bei jedem Push, auf jedem Branch
- `pull_request`: bei jedem Pull Request mit Ziel `main`
- `workflow_dispatch`: manuell per Knopf „Run workflow“ im Actions-Tab

### 2. Runner – wo läuft der Test?
Auf `ubuntu-latest`. Das ist eine frische virtuelle Maschine, die GitHub für jeden Lauf bereitstellt und danach wieder löscht. Docker und Docker Compose sind schon installiert. In öffentlichen Repos sind diese Runner kostenlos.

### 3. Job – wie heißt er und wie viele Steps hat er?
Es gibt einen Job mit der ID `stack-test` und dem Namen „Compose-Stack testen (Port 8085)“. Er hat **10 Steps**. Der Name darf nicht geändert werden, weil DoD 5 den Test an genau diesem Namen erkennt. Mit `timeout-minutes: 10` wird der Job nach spätestens 10 Minuten abgebrochen.

### 4. Action oder Shell-Befehl?
Nur **Step 1** benutzt eine Action (`uses: actions/checkout@v7`). Sie lädt den Code des Repos auf den Runner. Alle anderen Steps (2–10) führen mit `run:` normale Shell-Befehle aus, z. B. `docker compose`, `curl`, `jq` und `diff`.

### 5. Rechte – was bedeutet `permissions: contents: read`?
Der Workflow darf den Inhalt des Repos nur **lesen** und nichts verändern oder pushen. Das ist das Prinzip der **minimalen Rechte**. Wird ein Schritt oder eine Action kompromittiert, bleibt der mögliche Schaden klein.

### 6. Bedingung – was bewirkt `if: ${{ !github.event.repository.is_template }}`?
Der Job läuft nur, wenn das Repo **keine Vorlage** (Template) ist. In der Vorlage liegt noch kein Stack, der Test würde dort immer fehlschlagen. Deshalb wird er dort übersprungen, sonst wäre die Vorlage dauerhaft rot. In meinem eigenen Repo läuft der Job immer.

---

## Teil B – Fehler und Testlogik

### 7. In welchem Step schlägt der Test fehl?
| Fehler | Step | Begründung |
|---|---|---|
| a) Tabulator in der `compose.yml` | **Step 3** – YAML-Syntax prüfen | `docker compose config` kann die Datei nicht lesen |
| b) Ordner `html/` nicht committet | **Step 7** – Inhalt prüfen | `html/index.html` fehlt im Repo |
| c) `ports: "8085:8080"` | **Step 6** – Erreichbarkeit | Port 8085 ist veröffentlicht, aber der Webserver lauscht im Container auf Port 80, nicht auf 8080. Darum antwortet niemand. |
| d) `compose.yml` liegt in einem Unterordner | **Step 2** – Compose-Datei suchen | Es wird nur im Hauptordner gesucht |

### 8. Warum reicht es nicht, nur den Port zu prüfen?
Ein Server kann auf Port 8085 antworten und trotzdem die **falsche Seite** ausliefern, z. B. die Standardseite von Apache. Ein guter Test prüft das **erwartete Ergebnis**, nicht nur, ob etwas läuft. Deshalb vergleicht Step 7 die Antwort des Servers per `diff` mit meiner `html/index.html` und prüft vorher den HTTP-Status 200.

### 9. Wozu dient Step 10 „Aufräumen“?
`docker compose down -v` stoppt und entfernt Container, Netzwerke und Volumes. Wegen `if: always()` läuft er auch nach einem Fehler. Auf wiederverwendeten Runnern (z. B. selbst gehosteten) würden sonst alte Container übrig bleiben und den Port blockieren, sodass der nächste Lauf fehlschlägt.

---

## Teil C – CI/CD einordnen

### 10. Ist das CI, Continuous Delivery oder Continuous Deployment?
Das ist **Continuous Integration (CI)**. Bei jeder Änderung wird automatisch gebaut und getestet. Es wird aber **kein Artefakt** (z. B. Image in einer Registry) bereitgestellt, also kein Continuous Delivery, und **nichts automatisch ausgerollt**, also kein Continuous Deployment. Den LXC bringe ich selbst mit `git pull` und `docker compose up -d` auf den neuen Stand.

### 11. Welche Vorteile hat der automatische Test?
- Er läuft bei **jeder Änderung** automatisch, man kann ihn nicht vergessen.
- Er läuft **immer gleich** in einer sauberen Umgebung, und „bei mir geht's aber“ fällt weg.
- Fehler fallen **früh** auf, bevor sie in den Betrieb (`main`, LXC) kommen.
- Das Ergebnis ist für alle **sichtbar und dokumentiert** (Actions-Tab, Badges, Checks im Pull Request).

---

## Teil D – Für Experten

### 12. Warum gehört das Badge für DoD 3 zum Branch `dev`?
Ein Badge zeigt immer das Ergebnis des **letzten Laufs auf einem bestimmten Branch**. DoD 3 („Apache-Umstellung auf `dev` getestet“) gehört zu `dev`, deshalb nutzt das Badge `?branch=dev`. Solange es keinen Branch `dev` gibt, gibt es auch keinen Lauf, und das Badge bleibt grau. DoD 4 (Apache auf `main`) und DoD 5 (Pull Request) beziehen sich dagegen auf den Betriebsstand auf `main`.

### 13. Welche Informationen braucht die Prüfung für DoD 5 aus der GitHub-API?
- den zuletzt **gemergten Pull Request** von `dev` nach `main`, mit dem Zeitpunkt `merged_at` und dem letzten Commit `head.sha`
- das Ergebnis des Checks **„Compose-Stack testen (Port 8085)“** für genau diesen Commit, mit `completed_at`

DoD 5 ist nur grün, wenn dieser Check **erfolgreich** war **und vor dem Merge** abgeschlossen wurde (`completed_at` liegt vor `merged_at`). So ist nachgewiesen, dass nur nach grünem Test gemergt wurde.
