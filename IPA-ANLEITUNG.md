# Horizon aufs iPad bringen (IPA bauen ohne Mac)

## Warum hier keine `.ipa` entsteht

Ein iPad führt nur Code aus, der

1. mit Apples Compiler (clang) gegen das **iOS-SDK** als Mach-O/arm64 gebaut wurde und
2. von Apple **signiert** ist (Zertifikat + Provisioning-Profil mit der UDID deines iPads).

Beides gibt es nur mit macOS/Xcode bzw. mit deiner Apple-ID. Auf einem Windows-PC ist das
kein „umcoden"-Problem: auch ein nochmal neu geschriebener Client müsste durch Apples Toolchain.
Dieser Ordner enthält aber schon alles, was man dafür braucht (`packaging/ios`, `crates/horizon-ios*`,
Workflow `.github/workflows/ios.yml`) — es fehlt nur die Maschine, die baut.

Deshalb: bauen lässt du auf **GitHubs kostenlosen macOS-Runnern** (Weg A), signieren und
installieren dann hier auf Windows mit deiner Apple-ID (Schritt 3). Weg B ist der Weg mit Mac.

---

## Weg A — ohne Mac

### 1. Quellcode zu GitHub hochladen

Dieser Ordner ist bereits ein Git-Repository mit einem Commit (`main`). Es fehlt nur das Repo bei GitHub.

```bash
winget install --id GitHub.cli -e      # nur einmal; danach Terminal neu öffnen
gh auth login                          # GitHub.com → HTTPS → Login with a web browser
gh repo create horizon-ios --public --source . --remote origin --push
```

`--public` ist wichtig: bei öffentlichen Repos sind die macOS-Minuten gratis (private Repos
verbrauchen dein Monatskontingent zehnfach, und der Build braucht ~30–60 Minuten).

Ohne `gh`: Repo auf github.com anlegen und dann `git remote add origin <url>` +
`git push -u origin main`.

### 2. Den Build laufen lassen

Der Push auf `main` startet den Workflow `ios` automatisch. Beobachten und herunterladen:

```bash
gh run list --workflow ios.yml --limit 3
gh run watch                            # bis „ipa" grün ist
gh run download --name horizon-ios-game --dir ipa
```

Im Ordner `ipa` liegt dann **`Horizon-game-unsigned.ipa`** — das fertige Spiel als App-Bundle.
(Alternativ im Browser: Repo → Tab *Actions* → Lauf „ios" → unten *Artifacts*.)

Der Workflow verlangt Xcode 26 (Liquid-Glass-Launcher). `macos-26` bei GitHub hat das.
Bricht er dort ab, ist es eine Toolchain-Meldung aus `tools/ci/ios-toolchain.sh`, kein Fehler in
deinem Code.

### 3. Aufs iPad bringen (Signieren mit deiner Apple-ID)

Eine unsignierte `.ipa` installiert nirgends. Das Signieren übernimmt auf Windows **Sideloadly**
(gratis, <https://sideloadly.io>):

1. iPad per Kabel anschließen, „Diesem Computer vertrauen" bestätigen.
2. Am iPad **Entwicklermodus** an: *Einstellungen → Datenschutz & Sicherheit → Entwicklermodus*,
   Gerät neu starten, danach bestätigen (Pflicht ab iPadOS 16).
3. Sideloadly starten, `Horizon-game-unsigned.ipa` ins Fenster ziehen, deine **Apple-ID** eintragen
   (App-spezifisches Passwort, wenn 2FA an ist) und *Start* drücken.
4. Am iPad: *Einstellungen → Allgemein → VPN & Geräteverwaltung → Entwickler-App → Vertrauen*.

Grenzen einer **kostenlosen** Apple-ID: die App läuft **7 Tage**, dann neu signieren (Sideloadly
kann das per Auto-Refresh über WLAN). Ein bezahltes Developer-Programm (99 $/Jahr) macht daraus 1 Jahr.

---

## 4. Erster Start: Minecraft-Dateien fehlen

Die `.ipa` enthält absichtlich **keine** Mojang-Dateien (das Repo darf sie nicht weitergeben).
Beim Start zeigt das Spiel deshalb einen Import-Bildschirm mit Adresse und sechsstelliger PIN.
Die Spieldaten baust du auf dem PC und schickst sie rüber:

```bash
# einmalig, auf dem Rechner mit installiertem Minecraft (braucht Rust + JDK):
cargo run --release -p horizon-prepare -- --minecraft "<Pfad zur .minecraft>"
cargo run --release -p horizon-client -- --pack-resources .local/horizon-client/26.3 --out Horizon-26.3.zip
```

Dann die im Spiel angezeigte Adresse (`http://<ip>:<port>/`) am PC öffnen, ZIP auswählen, PIN
eintippen. Alternativ das ZIP per Kabel/Files-App in den Ordner `Horizon/Documents` kopieren —
das Spiel packt es beim nächsten Start selbst aus. Details: `packaging/ios/README.md`.

---

## Weg B — mit einem Mac

Dann entfällt GitHub komplett, und du installierst direkt aus Xcode:

```bash
rustup target add aarch64-apple-ios
brew install xcodegen
cd packaging/ios && xcodegen generate && open HorizonIOS.xcodeproj
```

In Xcode das Scheme **Horizon game** wählen, dein iPad als Ziel, beim Target ein Team
(deine Apple-ID) eintragen und *Run* drücken. Xcode baut das Spiel mit Cargo, packt es ins
Bundle und signiert es.

Es gibt zwei Schemes: **Horizon shell** ist nur die Prüf-App (zeigt vier Simulations-Proben an),
**Horizon game** ist das Spiel. Wenn du Zahlen auf schwarzem Grund siehst, ist es die falsche App.

---

## Was du erwarten kannst

Das ist ein Entwickler-Build, kein fertiges Spiel: Assets kommen von Hand (Schritt 4),
Touch-Steuerung ist noch nicht überall fertig, und auf einem echten iPad hat den Build
laut `packaging/ios/README.md` noch niemand laufen lassen. Erster Blick bei Problemen:
`Library/Application Support/horizon/logs/latest.log` im App-Container (Xcode → *Devices and
Simulators*, oder Dateien-App).
