# Proxmox Mail Gateway (PMG) - Custom Check mit Google Gemini AI

Ein intelligentes Custom Check Script für **Proxmox Mail Gateway (PMG)** zur KI-gestützten Spam- & Phishing-Erkennung mittels der **Google Gemini AI API**.

---

## 🌟 Highlights

- 🤖 **KI-Prüfung per Google Gemini API**: Analysiert Betreff, Absender und E-Mail-Inhalte (Plaintext & HTML) auf Phishing, Social Engineering und Spam.
- 🎛 **Verwaltung über die PMG Web GUI**: Steuerung der überprüften Empfänger-Domains und E-Mail-Adressen komfortabel über PMG *Who Objects* (`pmg-custom-check-gemini`).
- 🏷 **Erklärbarer Spam-Score**: Fügt `X-Gemini-AI-Score` und `X-Gemini-AI-Reason` (Begründung der KI) direkt in die Mail-Header ein (sichtbar in PMG Quarantäne & Mail-Client).
- ⚡ **Dynamisches Modell-Matching & Caching**: Fragt die neuesten Gemini Flash Modelle per API ab und wechselt bei Overload (HTTP 429/503) automatisch auf Fallback-Modelle.
- 🔒 **Sicher & Datenschutzkonform**: Verarbeitet nur relevante E-Mail-Inhalte bis zu einer konfigurierbaren Maximallänge.

---

## 📋 Voraussetzungen

- Proxmox Mail Gateway (Version 7.x, 8.x oder 9.x)
- Python 3.8+ (standardmäßig auf PMG installiert)
- Einen kostenlosen oder kostenpflichtigen API-Key aus dem [Google AI Studio](https://aistudio.google.com/)

---

## 🚀 Installation & Einrichtung

### 1. API-Key via Systemd Override konfigurieren

Erstelle den Systemd-Override-Ordner und hinterlege deinen Google Gemini API-Key in `/etc/systemd/system/pmg-smtp-filter.service.d/pmg-custom-check-gemini.conf`:

```bash
mkdir -p /etc/systemd/system/pmg-smtp-filter.service.d/
cat << 'EOF' > /etc/systemd/system/pmg-smtp-filter.service.d/pmg-custom-check-gemini.conf
[Service]
NoNewPrivileges=no
Environment="GEMINI_API_KEY=DEIN_GEMINI_API_KEY_HIER"
EOF
systemctl daemon-reload
```

### 2. Skript herunterladen und installieren

Lade das Skript herunter und mache es ausführbar:

```bash
wget -O /usr/local/bin/pmg-custom-check-gemini https://raw.githubusercontent.com/Lokutos/pmg-custom-check-gemini/main/usr/local/bin/pmg-custom-check-gemini
chmod +x /usr/local/bin/pmg-custom-check-gemini
```

### 3. PMG Custom Check aktivieren

Öffne `/etc/pmg/pmg.conf` und aktiviere `custom_check` sowie `custom_check_path` in der Sektion `admin`:

```ini
section: admin
        custom_check 1
        custom_check_path /usr/local/bin/pmg-custom-check-gemini
```

Schließe die Konfiguration ab und starte den PMG Filter-Dienst neu:

```bash
systemctl restart pmg-smtp-filter
```

---

## 🎛 Steuerung über die PMG Web-GUI (Who Objects)

1. Öffne die Proxmox Mail Gateway Web-Oberfläche.
2. Navigiere zu **Mail Filter ➔ Who Objects** (Wer-Objekte).
3. Klicke auf **Create** und erstelle ein neues Objekt mit folgendem Namen:
   - **Name:** `pmg-custom-check-gemini`
4. Füge diesem Objekt alle **Domains** (z. B. `example.com`) oder einzelnen **E-Mail-Adressen** hinzu, deren eingehende E-Mails per Gemini AI geprüft werden sollen.

> 💡 **Hinweis:** Wird das Who-Object gelöscht oder bleibt es leer, wird der KI-Check für alle Mails übersprungen.

---

## 🔍 Logging & Verifikation

Das Skript protokolliert alle Vorgänge über den Linux Syslog (`/var/log/mail.log` bzw. `journalctl`).

Live-Logs anzeigen:

```bash
journalctl -u pmg-smtp-filter -f
```

**Beispielhafter Log-Ablauf:**

```text
pmg-custom-check: PMG Who-Object 'pmg-custom-check-gemini' (ID 32) geladen: example.com
pmg-custom-check: Gemini AI Check aktiv für Empfänger 'user@example.com'
pmg-custom-check: Analyse erfolgreich (gemini-2.0-flash) | Absender: info@bad-spammer.com | Score: 8.5 | Grund: Phishing-Versuch mit vorgetäuschtem Passwort-Reset.
pmg-smtp-filter: hits=CustomCheck(8.5)
```

---

## 🛠 Fehlerbehebung (Troubleshooting)

### `pmgsh Gruppen-Abfrage Fehler: please run as root` oder `sudo: a password is required`
Falls `pmg-smtp-filter` als unprivilegierter Dienstbenutzer (`pmg-smtp-filter`) ausgeführt wird, kann `pmgsh` die PMG API nicht direkt ohne Root-Rechte abfragen. Das Skript versucht in diesem Fall automatisch ein Fallback über `sudo -n pmgsh`.

Erstelle bei diesem Fehler die Datei `/etc/sudoers.d/pmg-custom-check`, um `pmgsh` ohne Passworteingabe zu erlauben:

```bash
echo 'pmg-smtp-filter ALL=(ALL) NOPASSWD: /usr/bin/pmgsh' > /etc/sudoers.d/pmg-custom-check
chmod 0440 /etc/sudoers.d/pmg-custom-check
```

---

## 📄 Lizenz

Dieses Projekt steht unter der [MIT Lizenz](LICENSE).
