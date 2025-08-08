# P3D Postprozessor System – Entwicklerdokumentation

Dieses Dokument beschreibt das generische Postprozessor-System von P3D, mit dem du APT-Code (generischer Maschinen-Code) in maschinenspezifischen Code (z. B. G-Code) umwandeln kannst. Die Anleitung richtet sich an Entwickler mit C#-Kenntnissen, die ohne Vorwissen direkt eigene Postprozessoren erstellen möchten.

---

## Überblick

Das P3D Postprozessor-System stellt eine Schnittstelle (`IP3DPostProcessor`) bereit, die vom Postprozessor-Plugin implementiert wird. Die Aufgabe des Postprozessors ist es, die generischen Bewegungs- und Steuerdaten (APT-Code) in den spezifischen Maschinenbefehl umzuwandeln.

Die Ausgabe erfolgt zeilenweise mit einem internen Register-System, das formatierte Steuerzeichen (z. B. `X123.456`, `G01`, `F300`) effizient verwaltet.

---

## 1. Grundstruktur eines Postprozessors

Ein Postprozessor wird als C#-Klasse implementiert, die das Interface `IP3DPostProcessor` erfüllt. Die wichtigsten Methoden sind:

| Methode        | Zweck                                                       |
|----------------|-------------------------------------------------------------|
| `Initialize`   | Initialisierung zu Beginn der Verarbeitung (Parameter, Setup) |
| `Fedrat`       | Übergabe von Vorschubwerten                                 |
| `Goto`         | Lineare Bewegungsbefehle                                    |
| `Rapid`        | Schnelle Positionierungsbewegungen                         |
| `PPrint`       | Ausgabe von Kommentaren                                     |
| `MultAx`       | Steuerung von Mehr-Achsen-Modi                             |
| `Done`         | Abschluss der Verarbeitung (Nachbearbeitung)               |

---

## 2. Umgang mit Eingaben vom Benutzer

Das System stellt eine flexible Eingabemaske (`FormInput`) zur Verfügung, mit der du Parameter vom Benutzer zur Laufzeit abfragen kannst:

- Unterstützte Eingabetypen: Text, Checkbox, Integer, Float, ComboBox
- Beispielaufruf:

```csharp
var parameters = new List<InputFormParameter>
{
    new InputFormParameter { Name = "Username", Type = InputType.Text, Value = "" },
    new InputFormParameter { Name = "UseCooling", Type = InputType.Checkbox, Value = false },
    new InputFormParameter { Name = "Speed", Type = InputType.Integer, Value = 1000 },
    new InputFormParameter { Name = "Precision", Type = InputType.Float, Value = 0.01f },
    new InputFormParameter { Name = "ToolType", Type = InputType.ComboBox, Value = "Drill", DataSource = new string[] { "Drill", "Mill", "Lathe" } }
};

if (FormInput.TryGetValues("Parameter Eingabe", parameters, out Dictionary<string, object> result))
{
    // Werte aus 'result' verwenden
}
```

---

## 3. Registersystem: Kern der Codegenerierung

Die Ausgabe von Maschinen-Code erfolgt über das Registersystem. Es ermöglicht die deklarative Definition von Steuerzeichen und automatisiert Formatierung, Wertvergleich und Zeilenausgabe.

### 3.1 Register anlegen (`RegisterAdd`)

```csharp
void RegisterAdd(string name, string outputName, int decimalPlaces = 3, int leadingZeros = 0, int trailingZeros = 0)
```

- `name`: Interner Name für das Register (z. B. `"X"`, `"G"`)
- `outputName`: Kürzel im Maschinen-Code (z. B. `"X"`, `"G"`)
- `decimalPlaces`: Anzahl Dezimalstellen bei Zahlen (z. B. `4` für `X123.4567`)
- `leadingZeros`: Mindestanzahl führender Stellen vor Komma
- `trailingZeros`: Mindestanzahl Nachkommastellen, wird mit Nullen aufgefüllt

### 3.2 Werte setzen (`RegisterSetValue`)

```csharp
void RegisterSetValue(string registerName, object value, bool force = false)
```

- `registerName`: Name des Registers
- `value`: Wert, z. B. `float`, `int` oder `string`
- `force`: Erzwingt Ausgabe auch wenn Wert unverändert

### 3.3 Prüfen auf Änderungen (`ShouldOutblock`)

```csharp
bool ShouldOutblock(params string[] registerNames)
```

Gibt `true` zurück, wenn sich einer der angegebenen Registerwerte seit der letzten Ausgabe geändert hat.

### 3.4 Zeile ausgeben (`OutBlock`)

```csharp
void OutBlock()
```

Schreibt eine Zeile mit allen geänderten Registern in den Maschinen-Code.

### 3.5 Automatische Zeilennummern (`StartAutoLineNumber`)

```csharp
void StartAutoLineNumber(char lineNumberSymbol, int startNum, int interval, int maxNum)
```

Erzeugt bei jeder Ausgabe automatisch Zeilennummern (z. B. `N10`, `N20`, ...).

### 3.6 Zugriff auf generierten Code

- `string[] RegisterGetCode()` – Holt alle bisher generierten Codezeilen.
- `void RegisterSetCode(string[] code)` – Setzt bzw. überschreibt die gesamte Codeausgabe.

---

## 4. Verwendung von Wildcards (Platzhaltern) in Registern

Das Registersystem unterstützt sogenannte Wildcards (Platzhalter) in den Ausgabetexten der Register. Beispielsweise kannst du ein Register mit einem Ausgabeformat definieren, das Variablen enthält, welche bei der Ausgabe durch den aktuellen Registerwert ersetzt werden.

Beispiel:

```csharp
// Register mit Wildcard {@} definieren
RegisterAdd("I", "I=AC({@})");
```

- Hier wird `{@}` durch den aktuellen Wert des Registers `I` ersetzt.
- Wenn du also `RegisterSetValue("I", 5)` aufrufst, wird im Maschinen-Code `I=AC(5)` ausgegeben.
- Dies erlaubt flexible und komplexe Formate direkt im Registerausgabe-String.

---

## 5. Beispielablauf eines Befehls

```csharp
public void Goto(CLData data)
{
    RegisterSetValue("X", data["X"]);
    RegisterSetValue("Y", data["Y"]);
    RegisterSetValue("Z", data["Z"]);

    if (ShouldOutblock("X", "Y", "Z"))
    {
        RegisterSetValue("G", 1); // G1 – lineare Bewegung
        OutBlock();               // Ausgabe der Befehlszeile
    }
}
```

---

## 6. Zusätzliche Tipps

- Nutze `Output("...")` für Kommentare oder Befehle, die nicht im Register-System abgebildet sind.
- Achte darauf, Register nur bei Wertänderungen auszugeben, um sauberen und kompakten Code zu erzeugen.
- Verwalte globale Zustände (z. B. ob gerade Rapidbewegung aktiv ist) in Feldern der Postprozessor-Klasse.
- Nutze die `Done()`-Methode zur Nachbearbeitung des gesamten Codes vor der finalen Ausgabe.

---

## 7. Zusammenfassung der wichtigsten Methoden

| Methode              | Zweck                                               |
|----------------------|----------------------------------------------------|
| `Initialize(CLData)` | Setup & Eingabe, initialisieren von Registern      |
| `Fedrat(CLData)`     | Vorschub setzen                                    |
| `Goto(CLData)`       | Linearbewegung umsetzen                             |
| `Rapid(CLData)`      | Schnelle Positionierung                             |
| `PPrint(CLData)`     | Kommentar ausgeben                                  |
| `MultAx(CLData)`     | Mehr-Achsen-Modus setzen                            |
| `Done()`             | Abschlussarbeiten, Code final bearbeiten            |

---

## 8. Verwendete Typen

- **`CLData`**: Container mit Positions- und Zustandsinformationen der aktuellen Befehlszeile.
- **`InputFormParameter`**: Definiert Parameter für dynamische Eingabemasken.
- **`InputType`**: Enum mit Typen für Eingabefelder (Text, Checkbox, Integer, Float, ComboBox).

---

# Schlusswort

Das P3D Postprozessor-System bietet eine flexible und robuste Basis für maschinenspezifische Codeerzeugung. Durch das Registersystem mit Unterstützung von Wildcards und das modulare Interface können Entwickler mit C#-Erfahrung schnell eigene Postprozessoren für verschiedenste Maschinentypen schreiben.

Für Fragen oder Erweiterungen empfiehlt es sich, den Beispielcode eingehend zu studieren und die Methoden schrittweise zu implementieren.

---

*Ende der Dokumentation*
