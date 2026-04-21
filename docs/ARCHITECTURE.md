# ClubCore Architektur (Beschreibung + Begründung)

## 1. Zielbild

ClubCore ist als klassische 3-Schichten-Webanwendung aufgebaut:

1. **Präsentation:** Browser + Bootstrap/Jinja2 Templates
2. **Applikation:** Flask (Routing, Validierung, Authentifizierung, Geschäftslogik)
3. **Persistenz:** MariaDB über SQLAlchemy ORM

Im Deployment kommt davor ein Reverse Proxy (Nginx) mit TLS-Terminierung.

---

## 2. Komponenten und Verantwortlichkeiten

### 2.1 Flask-App (`app`-Container)

**Verantwortung**
- HTTP-Endpunkte (HTML + JSON)
- Login/Sitzungsverwaltung (`Flask-Login`)
- Formularvalidierung (`Flask-WTF`)
- Geschäftslogik für CRUD auf Mitglieder, Rollen und Events

**Warum so?**
- Flask ist leichtgewichtig und gut für administrative Backoffice-Anwendungen.
- Mit Blueprints bleibt die Struktur modular (auth, members, roles, events, api).
- Jinja2 + Bootstrap reduziert Frontend-Komplexität bei Admin-Tools.

### 2.2 Datenbank (`db`-Container)

**Verantwortung**
- Persistente Speicherung der Kerndaten
- Relationale Integrität (FKs, Constraints)

**Warum MariaDB/MySQL?**
- Weit verbreitet, stabil, guter Docker-Support.
- SQLAlchemy-Integration ist ausgereift.

### 2.3 Reverse Proxy (`nginx`-Container)

**Verantwortung**
- TLS-Terminierung (HTTPS)
- HTTP→HTTPS Redirect
- Weiterleitung an Flask (`proxy_pass`)

**Warum Nginx davor?**
- Entkoppelt TLS/HTTP-Funktionalität vom Python-Prozess.
- Bessere Kontrolle über Header, Timeouts und ggf. Caching.

### 2.4 Zertifikatsmanagement (`certbot`)

**Verantwortung**
- Initiales Ausstellen und Erneuern von Let's-Encrypt-Zertifikaten

**Warum getrennt?**
- Klare Trennung von Laufzeitverkehr (Nginx) und Zertifikatsjobs.
- Standardisiertes Vorgehen über Compose + Volumes.

---

## 3. Datenmodell (fachlich)

- `User`: Login-Benutzer, inkl. `is_admin`
- `MemberRole`: Vereinsrollen/Funktionen
- `Member`: Stammdaten der Mitglieder + Status
- `Event`: Veranstaltungstermine
- `MemberEventRegistration` (optional): Zuordnung Mitglied ↔ Event

**Begründung**
- Normalisierte Tabellen reduzieren Redundanz.
- Rollen werden referenziert statt als Freitext geführt.
- Optionales Join-Modell erlaubt spätere Erweiterungen (Teilnahmestatus, Notizen etc.).

---

## 4. Sicherheitsmodell

- Keine öffentliche Registrierung
- Zugriff nur für eingeloggte Nutzer
- Schreibzugriffe (CRUD) nur für Admins
- Passwortspeicherung als Hash (`werkzeug.security`)

**Begründung**
- Für Vereinsverwaltung ist ein geschlossenes Admin-System meist ausreichend.
- RBAC light (`is_admin`) ist für den Start einfach und robust.

---

## 5. Request-Flow

1. Browser ruft URL auf
2. Nginx nimmt Request an (80/443)
3. Bei HTTPS: Nginx terminiert TLS und proxyt intern auf Flask
4. Flask verarbeitet Route, nutzt SQLAlchemy für DB-Zugriff
5. Antwort als HTML (Jinja2) oder JSON (API)

---

## 6. Skalierung und Performance: Verbesserungsmöglichkeiten

Nachfolgend konkrete, priorisierte Maßnahmen für mehr Performance und Stabilität.

## 6.1 Quick Wins (kurzfristig)

1. **Gunicorn Worker-Setup tunen**
   - Startwert: `workers = 2 * CPU + 1`
   - `threads` bei I/O-lastigen Admin-Seiten prüfen
   - `timeout`, `graceful_timeout`, `max_requests` setzen

2. **DB-Indizes ergänzen**
   - Auf häufig gefilterten Feldern: `Member(last_name)`, `Member(status)`, `Event(event_date)`, `Member(email)`
   - Verbessert Listen-/Such-Queries spürbar

3. **N+1-Abfragen vermeiden**
   - Bei Listen mit Rollenbezug `joinedload(Member.role)` nutzen
   - Reduziert Anzahl SQL-Queries pro Seite

4. **Static Assets korrekt cachen**
   - Nginx Cache-Header für `/static` setzen
   - Optional Bootstrap lokal ausliefern (statt CDN), falls ohne Internet betrieben

## 6.2 Mittelfristig

1. **Redis als Cache + Session-Store**
   - Häufige Dashboard-/Listenabfragen kurz cachen
   - Sessions aus dem App-Prozess entkoppeln

2. **Read/Write-Optimierung in SQLAlchemy**
   - Nur benötigte Spalten selektieren (`with_entities`/`load_only`)
   - Pagination für große Listen erzwingen

3. **DB Connection Pooling feinjustieren**
   - `pool_size`, `max_overflow`, `pool_recycle` abhängig von Lastprofil konfigurieren

4. **Background Jobs**
   - Nichtkritische Aufgaben (E-Mails, Exporte, Reminder) via Celery/RQ auslagern

## 6.3 Langfristig

1. **Observability einführen**
   - Metriken: Request-Latenz, DB-Zeiten, Error-Rate
   - Logs zentralisieren (z. B. Loki/ELK)

2. **Horizontal skalieren**
   - Mehrere App-Container hinter Nginx
   - Sticky Sessions vermeiden (serverseitige Sessions / Redis)

3. **Optionales Read-Replica-Konzept**
   - Für stark leselastige Szenarien

---

## 7. Betriebs- und Qualitätsrichtlinien

- Migrationen versioniert mit `Flask-Migrate`
- Konfiguration ausschließlich über Umgebungsvariablen
- Regelmäßige Backups des MariaDB-Volumes
- Dependency- und Base-Image-Updates in festen Intervallen

---

## 8. Warum diese Architektur für den Projektkontext passt

- **Lern- und Praxisnähe:** gängiger Python-Webstack, verständlich für kleine Teams
- **Betriebssicherheit:** Containerisierung + klar getrennte Dienste
- **Erweiterbarkeit:** Blueprints/ORM erlauben spätere Module (Beiträge, Finanzen, Dokumente)
- **Kosten/Nutzen:** wenig Infrastruktur, trotzdem sauberer Produktionspfad mit HTTPS

---

## 9. Empfohlene nächste Schritte (Roadmap)

1. DB-Indizes + Pagination einführen
2. Gunicorn/Nginx-Timeouts und Workerzahl produktiv abstimmen
3. Automatische Let's-Encrypt-Erneuerung per Cron/Systemd-Timer aktivieren
4. Basis-Monitoring (Healthcheck, Uptime, Error-Alarm) ergänzen
