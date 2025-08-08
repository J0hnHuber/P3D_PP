🛠️ Anleitung zur Erstellung eines P3D Post-Prozessors
Ein P3D-Post-Prozessor ist eine C#-Klasse, die das Interface IP3DPostProcessor implementiert. Er dient als Übersetzer von APT-Code in maschinenspezifischen Code. Anhand des bereitgestellten Beispiels erkläre ich dir die einzelnen Schritte und Bestandteile.

1. Die Klasse und das Interface
Deine Post-Prozessor-Klasse muss das Interface IP3DPostProcessor implementieren. Dadurch wird sichergestellt, dass deine Klasse über alle notwendigen Methoden verfügt, die der Executor aufrufen wird, um die APT-Befehle zu verarbeiten.

C#

public class PostProcessor : IP3DPostProcessor
2. Globale Variablen
Globale Variablen werden verwendet, um den aktuellen Status der Maschine zu verfolgen.

_multAx: Ein bool, der den Zustand der mehrachsigen Bearbeitung speichert.

_isRapid: Ein bool, der angibt, ob die aktuelle Bewegung eine Eilgangbewegung (G0) ist.

C#

private bool _multAx = false;
private bool _isRapid = false;
3. Die Initialize-Methode
Diese Methode ist der Einstiegspunkt des Post-Prozessors und wird nur einmal am Anfang aufgerufen. Sie dient der Initialisierung.

StartAutoLineNumber(lineNumberSymbol: 'N', startNum: 10, interval: 10, maxNum: 5000);

Funktion: Konfiguriert die automatische Zeilennummerierung.

Argumente:

lineNumberSymbol: Das Symbol für die Zeilennummer (N).

startNum: Die Startnummer der Zeilen (hier 10).

interval: Der Inkrement-Wert für die Zeilennummerierung (hier 10).

maxNum: Die maximale Zeilennummer, bevor wieder bei startNum begonnen wird.

RegisterAdd("G", "G", decimalPlaces: 1, leadingZeros: 1);

Funktion: Definiert eine "Register"-Variable, die im G-Code ausgegeben wird.

Argumente:

"G": Der interne Name des Registers.

"G": Das Symbol, das im G-Code verwendet wird.

decimalPlaces: Anzahl der Nachkommastellen (hier 1).

leadingZeros: Anzahl der führenden Nullen (hier 1).

Output($"; ...\n;");

Funktion: Schreibt eine Zeile in den Ausgabepuffer.

Argument: Der String, der ausgegeben werden soll. Hier wird der Header mit Kommentaren (;) erzeugt.

4. Die Fedrat-Methode
Diese Methode wird aufgerufen, wenn der APT-Befehl FEDRAT (Vorschubgeschwindigkeit) auftritt.

_isRapid = false;: Setzt den Eilgang-Status auf false, da eine Vorschubbewegung bevorsteht.

RegisterSetValue("F", data["F"]);:

Funktion: Aktualisiert den Wert eines Registers.

Argumente:

"F": Der Name des zu aktualisierenden Registers.

data["F"]: Der neue Wert aus dem APT-Code.

Es gibt keinen OutBlock()-Aufruf, da der Vorschubwert erst zusammen mit der nächsten Bewegung ausgegeben werden soll.

5. Die Goto-Methode
Diese Methode wird für jede Bewegung (GOTO) aufgerufen. Hier werden die Koordinaten verarbeitet und ausgegeben.

RegisterSetValue("X", data["X"]);:

Funktion: Aktualisiert die X-Koordinate aus den data-Argumenten.

if (ShouldOutblock("X","Y","Z")):

Funktion: Überprüft, ob sich der Wert eines der angegebenen Register (hier X, Y oder Z) seit dem letzten OutBlock()-Aufruf geändert hat.

Argumente: Die Namen der Register, die überwacht werden sollen.

RegisterSetValue("G", cmd, true);:

Funktion: Setzt den G-Befehl (0 für Eilgang oder 1 für Vorschub). Der dritte Parameter true stellt sicher, dass dieses Register beim nächsten OutBlock()-Aufruf definitiv ausgegeben wird, auch wenn sich der Wert nicht geändert hat (was hier notwendig ist, da der G-Code-Befehl immer vor den Koordinaten stehen muss).

OutBlock();:

Funktion: Sammelt alle Register, deren ShouldOutblock-Flag auf true gesetzt ist, formatiert sie gemäß den Einstellungen aus RegisterAdd und schreibt sie als eine einzelne Zeile in den Ausgabepuffer.

6. Die Rapid-Methode
Diese Methode wird aufgerufen, wenn der APT-Befehl RAPID auftritt.

_isRapid = true;: Setzt den Eilgang-Status auf true.

7. Die PPrint-Methode
Diese Methode wird für den APT-Befehl PPRINT (Kommentar) aufgerufen.

Output($"; {data["Comment"]}");:

Funktion: Schreibt den Kommentar in den Ausgabepuffer, formatiert als G-Code-Kommentar (beginnend mit ;).

8. Die MultAx-Methode
Diese Methode behandelt den APT-Befehl MULTAX für mehrachsige Bewegungen.

if (data["State"].ToString() == "ON"): Überprüft den Zustand im APT-Code, um die globale Variable _multAx zu setzen.

9. Die Done-Methode
Diese Methode wird einmal ganz am Ende aufgerufen, nachdem alle APT-Befehle verarbeitet wurden.

string[] gcodeLines = RegisterGetCode();: Holt den gesamten generierten G-Code aus dem Puffer als Array.

Hier kannst du den fertigen Code weiterverarbeiten oder ändern (z. B. zusätzliche Zeilen am Ende hinzufügen oder eine spezielle Formatierung vornehmen).

RegisterSetCode(gcodeLines);: Schreibt den möglicherweise modifizierten Code zurück in den Puffer.

Mit diesem Gerüst kannst du deinen eigenen Post-Prozessor erstellen, indem du die Logik innerhalb der Methoden an die spezifischen Anforderungen deiner CNC-Maschine anpasst.
