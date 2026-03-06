# Codeberg Pages — Versuchsprotokoll (2026-03-06)

## Ziel

Hugo-Seite (Hextra) unter Custom Domain `fke.systems` via Codeberg Pages ausliefern.

## Ausgangslage

- Codeberg-Org: `floriankesslergmbh`
- Repo: `floriankesslergmbh/pages` (Default-Reponame für Org-Sites)
- Domain: `fke.systems`
- Stack: Hugo + Hextra v0.12.0, Go Modules

## DNS-Konfiguration (bei DNS-Provider gesetzt, verifiziert)

```
A     fke.systems   217.197.84.141
AAAA  fke.systems   2a0a:4580:103f:c0de::2
TXT   fke.systems   floriankesslergmbh.codeberg.page
```

Codeberg Pages TXT-Format: `[[branch.]repo.]user.codeberg.page`. Bei Repo-Name `pages` (Default) wird der Reponame weggelassen → `floriankesslergmbh.codeberg.page`.

## Versuch 1: Forgejo Actions (CI-basiert)

### Ansatz

Workflow `.forgejo/workflows/deploy.yaml` baut Hugo im Container `hugomods/hugo:latest` und pusht `public/` auf den `pages`-Branch. `.domains`-Datei mit `fke.systems` wird im Deploy-Step erzeugt.

### Probleme

1. **Actions musste erst aktiviert werden**: Repo Settings → Units → "Enable Actions" Checkbox.
2. **Docker-Image `hugomods/hugo:exts-0.157.0` existierte nicht** — `manifest unknown`. Fix: `hugomods/hugo:latest`.
3. **Kritisch: Der CI-Push auf den `pages`-Branch hat den neuen git-pages Server aktiviert.** Laut Doku: „Once you use one of the new deployment methods, your site will be served by the new server." Der neue git-pages Server unterstützt **keine Custom Domains**. Custom Domains funktionieren nur mit dem Legacy-v2-Server (`pages-server`).

### Ergebnis

TLS-Zertifikat wurde ausgestellt, aber `fke.systems` lieferte nur eine Codeberg-404-Seite (vom neuen git-pages Server, nicht von Legacy v2).

## Versuch 2: Lokaler Deploy (Legacy v2)

### Ansatz

Forgejo Actions Workflow entfernt. `deploy.sh` baut lokal und pusht `public/` direkt auf den `pages`-Branch (ohne CI):

```bash
#!/usr/bin/env bash
set -euo pipefail
cd "$(git rev-parse --show-toplevel)"
hugo --gc --minify
cd public
echo "fke.systems" > .domains
git init
git checkout -b pages
git add .
git commit -m "Deploy $(git -C .. rev-parse --short HEAD)"
git push --force git@codeberg.org:floriankesslergmbh/pages.git pages
```

### Probleme

1. Der git-pages Server hatte sich durch Versuch 1 bereits aktiviert und blockierte den Legacy-v2-Server.
2. `pages`-Branch gelöscht und neu gepusht → git-pages blieb aktiv (Header: `server: git-pages (git-pages)`).
3. Versuch, git-pages per DELETE-Request zu deregistrieren:
   - `curl -X DELETE https://floriankesslergmbh.codeberg.page/` → 401, braucht Auth
   - `Authorization: token <TOKEN>` → `unknown Authorization scheme`
   - `Authorization: Pages <TOKEN>` → braucht DNS-Challenge (`_git-pages-challenge.floriankesslergmbh.codeberg.page TXT`), nicht setzbar da `codeberg.page` nicht unsere Domain.
4. **Org und Repo komplett gelöscht und neu angelegt**, Actions deaktiviert, erneut deployed.
5. Ergebnis: `fke.systems` liefert jetzt HTTP 404 von `pages-server` (Legacy v2), nicht mehr von git-pages. Aber v2 holt den Content nicht ab — vermutlich fehlt ein initialer Trigger.

### Ergebnis

Legacy v2 antwortet, sieht aber den Content nicht. Kein Webhook konfiguriert (unklar ob v2 einen braucht oder automatisch fetcht). Nicht weiter verfolgt.

## Erkenntnisse

1. **Codeberg Pages hat zwei Systeme parallel**: Legacy v2 (`pages-server`) und neu (`git-pages`). Custom Domains gehen nur mit v2.
2. **Sobald git-pages einmal aktiviert wurde** (durch Webhook oder Forgejo Actions Push auf `pages`-Branch), übernimmt der neue Server. Es gibt keinen einfachen Weg zurück.
3. **git-pages Deregistrierung** erfordert DNS-Challenge auf `_git-pages-challenge.<subdomain>.codeberg.page` — unmöglich für Endnutzer.
4. **Forgejo Actions Limits** (falls später relevant): tiny=1CPU/2GB/2min, small=2CPU/4GB/5min, medium=4CPU/8GB/10min. Hugo-Build braucht <1s, `codeberg-tiny-lazy` reicht.
5. **Hugo-Image**: `hugomods/hugo:latest` funktioniert. Spezifische Versionstags wie `exts-0.157.0` existieren möglicherweise nicht.

## Offene Punkte

- **Hosting-Entscheidung steht aus.** Codeberg Pages mit Custom Domain ist aktuell nicht praktikabel. Alternativen: Linux-VM mit nginx + rsync, Cloudflare Pages, Netlify.
- **DNS-Records** für `fke.systems` zeigen noch auf Codeberg (A/AAAA/TXT). Müssen bei Hosting-Wechsel angepasst werden.
- **deploy.sh** muss je nach Hosting-Lösung angepasst werden (rsync statt git push, oder komplett entfallen bei CI-fähigem Hoster).
- **Codeberg bleibt als Git-Forge** — nur das Pages-Hosting wird ausgelagert.

## Repo-Status

- **Lokaler Pfad**: `/Users/fke/customers/floriankesslergmbh/pages/`
- **Remote**: `git@codeberg.org:floriankesslergmbh/pages.git`
- **main-Branch**: Hugo-Source (hugo.yaml, go.mod, content/, deploy.sh)
- **pages-Branch**: Generierter Output (HTML/CSS/JS + .domains)
- **Actions**: Deaktiviert
- **Webhook**: Keiner konfiguriert
