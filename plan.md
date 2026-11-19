# rpg-engine – Plan

## 1. Ziel & Umfang

| Punkt | Entscheidung |
|---|---|
| Zweck | Generische Solo-Engine; Gates ist nur eine mögliche Kampagne |
| Regeln | Eigenes System, D&D 5e und Gates dienen als Vorlage |
| Modell | Austauschbar (Ollama, Claude, …) über eine Schnittstelle |
| Oberfläche | Zuerst CLI, Web später |
| Mechanik durch die Engine | Proben & Kampf · Charakter & Inventar · Welt & NPCs · Quests & Verlauf |
| Funktionen | Hängen am Träger (Gegenstand mit Modi, Charakter mit Rassen-, Klassen- und erlangten Fähigkeiten) |
| Umfang einer Fähigkeit | Benannte Würfelformeln plus feste Folgen (Zustände, Dauer, Ladungen, Nutzungen pro Rast) |
| KI legt Fähigkeiten an | Ja, sie speichert direkt; die Engine meldet Neues, der Spieler kann nachträglich korrigieren |
| Erste Version | Eine Szene läuft komplett durch: Eingabe → Engine → Erzählung → Zustand gespeichert |

### Beispiel Funktionen

```
Gegenstand "Langschwert"
  Modus einhändig:  to_hit = d20 + STR + prof   damage = 1d8 + STR
  Modus zweihändig: to_hit = d20 + STR + prof   damage = 1d10 + STR

Charakter "X"
  Fähigkeiten:
    Dunkelsicht      (Herkunft: Rasse)
    Kampfrausch      (Herkunft: Klasse)
    Schweben         (Herkunft: im Spiel erlangt, von der KI angelegt)
      höhe    = 1d6 ft
      erfolg  = d20 + DEX + prof
```

## 2. Technische Grundentscheidungen

| Punkt | Entscheidung | Folge |
|---|---|---|
| Persistierung | Postgres | Fähigkeiten, Modi und Folgen als JSONB; lokal z. B. per Docker, Start über ein Skript |
| Schnittstelle | REST-ähnlich, Ressourcen + Befehle | Zustände lesen per GET; Spielhandlungen als Befehle (`POST /sessions/{id}/turns`); die Engine rechnet immer selbst; Erzählung per Streaming (z. B. SSE) |
| CLI-Anbindung | Von Anfang an über die API | Die Engine läuft als lokaler Server, das CLI ist ein echter Client; die Web-Oberfläche nutzt später dieselbe API |
| KI-Anbindung | Tool-Aufrufe des Modells, keine REST-Aufrufe | Die Tools binden an dieselbe Service-Schicht wie die API |

## 3. Würfelsystem

| Punkt | Entscheidung |
|---|---|
| PreparedDiceThrow (Name vorläufig) | Wird aus einem Formel-String erzeugt (`"2d8 + str"`); beim Erzeugen nur parsen und prüfen |
| Variablen (str, dex, prof, …) | Erst beim Würfeln aufgelöst: `roll(context)` holt die Werte vom aktuellen Charakter |
| Ergebnis von `roll()` | Ein Wurf-Objekt: Liste aus gewürfelten Werten und Modifikatoren, kennt `sum()` |
| Referenz | PreparedDiceThrows sind eigene Einträge in der DB; Waffen-Modi und Fähigkeiten verweisen per ID darauf; mehrere Träger können sich einen Wurf teilen |
| KI und Würfe | Die KI darf Würfe nur neu anlegen; Ändern bestehender Würfe ist vorerst nicht vorgesehen |
| Vorteil/Nachteil, Krit | Optionen bei `roll()`, z. B. `roll(context, advantage=True)`; das Wurf-Objekt vermerkt die Option |

```
throw = PreparedDiceThrow("2d8 + str")
wurf  = throw.roll(context=charakter_X)   # z. B. [5, 6, str +3]
wurf.sum()                                # 14
```

## 4. Datenmodell

| Punkt | Entscheidung |
|---|---|
| Charakterwerte | Frei definierbar pro Kampagne (Name + Zahl); Standardvorlage: STR, DEX, CON, INT, WIS, CHA, prof, Stufe/EP |
| Wertebindung | Die Engine kennt keine festen Werte; die Kampagne legt ihre Werte fest (z. B. ohne `dex`, mit `luck`). Formeln, Vorlagen und Charaktere sprechen Werte per Name an; Unbekanntes wird beim Anlegen abgelehnt |
| Rassen & Klassen | Vorlagen in der DB (Name, Fähigkeiten, Wertänderungen als JSONB `{"dex": +2}`), gehören zu einer Kampagne; der Charakter verweist darauf; die KI darf neue anlegen. Die D&D-Vorlage ist ein Startpaket zum Übernehmen und Anpassen |
| Gegenstände | Vorlage + Exemplar: Die Vorlage (z. B. „Langschwert“) trägt Modi und Würfe; das Exemplar gehört einem Charakter und hält eigene Daten (Anzahl, Ladungen, Zustand, eigener Name) |
