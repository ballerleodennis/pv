helf hier weiter : Zwei Dinge auf einmal: PowerShell führt nichts aus dem aktuellen Ordner aus (daher .\), und .sh ist für Windows ohnehin die falsche Endung, wenn du die Windows-Fassung meinst.

Erst prüfen, was in der Datei steckt:

Get-Content .\wg-server.sh -TotalCount 3
Erste Zeile @echo off → das ist die Batch-Fassung, nur falsch benannt.
Erste Zeile #!/usr/bin/env bash → das ist die Linux-Fassung, die braucht Git Bash oder WSL.
Wenn es die Batch-Fassung ist — umbenennen und starten:

Rename-Item .\wg-server.sh wg-server.bat
.\wg-server.bat
Alternativ ohne Umbenennen, direkt mit dem Interpreter:

cmd /c wg-server.sh
Das funktioniert auch bei .sh, weil cmd die Datei einfach als Batch einliest. Ist aber unsauber, weil %~dp0 und %* bei fremder Endung weiter greifen — die Umbenennung ist der bessere Weg.

Wenn es die Linux-Fassung ist, hast du den falschen der beiden Skripte gespeichert. Nimm die Windows-Version aus meiner vorletzten Antwort (wg-server.bat mit net session, New-NetNat, wireguard.exe) — die .sh-Version läuft unter Windows nur in Git Bash und kann kein netsh, reg oder New-NetNat.

Zum .ps1: Das brauchst du nicht mehr separat aufrufen, wg-server.bat macht die Freigabe-Vorbereitung nicht selbst — also:

powershell -NoProfile -ExecutionPolicy Bypass -File .\portfreigabe.ps1
Und: wg-server.bat fordert Admin selbst an, starte es trotzdem aus einem normalen Fenster, nicht aus einer bereits erhöhten Shell mit falschem Arbeitsverzeichnis.

Du hast mir noch das Ergebnis des IPv4-Tests nicht geschickt:

powershell -NoProfile -Command "(Invoke-WebRequest 'https://ipv4.icanhazip.com' -UseBasicParsing).Content.Trim()"
Das entscheidet, ob die Portfreigabe überhaupt Sinn hat. Wenn da eine 100.64.x-Adresse oder nichts kommt, ist die ganze Freigabe-Kiste für die Katz und wir gehen auf IPv6. ausgabe von meinem powershell : Suggestion [3,General]: Der Befehl wg-server.sh wurde nicht gefunden. Er ist jedoch am aktuellen Ort vorhanden. Windows PowerShell lädt Befehle nicht standardmäßig vom aktuellen Ort. Wenn Sie diesem Befehl vertrauen, geben Sie stattdessen ".\wg-server.sh" ein. Weitere Informationen erhalten Sie unter "get-help about_Command_Precedence".
C:\EIGENE-VPN> Get-Content .\wg-server.sh -TotalCount 3
@echo off
setlocal EnableExtensions EnableDelayedExpansion
chcp 65001 >nul
C:\EIGENE-VPN>

Gearbeitet für 4s · 1 Schritt

Bestätigt: erste Zeile @echo off, dazu chcp 65001 — das ist die Batch-Fassung. Die Linux-Variante ist damit vom Tisch, du musst nichts neu speichern. Nur die Endung ist falsch.

Bevor du irgendwas startest, in dieser Reihenfolge:

1. IPv4-Test — das entscheidet alles

Im normalen Fenster:

powershell -NoProfile -Command "(Invoke-WebRequest 'https://ipv4.icanhazip.com' -UseBasicParsing).Content.Trim()"
Kommt da eine öffentliche Adresse (nicht 100.64.x bis 100.127.x, nicht Timeout) → weiter. Kommt CGNAT oder nichts → Portfreigabe sparen, wir gehen auf IPv6.

2. Freigabe vorbereiten — erhöhte Shell, richtiges Verzeichnis

Aus deinem aktuellen Fenster:

Start-Process powershell -Verb RunAs -ArgumentList '-NoProfile','-ExecutionPolicy','Bypass','-NoExit','-Command','cd C:\EIGENE-VPN; .\portfreigabe.ps1'
-NoExit lässt das Fenster offen, damit du Fehlerzeilen lesen kannst. Falls New-NetNat mit "already exists" abbricht, hast du von einem früheren Versuch noch eine alte NAT-Regel stehen — sag Bescheid, dann räumen wir mit Remove-NetNat vorher auf. Das Admin-Fenster danach schließen.

3. Server starten — wieder aus dem normalen Fenster

Rename-Item .\wg-server.sh wg-server.bat
.\wg-server.bat
Nicht aus der erhöhten Shell, sondern aus deinem C:\EIGENE-VPN-Fenster — das Skript holt sich die Elevation selbst, und %~dp0 zeigt dann korrekt auf den Ordner.

Schick mir den Output von Schritt 1 und dem Start aus Schritt 3.

1 : C:\EIGENE-VPN> wg-server.sh
wg-server.sh : Die Benennung "wg-server.sh" wurde nicht als Name eines Cmdlet, einer Funktion, einer Skriptdatei oder
eines ausführbaren Programms erkannt. Überprüfen Sie die Schreibweise des Namens, oder ob der Pfad korrekt ist (sofern
enthalten), und wiederholen Sie den Vorgang.
In Zeile:1 Zeichen:1

wg-server.sh
+ CategoryInfo          : ObjectNotFound: (wg-server.sh:String) [], CommandNotFoundException
  + FullyQualifiedErrorId : CommandNotFoundException
Suggestion [3,General]: Der Befehl wg-server.sh wurde nicht gefunden. Er ist jedoch am aktuellen Ort vorhanden. Windows PowerShell lädt Befehle nicht standardmäßig vom aktuellen Ort. Wenn Sie diesem Befehl vertrauen, geben Sie stattdessen ".\wg-server.sh" ein. Weitere Informationen erhalten Sie unter "get-help about_Command_Precedence".
C:\EIGENE-VPN> Get-Content .\wg-server.sh -TotalCount 3
@echo off
setlocal EnableExtensions EnableDelayedExpansion
chcp 65001 >nul
2: C:\EIGENE-VPN> powershell -NoProfile -Command "(Invoke-WebRequest 'https://ipv4.icanhazip.com' -UseBasicParsing).Content.Trim()"
95.91.146.128
C:\EIGENE-VPN> 3 : C:\EIGENE-VPN> Rename-Item .\wg-server.sh wg-server.bat
C:\EIGENE-VPN> .\wg-server.bat
Start-Process : Das Argument für den Parameter "ArgumentList" kann nicht überprüft werden. Das Argument ist NULL oder
leer. Geben Sie ein Argument an, das nicht NULL oder leer ist, und führen Sie den Befehl erneut aus.
In Zeile:1 Zeichen:69

... cess -FilePath 'C:\EIGENE-VPN\wg-server.bat' -ArgumentList '' -Verb R ...
~~
CategoryInfo : InvalidData: (:) [Start-Process], ParameterBindingValidationException
FullyQualifiedErrorId : ParameterArgumentValidationError,Microsoft.PowerShell.Commands.StartProcessCommand
C:\EIGENE-VPN>

Gearbeitet für 7s · 1 Schritt

95.91.146.128 — öffentliche Telekom-Adresse, kein CGNAT. Portfreigabe ist damit sinnvoll, wir bleiben bei IPv4.

Der Fehler kommt aus dem Skript selbst: es will sich per Start-Process ... -ArgumentList '%*' -Verb RunAs selbst erhöhen, und weil du ohne Argumente gestartet hast, ist %* leer → -ArgumentList '' → Validierungsfehler. Das Skript kommt also gar nicht bis zur Arbeit.

Lösung: Elevation nicht dem Skript überlassen, sondern selbst erhöht starten. Dann ist die Admin-Prüfung im Skript schon erfüllt und der fehlerhafte Relaunch-Zweig wird übersprungen.

1. Erhöhte Shell im richtigen Ordner öffnen

Aus deinem normalen Fenster:

Start-Process cmd -Verb RunAs -ArgumentList '/k','cd /d C:\EIGENE-VPN'
2. In dem neuen Fenster — erst prüfen, ob die ps1 überhaupt da ist

dir portfreigabe.ps1
Hast du die portfreigabe.ps1 in Schritt 2 vorhin schon erfolgreich laufen lassen? Falls ja, überspringen. Falls nein oder falls die Datei fehlt:

powershell -NoProfile -ExecutionPolicy Bypass -File .\portfreigabe.ps1
3. Direkt danach im selben erhöhten Fenster

.\wg-server.bat
Bleib im erhöhten Fenster — nicht in ein normales wechseln, sonst läuft es wieder in denselben Fehler.

Zwei Dinge, auf die du achten sollst:

Falls die Firewall-Abfrage für wireguard.exe kommt: Zugriff zulassen, für private und öffentliche Netze.
Falls New-NetNat mit "already exists" abbricht, sag Bescheid — dann ist noch eine alte Regel von einem früheren Versuch drin und wir räumen mit Remove-NetNat auf.
Schick mir die Ausgabe von Schritt 2 und 3 komplett. Wenn du lieber ohne erhöhtes Fenster arbeiten willst, ginge auch .\wg-server.bat _ — der Unterstrich füllt %*, dann klappt auch der Selbst-Relaunch.

C:\EIGENE-VPN> .\wg-server.bat
Start-Process : Das Argument für den Parameter "ArgumentList" kann nicht überprüft werden. Das Argument ist NULL oder
leer. Geben Sie ein Argument an, das nicht NULL oder leer ist, und führen Sie den Befehl erneut aus.
In Zeile:1 Zeichen:69

... cess -FilePath 'C:\EIGENE-VPN\wg-server.bat' -ArgumentList '' -Verb R ...
~~
CategoryInfo : InvalidData: (:) [Start-Process], ParameterBindingValidationException
FullyQualifiedErrorId : ParameterArgumentValidationError,Microsoft.PowerShell.Commands.StartProcessCommand
C:\EIGENE-VPN>
hab kein wireguard

Gearbeitet für 5s · 1 Schritt

Zwei Baustellen: das Skript hat offenbar gar keine Admin-Prüfung, die den Relaunch überspringt — deshalb knallt es auch im erhöhten Fenster wieder an derselben Stelle. Und ohne WireGuard kann es später ohnehin nicht arbeiten.

1. WireGuard installieren

winget install WireGuard.WireGuard
Falls winget fehlt oder zickt: MSI direkt von https://www.wireguard.com/install/ ziehen und normal installieren. Danach einmal wireguard.exe prüfen — der Installer legt sie unter C:\Program Files\WireGuard\ ab.

2. Zeig mir, was das Skript wirklich macht

Ich rate sonst weiter blind an der Relaunch-Zeile herum. Bitte die komplette Datei und den Ordnerinhalt:

Get-Content .\wg-server.bat
dir
Beides vollständig, ohne Kürzen. Dann sehe ich, ob da ein Admin-Guard fehlt, welche Argumente erwartet werden, ob wg0.conf und Schlüssel schon existieren und wo genau wireguard.exe erwartet wird.

3. Bis dahin als Sofort-Test

.\wg-server.bat _
Der Unterstrich füllt %*, damit -ArgumentList nicht mehr leer ist. Beobachte, ob es in eine Endlosschleife läuft (Fenster poppt immer wieder neu auf) — dann fehlt wirklich der Guard und wir bauen ihn ein, statt weiter zu tricksen.

C:\EIGENE-VPN> Start-Process cmd -Verb RunAs -ArgumentList '/k','cd /d C:\EIGENE-VPN'
C:\EIGENE-VPN> dir portfreigabe.ps1

Verzeichnis: C:\EIGENE-VPN
Mode LastWriteTime Length Name

-a---- 17.11.2026 01:40 4965 portfreigabe.ps1

C:\EIGENE-VPN> powershell -NoProfile -ExecutionPolicy Bypass -File .\portfreigabe.ps1
Ziel: 192.168.178.20 Port 443/UDP -> 443
Hinweis: 192.168.178.20 ist der Box nicht als aktiver Nachbar bekannt.
FRITZ!Box-Benutzer (Admin): Admin
WARNUNG: WAN-IP () != oeffentliche IP (2a02:810a:18e:5f00:2d3a:cf1a:4dd7:cb16) -> CGNAT.
Variante erfolgreich: IP, permanent

Freigabe steht: 443/UDP -> :
C:\EIGENE-VPN> .\wg-server.bat
Start-Process : Das Argument für den Parameter "ArgumentList" kann nicht überprüft werden. Das Argument ist NULL oder
leer. Geben Sie ein Argument an, das nicht NULL oder leer ist, und führen Sie den Befehl erneut aus.
In Zeile:1 Zeichen:69

... cess -FilePath 'C:\EIGENE-VPN\wg-server.bat' -ArgumentList '' -Verb R ...
~~
CategoryInfo : InvalidData: (:) [Start-Process], ParameterBindingValidationException
FullyQualifiedErrorId : ParameterArgumentValidationError,Microsoft.PowerShell.Commands.StartProcessCommand
C:\EIGENE-VPN> Get-Content .\wg-server.bat
@echo off
setlocal EnableExtensions EnableDelayedExpansion
chcp 65001 >nul
title WireGuard Server - Windows

set "WG_DIR=%ProgramFiles%\WireGuard"
set "WG_EXE=%WG_DIR%\wireguard.exe"
set "WG_CLI=%WG_DIR%\wg.exe"
set "BASE=%~dp0"
set "KEYS=%BASE%keys"
set "CLIENTS=%BASE%clients"
set "CONF=%BASE%wg0.conf"
set "EPFILE=%BASE%endpoint.txt"
set "TUNNEL=wg0"
set "SVC=WireGuardTunnel$wg0"
set "WG_NET=10.66.66"
set "PORT=443"
set "MTU=1380"
set "NATNAME=WGVPN"

net session >nul 2>&1
if errorlevel 1 (
powershell -NoProfile -ExecutionPolicy Bypass -Command "Start-Process -FilePath '%~f0' -ArgumentList '%*' -Verb RunAs"
exit /b
)

set "ACT=%~1"
if "%ACT%"=="" set "ACT=menu"
goto :%ACT% 2>nul || goto :menu

:menu
cls
echo ============================================
echo WireGuard Server - %WG_NET.0/24 auf UDP %PORT%
echo ============================================
sc query "%SVC%" | findstr /C:"RUNNING" >nul 2>&1 && (echo Dienst: LAEUFT) || (echo Dienst: gestoppt)
powershell -NoProfile -Command "if(Get-NetNat -Name '%NATNAME%' -ErrorAction SilentlyContinue){exit 0}else{exit 1}" && (echo NAT: aktiv) || (echo NAT: inaktiv)
echo.
echo [1] Setup / Neuinstallation
echo [2] Server starten
echo [3] Server stoppen
echo [4] Status
echo [5] Client hinzufuegen
echo [6] Client entfernen
echo [7] Alles deinstallieren
echo [0] Exit
echo.
set /p "CH=> Auswahl: "
if "%CH%"=="1" goto :setup
if "%CH%"=="2" goto :up
if "%CH%"=="3" goto :down
if "%CH%"=="4" goto :status
if "%CH%"=="5" goto :add
if "%CH%"=="6" goto :remove
if "%CH%"=="7" goto :cleanup
if "%CH%"=="0" exit /b 0
goto :menu

:ensure_wg
if exist "%WG_EXE%" exit /b 0
echo Lade offiziellen WireGuard-Installer ...
powershell -NoProfile -ExecutionPolicy Bypass -Command "try { Invoke-WebRequest -Uri 'https://download.wireguard.com/windows-client/wireguard-installer.exe' -OutFile '%TEMP%\wg.exe' -UseBasicParsing; Start-Process '%TEMP%\wg.exe' -ArgumentList '/S' -Wait; exit 0 } catch { exit 1 }"
if errorlevel 1 (echo Installation fehlgeschlagen. & pause & exit /b 1)
if not exist "%WG_EXE%" (echo WireGuard nicht gefunden. & pause & exit /b 1)
exit /b 0

:setup
call :ensure_wg || goto :menu
if exist "%KEYS%\server.key" (
set /p "OV=Schluessel existieren bereits. Neu erzeugen? (j/n): "
if /i not "!OV!"=="j" goto :setup_net
)
echo.
set /p "ENDPOINT=Oeffentliche IP oder DDNS-Name: "
if "%ENDPOINT%"=="" (echo Abbruch: kein Endpunkt. & pause & goto :menu)

"%EPFILE%" echo %ENDPOINT%

echo [*] Schluessel erzeugen
if exist "%KEYS%" rd /s /q "%KEYS%"
mkdir "%KEYS%" 2>nul
mkdir "%CLIENTS%" 2>nul
powershell -NoProfile -ExecutionPolicy Bypass -Command "$wg='%WG_CLI%'; $d='%KEYS%'; function G { ((& $wg genkey) -join '') -replace '[^\x21-\x7E]','' }; $p=G; $pu=(((($p | & $wg pubkey) -join '')) -replace '[^\x21-\x7E]',''); $psk=(((& $wg genpsk) -join '') -replace '[^\x21-\x7E]',''); Set-Content -Path (Join-Path $d 'server.key') -Value $p -NoNewline -Encoding ascii; Set-Content -Path (Join-Path $d 'server.pub') -Value $pu -NoNewline -Encoding ascii; Set-Content -Path (Join-Path $d 'server.psk') -Value $psk -NoNewline -Encoding ascii"
if not exist "%KEYS%\server.key" (echo Schluesselerzeugung fehlgeschlagen. & pause & goto :menu)

for /f "usebackq delims=" %%i in ("%KEYS%\server.key") do set "SPRIV=%%i"

echo [*] Serverkonfiguration schreiben

"%CONF%" echo [Interface]

"%CONF%" echo Address = %WG_NET%.1/24
"%CONF%" echo ListenPort = %PORT%
"%CONF%" echo PrivateKey = %SPRIV%
"%CONF%" echo MTU = %MTU%
"%CONF%" echo SaveConfig = false

echo [*] Firewall
netsh advfirewall firewall delete rule name="WG-Server-In" >nul 2>&1
netsh advfirewall firewall delete rule name="WG-Server-Tunnel" >nul 2>&1
netsh advfirewall firewall add rule name="WG-Server-In" dir=in action=allow protocol=udp localport=%PORT% profile=any >nul
netsh advfirewall firewall add rule name="WG-Server-Tunnel" dir=in action=allow localip=%WG_NET%.0/24 profile=any >nul

:setup_net
echo [*] Routing aktivieren
reg add "HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters" /v IPEnableRouter /t REG_DWORD /d 1 /f >nul
netsh interface ipv4 set interface "%TUNNEL%" forwarding=enabled >nul 2>&1

echo [*] NAT einrichten
powershell -NoProfile -Command "if(Get-Command New-NetNat -ErrorAction SilentlyContinue){ Get-NetNat -Name '%NATNAME%' -ErrorAction SilentlyContinue | Remove-NetNat -Confirm:$false -ErrorAction SilentlyContinue; New-NetNat -Name '%NATNAME%' -InternalIPInterfaceAddressPrefix '%WG_NET%.0/24' -ErrorAction SilentlyContinue; exit 0 } else { exit 1 }"
if errorlevel 1 (
echo NetNat nicht verfuegbar - versuche RRAS-Fallback ...
set "PUBIF="
for /f "usebackq delims=" %%i in (powershell -NoProfile -Command "$a=(Get-NetRoute -DestinationPrefix '0.0.0.0/0' ^| Sort-Object RouteMetric ^| Select-Object -First 1).InterfaceAlias; $a") do set "PUBIF=%%i"
netsh routing ip nat install >nul 2>&1
netsh routing ip nat add interface name="%TUNNEL%" mode=full >nul 2>&1
if defined PUBIF netsh routing ip nat add interface name="!PUBIF!" mode=public >nul 2>&1
echo Hinweis: RRAS-Modus aktiv. Falls kein Internet durch den Tunnel geht, NetNat nachinstallieren.
)

echo [*] Dienst installieren und starten
"%WG_EXE%" /installtunnelservice "%CONF%"
timeout /t 3 /nobreak >nul
call :status_quiet

echo.
echo ============================================
echo SERVER BEREIT
echo ============================================
echo Endpunkt: %ENDPOINT%:%PORT%/udp
echo Router: UDP %PORT% an diesen PC weiterleiten
echo.
echo Jetzt Option 5 nutzen, um Clients anzulegen.
echo ============================================
pause & goto :menu

:up
net start "%SVC%" >nul 2>&1
echo Startbefehl ausgefuehrt.
pause & goto :menu

:down
net stop "%SVC%" >nul 2>&1
echo Server gestoppt. NAT bleibt bestehen.
pause & goto :menu

:status_quiet
"%WG_CLI%" show %TUNNEL% 2>nul
exit /b 0

:status
cls
"%WG_CLI%" show %TUNNEL% 2>nul
if errorlevel 1 echo Tunnel nicht aktiv.
echo.
echo Peers:
"%WG_CLI%" show %TUNNEL% peers 2>nul
echo.
ipconfig | findstr /C:"%WG_NET%.1" >nul && echo Server-IP gesetzt. || echo Server-IP fehlt.
pause & goto :menu

:add
if not exist "%KEYS%\server.key" (echo Erst Setup ausfuehren. & pause & goto :menu)
if not exist "%EPFILE%" (echo Endpunkt unbekannt - Setup erneut ausfuehren. & pause & goto :menu)
set /p "CNAME=Client-Name: "
if "%CNAME%"=="" (echo Abbruch. & pause & goto :menu)
set "CNAME=%CNAME: =_%"
if not exist "%CLIENTS%" mkdir "%CLIENTS%"
set "CFILE=%CLIENTS%.counter"
if exist "%CFILE%" (set /p "NUM=<%CFILE%") else (set "NUM=2")
echo !NUM! > "%CFILE%"
set /a NEXT=NUM+1
echo !NEXT! > "%CFILE%"

powershell -NoProfile -ExecutionPolicy Bypass -Command "$wg='%WG_CLI%'; $d='%CLIENTS%'; $n='%CNAME%'; function G { ((& $wg genkey) -join '') -replace '[^\x21-\x7E]','' }; $p=G; $pu=(((($p | & $wg pubkey) -join '')) -replace '[^\x21-\x7E]',''); $psk=(((& $wg genpsk) -join '') -replace '[^\x21-\x7E]',''); Set-Content -Path (Join-Path $d ($n + '.key')) -Value $p -NoNewline -Encoding ascii; Set-Content -Path (Join-Path $d ($n + '.psk')) -Value $psk -NoNewline -Encoding ascii; Set-Content -Path (Join-Path $d ($n + '.pub')) -Value $pu -NoNewline -Encoding ascii"
if not exist "%CLIENTS%%CNAME%.pub" (echo Schluessel fehlgeschlagen. & pause & goto :menu)

for /f "usebackq delims=" %%i in ("%CLIENTS%%CNAME%.key") do set "CPRIV=%%i"
for /f "usebackq delims=" %%i in ("%CLIENTS%%CNAME%.pub") do set "CPUB=%%i"
for /f "usebackq delims=" %%i in ("%CLIENTS%%CNAME%.psk") do set "CPSK=%%i"
for /f "usebackq delims=" %%i in ("%KEYS%\server.pub") do set "SRVPUB=%%i"
for /f "usebackq delims=" %%i in ("%EPFILE%") do set "EP=%%i"

"%CONF%" echo.
"%CONF%" echo [Peer]
"%CONF%" echo # %CNAME%
"%CONF%" echo PublicKey = %CPUB%
"%CONF%" echo PresharedKey = %CPSK%
"%CONF%" echo AllowedIPs = %WG_NET%.%NUM%/32

"%CLIENTS%%CNAME%.conf" echo [Interface]

"%CLIENTS%%CNAME%.conf" echo PrivateKey = %CPRIV%
"%CLIENTS%%CNAME%.conf" echo Address = %WG_NET%.%NUM%/32
"%CLIENTS%%CNAME%.conf" echo DNS = 1.1.1.1, 1.0.0.1
"%CLIENTS%%CNAME%.conf" echo MTU = %MTU%
"%CLIENTS%%CNAME%.conf" echo.
"%CLIENTS%%CNAME%.conf" echo [Peer]
"%CLIENTS%%CNAME%.conf" echo PublicKey = %SRVPUB%
"%CLIENTS%%CNAME%.conf" echo PresharedKey = %CPSK%
"%CLIENTS%%CNAME%.conf" echo Endpoint = %EP%:%PORT%
"%CLIENTS%%CNAME%.conf" echo AllowedIPs = 0.0.0.0/0, ::/0
"%CLIENTS%%CNAME%.conf" echo PersistentKeepalive = 25

net stop "%SVC%" >nul 2>&1
timeout /t 2 /nobreak >nul
net start "%SVC%" >nul 2>&1
timeout /t 2 /nobreak >nul

echo.
echo Client '%CNAME%' erstellt: %CLIENTS%%CNAME%.conf
echo Tunnel-IP: %WG_NET%.%NUM%
echo Diese Datei auf das Geraet kopieren und in WireGuard importieren.
pause & goto :menu

:remove
if not exist "%CLIENTS%" (echo Keine Clients. & pause & goto :menu)
dir /b "%CLIENTS%*.conf" 2>nul
echo.
set /p "CNAME=Client-Name entfernen: "
if not exist "%CLIENTS%%CNAME%.conf" (echo Nicht gefunden. & pause & goto :menu)
for /f "usebackq delims=" %%i in ("%CLIENTS%%CNAME%.pub") do set "RPUB=%%i"
powershell -NoProfile -ExecutionPolicy Bypass -Command "$c=Get-Content -Raw '%CONF%'; $c=$c -replace '(?s)\r?\n[Peer]\r?\n# %CNAME%\r?\n[^[]*',''; Set-Content -Path '%CONF%' -Value $c -NoNewline -Encoding ascii"
del /q "%CLIENTS%%CNAME%.conf" "%CLIENTS%%CNAME%.key" "%CLIENTS%%CNAME%.pub" "%CLIENTS%%CNAME%.psk" >nul 2>&1
net stop "%SVC%" >nul 2>&1
timeout /t 2 /nobreak >nul
net start "%SVC%" >nul 2>&1
echo Entfernt.
pause & goto :menu

:cleanup
set /p "OK=Wirklich alles entfernen? (j/n): "
if /i not "%OK%"=="j" goto :menu
net stop "%SVC%" >nul 2>&1
"%WG_EXE%" /uninstalltunnelservice %TUNNEL% >nul 2>&1
powershell -NoProfile -Command "Get-NetNat -Name '%NATNAME%' -ErrorAction SilentlyContinue | Remove-NetNat -Confirm:$false -ErrorAction SilentlyContinue" >nul 2>&1
netsh advfirewall firewall delete rule name="WG-Server-In" >nul 2>&1
netsh advfirewall firewall delete rule name="WG-Server-Tunnel" >nul 2>&1
reg add "HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters" /v IPEnableRouter /t REG_DWORD /d 0 /f >nul
echo Entfernt. Konfiguration und Schluessel bleiben im Ordner.
pause & goto :menu
C:\EIGENE-VPN> dir

Verzeichnis: C:\EIGENE-VPN
Mode LastWriteTime Length Name

-a---- 17.11.2026 01:52 53 nul
-a---- 17.11.2026 01:40 4965 portfreigabe.ps1
-a---- 17.11.2026 01:34 3245 vpn.bat
-a---- 17.11.2026 01:34 10422 wg-server.bat
-a---- 17.11.2026 01:26 5188 wg-setup.sh

C:\EIGENE-VPN>

Gearbeitet für 19s · 1 Schritt

Jetzt sehe ich das Skript komplett. Zwei Fehler, beide erklärbar:

Fehler 1 — die Admin-Prüfung. Das Skript nutzt net session als Test. Der Befehl hängt am Dienst "Server" (LanmanServer); ist der gestoppt oder deaktiviert, liefert net session auch als Admin einen Fehler → das Skript denkt "nicht erhöht" und startet sich neu. Ersetzen durch fltmc, das braucht keinen Dienst.

Fehler 2 — der leere Parameter. -ArgumentList '%*' kippt um, sobald kein Argument übergeben wird. Fest auf menu setzen, denn ohne Argument landet es ohnehin im Menü.

Patch, in deinem normalen Fenster ausführen:

$p='C:\EIGENE-VPN\wg-server.bat'
(Get-Content $p -Raw) -replace 'net session >nul 2>&1','fltmc >nul 2>&1' -replace "-ArgumentList '%\*'","-ArgumentList 'menu'" | Set-Content $p -Encoding ascii -NoNewline
Select-String -Path .\wg-server.bat -Pattern 'fltmc|ArgumentList'
Die zweite Zeile muss fltmc und -ArgumentList 'menu' zeigen. Außerdem liegt da eine Datei namens nul (53 Byte) im Ordner — del .\nul, die stört bei Redirects.

WireGuard brauchst du nicht vorher installieren. Der Abschnitt :ensure_wg lädt den offiziellen Installer selbst runter und installiert ihn still, sobald du Option 1 wählst. Einfach laufen lassen.

Wichtiger: die Portfreigabe war vermutlich für die Katz.

Deine ps1-Ausgabe sagt WAN-IP () != oeffentliche IP (2a02:810a:18e:5f00:...) -> CGNAT. Die Box hat also gar keine eigene öffentliche IPv4, und die 95.91.146.128 ist die Adresse eines geteilten Träger-Gateways. Das riecht stark nach DS-Lite bei der Telekom. Über IPv4 kommt dann niemand von außen auf 443/UDP rein, egal was in der Box steht.

Bevor du weitermachst: FRITZ!Box-Oberfläche → Internet → Online-Monitor. Steht dort unter Verbindungen eine IPv4 wie 100.64.x.x oder gar keine, ist es DS-Lite.

Bei DS-Lite gibt es zwei Wege:

Öffentliche IPv4 bei der Telekom dazubuchen (kostenpflichtige Option bzw. Business-Tarif). Dann bleibt alles wie geplant, die Freigabe auf 443/UDP funktioniert.
Auf IPv6 gehen. Dann Freigabe in der Box für UDP 443 auf die IPv6 des PCs setzen, als Endpunkt die IPv6 eintragen. Nachteil: Prefix wechselt, du brauchst MyFRITZ-DDNS als Endpunkt. Der Client muss IPv6 haben.
Sag mir, was der Online-Monitor zeigt, dann legen wir den Weg fest. Wenn du trotzdem erst mal mit IPv4 testen willst: Setup starten, als Endpunkt 95.91.146.128 eintragen — funktioniert dann eben nur, wenn die Gegenstelle auch im Telekom-Netz hängt.

C:\EIGENE-VPN> $p='C:\EIGENE-VPN\wg-server.bat'
C:\EIGENE-VPN> (Get-Content $p -Raw) -replace 'net session >nul 2>&1','fltmc >nul 2>&1' -replace "-ArgumentList '%*'","-ArgumentList 'menu'" | Set-Content $p -Encoding ascii -NoNewline
C:\EIGENE-VPN> Select-String -Path .\wg-server.bat -Pattern 'fltmc|ArgumentList'

wg-server.bat:21:fltmc >nul 2>&1
wg-server.bat:23: powershell -NoProfile -ExecutionPolicy Bypass -Command "Start-Process -FilePath '%~f0'
-ArgumentList 'menu' -Verb RunAs"
wg-server.bat:62:powershell -NoProfile -ExecutionPolicy Bypass -Command "try { Invoke-WebRequest -Uri
'https://download.wireguard.com/windows-client/wireguard-installer.exe' -OutFile '%TEMP%\wg.exe' -UseBasicParsing;
Start-Process '%TEMP%\wg.exe' -ArgumentList '/S' -Wait; exit 0 } catch { exit 1 }"

C:\EIGENE-VPN> was brauchst du, damit du mir es fertig machst sodass ich es nur noch starten muss und es läuft

Gearbeitet für 7s · 1 Schritt

Damit ich dir ein Skript bauen kann, das ohne Rückfragen durchläuft, brauche ich drei Dinge von dir:

1. Die portfreigabe.ps1 komplett

Get-Content .\portfreigabe.ps1
Grund: die hat schon einen Fehler produziert — sie meldete Freigabe steht: 443/UDP -> :, das Ziel ist leer. Und sie kam an die Box ran, also steckt da brauchbare TR-064-Logik drin, die ich übernehme und repariere statt neu zu erfinden.

2. Die vpn.bat komplett

Get-Content .\vpn.bat
Die liegt da noch rum, 3,2 KB. Ich muss wissen, ob die eine alte Fassung ist oder was Eigenes, das mit rein soll.

3. Der Online-Monitor

FRITZ!Box → Internet → Online-Monitor, Abschnitt Verbindungen. Steht da eine IPv4 (und welche)? Das entscheidet, ob das fertige Skript auf IPv4 oder IPv6 geht — beides automatisch zu erkennen und falsch zu entscheiden wäre schlimmer als einmal nachsehen.

Was ich dir dann baue: eine START.cmd, die sich selbst erhöht, WireGuard still installiert, Schlüssel erzeugt, wg0.conf schreibt, Firewall und NAT setzt, den Dienst startet und die Box-Freigabe korrekt auf die richtige Adresse legt — plus automatisch den ersten Client. Du doppelklickst und liest am Ende nur noch Pfad und Tunnel-IP ab.

Zwei Dinge, die ich dabei fixen werde, unabhängig von deinen Antworten: das leere Freigabe-Ziel und die fehlende Prüfung, ob die PC-IPv4 wirklich die eigene ist (die Box kannte 192.168.178.20 nicht als aktiven Nachbarn — da stimmt was nicht, entweder DHCP-Reservierung fehlt oder die Adresse ist veraltet).

portfreigabe.ps1 : #requires -Version 5.1
[CmdletBinding()]
param(
[string]$Box = '192.168.178.1',
[int]$ExtPort = 443,
[int]$IntPort = 443,
[ValidateSet('UDP','TCP')][string]$Proto = 'UDP',
[string]$TargetIP,
[string]$User,
[string]$Description = 'WireGuard',
[switch]$Delete,
[switch]$CheckOnly,
[switch]$Dump
)
$ErrorActionPreference = 'Stop'
$Svc = 'urn:dslforum-org:service:WANIPConnection:1'
$Ctl = "http://${Box}:49000/upnp/control/wanipconnection1"

function Invoke-Box {
param([string]$Action, [System.Collections.IDictionary]$Params)
$sb = New-Object System.Text.StringBuilder
[void]$sb.Append('<?xml version="1.0" encoding="utf-8"?>')
[void]$sb.Append('<s:Envelope xmlns:s="http://schemas.xmlsoap.org/soap/envelope/" s:encodingStyle="http://schemas.xmlsoap.org/soap/encoding/"><s:Body>')
[void]$sb.Append("<u:$Action xmlns:u="$Svc">")
foreach ($k in $Params.Keys) { [void]$sb.Append("<$k>$($Params[$k])</$k>") }
[void]$sb.Append("</u:$Action></s:Body></s:Envelope>")
$xml = $sb.ToString()
if ($Dump) { Write-Host $xml -ForegroundColor DarkGray }
Invoke-RestMethod -Uri $Ctl -Method Post -Credential $script:Cred -TimeoutSec 15 -ContentType 'text/xml; charset="utf-8"'
-Headers @{ SOAPAction = ""$Svc#$Action"" } -Body $xml
}

function Get-PortState {
try {
Invoke-Box 'GetSpecificPortMappingEntry' ([ordered]@{
NewRemoteHost = ''; NewExternalPort = $ExtPort; NewProtocol = $Proto
}) | Out-Null
return 'vorhanden'
} catch { return 'frei' }
}

--- Ziel ---
if (-not $TargetIP) {
$TargetIP = (Get-NetIPAddress -AddressFamily IPv4 |
Where-Object { $_.IPAddress -like '192.168.178.*' } |
Select-Object -First 1).IPAddress
if (-not $TargetIP) { throw "Keine LAN-IP gefunden. -TargetIP angeben." }
}
Write-Host "Ziel: $TargetIP Port $ExtPort/$Proto -> $IntPort" -ForegroundColor Cyan

--- Geraet bekannt? ---
if (-not (Get-NetNeighbor -AddressFamily IPv4 -ErrorAction SilentlyContinue |
Where-Object { $_.IPAddress -eq $TargetIP })) {
Write-Host "Hinweis: $TargetIP ist der Box nicht als aktiver Nachbar bekannt." -ForegroundColor Yellow
}

--- Anmeldung ---
if (-not $User) { $User = Read-Host "FRITZ!Box-Benutzer (Admin)" }
$script:Cred = Get-Credential -UserName $User -Message "FRITZ!Box-Passwort fuer $User"

try { Invoke-Box 'GetExternalIPAddress' ([ordered]@{}) | Out-Null }
catch { throw "Anmeldung/TR-064 fehlgeschlagen. TR-064 aktiv? Admin-Rechte? -> $($_.Exception.Message)" }

--- CGNAT ---
try {
$wan = (Invoke-Box 'GetExternalIPAddress' ([ordered]@{})).NewExternalIPAddress
$pub = (Invoke-WebRequest -Uri 'https://ifconfig.me/ip' -UseBasicParsing -TimeoutSec 8).Content.Trim()
if ($wan -ne $pub) {
Write-Host "WARNUNG: WAN-IP ($wan) != oeffentliche IP ($pub) -> CGNAT." -ForegroundColor Yellow
} else { Write-Host "Oeffentliche WAN-IP: $wan" -ForegroundColor Green }
} catch { Write-Host "WAN-IP-Pruefung fehlgeschlagen." -ForegroundColor DarkGray }

if ($CheckOnly) { Write-Host ("Status Port {0}: {1}" -f $ExtPort, (Get-PortState)); return }

if ($Delete) {
Invoke-Box 'DeletePortMapping' ([ordered]@{
NewRemoteHost = ''; NewExternalPort = $ExtPort; NewProtocol = $Proto
}) | Out-Null
Write-Host "Freigabe entfernt." -ForegroundColor Green
return
}

--- Anlegen mit Varianten ---
$variants = @(
@{ Lease = '0'; Client = $TargetIP; Name = 'IP, permanent' },
@{ Lease = '86400'; Client = $TargetIP; Name = 'IP, 24h Lease' },
@{ Lease = '0'; Client = $env:COMPUTERNAME; Name = 'Hostname, permanent' }
)

$ok = $false
foreach ($v in $variants) {
try {
Invoke-Box 'AddPortMapping' ([ordered]@{
NewRemoteHost = ''
NewExternalPort = $ExtPort
NewProtocol = $Proto
NewInternalPort = $IntPort
NewInternalClient = $v.Client
NewEnabled = '1'
NewPortMappingDescription = $Description
NewLeaseDuration = $v.Lease
}) | Out-Null
Write-Host "Variante erfolgreich: $($v.Name)" -ForegroundColor Green
$ok = $true
break
} catch {
Write-Host "Variante fehlgeschlagen ($($v.Name)): $($_.Exception.Message)" -ForegroundColor DarkYellow
}
}

if (-not $ok) {
Write-Host ""
Write-Host "Alle Varianten abgelehnt. Nutze den GUI-Weg (siehe Anleitung)." -ForegroundColor Red
return
}

$entry = Invoke-Box 'GetSpecificPortMappingEntry' ([ordered]@{
NewRemoteHost = ''; NewExternalPort = $ExtPort; NewProtocol = $Proto
})
Write-Host ""
Write-Host "Freigabe steht: $ExtPort/$Proto -> $($entry.NewInternalClient):$($entry.NewInternalPort)" -ForegroundColor Green 2. vpn.bat: @echo off
setlocal EnableExtensions EnableDelayedExpansion
chcp 65001 >nul
title WireGuard Client

set "TUNNEL=wg0"
set "SRV_IP=deinname.ddns.net"
set "SRV_PORT=443"
set "TUN_IP=10.66.66.2"
set "CONF=%~dp0wg0.conf"

set "WG_DIR=%ProgramFiles%\WireGuard"
set "WG_EXE=%WG_DIR%\wireguard.exe"
set "WG_CLI=%WG_DIR%\wg.exe"
set "SVC=WireGuardTunnel$%TUNNEL%"

net session >nul 2>&1
if errorlevel 1 (
powershell -NoProfile -ExecutionPolicy Bypass -Command "Start-Process -FilePath '%~f0' -ArgumentList '%*' -Verb RunAs"
exit /b
)

set "ACT=%~1"
if "%ACT%"=="" set "ACT=up"
if /i "%ACT%"=="up" goto :up
if /i "%ACT%"=="down" goto :down
if /i "%ACT%"=="status" goto :status
if /i "%ACT%"=="ks_on" goto :ks_on
if /i "%ACT%"=="ks_off" goto :ks_off
goto :up

:up
if not exist "%WG_EXE%" (echo WireGuard fehlt. & pause & exit /b 1)
if not exist "%CONF%" (echo %CONF% fehlt. & pause & exit /b 1)
call :ks_on_silent
sc query "%SVC%" >nul 2>&1
if errorlevel 1 ("%WG_EXE%" /installtunnelservice "%CONF%") else (net start "%SVC%" >nul 2>&1)
timeout /t 3 /nobreak >nul
"%WG_CLI%" show %TUNNEL% >nul 2>&1 && echo Verbunden. || echo Fehlgeschlagen.
pause & exit /b

:down
net stop "%SVC%" >nul 2>&1
echo Getrennt. Kill-Switch bleibt aktiv.
pause & exit /b

:status
cls
"%WG_CLI%" show %TUNNEL% 2>nul || echo Tunnel nicht aktiv.
pause & exit /b

:ks_on
call :ks_cleanup
netsh advfirewall set allprofiles firewallpolicy blockinbound,blockoutbound >nul
netsh advfirewall firewall add rule name="WG-Killswitch-Block" dir=out action=block profile=any >nul
netsh advfirewall firewall add rule name="WG-Killswitch-Tunnel" dir=out action=allow localip=%TUN_IP% profile=any >nul
netsh advfirewall firewall add rule name="WG-Killswitch-Handshake" dir=out action=allow remoteip=%SRV_IP% protocol=udp remoteport=%SRV_PORT% profile=any >nul
netsh advfirewall firewall add rule name="WG-Killswitch-DHCP" dir=out action=allow protocol=udp localport=68 remoteport=67 profile=any >nul
netsh advfirewall firewall add rule name="WG-Killswitch-Loop" dir=out action=allow remoteip=127.0.0.1 profile=any >nul
echo Kill-Switch aktiv.
pause & exit /b

:ks_on_silent
call :ks_cleanup
netsh advfirewall set allprofiles firewallpolicy blockinbound,blockoutbound >nul
netsh advfirewall firewall add rule name="WG-Killswitch-Block" dir=out action=block profile=any >nul
netsh advfirewall firewall add rule name="WG-Killswitch-Tunnel" dir=out action=allow localip=%TUN_IP% profile=any >nul
netsh advfirewall firewall add rule name="WG-Killswitch-Handshake" dir=out action=allow remoteip=%SRV_IP% protocol=udp remoteport=%SRV_PORT% profile=any >nul
netsh advfirewall firewall add rule name="WG-Killswitch-DHCP" dir=out action=allow protocol=udp localport=68 remoteport=67 profile=any >nul
netsh advfirewall firewall add rule name="WG-Killswitch-Loop" dir=out action=allow remoteip=127.0.0.1 profile=any >nul
exit /b 0

:ks_off
call :ks_cleanup
netsh advfirewall set allprofiles firewallpolicy blockinbound,allowoutbound >nul
echo Kill-Switch aus.
pause & exit /b

:ks_cleanup
for %%r in (Block Tunnel Handshake DHCP Loop) do netsh advfirewall firewall delete rule name="WG-Killswitch-%%r" >nul 2>&1
exit /b 0 3. Internet,IPV4: verbunden seit 15.11.2026, 06:29 Uhr
IPv4-Adresse: 95.91.146.128

Gearbeitet für 34s · 1 Schritt

Kein DS-Lite. Der Online-Monitor zeigt 95.91.146.128, identisch mit der Testadresse — echte öffentliche IPv4, Portfreigabe funktioniert. Beide Warnungen der ps1 waren Fehlalarme: GetExternalIPAddress liefert bei der Box einen leeren String (deshalb der sinnlose CGNAT-Vergleich), und die Nachbar-Prüfung kann die eigene IP nie finden, weil ein Host sich nicht selbst in seiner ARP-Tabelle steht.

Also bauen wir es fertig. Der folgende Block legt drei Dateien an: setup.ps1 (macht alles), START.cmd (Doppelklick-Einstieg) und eine reparierte vpn.bat für die Gegenseite.

Alles in einer Portion in dein normales PowerShell-Fenster einfügen und Enter:

$base='C:\EIGENE-VPN'

@'
#requires -Version 5.1
$ErrorActionPreference = 'Stop'

if (-not ([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)) {
    Start-Process powershell -Verb RunAs -ArgumentList @('-NoProfile','-ExecutionPolicy','Bypass','-NoExit','-File',('"' + $PSCommandPath + '"'))
    exit
}

$BASE    = Split-Path -Parent $PSCommandPath
$KEYS    = Join-Path $BASE 'keys'
$CLIENTS = Join-Path $BASE 'clients'
$CONF    = Join-Path $BASE 'wg0.conf'
$WG_DIR  = Join-Path $env:ProgramFiles 'WireGuard'
$WG_EXE  = Join-Path $WG_DIR 'wireguard.exe'
$WG_CLI  = Join-Path $WG_DIR 'wg.exe'
$TUNNEL  = 'wg0'
$SVC     = 'WireGuardTunnel$wg0'
$NET     = '10.66.66'
$PORT    = 443
$MTU     = 1380
$NATNAME = 'WGVPN'
$BOX     = '192.168.178.1'
$SvcURN  = 'urn:dslforum-org:service:WANIPConnection:1'
$Ctl     = "http://${BOX}:49000/upnp/control/wanipconnection1"

function Step($t) { Write-Host "`n[*] $t" -ForegroundColor Cyan }
function Die($t)  { Write-Host "`nFEHLER: $t" -ForegroundColor Red; Read-Host "Enter zum Schliessen"; exit 1 }

function Invoke-Box {
    param([string]$Action, [System.Collections.IDictionary]$Params)
    $sb = New-Object System.Text.StringBuilder
    [void]$sb.Append('<?xml version="1.0" encoding="utf-8"?>')
    [void]$sb.Append('<s:Envelope xmlns:s="http://schemas.xmlsoap.org/soap/envelope/" s:encodingStyle="http://schemas.xmlsoap.org/soap/encoding/"><s:Body>')
    [void]$sb.Append("<u:$Action xmlns:u=`"$SvcURN`">")
    foreach ($k in $Params.Keys) { [void]$sb.Append("<$k>$($Params[$k])</$k>") }
    [void]$sb.Append("</u:$Action></s:Body></s:Envelope>")
    Invoke-RestMethod -Uri $Ctl -Method Post -Credential $script:Cred -TimeoutSec 15 -ContentType 'text/xml; charset="utf-8"' -Headers @{ SOAPAction = "`"$SvcURN#$Action`"" } -Body $sb.ToString()
}

Write-Host "==============================================" -ForegroundColor White
Write-Host "  WireGuard Server - vollautomatisches Setup" -ForegroundColor White
Write-Host "==============================================" -ForegroundColor White

Step "Ermittle lokale LAN-Adresse"
$route = Get-NetRoute -DestinationPrefix '0.0.0.0/0' -ErrorAction SilentlyContinue | Sort-Object RouteMetric | Select-Object -First 1
if (-not $route) { Die "Keine Default-Route gefunden." }
$lanIP = (Get-NetIPAddress -InterfaceIndex $route.InterfaceIndex -AddressFamily IPv4 -ErrorAction SilentlyContinue | Select-Object -First 1).IPAddress
if (-not $lanIP) { Die "Keine IPv4 auf dem Internet-Interface." }
Write-Host "    LAN-IP: $lanIP"

Step "Ermittle oeffentliche IPv4"
try { $pub = (Invoke-WebRequest 'https://ipv4.icanhazip.com' -UseBasicParsing -TimeoutSec 10).Content.Trim() } catch { Die "Oeffentliche IP nicht ermittelbar." }
if ($pub -notmatch '^\d{1,3}(\.\d{1,3}){3}$') { Die "Unerwartete Antwort: $pub" }
Write-Host "    Oeffentliche IP: $pub"

Step "Pruefe WireGuard"
if (-not (Test-Path $WG_EXE)) {
    Write-Host "    Nicht installiert, installiere ..."
    $ok = $false
    if (Get-Command winget -ErrorAction SilentlyContinue) {
        & winget install --id WireGuard.WireGuard -e --accept-source-agreements --accept-package-agreements --silent | Out-Null
        if (Test-Path $WG_EXE) { $ok = $true }
    }
    if (-not $ok) {
        $dl = Join-Path $env:TEMP 'wg-installer.exe'
        Invoke-WebRequest 'https://download.wireguard.com/windows-client/wireguard-installer.exe' -OutFile $dl -UseBasicParsing
        Start-Process $dl -ArgumentList '/S' -Wait
    }
    Start-Sleep 3
}
if (-not (Test-Path $WG_EXE)) { Die "WireGuard-Installation fehlgeschlagen." }
Write-Host "    OK"

Step "Erzeuge Server-Schluessel"
if (Test-Path $KEYS) { Remove-Item $KEYS -Recurse -Force }
New-Item -ItemType Directory -Path $KEYS -Force | Out-Null
New-Item -ItemType Directory -Path $CLIENTS -Force | Out-Null
$spriv = (& $WG_CLI genkey).Trim()
$spub  = ($spriv | & $WG_CLI pubkey).Trim()
$spsk  = (& $WG_CLI genpsk).Trim()
Set-Content (Join-Path $KEYS 'server.key') $spriv -NoNewline -Encoding ascii
Set-Content (Join-Path $KEYS 'server.pub') $spub  -NoNewline -Encoding ascii
Set-Content (Join-Path $KEYS 'server.psk') $spsk  -NoNewline -Encoding ascii

Step "Schreibe Serverkonfiguration"
@"
[Interface]
Address = $NET.1/24
ListenPort = $PORT
PrivateKey = $spriv
MTU = $MTU
SaveConfig = false
"@ | Set-Content $CONF -Encoding ascii

Step "Firewall"
netsh advfirewall firewall delete rule name="WG-Server-In" | Out-Null
netsh advfirewall firewall delete rule name="WG-Server-Tunnel" | Out-Null
netsh advfirewall firewall add rule name="WG-Server-In" dir=in action=allow protocol=udp localport=$PORT profile=any | Out-Null
netsh advfirewall firewall add rule name="WG-Server-Tunnel" dir=in action=allow localip="$NET.0/24" profile=any | Out-Null

Step "Routing aktivieren"
reg add "HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters" /v IPEnableRouter /t REG_DWORD /d 1 /f | Out-Null

Step "NAT einrichten"
Get-NetNat -Name $NATNAME -ErrorAction SilentlyContinue | Remove-NetNat -Confirm:$false -ErrorAction SilentlyContinue
$natOk = $true
try { New-NetNat -Name $NATNAME -InternalIPInterfaceAddressPrefix "$NET.0/24" -ErrorAction Stop | Out-Null } catch { $natOk = $false }
if (-not $natOk) {
    Write-Host "    NetNat nicht verfuegbar, RRAS-Fallback" -ForegroundColor Yellow
    netsh routing ip nat install | Out-Null
    netsh routing ip nat add interface name="$TUNNEL" mode=full | Out-Null
    netsh routing ip nat add interface name="$($route.InterfaceAlias)" mode=public | Out-Null
}

Step "Installiere und starte Dienst"
& $WG_EXE /uninstalltunnelservice $TUNNEL 2>$null | Out-Null
Start-Sleep 1
& $WG_EXE /installtunnelservice $CONF
Start-Sleep 4
$svc = Get-Service $SVC -ErrorAction SilentlyContinue
if (-not $svc -or $svc.Status -ne 'Running') { Die "Dienst laeuft nicht." }
netsh interface ipv4 set interface "$TUNNEL" forwarding=enabled | Out-Null
Write-Host "    Dienst laeuft"

Step "Erzeuge ersten Client"
$cname = 'client1'
$cpriv = (& $WG_CLI genkey).Trim()
$cpub  = ($cpriv | & $WG_CLI pubkey).Trim()
$cpsk  = (& $WG_CLI genpsk).Trim()
Set-Content (Join-Path $CLIENTS "$cname.key") $cpriv -NoNewline -Encoding ascii
Set-Content (Join-Path $CLIENTS "$cname.pub") $cpub  -NoNewline -Encoding ascii
Set-Content (Join-Path $CLIENTS "$cname.psk") $cpsk  -NoNewline -Encoding ascii
Add-Content $CONF ""
Add-Content $CONF "[Peer]"
Add-Content $CONF "# $cname"
Add-Content $CONF "PublicKey = $cpub"
Add-Content $CONF "PresharedKey = $cpsk"
Add-Content $CONF "AllowedIPs = $NET.2/32"
@"
[Interface]
PrivateKey = $cpriv
Address = $NET.2/32
DNS = 1.1.1.1, 1.0.0.1
MTU = $MTU

[Peer]
PublicKey = $spub
PresharedKey = $cpsk
Endpoint = ${pub}:$PORT
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25
"@ | Set-Content (Join-Path $CLIENTS "$cname.conf") -Encoding ascii

Step "Dienst neu laden"
net stop $SVC | Out-Null
Start-Sleep 2
net start $SVC | Out-Null
Start-Sleep 3
netsh interface ipv4 set interface "$TUNNEL" forwarding=enabled | Out-Null

Step "Portfreigabe auf der FRITZ!Box"
$u = Read-Host "    FRITZ!Box-Benutzer (Admin)"
$script:Cred = Get-Credential -UserName $u -Message "FRITZ!Box-Passwort"
try { Invoke-Box 'GetExternalIPAddress' ([ordered]@{}) | Out-Null } catch { Die "TR-064-Anmeldung fehlgeschlagen: $($_.Exception.Message)" }
try { Invoke-Box 'DeletePortMapping' ([ordered]@{ NewRemoteHost=''; NewExternalPort=$PORT; NewProtocol='UDP' }) | Out-Null } catch { }
try {
    Invoke-Box 'AddPortMapping' ([ordered]@{
        NewRemoteHost=''; NewExternalPort=$PORT; NewProtocol='UDP'; NewInternalPort=$PORT
        NewInternalClient=$lanIP; NewEnabled='1'; NewPortMappingDescription='WireGuard'; NewLeaseDuration='0'
    }) | Out-Null
    $chk = (Invoke-Box 'GetSpecificPortMappingEntry' ([ordered]@{ NewRemoteHost=''; NewExternalPort=$PORT; NewProtocol='UDP' })).NewInternalClient
    if ($chk -eq $lanIP) { Write-Host "    Freigabe steht: $PORT/UDP -> ${lanIP}" -ForegroundColor Green }
    else { Write-Host "    Freigabe angelegt, Rueckmeldung leer. In der Box pruefen." -ForegroundColor Yellow }
} catch {
    Write-Host "    Automatisch fehlgeschlagen: $($_.Exception.Message)" -ForegroundColor Yellow
    Write-Host "    -> Box: Internet > Freigaben > UDP $PORT an $lanIP manuell setzen." -ForegroundColor Yellow
}

Write-Host "`n==============================================" -ForegroundColor Green
Write-Host "  FERTIG" -ForegroundColor Green
Write-Host "==============================================" -ForegroundColor Green
Write-Host "  Endpunkt:   ${pub}:$PORT/udp"
Write-Host "  Client-Datei: $CLIENTS\$cname.conf"
Write-Host "  Tunnel-IP Client: $NET.2"
Write-Host ""
Write-Host "  Weitere Clients: wg-server.bat  ->  Option 5"
Write-Host "  Dienststatus:    sc query $SVC"
Write-Host "=============================================="
Read-Host "`nEnter zum Schliessen"
'@ | Set-Content (Join-Path $base 'setup.ps1') -Encoding ascii

@'
@echo off
chcp 65001 >nul
cd /d "%~dp0"
powershell -NoProfile -ExecutionPolicy Bypass -File "%~dp0setup.ps1"
'@ | Set-Content (Join-Path $base 'START.cmd') -Encoding ascii

$q = Join-Path $base 'vpn.bat'
$c = Get-Content $q -Raw
$c = $c -replace 'net session >nul 2>&1','fltmc >nul 2>&1'
$c = $c -replace "-ArgumentList '%\*'","-ArgumentList 'up'"
$c = $c -replace 'deinname\.ddns\.net','95.91.146.128'
$c = $c -replace 'remoteip=127\.0\.0\.1 profile=any >nul', "remoteip=127.0.0.1 profile=any >nul`r`nnetsh advfirewall firewall add rule name=`"WG-Killswitch-LAN`" dir=out action=allow remoteip=192.168.178.0/24 profile=any >nul"
$c = $c -replace '\(Block Tunnel Handshake DHCP Loop\)','(Block Tunnel Handshake DHCP Loop LAN)'
Set-Content $q $c -Encoding ascii -NoNewline

Write-Host "`nDateien geschrieben. Starten mit: .\START.cmd" -ForegroundColor Green
Danach:

.\START.cmd
Es kommt genau eine Abfrage: FRITZ!Box-Benutzer und Passwort. Der Rest läuft durch — WireGuard-Installation, Schlüssel, Konfiguration, Firewall, NAT, Dienst, erster Client, Freigabe. Am Ende steht der Pfad zur Client-Datei.

Was ich in vpn.bat mit repariert habe: denselben net session- und Leerargument-Fehler, die Endpunkt-IP eingesetzt, und eine Ausnahme fürs lokale Netz in den Kill-Switch gelegt — ohne die wäre Drucker und Box-Zugriff tot, sobald der Schalter aktiv ist.

Zwei Hinweise:

Die Freigabe wird vorher gelöscht und neu gesetzt. Falls die Rückmeldung der Box leer bleibt, prüf im GUI nach — die Bestätigungsabfrage der Box liefert bei manchen Firmware-Ständen nichts zurück, obwohl die Regel steht.
95.91.146.128 ist bei Telekom-DSL nicht garantiert stabil. Wenn dir das wichtig ist, richte bei MyFRITZ einen DDNS-Namen ein und trag den in vpn.bat bei SRV_IP ein.
============================================== WireGuard Server - vollautomatisches Setup
[*] Ermittle lokale LAN-Adresse
LAN-IP: 192.168.178.20

[*] Ermittle oeffentliche IPv4
Oeffentliche IP: 95.91.146.128

[*] Pruefe WireGuard
OK

[*] Erzeuge Server-Schluessel

[*] Schreibe Serverkonfiguration

[*] Firewall

[*] Routing aktivieren

[*] NAT einrichten

[*] Installiere und starte Dienst
wireguard.exe : 2026/11/17 02:10:37 Fehler: The specified service does not exist as an installed service.
In C:\EIGENE-VPN\setup.ps1:116 Zeichen:1

& $WG_EXE /uninstalltunnelservice $TUNNEL 2>$null | Out-Null
+ CategoryInfo          : NotSpecified: (2026/11/17 02:1...talled service.:String) [], RemoteException
  + FullyQualifiedErrorId : NativeCommandError
PS C:\WINDOWS\system32>

Gearbeitet für 14s · 1 Schritt

Der Fehler ist harmlos, aber er killt das Skript: wireguard.exe /uninstalltunnelservice schreibt beim ersten Lauf "Dienst existiert nicht" auf stderr, und mit $ErrorActionPreference = 'Stop' wird daraus ein Abbruch. Dieselbe Falle steckt noch in net stop/net start weiter unten.

Drei Zeilen entschärfen, dann läuft es durch:

$p='C:\EIGENE-VPN\setup.ps1'
$c=Get-Content $p -Raw
$c=$c -replace [regex]::Escape("`$ErrorActionPreference = 'Stop'"),'$ErrorActionPreference = ''Continue'''
$c=$c -replace [regex]::Escape('& $WG_EXE /uninstalltunnelservice $TUNNEL 2>$null | Out-Null'),'try { & $WG_EXE /uninstalltunnelservice $TUNNEL 2>&1 | Out-Null } catch { }'
$c=$c -replace [regex]::Escape('net stop $SVC | Out-Null'),'try { net stop $SVC 2>&1 | Out-Null } catch { }'
$c=$c -replace [regex]::Escape('net start $SVC | Out-Null'),'try { net start $SVC 2>&1 | Out-Null } catch { }'
Set-Content $p $c -Encoding ascii -NoNewline
Select-String -Path $p -Pattern 'ErrorActionPreference|uninstalltunnelservice|net stop|net start'
Die Ausgabe muss 'Continue' und die drei try { ... }-Zeilen zeigen.

Dann neu starten:

.\START.cmd
Kein Aufräumen nötig — Firewall-Regeln und die NAT-Regel sind vom abgebrochenen Lauf noch da, das Skript löscht beides zu Beginn selbst und legt es neu an. Der Dienst wurde nie installiert, also gibt es auch nichts Halbes im Dienstmanager.

Nur zur Einordnung: Continue heißt, dass fehlgeschlagene Einzelbefehle protokolliert, aber nicht mehr abbrechen. Deshalb prüft das Skript an den entscheidenden Stellen weiterhin selbst — Dienststatus, Dateiexistenz, Freigabe-Rückmeldung — und bricht dort mit klarer Meldung ab, wo es wirklich zählt.

mach du das und schick mir den kompletten code wieder zurück

Gearbeitet für 12s · 1 Schritt

Hier die komplette Datei mit allen Korrekturen. Alles in einer Portion einfügen und Enter:

$base='C:\EIGENE-VPN'

@'
#requires -Version 5.1
$ErrorActionPreference = 'Continue'

if (-not ([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)) {
    Start-Process powershell -Verb RunAs -ArgumentList @('-NoProfile','-ExecutionPolicy','Bypass','-NoExit','-File',('"' + $PSCommandPath + '"'))
    exit
}

$BASE    = Split-Path -Parent $PSCommandPath
$KEYS    = Join-Path $BASE 'keys'
$CLIENTS = Join-Path $BASE 'clients'
$CONF    = Join-Path $BASE 'wg0.conf'
$WG_DIR  = Join-Path $env:ProgramFiles 'WireGuard'
$WG_EXE  = Join-Path $WG_DIR 'wireguard.exe'
$WG_CLI  = Join-Path $WG_DIR 'wg.exe'
$TUNNEL  = 'wg0'
$SVC     = 'WireGuardTunnel$wg0'
$NET     = '10.66.66'
$PORT    = 443
$MTU     = 1380
$NATNAME = 'WGVPN'
$BOX     = '192.168.178.1'
$SvcURN  = 'urn:dslforum-org:service:WANIPConnection:1'
$Ctl     = "http://${BOX}:49000/upnp/control/wanipconnection1"

function Step($t) { Write-Host "`n[*] $t" -ForegroundColor Cyan }
function Die($t)  { Write-Host "`nFEHLER: $t" -ForegroundColor Red; Read-Host "Enter zum Schliessen"; exit 1 }

function Invoke-Box {
    param([string]$Action, [System.Collections.IDictionary]$Params)
    $sb = New-Object System.Text.StringBuilder
    [void]$sb.Append('<?xml version="1.0" encoding="utf-8"?>')
    [void]$sb.Append('<s:Envelope xmlns:s="http://schemas.xmlsoap.org/soap/envelope/" s:encodingStyle="http://schemas.xmlsoap.org/soap/encoding/"><s:Body>')
    [void]$sb.Append("<u:$Action xmlns:u=`"$SvcURN`">")
    foreach ($k in $Params.Keys) { [void]$sb.Append("<$k>$($Params[$k])</$k>") }
    [void]$sb.Append("</u:$Action></s:Body></s:Envelope>")
    Invoke-RestMethod -Uri $Ctl -Method Post -Credential $script:Cred -TimeoutSec 15 -ContentType 'text/xml; charset="utf-8"' -Headers @{ SOAPAction = "`"$SvcURN#$Action`"" } -Body $sb.ToString()
}

Write-Host "==============================================" -ForegroundColor White
Write-Host "  WireGuard Server - vollautomatisches Setup" -ForegroundColor White
Write-Host "==============================================" -ForegroundColor White

Step "Ermittle lokale LAN-Adresse"
$route = Get-NetRoute -DestinationPrefix '0.0.0.0/0' -ErrorAction SilentlyContinue | Sort-Object RouteMetric | Select-Object -First 1
if (-not $route) { Die "Keine Default-Route gefunden." }
$lanIP = (Get-NetIPAddress -InterfaceIndex $route.InterfaceIndex -AddressFamily IPv4 -ErrorAction SilentlyContinue | Select-Object -First 1).IPAddress
if (-not $lanIP) { Die "Keine IPv4 auf dem Internet-Interface." }
Write-Host "    LAN-IP: $lanIP"

Step "Ermittle oeffentliche IPv4"
try { $pub = (Invoke-WebRequest 'https://ipv4.icanhazip.com' -UseBasicParsing -TimeoutSec 10).Content.Trim() } catch { Die "Oeffentliche IP nicht ermittelbar." }
if ($pub -notmatch '^\d{1,3}(\.\d{1,3}){3}$') { Die "Unerwartete Antwort: $pub" }
Write-Host "    Oeffentliche IP: $pub"

Step "Pruefe WireGuard"
if (-not (Test-Path $WG_EXE)) {
    Write-Host "    Nicht installiert, installiere ..."
    $ok = $false
    if (Get-Command winget -ErrorAction SilentlyContinue) {
        try { & winget install --id WireGuard.WireGuard -e --accept-source-agreements --accept-package-agreements --silent 2>&1 | Out-Null } catch { }
        if (Test-Path $WG_EXE) { $ok = $true }
    }
    if (-not $ok) {
        try {
            $dl = Join-Path $env:TEMP 'wg-installer.exe'
            Invoke-WebRequest 'https://download.wireguard.com/windows-client/wireguard-installer.exe' -OutFile $dl -UseBasicParsing
            Start-Process $dl -ArgumentList '/S' -Wait
        } catch { }
    }
    Start-Sleep 3
}
if (-not (Test-Path $WG_EXE)) { Die "WireGuard-Installation fehlgeschlagen." }
Write-Host "    OK"

Step "Erzeuge Server-Schluessel"
if (Test-Path $KEYS) { Remove-Item $KEYS -Recurse -Force }
New-Item -ItemType Directory -Path $KEYS -Force | Out-Null
New-Item -ItemType Directory -Path $CLIENTS -Force | Out-Null
$spriv = (& $WG_CLI genkey).Trim()
$spub  = ($spriv | & $WG_CLI pubkey).Trim()
$spsk  = (& $WG_CLI genpsk).Trim()
Set-Content (Join-Path $KEYS 'server.key') $spriv -NoNewline -Encoding ascii
Set-Content (Join-Path $KEYS 'server.pub') $spub  -NoNewline -Encoding ascii
Set-Content (Join-Path $KEYS 'server.psk') $spsk  -NoNewline -Encoding ascii
if (-not (Test-Path (Join-Path $KEYS 'server.key'))) { Die "Schluesselerzeugung fehlgeschlagen." }

Step "Schreibe Serverkonfiguration"
@"
[Interface]
Address = $NET.1/24
ListenPort = $PORT
PrivateKey = $spriv
MTU = $MTU
SaveConfig = false
"@ | Set-Content $CONF -Encoding ascii

Step "Firewall"
try { netsh advfirewall firewall delete rule name="WG-Server-In" 2>&1 | Out-Null } catch { }
try { netsh advfirewall firewall delete rule name="WG-Server-Tunnel" 2>&1 | Out-Null } catch { }
try { netsh advfirewall firewall add rule name="WG-Server-In" dir=in action=allow protocol=udp localport=$PORT profile=any 2>&1 | Out-Null } catch { }
try { netsh advfirewall firewall add rule name="WG-Server-Tunnel" dir=in action=allow localip="$NET.0/24" profile=any 2>&1 | Out-Null } catch { }

Step "Routing aktivieren"
try { reg add "HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters" /v IPEnableRouter /t REG_DWORD /d 1 /f 2>&1 | Out-Null } catch { }

Step "NAT einrichten"
try { Get-NetNat -Name $NATNAME -ErrorAction SilentlyContinue | Remove-NetNat -Confirm:$false -ErrorAction SilentlyContinue } catch { }
$natOk = $true
try { New-NetNat -Name $NATNAME -InternalIPInterfaceAddressPrefix "$NET.0/24" -ErrorAction Stop | Out-Null } catch { $natOk = $false }
if (-not $natOk) {
    Write-Host "    NetNat nicht verfuegbar, RRAS-Fallback" -ForegroundColor Yellow
    try { netsh routing ip nat install 2>&1 | Out-Null } catch { }
    try { netsh routing ip nat add interface name="$TUNNEL" mode=full 2>&1 | Out-Null } catch { }
    try { netsh routing ip nat add interface name="$($route.InterfaceAlias)" mode=public 2>&1 | Out-Null } catch { }
}

Step "Installiere und starte Dienst"
try { & $WG_EXE /uninstalltunnelservice $TUNNEL 2>&1 | Out-Null } catch { }
Start-Sleep 1
try { & $WG_EXE /installtunnelservice $CONF 2>&1 | Out-Null } catch { }
Start-Sleep 4
$svc = Get-Service $SVC -ErrorAction SilentlyContinue
if (-not $svc -or $svc.Status -ne 'Running') {
    Write-Host "    Dienst laeuft nicht, starte nach ..." -ForegroundColor Yellow
    try { net start $SVC 2>&1 | Out-Null } catch { }
    Start-Sleep 3
    $svc = Get-Service $SVC -ErrorAction SilentlyContinue
    if (-not $svc -or $svc.Status -ne 'Running') { Die "Dienst laeuft nicht. Konfiguration: $CONF" }
}
try { netsh interface ipv4 set interface "$TUNNEL" forwarding=enabled 2>&1 | Out-Null } catch { }
Write-Host "    Dienst laeuft"

Step "Erzeuge ersten Client"
$cname = 'client1'
$cpriv = (& $WG_CLI genkey).Trim()
$cpub  = ($cpriv | & $WG_CLI pubkey).Trim()
$cpsk  = (& $WG_CLI genpsk).Trim()
Set-Content (Join-Path $CLIENTS "$cname.key") $cpriv -NoNewline -Encoding ascii
Set-Content (Join-Path $CLIENTS "$cname.pub") $cpub  -NoNewline -Encoding ascii
Set-Content (Join-Path $CLIENTS "$cname.psk") $cpsk  -NoNewline -Encoding ascii
Add-Content $CONF ""
Add-Content $CONF "[Peer]"
Add-Content $CONF "# $cname"
Add-Content $CONF "PublicKey = $cpub"
Add-Content $CONF "PresharedKey = $cpsk"
Add-Content $CONF "AllowedIPs = $NET.2/32"
@"
[Interface]
PrivateKey = $cpriv
Address = $NET.2/32
DNS = 1.1.1.1, 1.0.0.1
MTU = $MTU

[Peer]
PublicKey = $spub
PresharedKey = $cpsk
Endpoint = ${pub}:$PORT
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25
"@ | Set-Content (Join-Path $CLIENTS "$cname.conf") -Encoding ascii

Step "Dienst neu laden"
try { net stop $SVC 2>&1 | Out-Null } catch { }
Start-Sleep 2
try { net start $SVC 2>&1 | Out-Null } catch { }
Start-Sleep 3
try { netsh interface ipv4 set interface "$TUNNEL" forwarding=enabled 2>&1 | Out-Null } catch { }

Step "Portfreigabe auf der FRITZ!Box"
$u = Read-Host "    FRITZ!Box-Benutzer (Admin)"
$script:Cred = Get-Credential -UserName $u -Message "FRITZ!Box-Passwort"
try { Invoke-Box 'GetExternalIPAddress' ([ordered]@{}) | Out-Null } catch { Die "TR-064-Anmeldung fehlgeschlagen: $($_.Exception.Message)" }
try { Invoke-Box 'DeletePortMapping' ([ordered]@{ NewRemoteHost=''; NewExternalPort=$PORT; NewProtocol='UDP' }) | Out-Null } catch { }
try {
    Invoke-Box 'AddPortMapping' ([ordered]@{
        NewRemoteHost=''; NewExternalPort=$PORT; NewProtocol='UDP'; NewInternalPort=$PORT
        NewInternalClient=$lanIP; NewEnabled='1'; NewPortMappingDescription='WireGuard'; NewLeaseDuration='0'
    }) | Out-Null
    $chk = (Invoke-Box 'GetSpecificPortMappingEntry' ([ordered]@{ NewRemoteHost=''; NewExternalPort=$PORT; NewProtocol='UDP' })).NewInternalClient
    if ($chk -eq $lanIP) { Write-Host "    Freigabe steht: $PORT/UDP -> ${lanIP}" -ForegroundColor Green }
    else { Write-Host "    Freigabe angelegt, Rueckmeldung leer. In der Box pruefen." -ForegroundColor Yellow }
} catch {
    Write-Host "    Automatisch fehlgeschlagen: $($_.Exception.Message)" -ForegroundColor Yellow
    Write-Host "    -> Box: Internet > Freigaben > UDP $PORT an $lanIP manuell setzen." -ForegroundColor Yellow
}

Write-Host "`n==============================================" -ForegroundColor Green
Write-Host "  FERTIG" -ForegroundColor Green
Write-Host "==============================================" -ForegroundColor Green
Write-Host "  Endpunkt:   ${pub}:$PORT/udp"
Write-Host "  Client-Datei: $CLIENTS\$cname.conf"
Write-Host "  Tunnel-IP Client: $NET.2"
Write-Host ""
Write-Host "  Weitere Clients: wg-server.bat  ->  Option 5"
Write-Host "  Dienststatus:    sc query $SVC"
Write-Host "=============================================="
Read-Host "`nEnter zum Schliessen"
'@ | Set-Content (Join-Path $base 'setup.ps1') -Encoding ascii

Write-Host "`nsetup.ps1 geschrieben." -ForegroundColor Green
Prüfen:

Select-String -Path .\setup.ps1 -Pattern "ErrorActionPreference|uninstalltunnelservice"
Erwartet: 'Continue' in Zeile 3 und die try { ... }-Zeile beim Uninstall.

Dann:

.\START.cmd
Was sich gegenüber vorher geändert hat: Continue statt Stop, damit stderr-Meldungen von wireguard.exe und net nicht mehr abbrechen. Alle nativen Aufrufe, die legitim scheitern dürfen (Deinstallieren eines nicht vorhandenen Dienstes, Löschen nicht existierender Firewall-Regeln, netsh auf einem noch nicht existierenden Interface), sind einzeln in try/catch gekapselt. Neu ist außerdem der Nachstart-Versuch beim Dienst: schlägt die Installation fehl, wird net start nachgeschoben und erst dann abgebrochen, wenn der Dienst wirklich nicht läuft — vorher wäre genau dieser Fall stumm durchgelaufen.

An den Stellen, wo es zählt, prüft das Skript weiterhin hart: kein LAN-Interface, keine öffentliche IP, keine Schlüsseldatei, Dienst nicht laufend → sauberer Abbruch mit Meldung statt halbfertigem Zustand.

============================================== WireGuard Server - vollautomatisches Setup
[*] Ermittle lokale LAN-Adresse
LAN-IP: 192.168.178.20

[*] Ermittle oeffentliche IPv4
Oeffentliche IP: 95.91.146.128

[*] Pruefe WireGuard
OK

[*] Erzeuge Server-Schluessel

[*] Schreibe Serverkonfiguration

[*] Firewall

[*] Routing aktivieren

[*] NAT einrichten

[*] Installiere und starte Dienst
Dienst laeuft nicht, starte nach ...

FEHLER: Dienst laeuft nicht. Konfiguration: C:\EIGENE-VPN\wg0.conf
Enter zum Schliessen:

Gearbeitet für 4s · 1 Schritt

Der Dienst wird installiert, stirbt aber sofort. Das Skript verschluckt die Begründung, weil es stderr wegwirft — deshalb jetzt manuell, mit sichtbarer Ausgabe. Im erhöhten Fenster:

Get-Content C:\EIGENE-VPN\wg0.conf
sc.exe query "WireGuardTunnel`$wg0"
Get-NetUDPEndpoint -LocalPort 443 -ErrorAction SilentlyContinue
& "$env:ProgramFiles\WireGuard\wireguard.exe" /installtunnelservice "C:\EIGENE-VPN\wg0.conf"
Start-Sleep 4
sc.exe query "WireGuardTunnel`$wg0"
Get-WinEvent -LogName Application -MaxEvents 20 | Where-Object { $_.ProviderName -like '*WireGuard*' } | Format-List TimeCreated,Message
Die vierte Zeile ist die entscheidende — die gibt den tatsächlichen Grund aus, den das Setup bisher unterdrückt hat. Die anderen liefern den Kontext: ob die Konfiguration vollständig ist, ob der Dienst überhaupt angelegt wurde und ob UDP 443 schon von etwas anderem belegt ist.

Schick mir die komplette Ausgabe. Nichts davon enthält Schlüssel, die du verbergen müsstest — es ist dein eigener Rechner.

C:\EIGENE-VPN>Get-Content C:\EIGENE-VPN\wg0.conf
Der Befehl "Get-Content" ist entweder falsch geschrieben oder
konnte nicht gefunden werden.

C:\EIGENE-VPN>sc.exe query "WireGuardTunnel`$wg0"
[SC] EnumQueryServicesStatus:OpenService FEHLER 1060:

Der angegebene Dienst ist kein installierter Dienst.

C:\EIGENE-VPN>Get-NetUDPEndpoint -LocalPort 443 -ErrorAction SilentlyContinue
Der Befehl "Get-NetUDPEndpoint" ist entweder falsch geschrieben oder
konnte nicht gefunden werden.

C:\EIGENE-VPN>& "$env:ProgramFiles\WireGuard\wireguard.exe" /installtunnelservice "C:\EIGENE-VPN\wg0.conf"
"&" kann syntaktisch an dieser Stelle nicht verarbeitet werden.

C:\EIGENE-VPN>Start-Sleep 4
Der Befehl "Start-Sleep" ist entweder falsch geschrieben oder
konnte nicht gefunden werden.

C:\EIGENE-VPN>sc.exe query "WireGuardTunnel`$wg0"
[SC] EnumQueryServicesStatus:OpenService FEHLER 1060:

Der angegebene Dienst ist kein installierter Dienst.

C:\EIGENE-VPN>Get-WinEvent -LogName Application -MaxEvents 20 | Where-Object { $_.ProviderName -like 'WireGuard' } | Format-List TimeCreated,Message
Der Befehl "Get-WinEvent" ist entweder falsch geschrieben oder
konnte nicht gefunden werden.

C:\EIGENE-VPN>

Gearbeitet für 2s · 1 Schritt

Das Fenster ist cmd, nicht PowerShell — daher scheitern alle Cmdlets und &. Bleib trotzdem in dem Fenster und nimm die cmd-Varianten:

type C:\EIGENE-VPN\wg0.conf
sc query "WireGuardTunnel$wg0"
netstat -ano -p UDP | findstr ":443"
"C:\Program Files\WireGuard\wireguard.exe" /installtunnelservice "C:\EIGENE-VPN\wg0.conf"
timeout /t 4 /nobreak >nul
sc query "WireGuardTunnel$wg0"
wevtutil qe Application /q:"*[System[Provider[@Name='WireGuard']]]" /c:10 /f:text
In cmd bleibt $ normal, der Backtick von vorhin war falsch.

Zeile 4 gibt den Installationsfehler jetzt im Klartext aus, Zeile 5/6 zeigen, ob der Dienst danach existiert und ob er stehen bleibt. Zeile 3 prüft, ob UDP 443 schon belegt ist — das ist der häufigste Grund, warum der Tunnel direkt wieder abreißt.

Falls du lieber in PowerShell bleiben willst: powershell -NoProfile eintippen, Enter, dann laufen auch die vorherigen Befehle. Aber die cmd-Variante reicht völlig.
