# Schwärzungs-Checkliste
Homelab-Doku & öffentliches GitHub-Repo · vor jedem Commit, Screenshot und Post

## 1. IMMER schwärzen

| Was | Beispiel / wo es auftaucht | Warum |
|---|---|---|
| Passwörter, Passphrasen, PINs | Installer, FRITZ!Box, WLAN-Key, MariaDB-User | Direkter Zugang |
| Private Keys, Tokens, API-Keys | ~/.ssh/id_ed25519, .env, GitHub-Tokens | Direkter Zugang (Public Keys sind unkritisch) |
| Öffentliche IP / Remote-Adressen | IP vom Provider, MyFRITZ!- oder DynDNS-Adresse | Macht dein Heimnetz aus dem Internet auffindbar |
| MAC-Adressen | net0: virtio=…, FRITZ!Box-Geräteliste, ip a | Eindeutige Gerätekennung |
| Seriennummern, UUIDs, IDs | smbios1, vmgenid, Typenschild, Lizenz-Keys | Eindeutige Kennung, kein Nutzen für Leser |
| Login-Benutzernamen | Shell-Prompt, user@pam, SSH-Befehle | Die Hälfte der Zugangsdaten |
| WLAN-Name (SSID) | FRITZ!Box-Screenshots | Verortet dich physisch |
| Daten Dritter (DSGVO!) | Fremde IPs, Namen, E-Mails in Logs; Geräte der Familie im Netz | Personenbezogene Daten anderer |
| Persönliches am Bildrand | Browser-Tabs, Lesezeichen, Taskleiste, Benachrichtigungen | Wird beim Screenshot gern übersehen |

## 2. NICHT schwärzen

| Was | Warum |
|---|---|
| Netzmaske / CIDR (255.255.255.0, /24) | Steht in Millionen Netzen, verrät nichts |
| Private IPs (192.168.x.x, 10.x.x.x) inkl. Gateway & DNS | Aus dem Internet nicht erreichbar, macht den IP-Plan nachvollziehbar. Einheitlich in der ganzen Doku! |
| Hostnamen im Heimnetz (pve01, lamp01, sw-buero) | Nur intern auflösbar |
| Ports, Dienste, Paketnamen, Versionen, Config-Keys | Das ist der fachliche Inhalt deiner Doku |
| Public Keys & Fingerprints | Sind zum Teilen gedacht |

## 3. WIE schwärzen

| Regel | Richtig | Falsch |
|---|---|---|
| Nur den sensiblen Teil | `net0: virtio=XX:XX:XX:XX:XX:XX,bridge=vmbr0` | Ganze Zeile löschen: der Beweis ist weg |
| Sichtbar markieren | `<redacted>`, `XX:XX:…`, `***` | Weiße Lücke: sieht aus wie ein Fehler |
| Konsole = Text | Ausgabe als Codeblock kopieren, Werte ersetzen | Screenshot einer Konsole |
| GUI-Bilder: deckend | Vollflächiger schwarzer Balken oder zuschneiden | Weichzeichnen oder Verpixeln: rekonstruierbar |

## 4. Check vor jedem Commit

- [ ] `git diff --staged` komplett gelesen, nicht nur überflogen
- [ ] Jeden Screenshot ganz angesehen, inklusive Ecken, Tabs und Taskleiste
- [ ] Suche nach MAC-Adressen: `git grep -nEi "([0-9a-f]{2}:){5}[0-9a-f]{2}"`
- [ ] Suche nach Geheimnissen: `git grep -niE "password|passwd|token|secret|PRIVATE KEY"`
- [ ] Keine .env, Keys oder Backups im Repo (ggf. .gitignore)

## 5. Notfall: doch etwas Geheimes gepusht?

1. Zuerst das Geheimnis ändern: Passwort ändern, Key oder Token widerrufen und neu erzeugen.
2. Danach aus der Git-History entfernen.

Begründung: Ein öffentliches Repo kann längst kopiert oder gecrawlt sein. Das Löschen aus der History allein macht ein Geheimnis nicht wieder geheim.
