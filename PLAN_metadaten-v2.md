# Plan: Metadaten auf Schema v2 umstellen (QUADRIGA-Space)

Status: offen · Erstellt: 2026-11-11 · Für die Übernahme durch einen anderen Agenten

## Kontext

QUADRIGA-Space ist ein fast unveränderter Abzug des alten `Book_Template` (Stand 2026-10-27).
Das Buch baut und deployt wieder (der Ubuntu-26.04-Fix für `IRkernel` ist auf `main`, PR #1),
aber die Metadaten sind auf dem alten Stand:

- Quelle ist `metadata.yml` mit `schema-version: 1.0.0` (Schema v1).
- Generierte Dateien heißen `metadata.jsonld` und `metadata.rdf`.
- `quadriga/metadata/*` ist deutlich älter als der aktuelle Stand im Template-Repo
  `quadriga-dk/quadriga-scaffolding` (`src/quadriga_scaffolding/data/`):
  `create_oersi_metadata.py` fehlt, alle anderen Skripte weichen ab.
- `metadata.yml` enthält noch Template-Werte: Titel „QUADRIGA-OER-Template …",
  Autor:innen des Templates, `identifier: https://doi.org/10.5281/zenodo.14970672`,
  `git`/`url` zeigen auf `Book_Template`.

Der Workflow „Update Metadata" läuft erfolgreich durch, erzeugt aber keine Änderungen,
weil er mit den alten Skripten auf den alten Daten arbeitet.

## Ziel

1. Metadaten auf **Schema v2** migrieren (`quadriga-metadata.yml`, `schemaVersion: 2.0.0`).
2. Skripte und Workflows aus `quadriga-scaffolding` übernehmen (dort ist der aktuelle Stand).
3. Generierte Dateien neu erzeugen: `CITATION.cff`, `CITATION.bib`, `.zenodo.json`,
   `quadriga-metadata.jsonld`, `quadriga-metadata.rdf` und die OERSI-Datei `metadata.yml`.
4. Deploy- und Update-Workflow auf die neuen Dateinamen umstellen.
5. Inhaltliche Metadaten auf QUADRIGA-Space anpassen (siehe „Offene Punkte").

## Schritte

### 1. Branch und Umgebung

```bash
cd ~/code/QUADRIGA-Space
git switch -c <name>/metadata-v2
python -m venv .venv && . .venv/bin/activate
pip install -r dev-requirements.txt   # enthält jsonschema für die Schema-Validierung
```

### 2. Skripte und Workflows übernehmen

Quelle der Wahrheit ist `quadriga-dk/quadriga-scaffolding`,
Verzeichnis `src/quadriga_scaffolding/data/` (lokal z. B. `~/code/quadriga-scaffolding`).

```bash
SC=~/code/quadriga-scaffolding/src/quadriga_scaffolding/data

# Metadaten-Skripte
cp $SC/quadriga/metadata/*.py quadriga/metadata/

# Workflows
cp $SC/.github/workflows/update-metadata.yml .github/workflows/
cp $SC/.github/workflows/deploy-book-python-sparql-r.yml .github/workflows/
```

Achtung beim Deploy-Workflow: Die Space-Version enthält den Ubuntu-26.04-Fix. Nach dem Kopieren
prüfen, dass er noch da ist:

```bash
grep -n "kernel_dir" .github/workflows/deploy-book-python-sparql-r.yml
```

Falls der Scaffold-Stand den Fix (noch) nicht enthält, den Block „Set up IRkernel" aus der
bisherigen Space-Version übernehmen:

```yaml
    - name: Set up IRkernel
      run: |
        kernel_dir="$(Rscript -e 'cat(system.file("kernelspec", package = "IRkernel"))')"
        jupyter kernelspec install --user --replace --name ir "$kernel_dir"
```

Optional, nur wenn benötigt: `_ext/video_figure.py`, `_static/video-figure.js`,
`_static/quadriga.css` aus dem Scaffold übernehmen. `_config.yml` **nicht** aus dem Scaffold
kopieren, die ist buch-spezifisch (Extensions dort selbst registrieren).

`__pycache__`-Ordner nicht committen.

### 3. `metadata.yml` (v1) nach `quadriga-metadata.yml` (v2) migrieren

Vor dem ersten `run_all` migrieren, denn danach ist `metadata.yml` die generierte OERSI-Datei.

Schlüssel-Mapping v1 → v2:

| v1 | v2 |
| --- | --- |
| `authors` (`given-names`, `family-names`, `credit`) | `authorLIST` (`givenName`, `familyName`, `creditLIST`) |
| `keywords` | `keywordLIST` (Einträge auch als `de:`/`en:`-Paare möglich) |
| `table-of-contents` | `tableOfContents` |
| `discipline` | `disciplineLIST` |
| `research-object-type` | `researchObjectTypeLIST` |
| `target-group` | `targetGroupLIST` |
| `time-required` | `timeRequired` |
| `contributors` | `contributorLIST` |
| `git` | `identifierGit` |
| `url` | `identifierOER` |
| `used-tools` | `usedToolLIST` |
| `chapters` | `chapterLIST` |
| `date-issued` | `dateIssued` |
| `date-modified` | `dateModified` |
| `context-of-creation` | `contextOfCreation` |
| `quality-assurance` | `qualityAssurance` |
| `learning-resource-type` | `learningResourceType` |
| `schema-version: 1.0.0` | `schemaVersion: 2.0.0` |
| `license: {code, content: {url, name}}` | `license:` (URL) und `licenseCode:` (URL) |

Kopfzeile der Datei auf das v2-Schema umstellen:

```yaml
# yaml-language-server: $schema=https://quadriga-dk.github.io/quadriga-schema/v2.0.0/schema.json
```

`qualityAssurance` muss ein Mapping mit `date` und `description` sein (kein String, keine Liste),
sonst bricht `create_zenodo_json` mit `AttributeError: 'list' object has no attribute 'get'` ab
(genau dieser Fehler tritt derzeit in Text-Fallstudie-4 auf).

Beispiel für einen v2-Autor:innen-Eintrag (siehe `Book_Template/quadriga-metadata.yml`):

```yaml
authorLIST:
- familyName: Schnaitter
  givenName: Hannes
  orcid: https://orcid.org/0000-0002-1602-6032
  affiliation: Humboldt-Universität zu Berlin, Institut für Bibliotheks- und Informationswissenschaft
  creditLIST:
  - Writing – original draft
  - Software
```

Nach der Migration: `metadata.yml` löschen (wird als OERSI-Datei neu erzeugt),
`metadata.jsonld` und `metadata.rdf` löschen.

### 4. Generierte Dateien erzeugen

```bash
python -m quadriga.metadata.run_all
```

Erzeugt bzw. aktualisiert: `CITATION.cff`, `CITATION.bib`, `.zenodo.json`,
`quadriga-metadata.jsonld`, `quadriga-metadata.rdf` und `metadata.yml` (OERSI,
Schema.org/LRMI für oersi.org).

Hinweise:

- `quadriga-metadata.yml` bleibt die einzige Quelle der Wahrheit.
- Die Lernziel-Extraktion sucht die Lernziel-Datei über `_toc.yml`. Fehlt sie, gibt es nur eine
  Warnung – kein Fehler.
- Fehlt `jsonschema`, wird die Schema-Validierung übersprungen (Warnung). Für einen echten
  Lauf `dev-requirements.txt` installieren.

### 5. Workflows an die neuen Dateinamen anpassen

`update-metadata.yml`, Schritt „Stage metadata files":

```yaml
for file in quadriga-metadata.yml metadata.yml CITATION.bib CITATION.cff .zenodo.json quadriga-metadata.jsonld quadriga-metadata.rdf; do
  [ -f "$file" ] && git add "$file"
done
```

`deploy-book-python-sparql-r.yml`, Schritt „Copy metadata files to build directory":

```yaml
cp quadriga-metadata.jsonld _build/html/quadriga-metadata.jsonld
cp quadriga-metadata.rdf _build/html/quadriga-metadata.rdf
```

(`inject_all_metadata` aus dem Scaffold liest die neuen Dateinamen bereits selbst.)

### 6. Inhaltliche Metadaten klären (Rückfrage an das Team)

Aktuell stehen im Buch noch Template-Werte. Zu klären und dann einzutragen:

- Titel des Buches (`title`, außerdem `_config.yml: title`).
- Autor:innen und Mitwirkende (`authorLIST`, `contributorLIST`).
- DOI/`identifier`, `identifierGit`, `identifierOER` – gibt es schon eine eigene DOI? Sonst
  vorerst Template-Werte stehen lassen und als TODO markieren.
- `qualityAssurance.description` (DOI des Qualitätssicherungs-Verfahrens).
- Lizenz: `license` (Inhalt, CC BY-SA 4.0) und `licenseCode` (Code, AGPL-3.0) prüfen.
- `_config.yml: url` zeigt noch auf `Book_Template`.

### 7. Prüfen

```bash
python -m quadriga.metadata.run_all     # fehlerfrei durchlaufen
git status                              # genau die erwarteten Dateien geändert
jupyter-book build .                    # Buch baut
grep -o 'application/ld+json' _build/html/index.html   # JSON-LD eingebunden
git grep -n "metadata.jsonld\|metadata.rdf\|schema-version"  # keine Altlasten mehr
```

Nach dem Merge:

- „Update Metadata" läuft grün und committet ggf. die generierten Dateien.
- „Deploy book with Python, SPARQL, and R" läuft grün (Ubuntu 26.04).
- Der Deploy-Job kopiert `quadriga-metadata.jsonld`/`.rdf` nach `_build/html`.

## Akzeptanzkriterien

- [ ] `quadriga-metadata.yml` mit `schemaVersion: 2.0.0` ist die Quelle; `metadata.yml` ist die
      generierte OERSI-Datei.
- [ ] `quadriga/metadata/*` entspricht dem Stand in `quadriga-scaffolding`.
- [ ] `run_all` läuft fehlerfrei, alle generierten Dateien sind aktuell und committed.
- [ ] Keine Verweise mehr auf `metadata.jsonld`, `metadata.rdf` oder `schema-version`.
- [ ] Deploy-Workflow enthält weiterhin den Ubuntu-26.04-`IRkernel`-Fix.
- [ ] Update-Metadata- und Deploy-Workflow sind auf `main` grün.

## Offene Punkte

1. Echte Metadaten-Werte (Titel, Beteiligte, DOI, Lizenz) – ohne diese bleiben Template-Angaben
   im Buch und in den generierten Dateien.
2. Soll QUADRIGA-Space überhaupt als eigenständige OER publiziert werden (eigene DOI), oder bleibt
   es eine Demo-/Spielwiese? Das entscheidet, ob `identifierOER` gesetzt wird.
3. Inhalt selbst ist noch Template-Text („QUADRIGA-OER-Template …") – nicht Teil dieses Plans.
