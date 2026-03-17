# Hands-on Workshop: KI-gestützte HR-Zeugniserstellung

## SAP GenAI Hub + SAP Build Process Automation

**Dauer:** ca. 2 Stunden
**Ziel:** Einen End-to-End-Prozess bauen, der per KI ein Zwischenzeugnis generiert

---

## Architektur-Überblick

```
Teilnehmer füllt Formular aus (Mitarbeiter auswählen)
        |
        v
SBPA holt Mitarbeiterdaten (Action: Get Employees)
        |
        v
Vorgesetzter prüft Daten und ergänzt Details (Approval Form)
        |
        v  [Approve]                         [Reject]
        |                                       |
GenAI Hub generiert Zeugnis (Orchestration)     Ende
        |
        v
Zeugnis wird angezeigt (Output Form)
        |
        v
E-Mail mit Zeugnis wird versendet
        |
        v
       Ende
```

---

## Was ist bereits vorbereitet?

| Komponente | Status | Details |
|---|---|---|
| SAP AI Core Deployment | RUNNING | Orchestration Deployment für alle Teilnehmer |
| Orchestration Config | Gespeichert | `hr-zwischenzeugnis-config` (Version 0.0.1) mit HR-Prompt |
| BTP Destination `AI_Core` | Konfiguriert | OAuth2ClientCredentials, zeigt auf das Deployment |
| BTP Destination `S4_Destination` | Konfiguriert | Zeigt auf den Employee Dummy Service |
| Action: Get Employees | Released | In der Action Library verfügbar |
| OpenAPI Spec für Orchestration | Bereitgestellt | Wird in Schritt 1 hochgeladen |

---

## Bereitgestellte Werte

Diese Werte werden im Workshop benötigt. Sie sind für alle Teilnehmer identisch:

| Parameter | Wert |
|---|---|
| **Deployment ID** | `d91f4280c50fbc2c` |
| **URL Prefix** | `/v2/inference/deployments/d91f4280c50fbc2c/v2` |
| **Config Name** | `hr-zwischenzeugnis-config` |
| **Config Version** | `0.0.1` |
| **Config Scenario** | `orchestration` |
| **Destination (AI)** | `AI_Core` |
| **Destination (Employees)** | `S4_Destination` |

---

## Schritt 1: AI Core Orchestration Action erstellen (15 Min)

In diesem Schritt erstellt ihr eine Action, die die SAP GenAI Hub Orchestration API anbindet. Damit lernt ihr, wie man eine beliebige REST API in SAP Build Process Automation integriert.

### 1.1 Action-Projekt anlegen

1. Öffne die **SAP Build Lobby**
2. Klick **Create**
3. Wähle **Build an Automated Process > Action**
4. Name: `AI Core Orchestration - [DEIN NAME]`
5. Lade die bereitgestellte **OpenAPI Spec** hoch (Datei: `sap-ai-core-orchestration-openapi.json`)
6. Klick **Create**

### 1.2 URL Prefix konfigurieren

Der URL Prefix verbindet die Action mit eurem spezifischen Deployment.

1. Im Action Editor: Klick oben rechts auf das **Zahnrad-Icon** (Project Settings)
2. Wähle links **URL Prefix**
3. Trage ein:
   ```
   /v2/inference/deployments/d91f4280c50fbc2c/v2
   ```
4. Klick **Save**

> **Erklaerung:** Die BTP Destination `AI_Core` zeigt auf `https://api.ai.prod.eu-central-1.aws.ml.hana.ondemand.com`. Der URL Prefix wird angehängt und ergibt zusammen mit dem Action-Pfad `/completion` die vollstaendige URL zum Orchestration Endpoint.

### 1.3 Input prüfen

1. Klick links auf die Action **Run an orchestrated completion inference request**
2. Gehe zum **Input** Tab
3. Prüfe, dass folgende Felder vorhanden sind:
   - **Parameter:** `AI-Resource-Group` (header, string) - Wert: `default`
   - **Body:**
     - `config_ref` (object) mit `name`, `version`, `scenario`
     - `placeholder_values` (object) mit den Prompt-Variablen

Falls die Felder fehlen oder falsch sind, klick unter Body auf **Add > From Sample JSON** und füge ein:

```json
{
  "config_ref": {
    "scenario": "orchestration",
    "name": "hr-zwischenzeugnis-config",
    "version": "0.0.1"
  },
  "placeholder_values": {
    "mitarbeiter_name": "Max Mustermann",
    "position": "Senior Consultant SAP BTP",
    "abteilung": "IT Consulting",
    "eintrittsdatum": "01.03.2022",
    "stichpunkte": "SAP Integration Suite, Teamleitung",
    "bewertung": "sehr gut"
  }
}
```

### 1.4 Output prüfen

1. Gehe zum **Output** Tab
2. Falls leer: Klick **Add > From Sample JSON** und füge ein:

```json
{
  "request_id": "abc-123",
  "intermediate_results": {
    "llm": {
      "id": "msg-123",
      "object": "chat.completion",
      "created": 1710000000,
      "model": "anthropic--claude-4.5-haiku",
      "choices": [
        {
          "index": 0,
          "message": {
            "role": "assistant",
            "content": "Zwischenzeugnis Text hier..."
          },
          "finish_reason": "stop"
        }
      ],
      "usage": {
        "completion_tokens": 500,
        "prompt_tokens": 200,
        "total_tokens": 700
      }
    }
  },
  "final_result": {
    "id": "msg-123",
    "object": "chat.completion",
    "created": 1710000000,
    "model": "anthropic--claude-4.5-haiku",
    "choices": [
      {
        "index": 0,
        "message": {
          "role": "assistant",
          "content": "Zwischenzeugnis Text hier..."
        },
        "finish_reason": "stop"
      }
    ],
    "usage": {
      "completion_tokens": 500,
      "prompt_tokens": 200,
      "total_tokens": 700
    }
  }
}
```

### 1.5 Action testen

1. Gehe zum **Test** Tab
2. Unter Connectivity: Wähle **Destination** und dann `AI_Core`
3. Im **Test Run** Bereich sollte bereits ein JSON-Body stehen. Falls nicht, füge den folgenden Test-Body ein:

```json
{
  "config_ref": {
    "name": "hr-zwischenzeugnis-config",
    "version": "0.0.1",
    "scenario": "orchestration"
  },
  "placeholder_values": {
    "position": "Senior Consultant SAP BTP",
    "abteilung": "IT Consulting",
    "bewertung": "sehr gut",
    "stichpunkte": "Einführung SAP Integration Suite, Teamleitung, Kundenprojekte bei DAX-Unternehmen",
    "eintrittsdatum": "01.03.2022",
    "mitarbeiter_name": "Max Mustermann"
  }
}
```

4. Klick **Test**
5. Warte auf die Antwort (kann 5-10 Sekunden dauern)
6. Du solltest **200 OK** und ein generiertes Zwischenzeugnis im Response Body sehen

> **Wichtig:** Falls du **404 Not Found** bekommst, prüfe den URL Prefix (Schritt 1.2). Falls du **400 Bad Request** bekommst, prüfe den Request Body.

### 1.6 Action speichern und releasen

1. Klick oben rechts auf **Save**
2. Klick auf **Release**
3. Die Action ist jetzt in der Action Library verfügbar und kann im Prozess verwendet werden

---

## Schritt 2: Projekt und Prozess anlegen (5 Min)

1. Zurück in der **SAP Build Lobby**: Klick **Create > Build an Automated Process > Process**
2. Projektname: `HR Zwischenzeugnis - [DEIN NAME]`
3. Prozessname: `Zeugnis-Prozess`
4. Bestätige mit **Create**

---

## Schritt 3: Custom Variable anlegen (2 Min)

1. Klick auf den **Prozess-Hintergrund** (nicht auf einen Step)
2. Rechts im Side Panel unter **Variables**: Klick **Configure** oder **+**
3. Neue Variable erstellen:
   - Name: `zeugnisText`
   - Type: `String`
4. Bestätige und schliesse den Dialog

---

## Schritt 4: Trigger-Formular erstellen (10 Min)

1. Klick auf den **Trigger**-Block im Prozess
2. Wähle **Blank Form**, Name: `Antragsformular_Zwischenzeugnis`
3. Das Formular oeffnet sich in einem neuen Tab

### Formular gestalten:

4. Ziehe ein **Dropdown**-Feld auf das Formular
5. Konfiguriere rechts:
   - Label: `Mitarbeiter auswählen`
   - **Data to display**: Wähle `Data Source`
   - Data Source: `get_Employees`
   - Destination Variable: `S4_Destination`
   - Available Data: `_name`
   - Haken bei **Required**
6. **Save**

---

## Schritt 5: Employee-Daten holen (5 Min)

1. Zurück im Prozess: Klick auf **+** nach dem Trigger
2. Wähle **Action > Browse Library**
3. Suche nach `Get entities from Employees` und füge sie hinzu
4. Konfiguration rechts:
   - **General** Tab: Destination Variable = `S4_Destination`
   - **Inputs** Tab: `$search` = Formularfeld `_name` (vom Trigger)

---

## Schritt 6: Approval-Formular erstellen (15 Min)

1. Klick auf **+** nach der Employee-Action
2. Wähle **Approval > Blank Approval**
3. Name: `Zwischenzeugnis Approval`

### Formular gestalten:

4. Füge folgende Felder hinzu:

| Feldtyp | Label | Read Only? |
|---|---|---|
| Text | Name | Ja |
| Text | Position | Ja |
| Text | Abteilung | Ja |
| Text | Eintrittsdatum | Ja |
| Text Area | Besondere Leistungen / Stichpunkte | Nein (editierbar!) |
| Dropdown (Manual) | Bewertung | Nein (editierbar!) |

5. Für das **Bewertung**-Dropdown: Wähle rechts `Manual` und füge Optionen hinzu:
   - `sehr gut`
   - `gut`
   - `befriedigend`
   - `ausreichend`
6. **Save**

### Approval Step konfigurieren:

7. Zurück im Prozess, klick auf den Approval Step
8. **General** Tab:
   - Subject: `Zwischenzeugnis genehmigen`
   - Recipients > Users: `Started By`
9. **Inputs** Tab - Mappe die Felder:
   - Name = `_name` (von Get Employees)
   - Abteilung = `department` (von Get Employees)
   - Eintrittsdatum = `startDate` (von Get Employees)
   - Besondere Leistungen = `specialAchievements` (von Get Employees)

---

## Schritt 7: GenAI Hub Action einbinden (10 Min)

1. Im **Approve**-Zweig: Klick auf **+**
2. Wähle **Action > Browse Library**
3. Suche nach `AI Core Orchestration` (eure eigene Action aus Schritt 1)

### General Tab:
- Destination Variable: `AI_Core`

### Inputs Tab:
- **config_ref** (Single Properties):
  - name: `hr-zwischenzeugnis-config`
  - scenario: `orchestration`
  - version: `0.0.1`
- **placeholder_values** (Single Properties):
  - mitarbeiter_name = Approval-Output `Name`
  - position = Approval-Output `Position`
  - abteilung = Approval-Output `Abteilung`
  - eintrittsdatum = Approval-Output `Eintrittsdatum`
  - stichpunkte = Approval-Output `Stichpunkte`
  - bewertung = Approval-Output `Bewertung`

### Outputs Tab:
4. Unter **Set Custom Variables**: Wähle die Variable `zeugnisText`
5. Mappe sie auf: `content` (unter result > final_result > choices > message > content)

> **Das ist der Trick:** Statt eines Script Tasks nutzen wir das Output-Mapping der Action, um den generierten Zeugnis-Text direkt in unsere Custom Variable zu schreiben. Kein Code nötig!

---

## Schritt 8: Output-Formular erstellen (10 Min)

1. Klick auf **+** nach der GenAI Action
2. Wähle **Form > Blank Form**
3. Name: `Zwischenzeugnis Output`

### Formular gestalten:

4. Füge eine **Überschrift (H1)** hinzu: `Zwischenzeugnis erfolgreich erstellt`
5. Füge ein **Text Area**-Feld hinzu:
   - Label: `Generiertes Zeugnis`
   - Haken bei **Read Only**
6. **Save**

### Output Step konfigurieren:

7. Zurück im Prozess:
   - **General** Tab:
     - Subject: `Zwischenzeugnis erfolgreich erstellt`
     - Recipients > Users: `Started By`
   - **Inputs** Tab:
     - Text Area = Custom Variable `zeugnisText`

---

## Schritt 9: E-Mail versenden (5 Min)

1. Klick auf **+** nach dem Output-Formular
2. Wähle **Mail**
3. Konfiguriere:
   - **An**: `Started By` > Email
   - **Betreff**: `Ihr Zwischenzeugnis wurde erstellt`
   - **Mail Body**: Klick auf `Edit Mail Body`
     - Schreibe: `Das Zwischenzeugnis ist fertig.`
     - Neue Zeile: `Zwischenzeugnis:`
     - Ziehe aus den **Custom Variables** links die Variable `zeugnisText` in den Body
   - Klick **Apply**

---

## Schritt 10: Prozess abschließen (3 Min)

1. Stelle sicher, dass beide Zweige (Approve und Reject) in einem **End**-Event enden
2. Klick oben rechts auf **Save**
3. Prüfe, dass der Status **Saved** (gruen) angezeigt wird

---

## Schritt 11: Release und Deploy (10 Min)

### Release:

1. Klick oben rechts auf **Release**
2. Falls ein Fehler kommt (`artifact status not valid`):
   - Öffne das genannte Artefakt (Tab oben)
   - Speichere es erneut
   - Versuche den Release nochmal
3. Nach erfolgreichem Release siehst du eine Versionsnummer

### Deploy:

4. Klick auf **Deploy**
5. Konfiguriere die **Destination Variables**:
   - `AI_Core`: Wähle die BTP Destination `AI_Core`
   - `S4_Destination`: Wähle die BTP Destination für den Employee Service
6. Bestätige mit **Deploy**

---

## Schritt 12: End-to-End Test (15 Min)

1. Öffne **My Inbox** in der SAP Build Lobby
2. Starte den Prozess ueber das Trigger-Formular
3. **Trigger-Formular**:
   - Wähle einen Mitarbeiter aus dem Dropdown
   - Klick **Submit**
4. **Inbox - Genehmigung**:
   - Öffne den Eintrag `Zwischenzeugnis genehmigen`
   - Prüfe die vorausgefüllten Mitarbeiterdaten
   - Ergaenze Stichpunkte (z.B. `SAP Integration Suite, Teamleitung, Kundenprojekte`)
   - Wähle eine Bewertung (z.B. `sehr gut`)
   - Klick **Approve**
5. **Warte ca. 5-10 Sekunden** (GenAI Hub generiert das Zeugnis)
6. **Inbox - Output**:
   - Öffne den Eintrag `Zwischenzeugnis erfolgreich erstellt`
   - Das generierte Zeugnis wird im Text Area angezeigt
7. **E-Mail prüfen** (optional)

---

## Troubleshooting

| Problem | Lösung |
|---|---|
| **404 Not Found** bei der GenAI Action | URL Prefix prüfen (Schritt 1.2). Ist das Deployment RUNNING? |
| **400 Bad Request** | Request Body prüfen. Stimmt der config_ref Name/Version? |
| **config_ref nicht gefunden** | Name, Scenario und Version der Orchestration Config prüfen |
| **Release: artifact status not valid** | Jedes Artefakt einzeln oeffnen und speichern |
| **Leeres Zeugnis im Output** | Outputs Tab der Action prüfen - ist `zeugnisText` auf `content` gemappt? |
| **Dropdown zeigt keine Mitarbeiter** | Destination Variable `S4_Destination` prüfen |
| **Action nicht in Library** | Action muss erst released sein (Schritt 1.6) |

---

## Zusammenfassung

In diesem Workshop habt ihr gelernt:

1. Wie man eine **externe REST API** (GenAI Hub) als Action in SBPA einbindet
2. Wie man in **SAP Build Process Automation** einen End-to-End Workflow baut
3. Wie man **Formulare** mit Datenbindung an externe Services erstellt
4. Wie man mit **config_ref** eine gespeicherte Orchestration Config referenziert
5. Wie man mit **Custom Variables und Output Mapping** LLM-Ergebnisse weiterverarbeitet
6. Wie man den Workflow **testet, released und deployed**

---

## Weitergehende Themen (Ausblick)

- **Eigenen Prompt erstellen**: Im GenAI Hub eigene Templates und Orchestration Configs bauen
- **Structured Output (json_schema)**: JSON-Output für programmatische Weiterverarbeitung
- **Content Filtering**: Azure Content Safety für Input/Output Filterung
- **Data Masking**: PII-Anonymisierung vor der LLM-Verarbeitung
- **Document Grounding**: RAG mit eigenen Firmendokumenten
- **PDF-Generierung**: Zeugnis als formatiertes PDF ausgeben
