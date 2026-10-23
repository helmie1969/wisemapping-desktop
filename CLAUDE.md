# CLAUDE.md

Hinweise für Claude Code in diesem Repository. Antworten und Erklärungen auf Deutsch; Code, Kommentare und Commit-Messages auf Englisch wie im übrigen Code.

## Projektziel

Fork von `wisemapping/wisemapping-desktop` als Grundlage für eine eigene Mind-Mapping-App:

- **Desktop-App** (Electron) mit lokaler Speicherung.
- **Browser-Version derselben App** – ohne Server, ohne Account, ohne Online-Speicher. Die Oberfläche wird geteilt; nur die Speicherschicht unterscheidet sich (siehe „Browser-Variante“).
- Eigene **Vorlagen/Themes** und eine **angepasste Oberfläche**; Funktionsausbau.

`wisemapping-open-source` (Java-Backend mit Accounts) wird bewusst nicht verwendet.

## Zwei Repos, nebeneinander

```
~/Dokumente/GitHub/
├── mindmap.code-workspace   # VS-Code-Workspace für beide Repos
├── wisemapping-desktop/     # dieses Repo
└── wisemapping-frontend/    # Fork von wisemapping/wisemapping-frontend (eigene CLAUDE.md)
```

Das Frontend **muss** als `../wisemapping-frontend` daneben liegen – Vite-Aliase und `scripts/setup-local.sh` verweisen fest darauf.

- Remotes: `origin` = eigener Fork (helmie1969), `upstream` = Original (Push gesperrt).
- Branches: gearbeitet wird auf `dev`; `main` bleibt Spiegel von `upstream/main`.
- Änderungen am Editor, an Themes oder an der Mindmap-Logik gehören ins Frontend-Repo (`packages/editor`, `packages/mindplot`), nicht hierher.

## Architektur

- `src/main/` – Electron-Hauptprozess. `main.ts` registriert die IPC-Kanäle (`mindmap:list|load|save|create|delete|import|export`, `dialog:save-file`, `app:get-storage-path`, `app:open-external`) und das native Menü. `fileManager.ts` speichert jede Mindmap als `<uuid>.wxml` in `app.getPath('userData')` – unter Linux `~/.config/WiseMapping/` (die README nennt veraltet `~/Documents/WiseMapping/`).
- `src/preload/preload.ts` – stellt `window.electron.ipcRenderer` mit Kanal-Whitelist bereit.
- `src/renderer/src/` – React-19-Oberfläche mit MUI: `App.tsx` schaltet zwischen `HomeScreen` (Mindmap-Liste) und `EditorScreen` (bettet `<Editor>` aus `@wisemapping/editor` ein) um; `ExportDialog`/`ImportDialog`; `theme.ts` für das MUI-Theme.
- `src/renderer/src/persistence/` – `ElectronPersistenceManager` (erweitert `PersistenceManager` aus `@wisemapping/mindplot`, lädt/speichert über IPC) und `MapInfoImpl` (implementiert `MapInfo` aus `@wisemapping/editor`).

### Wie das Frontend eingebunden ist

Zwei getrennte Wege – wichtig, um zu wissen, wann was neu gebaut werden muss:

1. **Laufzeit/Bundle:** `electron.vite.config.ts` biegt `@wisemapping/editor|mindplot|web2d` im Renderer per Alias direkt auf `../wisemapping-frontend/packages/*/src` um. Änderungen am Frontend-Quellcode erscheinen in `yarn dev` sofort, ohne Neubau.
2. **Typen (`tsc`, VS-Code-IntelliSense):** kommen aus `node_modules/@wisemapping/*/dist/*.d.ts`, installiert von `yarn deps`. Nach Änderungen an der **öffentlichen API** des Frontends `yarn deps` erneut ausführen, sonst prüft `yarn typecheck` gegen veraltete Typen.

Keine `declare module '@wisemapping/…'`-Platzhalter in `env.d.ts` einführen – sie überdecken die echten Typen und haben früher API-Abweichungen versteckt.

## Befehle

```bash
yarn deps        # Frontend bauen (inkl. .d.ts), packen, in node_modules installieren
yarn dev         # Entwicklungsmodus (Vite-Server :5173 + Electron-Fenster)
yarn typecheck   # tsc --noEmit
yarn build       # Produktions-Build nach out/
yarn dist:linux  # Installationspaket bauen
yarn test        # Playwright-E2E (tests/e2e)
```

## Fallstricke

- **Abhängigkeiten nur mit `yarn deps` installieren**, nie `yarn install`/`npm install` direkt: die `@wisemapping/*`-Pakete in Version 6.x gibt es nicht auf npm. `yarn deps` nutzt intern npm.
- **`package-lock.json` wird von `yarn deps` gelöscht** und erscheint als `D` in `git status`. Diese Löschung nie committen; beim Committen gezielt Dateien stagen statt `git add -A`.
- **Electron unter Node 26 / npm 12:** npm blockiert Install-Scripts, und `node node_modules/electron/install.js` bricht beim Entpacken still ab (nur `snapshot_blob.bin` in `dist/`). Symptom bei `yarn dev`: `Error: Electron uninstall`. Abhilfe: Zip aus `~/.cache/electron/*/electron-v<version>-linux-x64.zip` nach `node_modules/electron/dist/` entpacken, `dist/electron.d.ts` nach `node_modules/electron/` verschieben und `node_modules/electron/path.txt` mit Inhalt `electron` (ohne Zeilenumbruch) anlegen. Nach jedem `yarn deps` prüfen.
- **Yarn-Versionen:** hier Yarn 1, im Frontend Yarn 4 – beides über Corepack (in `~/.local/bin`). Bei Rückfragen von Corepack `COREPACK_ENABLE_DOWNLOAD_PROMPT=0` setzen.
- **`yarn lint` ist im Original kaputt** (ESLint 9 ohne Flat-Config). `yarn typecheck` ist die verlässliche Prüfung.
- **`http://localhost:5173` im normalen Browser bleibt leer** (`window.electron` fehlt). Das ist erwartet und genau die Stelle, die die Browser-Variante ersetzen muss.
- Electron meldet unter Wayland harmlose Fehler (`wayland_wp_color_manager`, `ZINK`, `Autofill.enable`).

## Browser-Variante (geplant)

Alle Speicherzugriffe laufen heute über `window.electron.ipcRenderer` – in `ElectronPersistenceManager`, aber auch direkt in `HomeScreen.tsx`, `EditorScreen.tsx` und den Dialogen. Vorgehen:

1. Eine Speicher-Schnittstelle einführen (Liste, Laden, Speichern, Anlegen, Löschen, Import/Export) und alle direkten `ipcRenderer`-Aufrufe im Renderer dahinter verbergen.
2. Zwei Implementierungen: Electron (IPC, wie heute) und Browser (IndexedDB; Import/Export über Datei-Upload/-Download bzw. File System Access API).
3. Eigener Vite-Build des Renderers als statische Seite (z. B. für GitHub Pages), der die Browser-Implementierung wählt.

## Lizenz und Branding

WiseMapping Public License (Apache 2.0 + Zusatz): **Jeder Bildschirm** muss sichtbar „powered by WiseMapping“ mit Link auf https://www.wisemapping.com zeigen, immer an derselben Position. Der Name „WiseMapping“ darf nicht als Marke für die eigene App verwendet werden – eigener App-Name nötig (`app.setName` in `main.ts`, `electron-builder.yml`, `package.json`). Das vorhandene Logo sitzt in `packages/editor/src/components/creator-info-pane` im Frontend; bei UI-Umbauten den Hinweis erhalten.
