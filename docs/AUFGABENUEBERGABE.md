# Aufgabendokumentation - Betrieb und Updates LibreChatNeo

## 0. Dokument-Info
- Erstellt von / Datum: Thilo Thum, 2026-03-23
- Zuletzt aktualisiert von / Datum: Thilo Thum, 2026-03-23
- Naechste Ueberpruefung: 2027-03-23

## 1. Worum geht es?
- Name der Aufgabe: Betrieb und Updates LibreChatNeo (Docker Stack)
- Abteilung / Bereich: Engineering / DevOps
- Kurzbeschreibung: Diese Aufgabe stellt sicher, dass die LibreChatNeo-Instanz laeuft,
  sicher bleibt und regelmaessig mit Upstream-LibreChat synchronisiert wird. Dazu
  gehoeren Code-Updates, Image-Builds, Deployments und Funktionschecks. Ohne diese
  Aufgabe drohen Ausfaelle, veraltete Abhaengigkeiten und Funktionsbrueche (z.B.
  lokale Websuche, MCP-Tools).
- Gehoert diese Aufgabe zu einem groesseren Prozess?
  Ja, Bestandteil des laufenden Betriebs der internen AI-Plattform.
- Relevante Links:
  - `README.md`
  - `docs/NEO_CHANGES.md`
  - `docs/local-web-search.md`
  - `docker-compose.yml`
  - `deploy-compose.yml`
  - `config/deployed-update.js`
  - Upstream: https://github.com/danny-avila/LibreChat
  - Fork: https://github.com/neomint-research/LibreChatNeo

## 2. Wer ist zustaendig?
- Hauptverantwortlich: Meik Schürmann
- Vertretung: Sebastian Krüßmann
- Wer liefert etwas zu, wer bekommt ein Ergebnis?
  - Zulieferung: Produkt / Fachbereich liefert Anforderungen, Security liefert Hinweise.
  - Ergebnis: Betriebsteam und Nutzer erhalten eine laufende, aktuelle Instanz.
- Wen kontaktieren, wenn etwas nicht weitergeht?
  - Meik Schürmann

## 3. Teilaufgaben

### Teilaufgabe 1: Upstream Sync und Merge
- Was ist das und wozu dient es?
  Synchronisiert die Fork mit upstream/main, um Bugfixes und neue Features zu uebernehmen.
- Wer macht es?
  Maintainer / DevOps (noch zu benennen).
- Wie geht man vor?
  1) `git fetch upstream`
  2) `git checkout main`
  3) `git merge upstream/main`
  4) Konflikte loesen und relevante Dateien pruefen (siehe Abschnitt 1 und 4).
- Wann / wie oft?
  Anlassbezogen bei Releases, mindestens monatlich.
- Was ist dabei zu beachten?
  Hotspots: `docker-compose.yml`, `deploy-compose.yml`, `docker/entrypoint.sh`,
  `api/server/services/WebSearch/*`, `packages/api/src/mcp/*`,
  `client/src/components/Chat/*`, `librechat.example.yaml`.
- Stand: Ablauf beschrieben, Zuständigkeit offen.

### Teilaufgabe 2: Image Build und Publish (GHCR)
- Was ist das und wozu dient es?
  Baut und veroeffentlicht API, ws-local und SearxNG Images fuer Deployments.
- Wer macht es?
  Maintainer / DevOps (noch zu benennen).
- Wie geht man vor?
  1) Lokale Builds: `docker compose build api ws-local searxng`
  2) CI: `.github/workflows/neo-publish-ghcr.yml` ausloesen oder laufen lassen.
  3) Pruefen, dass die Tags in GHCR verfuegbar sind.
- Wann / wie oft?
  Nach relevanten Codeaenderungen oder bei Upstream-Merge.
- Was ist dabei zu beachten?
  Image-Namen in `docker-compose.yml` und `deploy-compose.yml` muessen passen.
- Stand: Ablauf beschrieben, Zuständigkeit offen.

### Teilaufgabe 3: Deployment Update
- Was ist das und wozu dient es?
  Zieht neue Images und startet die produktive Instanz neu.
- Wer macht es?
  Ops / DevOps (noch zu benennen).
- Wie geht man vor?
  Option A (manuell):
  1) `docker compose -f deploy-compose.yml pull`
  2) `docker compose -f deploy-compose.yml up -d`
  Option B (Script):
  - `npm run update:deployed` (nutzt `config/deployed-update.js`)
- Wann / wie oft?
  Nach Image-Publish oder sicherheitsrelevanten Updates.
- Was ist dabei zu beachten?
  Das Script nutzt `sudo docker-compose` (klassisches CLI) und zieht nur `api`.
  Bei neuen ws-local / searxng Images ggf. manuell `pull` ausfuehren.
- Stand: Ablauf beschrieben, Zuständigkeit offen.

### Teilaufgabe 4: Smoke Tests und Monitoring
- Was ist das und wozu dient es?
  Prueft, dass Kernfunktionen nach Updates funktionieren (Chat, MCP, Websuche).
- Wer macht es?
  Maintainer
- Wie geht man vor?
  1) `docker compose ps`
  2) `docker compose logs -f api ws-local searxng`
  3) UI-Check: Login/Chat, Modellwahl, MCP-Tools laden, Websuche testen.
- Wann / wie oft?
  Nach jedem Deployment und bei stoerungsbezogenen Einsaetzen.
- Was ist dabei zu beachten?
  Websuche haengt an `ws-local` und `searxng`. Bei Fehlern zuerst diese Logs pruefen.
- Stand: Ablauf beschrieben, Zuständigkeit offen.

## 4. So wird's gemacht
- Schritt-fuer-Schritt-Ablauf:
  1) Docker-Status pruefen und sicherstellen, dass Services laufen.
  2) Upstream Sync durchfuehren und Konflikte loesen (Teilaufgabe 1).
  3) Images bauen und publishen (Teilaufgabe 2).
  4) Deployment aktualisieren (Teilaufgabe 3).
  5) Smoke Tests ausfuehren (Teilaufgabe 4).
- Was tun, wenn etwas nicht wie erwartet laeuft?
  - Container startet nicht: `docker compose logs <service>` pruefen.
  - Websuche liefert nichts: `ws-local` und `searxng` Logs pruefen.
  - MCP-Tools fehlen: `api` Logs pruefen und `librechat.example.yaml` kontrollieren.
  - `docker-compose` fehlt: auf `docker compose` (neues CLI) umstellen.
- Wie lange dauert es in der Regel?
  30-90 Minuten inkl. Pull, Restart und Checks.
- Stand: Ablauf beschrieben.

## 5. Wann und wie oft?
- Rhythmus: Anlassbezogen, mindestens monatlicher Upstream-Check.
- Konkrete Termine oder Vorlaufzeiten:
  - Security Fixes: sofort.
  - Feature Updates: bei Bedarf, spaetestens alle 4-6 Wochen.
- Fristen mit rechtlichem oder vertraglichem Hintergrund:
  Noch nicht definiert.
- Was passiert, wenn eine Frist gerissen wird?
  Risiko von Sicherheitsluecken oder Funktionsverlusten.
- Stand: Rhythmus definiert, Fristen offen.

## 6. Relevante Links und Zugaenge
- Programme / Systeme:
  - Git, GitHub, Docker, Docker Compose, GHCR
  - Node.js + npm (fuer Scripts)
- Wo sind Zugangsdaten hinterlegt?
  - Passwortmanager: Noch zu benennen (z.B. "1Password / Vault <Name>").
  - `.env` auf dem Host fuer Laufzeit-Keys.
- Lizenzen, Hardware, Schluessel, Zugangskarten:
  - Keine spezifischen Hardware-Abhaengigkeiten dokumentiert.
- Externe Dienstleister:
  - GitHub / GHCR (Zugang ueber GitHub Org).
  - Optional: Cloud-Provider (noch zu benennen).
- Wo liegen relevante Dateien, Vorlagen und Formulare?
  - `deploy-compose.yml`, `docker-compose.yml`, `.env`, `librechat.yaml`
  - `docs/NEO_CHANGES.md`, `docs/local-web-search.md`
  - `config/deployed-update.js`
- Bestehende Anleitungen oder Dokumentation:
  - `docs/NEO_CHANGES.md` (Fork-Differenzen und Update-Guide)
  - `docs/local-web-search.md` (Websuche-Stack)
  - `README.md` (Setup)
- Stand: Teilweise dokumentiert, Zugaenge offen.

## 7. Woran erkennt man, dass es gut gemacht wurde?
- Ergebnis, wenn alles stimmt:
  - UI unter `http://localhost` erreichbar (oder produktive Domain).
  - API laeuft auf `:3080`.
  - MCP-Toolliste laedt, und Tools sind nutzbar.
  - Websuche liefert Ergebnisse mit Zitaten.
- Vorgaben, Standards oder Normen:
  - Interne Betriebsstandards (noch zu benennen).
  - Relevante Doku: `docs/NEO_CHANGES.md`, `docs/local-web-search.md`.
- Wer schaut drueber oder nimmt ab?
  - Noch zu benennen (Rolle: Tech Lead / Product).
- Stand: Kriterien definiert, Abnahme offen.

## 8. Rechtliches und Datenschutz
- Gesetzliche oder vertragliche Anforderungen:
  - Noch zu pruefen (z.B. DSGVO/GDPR, AV-Vertraege).
- Personenbezogene Daten?
  - Ja, Chatverlaeufe und Nutzerdaten liegen in MongoDB.
  - Loesch- und Aufbewahrungsregeln sind zu definieren und zu dokumentieren.
- Was muss dokumentiert oder nachgewiesen werden?
  - Backup- und Restore-Prozesse.
  - Zugriffsprotokolle, falls erforderlich.
- Braucht es Freigaben oder Unterschriften?
  - Noch zu klaeren.
- Stand: Noch nicht vollstaendig geprueft.

## 9. Was man wissen sollte, aber nirgends steht
- `config/deployed-update.js` nutzt `sudo docker-compose` und zieht nur `api`.
  Wenn `ws-local` oder `searxng` aktualisiert wurden, manuell `pull` ausfuehren.
- `deploy-compose.yml` bindet `./librechat.yaml` (nicht die Example-Datei) ins API-Image.
- `docker/entrypoint.sh` startet MongoDB intern, wenn `MONGO_URI` fehlt.
- `ws-local` nutzt Playwright; Versionen in `ws-local/Dockerfile` und
  `ws-local/package.json` muessen zusammenpassen.
- STT-Mikrofon-Button ist in der UI deaktiviert (`AudioRecorder.tsx`).
- Stand: Wichtige Punkte erfasst, ggf. ergaenzen.

## 10. Was tun, wenn etwas schieflaeuft?
- Haeufige Ausnahmesituationen:
  - Container starten nicht -> Logs pruefen.
  - Websuche liefert leer -> `searxng` und `ws-local` Logs pruefen.
  - MCP-Tools fehlen -> `api` Logs und `librechat.example.yaml` pruefen.
  - Merge-Konflikte -> Hotspot-Dateien priorisiert pruefen.
- Wen kontaktieren bei welchem Problem?
  - Noch zu benennen (z.B. DevOps fuer Infrastruktur, Maintainer fuer Code).
- Was tun, wenn die zustaendige Person unerwartet ausfaellt?
  - Vertretung uebernimmt und nutzt dieses Dokument + `docs/NEO_CHANGES.md`.
- Stand: Grundsaetze definiert, Kontakte offen.

## 11. Was haengt dran?
- Was muss vorher erledigt oder vorhanden sein?
  - Docker und Compose installiert, GHCR Zugriff, `.env` gepflegt.
- Was folgt danach?
  - Stakeholder informieren, ggf. Release Notes aktualisieren.
- Kritische Abhaengigkeiten?
  - GHCR (Image Pull), MongoDB-Volume, SearxNG Settings, Netzwerkports.

## 12. Offene Punkte
- Hauptverantwortliche und Vertretung benennen.
- Passwortmanager / Secret-Storage und Zugriffsrechte dokumentieren.
- Rechtliches (DSGVO/GDPR) und Aufbewahrung pruefen und dokumentieren.
- Backup/Restore-Prozess definieren (MongoDB, Uploads, Logs).
- SLAs oder Wartungsfenster festlegen.
- Abnahmeverantwortliche Person definieren.
