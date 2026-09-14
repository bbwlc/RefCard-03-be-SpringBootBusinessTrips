# EX-04 – Lokaler Datei-Storage für Mitarbeiterfotos (Vorstufe zu S3)

## Lernziele

Nach dieser Übung könnt ihr:

- einen Spring-Boot-Endpoint für Datei-Upload (`multipart/form-data`) und -Download schreiben
- Dateien sicher auf einem Dateisystem ablegen (Grössen-/Typ-Validierung, Schutz vor Path Traversal, kollisionsfreie Dateinamen)
- den Ablagepfad über eine Property nach aussen konfigurierbar machen, statt ihn im Code fest zu verdrahten
- ein Docker-**Volume** für persistenten Storage einrichten und von einem zustandslosen Container-Dateisystem abgrenzen
- den EC2-Deploy aus EX-01 so anpassen, dass hochgeladene Dateien einen Redeploy überleben
- begründen, warum genau dieser Ansatz **nicht** unverändert nach ECS/Fargate (EX-03) migriert werden kann — das ist die Brücke zu RefCard 05 (S3)

## Voraussetzungen

- Abgeschlossene [EX-01](./EX-01-deploy-AWS-EC2.md) — laufende EC2-Instanz, funktionierende Deploy-Pipeline
- Empfohlen, aber nicht zwingend: [EX-02](./EX-02-create-Docker-Image-DockerHub.md) — der Docker-Teil dieser Übung (Schritt 6) baut auf dem dortigen `Dockerfile` auf
- Java 21 und MariaDB lokal lauffähig (siehe Haupt-`README`)

---

## Warum lokal/Volume und nicht gleich S3?

Diese Übung ist **bewusst** eine Zwischenstufe. Ein Storage-Endpoint, der Dateien auf das lokale Dateisystem schreibt, lässt sich mit reinem Spring-Boot-Grundwissen (kein AWS-SDK, kein zusätzlicher Account) umsetzen und zeigt die eigentlich interessanten Konzepte (Multipart-Handling, Validierung, Pfad-Sicherheit) ohne AWS-Rauschen. Erst wenn klar ist, **warum** ein lokales Volume in der Cloud an Grenzen stösst (siehe Abschnitt am Ende), ergibt der Umstieg auf S3 in RefCard 05 wirklich Sinn — sonst wäre S3 nur "ein weiterer Konfigurationswert", ohne dass der Grund dafür verstanden wurde.

---

## Schritt 1: `Employee` um eine Foto-Referenz erweitern

In `Employee` (`src/main/java/ch/bbw/trips/repo/Employee.java`) ein neues Feld ergänzen:

```java
private String profilePictureFilename;
```

Mit den passenden Gettern/Settern (bzw. via Lombok, falls ihr die bestehenden manuellen Getter/Setter im Zuge dessen aufräumt — siehe `docs/reviews/CR-01-initial.md` zu bereits bekannten Lombok-Inkonsistenzen in dieser Klasse).

> Bewusst wird **nur der Dateiname** in der Datenbank gespeichert, nicht die Bilddaten selbst (kein `@Lob byte[]`). Die Datei liegt auf dem Dateisystem, die DB hält nur die Referenz — dasselbe Prinzip, das später mit einem S3-Objektschlüssel identisch wiederkehrt (RefCard 05).

---

## Schritt 2: Storage-Pfad konfigurierbar machen

In `application.properties`:

```properties
app.storage.location=./uploads
spring.servlet.multipart.max-file-size=5MB
spring.servlet.multipart.max-request-size=5MB
```

`app.storage.location` wird bewusst **nicht** hartcodiert, sondern als eigene, frei benennbare Property eingeführt — analog zu `spring.datasource.url`, das schon aus EX-02 bekannt per Umgebungsvariable überschrieben werden kann (`APP_STORAGE_LOCATION` beim `docker run`/systemd-Unit).

`./uploads` lokal zum `.gitignore` hinzufügen, damit testweise hochgeladene Dateien nicht versehentlich committet werden:

```
uploads/
```

---

## Schritt 3: `FileStorageService` schreiben

Neue Klasse `ch.bbw.trips.storage.FileStorageService` (Paket muss unter `ch.bbw.trips.**` liegen, siehe Haupt-`CLAUDE.md`/Package-Scanning-Hinweis):

```java
@Service
public class FileStorageService {

    private final Path root;
    private static final List<String> ALLOWED_TYPES =
        List.of("image/png", "image/jpeg", "image/webp");

    public FileStorageService(@Value("${app.storage.location}") String location) throws IOException {
        this.root = Paths.get(location).toAbsolutePath().normalize();
        Files.createDirectories(root);
    }

    public String store(MultipartFile file) throws IOException {
        if (file.isEmpty()) {
            throw new IllegalArgumentException("Datei ist leer");
        }
        if (!ALLOWED_TYPES.contains(file.getContentType())) {
            throw new IllegalArgumentException("Nicht unterstützter Dateityp: " + file.getContentType());
        }

        String extension = StringUtils.getFilenameExtension(file.getOriginalFilename());
        String filename = UUID.randomUUID() + "." + extension;

        Path target = root.resolve(filename).normalize();
        if (!target.getParent().equals(root)) {
            throw new IllegalArgumentException("Ungültiger Dateipfad");
        }

        Files.copy(file.getInputStream(), target, StandardCopyOption.REPLACE_EXISTING);
        return filename;
    }

    public Resource load(String filename) throws MalformedURLException {
        Path file = root.resolve(filename).normalize();
        if (!file.getParent().equals(root)) {
            throw new IllegalArgumentException("Ungültiger Dateipfad");
        }
        return new UrlResource(file.toUri());
    }
}
```

Drei sicherheitsrelevante Punkte, bewusst nicht optional:

- **`ALLOWED_TYPES`-Whitelist** — ohne sie könnte über diesen Endpoint jede beliebige Datei (z. B. eine `.jsp`/`.php`, falls die Instanz je eine andere Anwendung daneben betreibt) abgelegt werden.
- **UUID statt Original-Dateiname** — verhindert sowohl Namenskollisionen (zwei Mitarbeitende laden `foto.jpg` hoch) als auch, dass der Original-Dateiname selbst zum Angriffsvektor wird (z. B. `../../etc/passwd` als "Dateiname").
- **`normalize()` + Prüfung, dass der aufgelöste Pfad noch unterhalb von `root` liegt** — das eigentliche Path-Traversal-Schutzmuster. Ohne diese Prüfung würde `resolve()` einen manipulierten Dateinamen anstandslos ausserhalb von `root` auflösen.

---

## Schritt 4: REST-Endpoints in `EmployeeController`

```java
@PostMapping("/v1/employees/{id}/photo")
public ResponseEntity<Void> uploadPhoto(@PathVariable Long id, @RequestParam("file") MultipartFile file) throws IOException {
    Employee employee = employeeRepository.findById(id)
        .orElseThrow(() -> new TriptNotFoundException(id));
    String filename = fileStorageService.store(file);
    employee.setProfilePictureFilename(filename);
    employeeRepository.save(employee);
    return ResponseEntity.ok().build();
}

@GetMapping("/v1/employees/{id}/photo")
public ResponseEntity<Resource> downloadPhoto(@PathVariable Long id) throws MalformedURLException {
    Employee employee = employeeRepository.findById(id)
        .orElseThrow(() -> new TriptNotFoundException(id));
    if (employee.getProfilePictureFilename() == null) {
        return ResponseEntity.notFound().build();
    }
    Resource resource = fileStorageService.load(employee.getProfilePictureFilename());
    return ResponseEntity.ok()
        .contentType(MediaType.IMAGE_JPEG)
        .body(resource);
}
```

> `MediaType.IMAGE_JPEG` ist hier vereinfacht fest verdrahtet — wer mehrere Bildformate zulässt (Schritt 3 erlaubt PNG/WebP), sollte den tatsächlich gespeicherten Content-Type mit ablegen (z. B. als zweites Feld auf `Employee`, oder aus der Dateiendung ableiten) statt ihn zu raten.

---

## Schritt 5: Lokal testen

```bash
curl -X POST http://localhost:8080/v1/employees/1/photo \
  -F "file=@/pfad/zu/testbild.jpg"

curl http://localhost:8080/v1/employees/1/photo -o downloaded.jpg
```

Kontrolle, dass die Datei tatsächlich unter `app.storage.location` liegt:

```bash
ls -la uploads/
```

---

## Schritt 6: Docker-Volume für Persistenz

Ohne zusätzliche Vorkehrung landen hochgeladene Dateien **im beschreibbaren Layer des Containers** — sie gehen verloren, sobald der Container entfernt wird (`docker rm`), und werden bei jedem `docker run` eines frischen Containers wieder leer. Das gilt unabhängig davon, ob der Container neu gebaut oder nur neu gestartet wurde.

Im `Dockerfile` aus EX-02 den Storage-Pfad als Volume markieren:

```dockerfile
VOLUME ["/app/uploads"]
```

Beim lokalen Testlauf ein benanntes Volume mounten, das unabhängig vom Container-Lebenszyklus existiert:

```bash
docker volume create biztrips-uploads

docker run --rm -p 8080:8080 \
  --network docker_default \
  -v biztrips-uploads:/app/uploads \
  -e APP_STORAGE_LOCATION=/app/uploads \
  -e SPRING_PROFILES_ACTIVE=mariadb \
  -e SPRING_DATASOURCE_URL="jdbc:mariadb://bbw-mariadb-trips-dev:3306/db_biztrips" \
  "$DOCKERHUB_USER/biztrips-backend:prod01"
```

Container entfernen und mit demselben Volume neu starten — die zuvor hochgeladene Datei muss weiterhin abrufbar sein:

```bash
docker rm -f $(docker ps -q --filter ancestor="$DOCKERHUB_USER/biztrips-backend:prod01")
# erneut mit -v biztrips-uploads:/app/uploads starten, Foto erneut abrufen
```

> **Ein benanntes Volume ist an genau den Docker-Host gebunden, auf dem es angelegt wurde.** Läuft der Container auf einer anderen Maschine (zweite EC2-Instanz, oder — wie in EX-03 — ein Fargate-Task ohne feste Host-Zuordnung), ist das Volume dort nicht vorhanden. Das ist bereits ein erster Vorgeschmack auf die Einschränkung, die weiter unten ausführlicher erklärt wird.

---

## Schritt 7: EC2-Deploy anpassen, damit Uploads einen Redeploy überleben

Der `deploy`-Job aus EX-01 überträgt bei jedem Push ausschliesslich die `.jar`-Datei (`rsync ... app.jar`) — das Upload-Verzeichnis auf der Instanz bleibt davon unberührt, **solange** `app.storage.location` auf einen Pfad ausserhalb des Arbeitsverzeichnisses zeigt, aus dem der systemd-Service die Jar startet.

Auf der EC2-Instanz einmalig ein festes Verzeichnis anlegen:

```bash
sudo mkdir -p /opt/businesstrips/uploads
sudo chown ubuntu:ubuntu /opt/businesstrips/uploads
```

In der systemd-Unit (`/etc/systemd/system/businesstrips.service`, siehe EX-01 Schritt 3.4) eine Umgebungsvariable ergänzen:

```ini
Environment=APP_STORAGE_LOCATION=/opt/businesstrips/uploads
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart businesstrips.service
```

Damit zeigt `app.storage.location` dauerhaft auf ein Verzeichnis, das der Deploy-Job nie anfasst — ein neuer `.jar`-Redeploy überschreibt weiterhin nur `app.jar`, nicht die bereits hochgeladenen Fotos.

---

## Bekannte Stolpersteine

**`413 Payload Too Large` beim Upload**
`spring.servlet.multipart.max-file-size`/`max-request-size` (Schritt 2) sind kleiner als die hochgeladene Datei, oder ein davorliegender Reverse Proxy (falls später einer ergänzt wird) hat ein eigenes, niedrigeres Limit.

**Download liefert `404`, obwohl der Upload mit `200 OK` bestätigt wurde**
Meist läuft die Applikation zwischen Upload und Download mit unterschiedlichem `app.storage.location` (z. B. einmal lokal mit `./uploads`, einmal im Container mit `/app/uploads`, ohne dass `APP_STORAGE_LOCATION` gesetzt wurde) — die Datei liegt dann physisch an einem anderen Ort als dort, wo der Download-Endpoint sucht.

**Nach einem Container-Neustart sind alle zuvor hochgeladenen Fotos weg**
Container wurde ohne `-v biztrips-uploads:/app/uploads` gestartet (Schritt 6) — ohne explizites Volume landen Schreibvorgänge im flüchtigen Container-Layer.

**`IllegalArgumentException: Ungültiger Dateipfad` bei völlig normalen Dateinamen**
Prüft, ob `root` in `FileStorageService` tatsächlich mit `toAbsolutePath().normalize()` aufgelöst wurde (Konstruktor, Schritt 3) — vergleicht man einen bereits normalisierten `target`-Pfad mit einem nicht-normalisierten `root`, schlägt der `equals()`-Vergleich auch bei legitimen Dateinamen fehl.

---

## Warum das (noch) nicht 1:1 nach ECS/Fargate (EX-03) passt

| | Lokal/Volume (diese Übung) | ECS/Fargate (EX-03) |
| --- | --- | --- |
| **Anzahl Tasks** | Ein Server, ein Dateisystem, eine Wahrheit | `desired-count: 2` (EX-03) — mehrere gleichzeitig laufende Tasks, jeder mit eigenem, isoliertem Container-Dateisystem |
| **Sichtbarkeit von Uploads** | Ein hochgeladenes Foto ist sofort von derselben Instanz wieder abrufbar | Task A speichert lokal — Task B (eine andere, gleichberechtigte Kopie hinter demselben Load Balancer) sieht die Datei **nicht**, da kein gemeinsames Dateisystem existiert |
| **Self-Healing** | Kein automatischer Ersatz bei Absturz | Ersetzt ECS einen abgestürzten Task automatisch (siehe EX-03), verliert der neue Task alle zuvor lokal gespeicherten Dateien des alten |
| **Rolling Deployment** | Kein Rolling Deployment, ein Prozess wird neu gestartet | Ein neues Deployment (EX-03 Schritt 8) ersetzt Tasks nacheinander — jeder neue Task startet mit leerem Container-Dateisystem |

Kurz: Sobald mehr als **ein** Task/Server gleichzeitig läuft oder Tasks jederzeit ersetzt werden können, ist "auf dem lokalen Dateisystem ablegen" keine verlässliche Ablagestrategie mehr — unabhängig davon, wie sauber der Code in dieser Übung sonst ist. Genau diese Lücke schliesst **S3** in RefCard 05: ein von allen Tasks gleichermassen erreichbarer, nicht an einen einzelnen Container gebundener Speicherort. Der `FileStorageService` aus Schritt 3 bleibt dabei als **Interface/Konzept** bestehen (`store(MultipartFile)` → Referenz, `load(String)` → `Resource`) — in RefCard 05 bekommt er lediglich eine zweite Implementierung, die statt `Files.copy(...)` das AWS-SDK (`S3Client.putObject(...)`) aufruft.

---

## Reflexionsfragen

1. Warum wird in `Employee` nur der Dateiname gespeichert und nicht das Bild selbst als `byte[]` in der Datenbank? Welche Nachteile hätte die Alternative (grosse `BLOB`-Spalte)?
2. Was genau leistet die Prüfung `target.getParent().equals(root)` in `FileStorageService.store(...)`, die über die reine `ALLOWED_TYPES`-Prüfung hinausgeht — welcher konkrete Angriff wird damit verhindert?
3. Warum überlebt ein benanntes Docker-Volume (Schritt 6) einen `docker rm` des Containers, aber nicht den Wechsel auf eine andere physische/virtuelle Maschine?
4. In Schritt 7 wird `APP_STORAGE_LOCATION` bewusst auf ein Verzeichnis **ausserhalb** des Pfads gesetzt, aus dem `app.jar` gestartet wird. Was würde konkret passieren, läge das Upload-Verzeichnis stattdessen direkt neben `app.jar` im selben, bei jedem Deploy überschriebenen Arbeitsverzeichnis?
5. Die Tabelle oben nennt `desired-count: 2` aus EX-03 als Kernproblem für lokalen Storage. Würde das Problem verschwinden, wenn man `desired-count` in EX-03 auf `1` zurücksetzen würde? Was ginge dabei verloren, das EX-03 explizit als Vorteil von ECS gegenüber EX-01 hervorhebt?
6. Welche der drei Methoden in `FileStorageService` (`Konstruktor`, `store`, `load`) müssten sich ändern, wenn Schritt 3 später (RefCard 05) durch eine S3-basierte Implementierung ersetzt wird — und welche Signatur (Parameter/Rückgabetyp) könnte dabei unverändert bleiben?
