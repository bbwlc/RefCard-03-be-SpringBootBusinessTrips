# Backend-Deployment auf ECS – TODO

## Entscheidung

Das Backend läuft im **gleichen ECS-Cluster** (`biztrips-cluster`), aber als **eigener Service** mit eigenem Container.

### Warum ein eigener Service

- **Unabhängige Deployments:** Eine Frontend-Änderung startet das Backend nicht neu, und umgekehrt.
- **Eigene Skalierung:** Das Frontend ist nginx mit statischen Dateien und braucht fast nichts. Das Backend braucht meist mehr CPU und RAM.
- **Ein Container pro Aufgabe:** Das ist das übliche ECS-Muster. Zwei Prozesse in einem Container machen Health-Checks und Logs unübersichtlich.

### Warum derselbe Cluster

Ein Cluster ist nur ein logischer Namespace. Bei Fargate kostet er nichts extra und verursacht keinen zusätzlichen Verwaltungsaufwand.

### ALB: einen teilen (empfohlen) oder zwei

| Variante | Vorteile | Nachteile |
| --- | --- | --- |
| **Einen ALB teilen** | Günstiger. Frontend und API liegen auf demselben Origin, daher kein CORS-Problem. | Listener-Regeln müssen gepflegt werden. |
| **Zwei ALBs** | Sauber getrennt. | Doppelte Kosten, CORS im Backend nötig. |

Empfehlung: Der bestehende Listener bekommt die Regel `/api/*` → Backend-Target-Group. Alles andere geht weiter an das Frontend.

## Offene Angaben

- [ ] Port des Backends
- [ ] Pfad für den Health-Check (z. B. `/api/health`)
- [ ] Umgebungsvariablen und Secrets des Backends (DB-Zugang etc.)

## Schritte

Region `us-east-1`, Learner Lab (Rolle `LabRole`, Profil `awsacademy`).

### 1. ECR-Repository

- [ ] `aws ecr create-repository --repository-name biztrips-api`

### 2. Task Definition

- [ ] `task-definition-api.json` anlegen (Family `biztrips-api`, Fargate, `awsvpc`, `executionRoleArn` = `LabRole`)
- [ ] Container-Port und Log-Gruppe `/ecs/biztrips-api` setzen
- [ ] Umgebungsvariablen und Secrets eintragen

### 3. Netzwerk

- [ ] Security Group `biztrips-api-sg`: eingehend nur vom ALB (`biztrips-alb-sg`) auf dem Backend-Port
- [ ] Target Group `biztrips-api-tg`: Typ `ip`, Backend-Port, Health-Check-Pfad
- [ ] Listener-Regel auf dem bestehenden ALB: `/api/*` → `biztrips-api-tg`

### 4. ECS-Service

- [ ] Task Definition registrieren
- [ ] Service `biztrips-api-service` im Cluster `biztrips-cluster` anlegen
  - `desired-count 2`, Fargate, Public IP `ENABLED`
  - Target Group `biztrips-api-tg` anhängen

### 5. Pipeline (GitHub Actions)

- [ ] Eigenes Repo bzw. eigener Workflow fürs Backend
- [ ] Job analog zu `deploy-ecs` im Frontend: ECR-Login, Image bauen und pushen, Task Definition rendern, Service deployen
- [ ] Die Secrets `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` und `AWS_SESSION_TOKEN` auch im Backend-Repo setzen
- [ ] Die Learner-Lab-Secrets laufen beim Ende der Lab-Sitzung ab und müssen dann neu gesetzt werden

### 6. Frontend anpassen

- [ ] `VITE_API_BASE_URL` als Repository-Variable auf `/api` setzen (bei gemeinsamem ALB)
- [ ] Prüfen, wie `src/components/tripsService.js` die URL zusammensetzt
- [ ] Frontend neu bauen, denn Vite schreibt den Wert beim Build fest ins JS-Bundle
- [ ] Bei zwei ALBs stattdessen die API-URL des zweiten ALBs eintragen und CORS im Backend aktivieren

### 7. Test

- [ ] `curl http://<alb-dns>/api/...` liefert eine Antwort des Backends
- [ ] Das Frontend lädt die Daten über `/api`
- [ ] Beide Services zeigen `running == desired` in ECS
