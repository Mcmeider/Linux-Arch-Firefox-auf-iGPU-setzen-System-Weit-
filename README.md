# Linux-Arch-Firefox-auf-iGPU-setzen-System-Weit-
Linux-Arch: Firefox auf iGPU setzen (System-Weit)

README: Firefox VRAM-Offloading auf iGPU (Systemweit & Updatesicher)


Zweck

Diese Konfiguration zwingt Firefox und alle darauf basierenden "Ice" WebApps (SSB), dauerhaft die integrierte Grafikeinheit (iGPU) zu nutzen.
Dies entlastet den VRAM der dedizierten Grafikkarte (dGPU) für Gaming oder AI-Anwendungen (LLMs), während der Browser effizient auf der iGPU läuft.

Voraussetzungen

Linux-System (Arch/EndeavourOS empfohlen, funktioniert aber fast überall).
Dual-GPU Setup (z.B. AMD Ryzen iGPU + AMD Radeon dGPU).
Root/Sudo-Rechte.

Schritt 1: Hardware-IDs ermitteln

Da der Linux-Kernel bei zwei Grafikkarten des gleichen Herstellers oft die Zuordnung (card0 vs card1) vertauscht, nutzen wir die eindeutige Vendor:Device ID.
Terminal öffnen.
Folgenden Befehl eingeben:
Bash
lspci -nn | grep -E "VGA|Display"


Notiere die ID der iGPU (meist der Eintrag, der nicht an den Hauptmonitoren hängt oder als "Integrated" bekannt ist).
Format: [VVVV:DDDD] (z.B. [1002:13c0])
Beispiel Ryzen 7000/9000: 1002:13c0
Beispiel RX 6800 (dGPU): 1002:73bf (Diese wollen wir NICHT).
Wichtig: Ersetze in den folgenden Schritten 1002:13c0 durch DEINE ermittelte ID!

Schritt 2: Den Wrapper erstellen

Wir erstellen ein Start-Skript in /usr/local/bin. Dieser Pfad hat im System Vorrang vor dem Standardpfad /usr/bin. Dadurch fangen wir jeden Aufruf von "firefox" ab, ohne Systemdateien zu überschreiben.
Datei erstellen:
Bash
sudo nano /usr/local/bin/firefox


Inhalt einfügen:
Bash
#!/bin/bash
# Wrapper für Firefox iGPU Offloading
# Zwingt Rendering auf ID 1002:13c0 (Ryzen iGPU)

export DRI_PRIME=1002:13c0

# Startet das originale Binary und reicht alle Argumente durch
exec /usr/bin/firefox "$@"


Speichern (Strg+O, Enter) und Beenden (Strg+X).
Ausführbar machen:
Bash
sudo chmod +x /usr/local/bin/firefox



Schritt 3: Startmenü & Taskleiste anpassen (Systemweit)

Standardmäßige .desktop-Dateien nutzen oft den absoluten Pfad (Exec=/usr/bin/firefox), wodurch unser Wrapper umgangen würde. Wir erstellen eine "Shadow-Kopie" mit höherer Priorität.
Verzeichnis für systemweite lokale Apps erstellen (falls nicht vorhanden):
Bash
sudo mkdir -p /usr/local/share/applications


Original-Verknüpfung kopieren:
Bash
sudo cp /usr/share/applications/firefox.desktop /usr/local/share/applications/firefox.desktop


Pfade im Eintrag auf unseren Wrapper umbiegen (Search & Replace):
Bash
sudo sed -i 's|Exec=/usr/bin/firefox|Exec=/usr/local/bin/firefox|g' /usr/local/share/applications/firefox.desktop
sudo sed -i 's|Exec=firefox|Exec=/usr/local/bin/firefox|g' /usr/local/share/applications/firefox.desktop


Desktop-Datenbank aktualisieren:
Bash
sudo update-desktop-database /usr/local/share/applications



Schritt 4: Bestehende "Ice" WebApps patchen

WebApps, die mit dem Tool "Ice" (Peppermint/Endeavour) erstellt wurden, haben ihre eigenen Startdateien im Home-Verzeichnis des Nutzers. Auch diese müssen auf den Wrapper zeigen.
Alle bestehenden WebApp-Verknüpfungen des Users patchen:
Bash
sed -i 's|Exec=/usr/bin/firefox|Exec=/usr/local/bin/firefox|g' ~/.local/share/applications/*.desktop
sed -i 's|Exec=firefox|Exec=/usr/local/bin/firefox|g' ~/.local/share/applications/*.desktop


KDE/System-Cache neu einlesen:
Bash
kbuildsycoca6


Hinweis für neue WebApps: Wenn du in Zukunft eine neue WebApp mit Ice erstellst, wähle im Dropdown nicht "Firefox" (falls es den Systempfad nutzt), sondern gib unter "Custom" den Pfad /usr/local/bin/firefox an, oder führe den obigen Befehl nach der Erstellung erneut aus.

Schritt 5: Verifizierung

Stelle sicher, dass Firefox komplett geschlossen ist:
Bash
killall firefox


Starten Sie Firefox (egal ob über Terminal, Icon oder WebApp).
Öffnen Sie einen Tab mit der Adresse: about:support.
Scrollen Sie zum Abschnitt Grafik.
Prüfen Sie den Eintrag GPU 1.
✅ Erfolg: Dort steht "Ryzen" / "Raphael" / "Integrated" (und die ID 0x13c0).
❌ Fehler: Dort steht "Navi" / "Radeon RX" (die dGPU).

Deinstallation (Rückgängig machen)

Um den Ursprungszustand wiederherzustellen:

Bash


# 1. Wrapper löschen
sudo rm /usr/local/bin/firefox

# 2. Globale Desktop-Override löschen
sudo rm /usr/local/share/applications/firefox.desktop
sudo update-desktop-database /usr/local/share/applications

# 3. WebApps zurücksetzen (optional, nutzen dann wieder System-Standard)
sed -i 's|Exec=/usr/local/bin/firefox|Exec=/usr/bin/firefox|g' ~/.local/share/applications/*.desktop


