# rpg-engine – Plan

## 1. Ziel & Umfang

| Punkt | Entscheidung |
|---|---|
| Zweck | Generische Solo-Engine; Gates ist nur eine mögliche Kampagne |
| Regeln | Eigenes System, D&D 5e und Gates dienen als Vorlage |
| Modell | Austauschbar (Ollama, Claude, …); Sache des KI-Clients, nicht der Engine |
| Oberfläche | Nur CLI, keine Web-Oberfläche geplant |
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
| Schnittstelle | REST-ähnlich, Ressourcen + Befehle | Zustände lesen per GET; Spielhandlungen als Befehle (`POST /sessions/{id}/turns`); die Engine rechnet immer selbst und liefert Ergebnisse (Würfe, Zustände), keine Erzählung |
| CLI-Anbindung | Von Anfang an über die API | Die Engine läuft als lokaler Server, das CLI ist ein echter Client |
| KI-Anbindung | Die KI ist ein normaler API-Client | Der Engine ist egal, ob Skript, KI oder Mensch (CLI) aufruft; sie enthält keinen KI-Code. Ein eigener Spielleiter-Client (oder ein MCP-Server über der API) setzt Tool-Aufrufe des Modells in API-Aufrufe um; Erzählung und Streaming liegen dort |

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
| Exemplar-Erweiterung | Ein Exemplar kann zusätzliche Fähigkeiten und Modi bekommen (z. B. Verzauberung `+1d6 Feuer`); die Vorlage bleibt unverändert |
| Aktive Zustände | Eigene Einträge am Charakter: Name, Quelle (Fähigkeit/Gegenstand/Gegner), Dauer (Runden/Rast/dauerhaft), Wirkung (Wertänderungen, Vorteil/Nachteil); die Engine zählt herunter und entfernt Abgelaufenes |
| Figuren | Ein gemeinsames Modell (Werte, Fähigkeiten, Gegenstände, Zustände) mit Typ: **Spieler**, **NPC** (benannt, taucht mehrfach auf), **TC** (temporärer Charakter, z. B. ein Wolf aus einer Zufallsbegegnung). Ein TC kann zum NPC befördert werden |
| Kreaturen | Vorlagen (wie Rassen/Klassen), aus denen TCs erzeugt werden |
| Kennungen | NPC: Name, eindeutig in der ganzen Kampagne. TC: kein Name, nur eine Kennung (z. B. `wolf_01`), eindeutig pro Begegnung |
| TC-Lebensdauer | Ein TC wird nach der Begegnung gelöscht, außer er wurde zum NPC befördert |
| Szenen-Protokoll | Die Szene speichert, wie viele von welcher Kreatur vorkamen und was mit ihnen geschah |
| TC-Ausgang | Feste Ausgänge (getötet, geflohen, verschont, gezähmt, befördert) plus optionale Notiz; auswertbar (z. B. „12 Wölfe getötet“) |
| Orte | Baumstruktur (z. B. Kontinent → Stadt → Taverne) mit Beschreibung und Notizen; NPCs und Gegenstände können einem Ort zugeordnet sein; die KI darf neue Orte anlegen |
| Quests | Baumstruktur: Teilziele sind selbst Quests (gleicher Typ). Jede hat Titel, Beschreibung, Status (offen/aktiv/erledigt/gescheitert), optional Verknüpfungen zu Auftraggeber-NPC, Orten, Belohnungen; die KI darf Quests anlegen und den Status ändern |
| Quest-Status | Wird manuell bzw. von der KI gesetzt, nicht automatisch aus Teilzielen abgeleitet; die Engine meldet nur Hinweise (z. B. „Alle Teilziele erledigt“) – so bleiben optionale Teilziele und alternative Wege möglich |
| Verlauf | Kampagne → Szene; dauerhaft gespeichert werden pro Szene nur Zusammenfassung und Szenen-Protokoll |
| Züge | Die Engine verwaltet Züge im Speicher, persistiert sie aber nicht |
| Kampfstart | Die KI startet einen Kampf per Tool mit einer Liste aus Figuren (ID/Name) und ihrer Initiative; die Engine führt daraus die Reihenfolge |
| Aktionsökonomie | Die Engine merkt sich pro Figur und Runde, was genutzt wurde (Bewegung, Aktion, Bonusaktion, Reaktion …) |
| Kampf-Schnappschuss | Laufender Kampf (Reihenfolge, aktive Figur, genutzte Aktionen) wird als temporärer Schnappschuss gespeichert, nach Kampfende gelöscht; erlaubt Fortsetzen nach Server-Neustart |
| Szenenende | Die KI beendet eine Szene per API-Aufruf, wenn die Erzählung es nahelegt (Ortswechsel, Zeitsprung …); der Spieler sagt es der KI oder ruft selbst auf. Die Zusammenfassung ist Pflichtparameter: fehlt sie, lehnt die Engine ab (z. B. 422 mit Hinweis); ein leerer String wird akzeptiert, die Antwort enthält aber eine Warnung „Zusammenfassung leer“ |

## 5. Sprache & Werkzeuge

| Punkt | Entscheidung | Begründung |
|---|---|---|
| Sprache | Rust | Prioritäten: Typzwang, klare Strukturen, Clean-Code-Zwang, Performance/Speicher, wenig Overhead. Komposition + Traits + Enums statt Vererbung passen zum Datenmodell (Figur mit Typ, Vorlage + Exemplar) |
| DB-Anbindung | sqlx (Postgres) | SQL selbst geschrieben, beim Kompilieren gegen das Schema geprüft; JSONB per serde; Migrationen eingebaut |
| DB-Abstraktion | Repository-Traits (z. B. `FigurRepo`, `SzeneRepo`) | Die Engine kennt nur die Traits; zuerst Postgres-Umsetzung, weitere DBs (SQLite, MongoDB …) später ohne Engine-Änderung |
| API-Framework | Axum (auf Tokio) | Verbreitet, schlank, typsichere Extraktoren, passt zu sqlx und serde |

## 6. Architektur

```mermaid
flowchart TB
    subgraph Clients["Clients (eigene Programme)"]
        CLI["CLI<br/>clap / rustyline"]
        KI["Spielleiter-Client<br/>ruft Modell auf (Ollama, Claude …)<br/>Tool-Aufrufe → API-Aufrufe"]
        Skript["Skripte / Tests"]
    end

    subgraph Engine["Engine-Server (Rust, Tokio)"]
        API["API-Schicht · Axum<br/>Routen, DTOs (serde), Validierung,<br/>Fehler → HTTP (z. B. 422), Warnungen"]
        SVC["Service-Schicht<br/>Anwendungsfälle: Probe würfeln, Kampf starten,<br/>Zug ausführen, Szene beenden, TC befördern …"]
        DOM["Domäne (reine Logik, kein I/O)<br/>Figur, Fähigkeit, Gegenstand, Zustand, Quest, Ort<br/>PreparedDiceThrow, Regeln"]
        MEM["Laufzeit-Zustand (Speicher)<br/>aktive Kämpfe: Reihenfolge, aktive Figur,<br/>genutzte Aktionen"]
        REPO["Repository-Traits<br/>FigurRepo, SzeneRepo, KampfRepo …"]
        PG["sqlx-Umsetzung (Postgres)<br/>SQL beim Kompilieren geprüft, JSONB ↔ serde"]
    end

    DB[("PostgreSQL<br/>Tabellen + JSONB<br/>inkl. Kampf-Schnappschuss")]

    CLI -- "HTTP/JSON (reqwest)" --> API
    KI -- "HTTP/JSON" --> API
    Skript -- "HTTP/JSON" --> API
    API --> SVC
    SVC --> DOM
    SVC <--> MEM
    SVC --> REPO
    REPO -. "implementiert durch" .-> PG
    PG --> DB
    MEM -. "Schnappschuss über KampfRepo" .-> REPO
```

Abhängigkeiten zeigen nur nach unten: Die Domäne kennt weder Axum noch sqlx; die Service-Schicht kennt nur die Repository-Traits.

## 7. ERD

Quelle: `docs/erd.mmd`

```mermaid
erDiagram
    KAMPAGNE ||--o{ WERT_DEF : "definiert"
    KAMPAGNE ||--o{ VORLAGE : "hat"
    KAMPAGNE ||--o{ WURF : "hat"
    KAMPAGNE ||--o{ FAEHIGKEIT : "hat"
    KAMPAGNE ||--o{ GEGENSTAND_VORLAGE : "hat"
    KAMPAGNE ||--o{ FIGUR : "hat"
    KAMPAGNE ||--o{ ORT : "hat"
    KAMPAGNE ||--o{ QUEST : "hat"
    KAMPAGNE ||--o{ SZENE : "hat"

    FAEHIGKEIT ||--o{ FAEHIGKEIT_WURF : "benannte Formeln"
    WURF ||--o{ FAEHIGKEIT_WURF : "genutzt von"
    WURF ||--o{ MODUS_WURF : "genutzt von"

    VORLAGE ||--o{ VORLAGE_FAEHIGKEIT : "gibt"
    FAEHIGKEIT ||--o{ VORLAGE_FAEHIGKEIT : "in"

    FIGUR ||--o{ FIGUR_VORLAGE : "Rasse/Klasse"
    VORLAGE ||--o{ FIGUR_VORLAGE : "von"
    VORLAGE |o--o{ FIGUR : "Kreatur eines TC"
    FIGUR ||--o{ FIGUR_FAEHIGKEIT : "besitzt"
    FAEHIGKEIT ||--o{ FIGUR_FAEHIGKEIT : "in"
    FIGUR ||--o{ ZUSTAND : "aktiv"
    ORT |o--o{ FIGUR : "befindet sich"

    GEGENSTAND_VORLAGE ||--o{ MODUS : "Standard-Modi"
    GEGENSTAND_VORLAGE ||--o{ GEGENSTAND_EXEMPLAR : "Exemplare"
    GEGENSTAND_EXEMPLAR |o--o{ MODUS : "Zusatz-Modi"
    MODUS ||--o{ MODUS_WURF : "Formeln"
    GEGENSTAND_EXEMPLAR ||--o{ EXEMPLAR_FAEHIGKEIT : "Zusatz"
    FAEHIGKEIT ||--o{ EXEMPLAR_FAEHIGKEIT : "in"
    FIGUR |o--o{ GEGENSTAND_EXEMPLAR : "besitzt"
    ORT |o--o{ GEGENSTAND_EXEMPLAR : "liegt in"

    ORT |o--o{ ORT : "enthält"
    QUEST |o--o{ QUEST : "Teilziele"
    FIGUR |o--o{ QUEST : "Auftraggeber"
    QUEST ||--o{ QUEST_ORT : ""
    ORT ||--o{ QUEST_ORT : ""

    ORT |o--o{ SZENE : "spielt in"
    SZENE ||--o{ BEGEGNUNG : "hat"
    BEGEGNUNG |o--o{ FIGUR : "TCs"
    SZENE ||--o{ SZENEN_PROTOKOLL : "Kreaturen"
    VORLAGE ||--o{ SZENEN_PROTOKOLL : "Kreatur"

    KAMPAGNE {
        uuid id PK
        text name
    }
    WERT_DEF {
        uuid id PK
        uuid kampagne_id FK
        text name "z. B. str, luck; unique je Kampagne"
        text beschreibung
    }
    WURF {
        uuid id PK
        uuid kampagne_id FK
        text formel "z. B. 2d8 + str"
    }
    FAEHIGKEIT {
        uuid id PK
        uuid kampagne_id FK
        text name
        text beschreibung
        jsonb folgen "Zustände, Dauer, Ladungen, Nutzungen/Rast"
        bool von_ki_angelegt
    }
    FAEHIGKEIT_WURF {
        uuid faehigkeit_id PK,FK
        text name PK "z. B. erfolg, hoehe"
        uuid wurf_id FK
    }
    VORLAGE {
        uuid id PK
        uuid kampagne_id FK
        text art "rasse | klasse | kreatur"
        text name
        jsonb wertaenderungen "Rasse/Klasse: {dex: +2}"
        jsonb grundwerte "Kreatur: Startwerte"
    }
    VORLAGE_FAEHIGKEIT {
        uuid vorlage_id PK,FK
        uuid faehigkeit_id PK,FK
    }
    FIGUR {
        uuid id PK
        uuid kampagne_id FK
        text typ "spieler | npc | tc"
        text name "NPC: unique je Kampagne"
        text kennung "TC: z. B. wolf_01, unique je Begegnung"
        uuid kreatur_id FK "nur TC"
        uuid begegnung_id FK "nur TC"
        uuid ort_id FK
        jsonb werte "{str: 14, luck: 3, hp: 20}"
    }
    FIGUR_VORLAGE {
        uuid figur_id PK,FK
        uuid vorlage_id PK,FK
    }
    FIGUR_FAEHIGKEIT {
        uuid figur_id PK,FK
        uuid faehigkeit_id PK,FK
        text herkunft "rasse | klasse | erlangt | kreatur"
        int nutzungen_uebrig
    }
    ZUSTAND {
        uuid id PK
        uuid figur_id FK
        text name
        text quelle_art "faehigkeit | gegenstand | figur"
        uuid quelle_id
        text dauer_art "runden | rast | dauerhaft"
        int dauer_rest
        jsonb wirkung "Wertänderungen, Vor-/Nachteil"
    }
    GEGENSTAND_VORLAGE {
        uuid id PK
        uuid kampagne_id FK
        text name
        text beschreibung
    }
    GEGENSTAND_EXEMPLAR {
        uuid id PK
        uuid vorlage_id FK
        uuid besitzer_id FK "Figur, optional"
        uuid ort_id FK "optional"
        text eigener_name
        int anzahl
        int ladungen
        text zustand
    }
    MODUS {
        uuid id PK
        uuid gegenstand_vorlage_id FK "entweder"
        uuid exemplar_id FK "oder"
        text name "z. B. zweihändig"
    }
    MODUS_WURF {
        uuid modus_id PK,FK
        text name PK "to_hit, damage"
        uuid wurf_id FK
    }
    EXEMPLAR_FAEHIGKEIT {
        uuid exemplar_id PK,FK
        uuid faehigkeit_id PK,FK
    }
    ORT {
        uuid id PK
        uuid kampagne_id FK
        uuid parent_id FK
        text name
        text beschreibung
        text notizen
    }
    QUEST {
        uuid id PK
        uuid kampagne_id FK
        uuid parent_id FK
        text titel
        text beschreibung
        text status "offen | aktiv | erledigt | gescheitert"
        uuid auftraggeber_id FK
        jsonb belohnung
    }
    QUEST_ORT {
        uuid quest_id PK,FK
        uuid ort_id PK,FK
    }
    SZENE {
        uuid id PK
        uuid kampagne_id FK
        int nummer
        uuid ort_id FK
        text zusammenfassung "Pflicht beim Beenden"
        timestamptz beendet_am
    }
    BEGEGNUNG {
        uuid id PK
        uuid szene_id FK
        bool aktiv
        jsonb kampf_schnappschuss "Reihenfolge, aktive Figur, Aktionen; nach Ende null"
    }
    SZENEN_PROTOKOLL {
        uuid szene_id PK,FK
        uuid kreatur_id PK,FK
        text ausgang PK "getötet | geflohen | verschont | gezähmt | befördert"
        int anzahl
        text notiz
    }
```

Annahmen (noch zu bestätigen):
- Rasse, Klasse und Kreatur teilen eine Tabelle `VORLAGE` (Spalte `art`).
- Werte einer Figur liegen als JSONB; die Engine prüft die Namen gegen `WERT_DEF`.
- Begegnung ist eigene Tabelle: Rahmen für TC-Kennungen (`unique(begegnung_id, kennung)`) und Kampf-Schnappschuss. TCs werden mit der Begegnung gelöscht, außer befördert.
- NPC-Name eindeutig per partiellem Unique-Index (`where typ = 'npc'`).
