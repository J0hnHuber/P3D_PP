# Dokumentation: Erstellung eines Postprozessors für P3D mit der Visual Studio Vorlage (Vollständige Beschreibungen)

Diese Dokumentation beschreibt detailliert, wie Sie die von P3D bereitgestellte Visual Studio Vorlage verwenden können, um effizient einen benutzerdefinierten Postprozessor zu erstellen. Die Vorlage enthält bereits die notwendige DLL-Referenz und eine Basisklasse, die von `IP3DPostProcessor` erbt und alle erforderlichen Funktionen integriert hat.

## Vorteile der Vorlage

Die Verwendung der Visual Studio Vorlage bietet folgende Vorteile:

* **Zeitersparnis:** Das manuelle Hinzufügen der `P3D_Postprocessors.dll`-Referenz und das Erstellen der Grundstruktur der `PostProcessor`-Klasse mit allen Methoden der `IP3DPostProcessor`-Schnittstelle entfällt.
* **Standardisierte Struktur:** Die Vorlage gewährleistet eine konsistente und übersichtliche Organisation Ihres Postprozessorprojekts.
* **Direktstart:** Sie können sich unmittelbar auf die Implementierung der spezifischen G-Code-Generierungslogik für Ihre CNC-Maschine konzentrieren.

## Schritte zur Verwendung der Postprozessor-Vorlage

1.  **Beschaffen Sie die P3D Postprozessor Vorlage:**
    * Die Visual Studio Vorlage wird Ihnen üblicherweise vom DTR-Team als ZIP-Datei oder über einen anderen Kommunikationskanal bereitgestellt.
    * Speichern Sie die Vorlagendatei an einem leicht zugänglichen Ort auf Ihrem lokalen Rechner.

2.  **Installieren Sie die Vorlage in Visual Studio:**
    * Öffnen Sie Ihre Installation von Visual Studio.
    * Navigieren Sie im Menü zu "Erweiterungen" und wählen Sie "Erweiterungen verwalten...".
    * Im Dialogfenster "Erweiterungen verwalten" wählen Sie im linken Bereich "Online" und suchen Sie im Suchfeld rechts oben nach "P3D Postprocessor" (der exakte Name der Erweiterung kann leicht variieren).
    * Sollten Sie die Vorlage als ZIP-Datei erhalten haben, wählen Sie im linken Bereich "Aus Datei installieren..." und navigieren Sie zu der heruntergeladenen ZIP-Datei. Klicken Sie auf "Öffnen" und folgen Sie den Installationsanweisungen.
    * Nachdem die Installation abgeschlossen ist, werden Sie möglicherweise aufgefordert, Visual Studio neu zu starten. Tun Sie dies, um sicherzustellen, dass die Vorlage korrekt geladen wurde.

3.  **Erstellen Sie ein neues Projekt mit der P3D Postprozessor Vorlage:**
    * Klicken Sie in Visual Studio auf "Neues Projekt erstellen".
    * Im Fenster "Neues Projekt erstellen" suchen Sie im Suchfeld nach "P3D Postprocessor". Die Vorlage sollte unter den C#-Vorlagen angezeigt werden.
    * Wählen Sie die "P3D Postprocessor" Vorlage aus und klicken Sie auf "Weiter".
    * Im nächsten Schritt geben Sie Ihrem Projekt einen aussagekräftigen Namen (z. B. `P3DPostProcessor_MyMachine`) und wählen Sie einen geeigneten Speicherort für das Projekt. Klicken Sie anschließend auf "Erstellen".

4.  **Überprüfen Sie die Projektstruktur:**
    * Nachdem das Projekt erstellt wurde, überprüfen Sie den Projektmappen-Explorer (normalerweise auf der rechten Seite von Visual Studio). Sie sollten Folgendes sehen:
        * Ein Projekt mit dem von Ihnen gewählten Namen.
        * Unter "Abhängigkeiten" oder "Referenzen" sollte die `P3D_Postprocessors.dll` bereits als Referenz hinzugefügt sein.
        * Eine C#-Datei namens `PostProcessor.cs` (oder ähnlich).
    * Öffnen Sie die Datei `PostProcessor.cs` durch Doppelklick. Sie werden feststellen, dass die Klasse `PostProcessor` die `IP3DPostProcessor`-Schnittstelle implementiert und alle in dieser Schnittstelle definierten Methoden bereits als leere oder mit grundlegenden Kommentaren versehene Methoden vorhanden sind. Dies ist der Ausgangspunkt für Ihre Postprozessor-Implementierung.

5.  **Implementieren Sie die spezifische Logik für Ihre Maschine:**
    * Bearbeiten Sie die einzelnen Methoden in der `PostProcessor.cs`-Datei, um das gewünschte Verhalten Ihres Postprozessors zu definieren.

        * **`public void Initialize()`:**
            * Diese Methode wird **einmalig** zu Beginn des Postprozessierungsprozesses aufgerufen.
            * Hier registrieren Sie die G-Code-Wörter, die Ihr Postprozessor verwenden soll, mithilfe der statischen Methode `Register.RegisterAdd()`. Für jedes G-Code-Wort definieren Sie einen internen Schlüssel (`key`), das tatsächliche G-Code-Symbol (`symbol`) und optionale Formatierungsanweisungen wie führende/nachfolgende Nullen (`leadingZeros`, `trailingZeros`) und ob immer ein Vorzeichen ausgegeben werden soll (`alwaysSign`).
            * Verwenden Sie die statische Methode `Register.Output()` um den Header Ihres G-Code-Programms auszugeben. Dies können Kommentare mit Informationen zum Postprozessor, dem Erstellungsdatum und anderen relevanten Details sein.

            ```csharp
            public void Initialize()
            {
                // G-Code-Wörter für grundlegende Bewegungen und Vorschub registrieren
                RegisterAdd("G", "G", 1); // G-Befehl mit einer führenden Null (z.B. G01)
                RegisterAdd("X", "X", trailingZeros: 3); // X-Koordinate mit drei Nachkommastellen
                RegisterAdd("Y", "Y", trailingZeros: 3); // Y-Koordinate mit drei Nachkommastellen
                RegisterAdd("Z", "Z", trailingZeros: 3); // Z-Koordinate mit drei Nachkommastellen
                RegisterAdd("F", "F"); // Vorschubgeschwindigkeit ohne spezielle Formatierung

                // Header des G-Code-Programms ausgeben
                Output($"; Code generiert mit P3D Postprozessor für Ihre Maschine");
                Output($"; Erstellungsdatum: {DateTime.Now.ToString("dd.MM.yyyy HH:mm:ss")}");
                Output(";");
            }
            ```

        * **`public void Goto(CLData data)`:**
            * Diese Methode wird für **interpolierte Bewegungen** aufgerufen (in der Regel geradlinige Bewegungen mit Vorschub, entsprechend G1).
            * Das `data`-Objekt enthält die Zielkoordinaten (unter den Schlüsseln "X", "Y", "Z") und möglicherweise andere relevante Informationen für die Bewegung.
            * Verwenden Sie `Register.RegisterSetValue()` um die extrahierten Koordinatenwerte den entsprechenden G-Code-Wörtern ("X", "Y", "Z") zuzuweisen.
            * Rufen Sie `Register.ShouldOutblock("X", "Y", "Z")` auf, um zu prüfen, ob sich mindestens eine der Positionskoordinaten seit dem letzten Aufruf von `Register.OutBlock()` geändert hat. Wenn dies der Fall ist, muss ein neuer G-Code-Block ausgegeben werden.
            * Setzen Sie den G-Code-Befehl für Vorschubbewegung (in der Regel "G1") mithilfe von `Register.RegisterSetValue("G", 1, true)`. Das `true`-Argument bewirkt, dass "G1" im nächsten Aufruf von `Register.OutBlock()` ausgegeben wird, falls eine Positionsänderung stattgefunden hat.
            * Rufen Sie `Register.OutBlock()` auf, um eine neue G-Code-Zeile zu generieren, die die geänderten G-Code-Wörter und ihre aktuellen Werte enthält.

            ```csharp
            private bool _isRapid = false; // Hilfsvariable zur Verfolgung des Bewegungsmodus

            public void Goto(CLData data)
            {
                RegisterSetValue("X", data["X"]);
                RegisterSetValue("Y", data["Y"]);
                RegisterSetValue("Z", data["Z"]);

                if (Register.ShouldOutblock("X", "Y", "Z"))
                {
                    RegisterSetValue("G", _isRapid ? 0 : 1, true); // Setze G0 für Rapid, G1 für Goto
                    Register.OutBlock();
                }
            }
            ```

        * **`public void Fedrat(CLData data)`:**
            * Diese Methode wird aufgerufen, wenn sich die **Vorschubgeschwindigkeit** ändert.
            * Das `data`-Objekt enthält den neuen Vorschubwert unter dem Schlüssel "F".
            * Verwenden Sie `Register.RegisterSetValue("F", data["F"])` um den Vorschubwert dem "F"-G-Code-Wort zuzuweisen. Beachten Sie, dass hier üblicherweise **kein** `Register.OutBlock()` aufgerufen wird. Der Vorschubwert wird in der Regel zusammen mit dem nächsten Bewegungsbefehl (G0 oder G1) ausgegeben.

            ```csharp
            public void Fedrat(CLData data)
            {
                RegisterSetValue("F", data["F"]);
                // Der Vorschub wird im nächsten Bewegungsbefehl ausgegeben
            }
            ```

        * **`public void Rapid(CLData data)`:**
            * Diese Methode wird für **Eilgangbewegungen** (schnelle, nicht-bearbeitende Bewegungen, entsprechend G0) aufgerufen.
            * Das `data`-Objekt enthält die Zielkoordinaten (unter den Schlüsseln "X", "Y", "Z").
            * Setzen Sie die interne Hilfsvariable `_isRapid` auf `true`, um anzuzeigen, dass die nächste Bewegung ein Eilgang ist.
            * Verwenden Sie `Register.RegisterSetValue()` um die Zielkoordinaten zuzuweisen.
            * Rufen Sie `Register.ShouldOutblock("X", "Y", "Z")` auf und geben Sie bei Bedarf einen neuen G-Code-Block mit `Register.RegisterSetValue("G", 0, true)` (für G0) und `Register.OutBlock()` aus.
            * Setzen Sie `_isRapid` nach der Ausgabe wieder auf `false`, da die nächste Bewegung möglicherweise wieder ein `Goto` (G1) sein kann.

            ```csharp
            private bool _isRapid = false;

            public void Rapid(CLData data)
            {
                _isRapid = true;
                RegisterSetValue("X", data["X"]);
                RegisterSetValue("Y", data["Y"]);
                RegisterSetValue("Z", data["Z"]);

                if (Register.ShouldOutblock("X", "Y", "Z"))
                {
                    RegisterSetValue("G", 0, true);
                    Register.OutBlock();
                }
                _isRapid = false;
            }
            ```

        * **`public void PPrint(CLData data)`:**
            * Diese Methode wird für **Kommentare** im CAM-Programm aufgerufen.
            * Das `data`-Objekt enthält den Kommentartext unter dem Schlüssel "Comment".
            * Verwenden Sie `Register.Output($"; {data["Comment"]}")` um den Kommentar in der G-Code-Datei auszugeben. Kommentare in G-Code beginnen üblicherweise mit einem Semikolon (;).

            ```csharp
            public void PPrint(CLData data)
            {
                Register.Output($"; {data["Comment"]}");
            }
            ```

        * **`public void MultAx(CLData data)`:**
            * Diese Methode wird aufgerufen, um den **Multi-Achsen-Modus** zu aktivieren oder zu deaktivieren.
            * Das `data`-Objekt enthält den Zustand (z. B. "ON" oder "OFF") unter dem Schlüssel "State".
            * Implementieren Sie hier die Logik, um spezifische G-Code-Befehle auszugeben oder interne Zustände Ihres Postprozessors zu verwalten, die für die Handhabung von zusätzlichen Achsen relevant sind. Die genauen Befehle hängen stark von Ihrer Maschinensteuerung ab.

            ```csharp
            private bool _multAx = false;

            public void MultAx(CLData data)
            {
                if (data["State"].ToString() == "ON")
                {
                    _multAx = true;
                    // Geben Sie hier ggf. G-Code zum Aktivieren des Multi-Achsen-Modus aus
                    // Register.Output("...");
                }
                else
                {
                    _multAx = false;
                    // Geben Sie hier ggf. G-Code zum Deaktivieren des Multi-Achsen-Modus aus
                    // Register.Output("...");
                }
            }
            ```

        * **`public void Done()`:**
            * Diese Methode wird **einmalig** aufgerufen, nachdem alle `CLData`-Objekte verarbeitet wurden.
            * Hier können Sie abschließende G-Code-Befehle ausgeben, wie z. B. das Programmende (M30), Spindelstopp (M05), Kühlmittel aus (M09) oder das Zurückfahren der Maschine in eine sichere Ausgangsposition.
            * Die auskommentierten Codezeilen in Ihrem Beispiel deuten darauf hin, dass Sie hier auch die Möglichkeit haben, den **gesamten generierten G-Code** zu bearbeiten, bevor er an den P3D-Slicer zurückgegeben wird. Verwenden Sie `Register.RegisterGetCode()` um den generierten Code als Array von Strings abzurufen und `Register.RegisterSetCode(string[])` um den bearbeiteten Code zurückzuschreiben.

            ```csharp
            public void Done()
            {
                // Hier können Sie den gesamten generierten G-Code nachbearbeiten
                /*
                string[] gcodeLines = Register.RegisterGetCode();
                for (int i = 0; i < gcodeLines.Length; i++)
                {
                    // Führen Sie hier Ihre Nachbearbeitungsschritte durch
                }
                Register.RegisterSetCode(gcodeLines);
                */

                // Abschließende G-Code-Befehle ausgeben
                Register.Output("M30"); // Programmende
            }
            ```

6.  **Kompilieren Sie Ihre Klassenbibliothek:**
    * Nachdem Sie die Implementierung abgeschlossen haben, erstellen Sie Ihr Projekt in Visual Studio. Wählen Sie im Menü "Build" -> "Projektmappe erstellen".
    * Die kompilierte Postprozessor-DLL-Datei (`P3DPostProcessor_MyMachine.dll` oder ähnlich) wird im Ausgabeverzeichnis Ihres Projekts gefunden (normalerweise unter `bin\Debug` oder `bin\Release`).

7.  **Konfigurieren Sie P3D, um Ihren Postprozessor zu verwenden:**
    * Starten Sie Ihren P3D-Slicer.
    * Öffnen Sie die Einstellungen oder Konfiguration für die Postprozessoren. Der genaue Speicherort dieser Einstellungen hängt von der P3D-Version ab. Suchen Sie nach einem Abschnitt, der "Postprozessoren", "Maschinenkonfiguration" oder ähnlich heißt.
    * Fügen Sie einen neuen benutzerdefinierten Postprozessor hinzu.
    * Navigieren Sie zu dem Ausgabeverzeichnis Ihres Visual Studio Projekts und wählen Sie die erstellte `.dll`-Datei aus.
    * Konfigurieren Sie gegebenenfalls weitere spezifische Einstellungen für Ihren Postprozessor innerhalb der P3D-Oberfläche.

## Zusammenfassung der Vorteile der Vorlage

Die P3D Visual Studio Vorlage bietet einen erheblichen Vorteil, indem sie die anfängliche Einrichtung eines Postprozessorprojekts automatisiert. Sie erhalten eine sofort einsatzbereite Struktur mit den notwendigen Referenzen und einer Basisimplementierung der `IP3DPostProcessor`-Schnittstelle. Dies ermöglicht Ihnen, sich direkt auf die Entwicklung der maschinenspezifischen Logik zu konzentrieren und beschleunigt den Prozess der Erstellung eines benutzerdefinierten Postprozessors für Ihren P3D-Slicer. Die detaillierten Beschreibungen der einzelnen Methoden und die Erklärungen zum `Register`-System in dieser Dokumentation helfen Ihnen dabei, die Funktionalität der Vorlage optimal zu nutzen und einen präzisen und zuverlässigen Postprozessor für Ihre CNC-Maschine zu erstellen.
