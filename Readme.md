# Dokumentation: Erstellung eines Postprozessors für P3D

Diese Dokumentation beschreibt die Schritte und Konzepte, die zum Erstellen eines benutzerdefinierten Postprozessors für den P3D-Slicer erforderlich sind. Postprozessoren sind entscheidend, um die vom Slicer generierten Rohdaten (CLData) in spezifischen G-Code zu übersetzen, der von Ihrer CNC-Maschine verstanden wird.

## Grundlagen

Ein P3D-Postprozessor ist eine .NET-Klassenbibliothek (DLL), die die Schnittstelle `IP3DPostProcessor` implementiert. Diese Schnittstelle definiert verschiedene Methoden, die während des Postprozessierungsprozesses vom P3D-Slicer aufgerufen werden.

Die Kommunikation zwischen dem Slicer und Ihrem Postprozessor erfolgt über das `CLData`-Objekt. Dieses Objekt ist ein Dictionary, das Informationen über die aktuelle Operation (z. B. Bewegung, Vorschub, Kommentar) enthält. Die Schlüssel und Werte in diesem Dictionary hängen von der Art der Operation ab.

Der Postprozessor verwendet das statische `Register`-System, um G-Code-Wörter und deren Werte zu verwalten. Dieses System ermöglicht es, G-Code-Befehle schrittweise aufzubauen und effizient auszugeben.

## Schritte zur Erstellung eines Postprozessors

1.  **Erstellen Sie ein neues .NET-Klassenbibliotheksprojekt:**
    * Öffnen Sie Visual Studio (oder eine andere geeignete .NET-Entwicklungsumgebung).
    * Erstellen Sie ein neues Projekt vom Typ "Klassenbibliothek (.NET Framework)" oder "Klassenbibliothek (.NET Standard)". Es wird empfohlen, .NET Framework zu verwenden, um Kompatibilität zu gewährleisten.
    * Geben Sie dem Projekt einen aussagekräftigen Namen, z. B. `P3DPostProcessor_MyMachine`.

2.  **Fügen Sie die P3D-Postprozessor-DLL als Referenz hinzu:**
    * Klicken Sie im Projektmappen-Explorer mit der rechten Maustaste auf "Abhängigkeiten" oder "Referenzen" und wählen Sie "Referenz hinzufügen".
    * Navigieren Sie zum Speicherort der `P3D_Postprocessors.dll` (die Ihnen vom P3D-Team bereitgestellt wurde) und wählen Sie sie aus.
    * Klicken Sie auf "Hinzufügen".

3.  **Erstellen Sie Ihre Postprozessor-Klasse:**
    * Fügen Sie Ihrem Projekt eine neue C#-Klassendatei hinzu (falls noch keine vorhanden ist). Nennen Sie sie z. B. `PostProcessor.cs`.
    * Implementieren Sie die `IP3DPostProcessor`-Schnittstelle in Ihrer Klasse:

    ```csharp
    using P3D_Postprocessors;
    using static P3D_Postprocessors.Register;
    using System;

    namespace P3DPostProcessor_MyMachine // Ersetzen Sie dies durch Ihren Namespace
    {
        public class PostProcessor : IP3DPostProcessor
        {
            // Implementieren Sie die Methoden der IP3DPostProcessor-Schnittstelle hier
            public void Initialize() { /* ... */ }
            public void Goto(CLData data) { /* ... */ }
            public void Fedrat(CLData data) { /* ... */ }
            public void Rapid(CLData data) { /* ... */ }
            public void PPrint(CLData data) { /* ... */ }
            public void MultAx(CLData data) { /* ... */ }
            public void Done() { /* ... */ }
        }
    }
    ```

4.  **Implementieren Sie die `Initialize()`-Methode:**
    * Diese Methode wird zu Beginn des Postprozessierungsprozesses aufgerufen.
    * Verwenden Sie `RegisterAdd()`, um die G-Code-Wörter zu definieren, die Ihr Postprozessor verwenden wird (z. B. "G", "X", "Y", "Z", "F"). Sie können auch Formatierungsoptionen wie führende/nachfolgende Nullen und das Vorzeichenverhalten festlegen.
    * Geben Sie hier typischerweise den Header des G-Code-Programms aus (z. B. Kommentare mit Informationen zum Generator und Datum). Verwenden Sie `Output()` dafür.

    ```csharp
    public void Initialize()
    {
        // G-Code-Wörter registrieren
        RegisterAdd("G", "G", 1);
        RegisterAdd("X", "X", trailingZeros: 3);
        RegisterAdd("Y", "Y", trailingZeros: 3);
        RegisterAdd("Z", "Z", trailingZeros: 3);
        RegisterAdd("F", "F");

        // Header ausgeben
        Output($"; Code generiert mit P3D Postprozessor für Meine Maschine");
        Output($"; Datum: {DateTime.Now.ToString("dd.MM.yyyy HH:mm:ss")}");
        Output(";");
    }
    ```

5.  **Implementieren Sie die Bewegungsbefehle (`Goto()`, `Rapid()`):**
    * **`Goto(CLData data)`:** Wird für interpolierte Bewegungen (in der Regel G1) aufgerufen.
        * Extrahieren Sie die Zielkoordinaten (X, Y, Z) aus dem `data`-Objekt mithilfe des Indexers (z. B. `data["X"]`).
        * Verwenden Sie `RegisterSetValue()`, um die Werte für die entsprechenden G-Code-Wörter ("X", "Y", "Z") zu setzen.
        * Verwenden Sie `ShouldOutblock()`, um zu prüfen, ob sich mindestens einer der registrierten Werte seit dem letzten `OutBlock()`-Aufruf geändert hat. Wenn ja, geben Sie einen neuen G-Code-Block aus.
        * Setzen Sie den `G`-Befehl auf `1` (für Vorschubbewegung) mithilfe von `RegisterSetValue("G", 1, true)`. Das `true`-Argument erzwingt, dass "G1" im nächsten `OutBlock()` ausgegeben wird, falls eine Bewegung stattfindet.
        * Rufen Sie `OutBlock()` auf, um die akkumulierten G-Code-Wörter als eine Zeile auszugeben.

    * **`Rapid(CLData data)`:** Wird für Eilgangbewegungen (G0) aufgerufen.
        * Ähnlich wie `Goto()`, extrahieren Sie die Zielkoordinaten und setzen Sie die entsprechenden Registerwerte.
        * Setzen Sie den `G`-Befehl auf `0` (für Eilgang) mithilfe von `RegisterSetValue("G", 0, true)`.
        * Rufen Sie `OutBlock()` auf.

    ```csharp
    private bool _isRapid = false; // Hilfsvariable, um den Bewegungsmodus zu verfolgen

    public void Goto(CLData data)
    {
        RegisterSetValue("X", data["X"]);
        RegisterSetValue("Y", data["Y"]);
        RegisterSetValue("Z", data["Z"]);

        if (ShouldOutblock("X", "Y", "Z"))
        {
            RegisterSetValue("G", _isRapid ? 0 : 1, true); // G0 für Rapid, G1 für Goto
            OutBlock();
        }
    }

    public void Rapid(CLData data)
    {
        _isRapid = true;
        RegisterSetValue("X", data["X"]);
        RegisterSetValue("Y", data["Y"]);
        RegisterSetValue("Z", data["Z"]);

        if (ShouldOutblock("X", "Y", "Z"))
        {
            RegisterSetValue("G", 0, true);
            OutBlock();
        }
        _isRapid = false; // Nach der Ausgabe zurücksetzen, da nächste Bewegung wieder Goto sein könnte
    }
    ```

6.  **Implementieren Sie die Vorschubgeschwindigkeit (`Fedrat()`):**
    * **`Fedrat(CLData data)`:** Wird aufgerufen, wenn sich die Vorschubgeschwindigkeit ändert.
        * Extrahieren Sie den Vorschubwert aus `data["F"]`.
        * Verwenden Sie `RegisterSetValue("F", data["F"])`, um den Wert für das "F"-Wort zu setzen. Beachten Sie, dass hier kein `OutBlock()` aufgerufen wird. Der Vorschub wird typischerweise in der nächsten Zeile zusammen mit einem Bewegungsbefehl ausgegeben.

    ```csharp
    public void Fedrat(CLData data)
    {
        RegisterSetValue("F", data["F"]);
        // Vorschub wird im nächsten Bewegungsbefehl ausgegeben
    }
    ```

7.  **Implementieren Sie Kommentare (`PPrint()`):**
    * **`PPrint(CLData data)`:** Wird für Kommentare aufgerufen.
        * Extrahieren Sie den Kommentartext aus `data["Comment"]`.
        * Verwenden Sie `Output($"; {data["Comment"]}")`, um den Kommentar mit einem Semikolon (`;`) als G-Code-Kommentar auszugeben.

    ```csharp
    public void PPrint(CLData data)
    {
        Output($"; {data["Comment"]}");
    }
    ```

8.  **Implementieren Sie andere spezifische Befehle (`MultAx()`, etc.):**
    * **`MultAx(CLData data)`:** In Ihrem Beispiel wird dies verwendet, um einen Multi-Achsen-Modus zu aktivieren oder zu deaktivieren. Die genaue Implementierung hängt von Ihrer Maschinensteuerung ab. Möglicherweise müssen Sie hier spezifische G-Code-Befehle ausgeben oder interne Zustände verwalten, die in anderen Methoden verwendet werden.

    ```csharp
    private bool _multAx = false;

    public void MultAx(CLData data)
    {
        if (data["State"].ToString() == "ON")
        {
            _multAx = true;
            // Geben Sie hier ggf. G-Code zum Aktivieren des Multi-Achsen-Modus aus
            // Output("...");
        }
        else
        {
            _multAx = false;
            // Geben Sie hier ggf. G-Code zum Deaktivieren des Multi-Achsen-Modus aus
            // Output("...");
        }
    }
    ```

9.  **Implementieren Sie die `Done()`-Methode:**
    * **`Done()`:** Wird aufgerufen, nachdem alle CLData-Objekte verarbeitet wurden.
    * Hier können Sie abschließende G-Code-Befehle ausgeben (z. B. Programmende M30, Spindelstopp M05, Kühlmittel aus M09).
    * Ihr Beispielcode deutet darauf hin, dass Sie hier auch die Möglichkeit haben, den gesamten generierten G-Code zu bearbeiten, bevor er an den Slicer zurückgegeben wird. Verwenden Sie `RegisterGetCode()` um den generierten Code als String-Array zu erhalten und `RegisterSetCode()` um ihn nach der Bearbeitung wieder zu setzen.

    ```csharp
    public void Done()
    {
        Output("M30"); // Programmende
    }
    ```

10. **Kompilieren Sie Ihre Klassenbibliothek:**
    * Erstellen Sie Ihr Projekt in Visual Studio (oder Ihrer Entwicklungsumgebung). Dadurch wird die `P3DPostProcessor_MyMachine.dll`-Datei (oder ein ähnlicher Name) im Ausgabeverzeichnis (z. B. `bin\Debug` oder `bin\Release`) erstellt.

11. **Konfigurieren Sie P3D, um Ihren Postprozessor zu verwenden:**
    * Öffnen Sie die Einstellungen in Ihrem P3D-Slicer.
    * Suchen Sie den Abschnitt für die Postprozessoren.
    * Fügen Sie einen neuen benutzerdefinierten Postprozessor hinzu.
    * Wählen Sie die von Ihnen erstellte `P3DPostProcessor_MyMachine.dll`-Datei aus.
    * Konfigurieren Sie ggf. weitere spezifische Einstellungen für Ihren Postprozessor innerhalb von P3D.

## Das `Register`-System im Detail

Das statische `Register`-System dient als zentrale Verwaltung für G-Code-Wörter und deren Werte.

* **`RegisterAdd(string key, string symbol, int leadingZeros = 0, int trailingZeros = 0, bool alwaysSign = false, object value = null)`:** Fügt ein neues G-Code-Wort zum Register hinzu oder aktualisiert ein vorhandenes.
    * `key`: Ein interner Schlüssel zur Identifizierung des Worts (z. B. "X", "F").
    * `symbol`: Das tatsächliche G-Code-Symbol (z. B. "X", "F"). Kann auch ein Formatstring mit `{@}` sein, der durch den Wert ersetzt wird (z. B. "S{@}").
    * `leadingZeros`: Anzahl der führenden Nullen für numerische Werte.
    * `trailingZeros`: Anzahl der nachfolgenden Nullen für numerische Werte (wird normalerweise nicht direkt verwendet, kann aber in benutzerdefinierten Formatierungen nützlich sein).
    * `alwaysSign`: Erzwingt die Ausgabe eines Vorzeichens (+/-) auch für positive Werte.
    * `value`: Ein optionaler initialer Wert.

* **`ShouldOutblock(params string[] keys)`:** Gibt `true` zurück, wenn sich der Wert mindestens eines der angegebenen G-Code-Wörter seit dem letzten Aufruf von `OutBlock()` geändert hat und daher eine Ausgabe im nächsten Block erforderlich ist.

* **`RegisterSetValue(string key, object value, bool requireOutblock = false)`:** Setzt den Wert für das angegebene G-Code-Wort.
    * `requireOutblock`: Wenn `true`, wird `ShouldOutblock` für dieses Wort automatisch auf `true` gesetzt, um die Ausgabe im nächsten `OutBlock()` zu erzwingen.

* **`RegisterTryGetValue(string key, out object value, bool current = true)`:** Versucht, den aktuellen oder vorherigen Wert eines G-Code-Worts abzurufen.

* **`Output(string line)`:** Fügt eine einzelne Zeile direkt zum Ausgabecode hinzu (z. B. für Kommentare oder Befehle ohne zugehöriges Registerwort).

* **`OutBlock()`:** Generiert eine G-Code-Zeile, die alle G-Code-Wörter enthält, deren Werte sich seit dem letzten Aufruf geändert haben (und für die `ShouldOutblock` `true` ist). Nach der Ausgabe werden die `ShouldOutblock`-Flags zurückgesetzt.

* **`RegisterGetCode()`:** Gibt den gesamten generierten G-Code als Array von Strings zurück.

* **`RegisterSetCode(string[] code)`:** Überschreibt den bisher generierten G-Code mit dem übergebenen Array. Dies ist nützlich für die Nachbearbeitung des gesamten Codes in der `Done()`-Methode.

## Das `CLData`-Objekt

Das `CLData`-Objekt ist ein Dictionary, das die Informationen für den aktuellen Verarbeitungsschritt enthält. Die Schlüssel und Werte variieren je nach Art der Operation. Einige häufige Schlüssel sind:

* `X`, `Y`, `Z`: Zielkoordinaten für Bewegungen.
* `F`: Vorschubgeschwindigkeit.
* `Comment`: Kommentartext.
* `State`: Zustand (z. B. "ON", "OFF" für `MultAx`).
* `R`: Radius für Kreisbewegungen.
* `CircleDir`: Richtung für Kreisbewegungen (true für CW, false für CCW).
* Andere operation-spezifische Parameter.

Verwenden Sie den Indexer (`data["Schlüssel"]`) und `ContainsKey()` um auf die Daten im `CLData`-Objekt zuzugreifen.

## Wichtige Überlegungen

* **Maschinenspezifische G-Code-Syntax:** Stellen Sie sicher, dass der generierte G-Code exakt der Syntax entspricht, die Ihre CNC-Maschinensteuerung erwartet. Dies umfasst G- und M-Codes, Adressformate und spezielle Befehle.
* **Interpolationstypen:** Berücksichtigen Sie lineare (G0, G1) und zirkulare (G02, G03) Interpolation sowie Helixbewegungen, falls Ihre Maschine diese unterstützt und Ihr CAM-System sie generiert.
* **Werkzeugwechsel (M06):** Implementieren Sie die Logik für Werkzeugwechsel, falls erforderlich, einschließlich des Ausgebens des `M06`-Befehls und ggf. zugehöriger Parameter (Werkzeugnummer).
* **Spindelsteuerung (M03, M04, M05):** Implementieren Sie die Steuerung für Spindeldrehung (vorwärts, rückwärts, Stopp) und Drehzahl (S-Wort).
* **Kühlmittelsteuerung (M08, M09):** Implementieren Sie das Ein- und Ausschalten des Kühlmittels.
* **Arbeitskoordinatensysteme (G54-G59, etc.):** Berücksichtigen Sie die Unterstützung für verschiedene Arbeitskoordinatensysteme.
* **Sicherheit:** Fügen Sie ggf. Sicherheitsbefehle zum Start und Ende des Programms hinzu (z. B. Rückzug in eine sichere Position).
* **Testen:** Testen Sie Ihren Postprozessor gründlich mit verschiedenen Arten von CAM-Pfaden, um sicherzustellen, dass der generierte G-Code korrekt ist und Ihre Maschine sicher betrieben werden kann. Verwenden Sie wenn möglich eine G-Code-Simulation, bevor Sie den Code auf der realen Maschine ausführen.

Durch das Verständnis dieser Konzepte und die schrittweise Implementierung der `IP3DPostProcessor`-Schnittstelle können Sie einen maßgeschneiderten Postprozessor erstellen, der die Ausgabe von P3D perfekt an die Anforderungen Ihrer CNC-Maschine anpasst.
