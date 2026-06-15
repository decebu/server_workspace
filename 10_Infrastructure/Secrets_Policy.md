# Secrets- & Schlüssel-Policy (Homelab)

Systematik für den Umgang mit Secrets (API-Tokens, SSH-Keys, Passwörter, Zertifikate) im
Homelab. Verallgemeinert aus dem Fleet-Manager-Umbau (siehe `Fleet_Manager_Setup.md`) als
wiederverwendbares Muster für künftige Services.

## Grundsätze
1. **Möglichst nicht in git** — auch git-crypt schützt nur Remote/Repo, lokal liegt der Wert im Klartext.
2. **Möglichst kein Klartext at rest** — verschlüsselt ablegen, erst zur Laufzeit entschlüsseln.
3. **Blast-Radius begrenzen** — ein geleaktes Secret soll möglichst wenig können.
4. **Ehrlich über Grenzen** — perfekte Geheimhaltung gegen lokalen Root/Docker-Zugriff gibt es nicht; Aufwand am Risiko ausrichten.

## Entscheidungsbaum: wo lebt ein Secret?

| Fall | Ablage | Beispiel |
|---|---|---|
| Service-Secret (Container-Env, Token, Key) | **nicht in git**, age-verschlüsselt at rest, **Runtime-Injektion** | Kestra `SECRET_ANSIBLE_PRIVATE_KEY`, `SECRET_NTFY_TOKEN` |
| Muss versioniert mit ins Repo | **git-crypt** (nur wenn nötig) | verschlüsselte `.env`/Inventory |
| Host-lokales Daemon-Secret, nicht-interaktiv gebraucht | **root-only Datei** (`chmod 600`), nicht in git | `/root/.restic_pw`, `/root/.config/fleet-backup.env` |
| Datei-Mount-Secret, das beim Boot da sein muss | **Host-Datei `chmod 600`**, gitignored (kein ephemeral, sonst Reboot-Problem) | `wg0.conf`, `rsyncd.secrets` |
| Zukunft: zentral, auditierbar | externer Secret-Manager / Vault / SSH-CA | (noch nicht umgesetzt) |

## Muster A — age + Runtime-Injektion (bevorzugt für Service-Secrets)
- **age-Identity** (passphrase-geschützt) = Root-of-Trust: `~/.config/age/keys.txt.age`, Host-only, in keinem git.
- **Secret-Datei** age-verschlüsselt: `*.age` (Werte z.B. base64), nicht in git.
- **Generierung** verschlüsselt an den **Recipient** (Public Key) — braucht die Passphrase nicht.
- **Injektion** zur Laufzeit: `age -d` → Shell-Env → Compose interpoliert (`environment: - X=${X}`), **kein** Klartext-File.
- **Guard:** `${X:?fehlt}` lässt einen Start ohne geladene Secrets bewusst fehlschlagen.
- Reboots überstehen das (Docker behält die Container-Env); nur (Re)Create braucht die Passphrase.

## Muster B — git-crypt (nur wenn etwas zwingend ins Repo muss)
- Transparent ver-/entschlüsselt über `.gitattributes`-Filter.
- **Grenzen:** lokal entsperrt = Klartext lesbar; alte Blobs bleiben in der **Historie** (Entfernen nur via `git filter-repo`).
- Aufpassen: Suffixe, die `.gitattributes` **nicht** erfasst (z.B. `.secrets`), landen sonst im **Klartext** in git.

## Blast-Radius senken
- **SSH-Keys:** `from="<mgmt-IP>"` in `authorized_keys` → Key nur von der erwarteten Quelle nutzbar.
- **sudo:** eingeschränkte Befehle statt `NOPASSWD: ALL`, wo praktikabel.
- **Tokens:** kleinster Scope (z.B. ntfy publish-only auf ein Topic).
- **Langfristig:** SSH-CA mit kurzlebigen Zertifikaten → kein dauerhafter Private Key.

## Guardrails gegen versehentliches Auslesen (auch durch KI-Tools)
- Repo-`CLAUDE.md`: Leseverbot für Secret-Pfade (weiche Regel).
- `.claude/settings.json` `permissions.deny`: `Read(...)` auf Secret-Globs (harte Regel — greift aber nur für das `Read`-Tool, nicht für `Bash`/`docker inspect`).
- **Ehrliche Grenze:** Solange ein Secret als Container-Env läuft, ist es per `docker inspect`/`printenv` für jeden mit Docker-Zugriff lesbar. Das verhindert dieser Aufbau nicht — er reduziert „in git" und „Klartext at rest".

## Rotation (generisches Vorgehen)
1. Neues Geheimnis erzeugen.
2. **Überlappung:** neues neben dem alten gültig machen (z.B. Pubkey verteilen), **bevor** das alte entfernt wird → kein Aussperren.
3. Secret an der Laufzeit-Quelle aktualisieren (age-Env etc.).
4. Verifizieren (Funktion testen, nicht den Wert anzeigen).
5. Altes Geheimnis entwerten/entfernen.
(Konkretes Beispiel: Ansible-Key + ntfy-Token in `Fleet_Manager_Setup.md`, Abschnitt Secret-Rotation.)

## Root-of-Trust sichern
Der eine Schlüssel, der alles entsperrt (age-Identity + Passphrase), ist ein **Single Point of
Failure**: extern sichern (Passwortmanager/Offline-Backup). Geht er verloren, sind alle
`*.age` unbrauchbar. Ebenso: lokale root-only Repo-Passwörter (z.B. `/root/.restic_pw`) gehören
extern gesichert — sie liegen bewusst **nicht** im eigenen Backup.

## Reifegrad / „hinreichend?"
**Abgedeckt:** raus aus git, Klartext-at-rest vermieden (Service-Secrets), Blast-Radius via
`from=`, Rotation mit Überlappung + Aussperr-Schutz, Guardrails, Root-of-Trust-Backup.

**Offene Lücken / nächste Stufen:**
- git-**Historie** enthält ggf. noch alte Blobs → optionaler `git filter-repo`-Purge.
- `docker inspect`-Exposure laufender Env-Secrets (inhärent).
- Kein **zentraler** Secret-Manager/Vault (pro-Host age-Identities, dezentral).
- **SSH-CA** mit kurzlebigen Certs noch nicht umgesetzt (würde dauerhafte Private Keys eliminieren).

Für ein Single-Admin-Homelab ist der aktuelle Stand angemessen; Vault/CA lohnen erst bei mehr Hosts/Personen.

## Checkliste — neues Secret / neuer Service
- [ ] Wo lebt es? (Entscheidungsbaum oben) — Default: age + Runtime-Injektion, **nicht** in git
- [ ] Falls Datei doch ins Repo muss: git-crypt-Filter greift wirklich? (`.gitattributes` prüfen)
- [ ] Kleinster nötiger Scope vergeben
- [ ] Blast-Radius begrenzt (`from=`, eingeschränktes sudo, Topic-Scope)
- [ ] Secret-Pfad in `CLAUDE.md` + `.claude/settings.json` deny aufgenommen
- [ ] Rotationsweg bekannt/dokumentiert
- [ ] Root-of-Trust + lokale root-only Passwörter extern gesichert
- [ ] Im Backup enthalten? (bzw. bewusst ausgenommen, wie Repo-Passwörter)
