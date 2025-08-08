# 🛠️ P3D Postprozessor-Plugin-System – Entwicklerdokumentation

## 📌 Ziel dieser Anleitung

Diese Dokumentation richtet sich an Entwickler mit C#-Erfahrung, die ein Plugin für das **P3D CAM-System** schreiben möchten. Sie beschreibt die Struktur, Schnittstellen und Konzepte des **Postprozessor-Systems**, sodass du **ohne Beispielcode** direkt loslegen kannst.

---

## 📁 Projektstruktur

Ein Postprozessor besteht aus einer .NET-Klasse, die das Interface `IP3DPostProcessor` implementiert. Diese Klasse wird **per Reflection** aus einer externen DLL geladen und verarbeitet APT-kompatiblen Code aus P3D zu fertigem G-Code.

### Anforderungen:
- Sprache: **C# (.NET 4.8 oder höher empfohlen)**
- Ausgabe: **DLL** mit mindestens einer Klasse, die `IP3DPostProcessor` implementiert
- Abhängigkeiten: Referenz auf die `P3DPP.dll` (enthält Interfaces und Basisklassen)

---

## 🔌 Interface: `IP3DPostProcessor`

Folgende Methoden muss dein Postprozessor implementieren:

| Methode       | Wird aufgerufen bei …            | Zweck                                               |
|---------------|----------------------------------|------------------------------------------------------|
| `Initialize`  | Start der Verarbeitung           | Setup, Benutzerabfragen, Register definieren        |
| `Fedrat`      | `FEDRAT` Anweisung               | Feedrate (Vorschub) setzen                          |
| `Goto`        | `GOTO` Anweisung                 | Bewegungskommandos erzeugen                         |
| `Rapid`       | `RAPID` Anweisung                | Schnelle Bewegung aktivieren                        |
| `PPrint`      | `PPRINT` Kommentar                | Kommentar in G-Code schreiben                       |
| `MultAx`      | `MULTAX` Maschinenmodus          | Zustand für Mehrachsenbetrieb setzen                |
| `Done`        | Ende der Verarbeitung            | G-Code manipulieren, Nachbearbeitung                |

---

## 🧰 Das `Register`-System

Die zentrale Komponente zur G-Code-Generierung ist das **`Register`-System**. Du arbeitest nicht direkt mit Strings, sondern setzt Werte für benannte "Register", die dann als G-Code zusammengesetzt werden.

### 🔧 Typische Befehle:

| Funktion                        | Beschreibung                                          |
|---------------------------------|-------------------------------------------------------|
| `RegisterAdd(name, ...)`        | Neuen Register (z. B. "X") definieren                 |
| `RegisterSetValue(name, value)` | Wert in ein Register schreiben                       |
| `OutBlock()`                    | Aktuelle Registerwerte als G-Code-Zeile ausgeben     |
| `ShouldOutblock(params)`        | Gibt true zurück, wenn Registerwerte sich geändert haben |
| `StartAutoLineNumber(...)`      | Automatische Zeilennummern wie `N10`, `N20`, ...     |
| `Output(string)`                | Rohtext direkt in die Ausgabe schreiben              |
| `RegisterGetCode()`             | Gesamten generierten G-Code abrufen (array of strings) |
| `RegisterSetCode(lines)`        | G-Code nachbearbeiten                                |

---

## 🧩 Datenzugriff: `CLData`

Alle Methoden wie `Fedrat(CLData data)` erhalten ein `CLData`-Objekt, das alle **Parameter der APT-Anweisung** enthält.

### Zugriff:
```csharp
var feedrate = data["F"];         // Zugriff auf numerische Werte
var comment = data["Comment"];    // Zugriff auf Strings
```

**Typische Schlüssel:**
- `X`, `Y`, `Z`
- `F` (Feedrate)
- `Comment` (bei `PPRINT`)
- `State` (bei `MULTAX`)

---

## 📋 Benutzerabfragen: `FormInput`

Du kannst beim Start über ein modales Dialogfenster Benutzereingaben abfragen. Die Eingaben werden als Dictionary zurückgegeben.

### Schritte:
1. Erstelle eine Liste von `InputFormParameter`
2. Übergib sie an `FormInput.TryGetValues(...)`
3. Werte das Ergebnis aus

### Unterstützte Eingabetypen:
- `Text` (freier Text)
- `Integer` (Ganzzahl)
- `Float` (Gleitkomma)
- `Checkbox` (bool)
- `ComboBox` (Auswahl aus Liste)

---

## 🔄 Ablauf beim Ausführen

P3D ruft dein Plugin automatisch wie folgt auf:

1. `Initialize(CLData data)`
2. Für jede APT-Anweisung:
    - Z. B. `Fedrat(data)`, `Goto(data)` etc.
3. `Done()` nach der letzten Zeile

---

## ✅ Mindestanforderungen an dein Plugin

Damit dein Plugin funktioniert, musst du:
- Eine öffentliche Klasse mit Standard-Konstruktor haben
- Diese Klasse muss `IP3DPostProcessor` implementieren
- Du musst `Initialize()` korrekt implementieren
- Mindestens ein Register definieren (z. B. `X`, `Y`, `Z`)

---

## 💡 Erweiterungsideen

Du kannst dein Plugin beliebig erweitern:
- Eigene State-Variablen (z. B. für Modi oder Flags)
- Zusatzlogik wie Werkzeugwechsel, Sicherheitsabfragen
- Erkennung von Sonderbewegungen (z. B. Helix)
- Ausgabe von Maschinen-spezifischem G-Code

---

## 🧪 Debugging-Tipps

- Verwende `MessageBox.Show(...)` zum Debuggen
- Gib Zwischenschritte als Kommentare per `Output()` aus
- Manipuliere den G-Code in `Done()` bevor er gespeichert wird

---

## 📦 Deployment

- Erzeuge eine `.dll` mit deiner Plugin-Klasse
- Platziere die DLL im Plugin-Ordner von P3D
- Starte P3D – dein Plugin wird automatisch geladen
- P3D sucht nach Klassen mit `IP3DPostProcessor` Interface

---

## 📘 Fazit

Das Postprozessor-System in P3D ist flexibel und modular. Durch die Kombination aus Registersystem, Benutzerabfragen und Datenmanipulation in `Done()` kannst du jede Art von Maschinenpostprozessor für G-Code, Heidenhain, Fanuc, etc. erzeugen.

```csharp
// Beispiel-Struktur (nur zur Orientierung)
public class MyPostprocessor : IP3DPostProcessor
{
    public void Initialize(CLData data) { /* Setup */ }
    public void Fedrat(CLData data) { /* Vorschub */ }
    public void Goto(CLData data) { /* Bewegung */ }
    public void Done() { /* Nachbearbeitung */ }
}
```

> Bei richtiger Nutzung brauchst du **keine G-Code-Strings selbst formatieren** – das übernimmt das Registersystem vollständig für dich.

---

## 🏢 Lizenz / Hinweise

Dieses System ist Teil von **P3D** (© Datentechnik Reitz GmbH & Co. KG).  
Verwendung nur im Rahmen der P3D CAM Software.

