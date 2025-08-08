# 📄 P3D Postprozessor System – Dokumentation

## 📚 Inhalt

- [Einleitung](#einleitung)
- [Systemarchitektur](#systemarchitektur)
- [Projektstruktur](#projektstruktur)
- [Postprozessor Interface (IP3DPostProcessor)](#postprozessor-interface-ip3dpostprocessor)
- [Register-System](#register-system)
- [APT Parser](#apt-parser)
- [Dynamische Eingabe-Formulare](#dynamische-eingabe-formulare)
- [Postprozessor-Beispiel (PostProcessor.cs)](#postprozessor-beispiel-postprocessorcs)
- [Test & Ausführung](#test--ausführung)
- [Erweiterungsideen](#erweiterungsideen)

---

## Einleitung

Das P3D-Postprozessor-System ist ein flexibles und erweiterbares Framework zum Ausführen von benutzerdefinierten Postprozessoren auf Basis einer APT-ähnlichen Eingabedatei. Es unterstützt dynamisches Parsen, eine Plugin-basierte Architektur sowie eine UI-basierte Parametereingabe.

---

## Systemarchitektur

```
+-----------------------+
|       P3D_PP_Executor |
+----------+------------+
           |
           | lädt DLL (Postprozessor Plugin)
           v
+----------+------------+
|  IP3DPostProcessor    |
|  (PostProcessor.cs)   |
+----------+------------+
           |
           v
+----------+------------+
|   Register-System     |
+-----------------------+
           |
           v
+-----------------------+
|  APT-Parser           |
+-----------------------+
           |
           v
+-----------------------+
|  Ausgabe: G-Code TXT  |
+-----------------------+
```

---

## Projektstruktur

- `P3D_PP_Executor`  
  Hauptprogramm, das APT-Dateien einliest, DLL lädt und den Postprozessor ausführt.

- `P3D_Postprocessors`  
  Enthält das zentrale `IP3DPostProcessor`-Interface, das Register-System und Hilfsklassen wie `CLData`.

- `P3DPP`  
  Dynamische Eingabeformulare über `FormInput` mit Unterstützung für verschiedene Eingabetypen.

---

## Postprozessor Interface (`IP3DPostProcessor`)

```csharp
public interface IP3DPostProcessor
{
    void Initialize(CLData data);
    void Goto(CLData data);
    void Fedrat(CLData data);
    void Rapid(CLData data);
    void PPrint(CLData data);
    void MultAx(CLData data);
    void Done();
}
```

Diese Methoden werden je nach APT-Befehl aufgerufen. Alle Argumente werden über `CLData` übergeben.

---

## Register-System

Zentrale Komponente für das Erstellen von G-Code Zeilen. Bietet:

### Beispiel: Register initialisieren
```csharp
RegisterAdd("X", "X", decimalPlaces: 4, trailingZeros: 5);
RegisterAdd("F", "F");
```

### Werte setzen
```csharp
RegisterSetValue("X", 12.3456);
```

### Outblock erzeugen
```csharp
OutBlock();
```

### G-Code abrufen oder überschreiben
```csharp
string[] code = RegisterGetCode();
RegisterSetCode(code);
```

---

## APT Parser

Das APT-ähnliche Format wird in `AptCommand`-Objekte umgewandelt:

### Unterstützte Befehle:

- `GOTO / x, y, z`
- `FEDRAT / value`
- `RAPID`
- `PPRINT / 'comment'`
- `MULTAX / ON|OFF`

### Beispiel:
```text
PPRINT / 'Hello World'
FEDRAT / 500
GOTO / 12.5, 25.7, -1.0
```

Jeder Befehl wird in ein `AptCommand`-Objekt mit Argument-Dictionary übersetzt.

---

## Dynamische Eingabeformulare

Die Klasse `FormInput` erlaubt benutzerdefinierte Eingaben über ein dynamisch generiertes Formular.

### Unterstützte Typen (`InputType`)
- `Text`
- `Checkbox`
- `Integer`
- `Float`
- `ComboBox`

### Beispiel-Aufruf:
```csharp
var parameters = new List<InputFormParameter>
{
    new InputFormParameter { Name = "Speed", Type = InputType.Float, Value = 100.0f },
    new InputFormParameter { Name = "Enable Feature", Type = InputType.Checkbox, Value = true },
    new InputFormParameter { Name = "Mode", Type = InputType.ComboBox, Value = "Fast", DataSource = new[] { "Fast", "Slow" } }
};

if (FormInput.TryGetValues("Settings", parameters, out var values)) {
    // Verarbeitung...
}
```

---

## Postprozessor-Beispiel (`PostProcessor.cs`)

Ein Beispiel eines konkreten Postprozessors.

### Wichtige Eigenschaften:

- `_multAx` und `_isRapid` speichern Zustand
- `GetInputValues()` fragt UI-Parameter beim Start ab
- `Initialize()` initialisiert Register und Kopfzeile
- `Goto()`, `Fedrat()`, `Rapid()` etc. setzen Register und erzeugen G-Code-Zeilen

### Beispiel OutBlock:
```csharp
RegisterSetValue("X", data["X"]);
RegisterSetValue("G", 1, true);
OutBlock();
```

---

## Test & Ausführung

### APT-Testdatei
```text
PPRINT / 'Start'
FEDRAT / 500
GOTO / 12.5, 25.7, -1.0
RAPID
GOTO / 30.0, 50.0, 10.0
```

### Programmstart (Testmode):
```csharp
var ppPath = "../../../TestPP/bin/Debug/TestPP.dll";
var aptPath = "../../../p3dAPT.txt";
var outPutPath = "../../../test.txt";
```

Die Ausgabe wird in `test.txt` geschrieben.

---

## Erweiterungsideen

- [ ] Unterstützung für weitere APT-Befehle (`CIRCLE`, `SPINDL`, etc.)
- [ ] Validierung von FormInput-Feldern (z. B. Pflichtfelder)
- [ ] Presets für Eingabemasken
- [ ] Erweiterte G-Code Templates (z. B. Tool-Change Handling)
- [ ] Plugin-Auswahl bei mehreren Typen in DLL
- [ ] Unterstützung für mehr als einen Postprozessor pro DLL

---

## Lizenz / Info

© Datentechnik Reitz GmbH & Co. KG  
Dieses System ist intern für P3D Postprozessor-Verarbeitung gedacht.
