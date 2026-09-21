# Httpd

## Setup

```bash
sudo apt update
sudo apt upgrade
sudo apt install php-fpm apache2
sudo systemctl enable --now apache2
```

## Creazione del Mini-Sito di Test

Per applicare il principio del minimo privilegio:

- La proprietà dei file va assegnata a `root:www-data`.
- Le **directory** devono avere permessi `755` (`rwxr-xr-x`).
- I **file** devono avere permessi `644` (`rw-r--r--`).

In questo modo il processo del server web (`www-data`) può **leggere ed eseguire** gli script, ma **non può sovrascrivere o iniettare codice** nei file del sito.

```bash
# 1. Crea le directory del progetto (inclusa la cartella riservata config)
sudo mkdir -p /var/www/mini-site/public/config
sudo mkdir -p /var/www/mini-site/public/admin
sudo mkdir -p /var/www/mini-site/public/protected

# 2. Crea un file index.php minimale
sudo vim /var/www/mini-site/public/index.php
```

```php
<?php
header("X-Custom-Header: MiniSiteTest");
?>
<!DOCTYPE html>
<html lang="it">
<head>
  <meta charset="UTF-8">
  <title>Mini-Sito PHP Test</title>
  <style>
    body { font-family: sans-serif; background: #f4f4f9; padding: 2rem; color: #333; }
    .card { background: white; padding: 1.5rem; border-radius: 8px; box-shadow: 0 2px 5px rgba(0,0,0,0.1); }
    .success { color: #2e7d32; font-weight: bold; }
  </style>
</head>
<body>
  <div class="card">
    <h1>Hello World! 👋</h1>
    <p class="success">✔ Il mini-sito PHP funziona correttamente!</p>
    <hr>
    <ul>
      <li><strong>SAPI usata:</strong> <?php echo php_sapi_name(); ?></li>
      <li><strong>Versione PHP:</strong> <?php echo phpversion(); ?></li>
      <li><strong>Server Web:</strong> <?php echo $_SERVER['SERVER_SOFTWARE'] ?? 'N/D'; ?></li>
      <li><strong>Data/Ora Server:</strong> <?php echo date('Y-m-d H:i:s'); ?></li>
    </ul>
  </div>
</body>
</html>
```

```bash
# 3. Crea file sensibili di configurazione per testare i blocchi di accesso
sudo bash -c 'echo "DB_PASSWORD=secret" > /var/www/mini-site/public/config/db.ini'
sudo bash -c 'echo "SECRET_KEY=12345" > /var/www/mini-site/public/.env'

# 4. Imposta la proprietà corretta (root proprietario, www-data gruppo)
sudo chown -R root:www-data /var/www/mini-site

# 5. Applica permessi differenziati tra cartelle (755) e file (644)
sudo find /var/www/mini-site -type d -exec chmod 755 {} \;
sudo find /var/www/mini-site -type f -exec chmod 644 {} \;
```

## Multi-Processing Modules (MPM)

Apache gestisce le richieste dei client usando gli **MPM**. La scelta del modulo determina le prestazioni la modalità di esecuzione delle applicazioni (es. PHP).

| MPM                   | Gestione connessioni                                                              | Integrazione PHP                                               | Quando usarlo                                                                                      |
| --------------------- | --------------------------------------------------------------------------------- | -------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| **Prefork**           | 1 processo per ogni richiesta (1 processo = 1 client). Non thread-safe.           | Usa `mod_php` (PHP caricato direttamente nel processo Apache). | **Obsoleto/Sconsigliato**. Consuma molta RAM ed elimina i vantaggi dell'architettura multi-thread. |
| **Worker**            | Processi multipli, ciascuno con più thread. I/O sincrono / bloccante.             | Usa **PHP-FPM** tramite socket Unix/TCP (`proxy_fcgi`).        | Transizionale. Buon throughput ma fatica con connessioni Keep-Alive prolungate.                    |
| **Event** (_Default_) | Multi-processo e multi-thread con thread dedicato per I/O asincrono (Keep-Alive). | Usa **PHP-FPM** tramite socket Unix/TCP (`proxy_fcgi`).        | **Standard moderno**. Minimizza l'uso della RAM e gestisce elevate moli di connessioni simultanee. |

_(FPM sta per FastCGI Process Manager)_

> **Regola d'oro**: In qualsiasi installazione moderna con PHP si utilizza **MPM Event + PHP-FPM**. Evitare `mod_php` poiché forza l'uso di Prefork e satura rapidamente la memoria del server.

## VirtualHost HTTP di Base

```bash
vim /etc/apache2/sites-available/mini-site.conf
```

```apache
<VirtualHost *:80>
  ServerName localhost
  DocumentRoot /var/www/mini-site/public

  # INTEGRAZIONE PHP-FPM
  <FilesMatch \.php$>
    SetHandler "proxy:unix:/run/php/php8.1-fpm.sock|fcgi://localhost"
  </FilesMatch>

  # HEADER DI SICUREZZA
  # HSTS (1 anno): Forza l'uso esclusivo di HTTPS nel browser
  Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"

  # Anti-MIME-Sniffing: Impedisce al browser di "indovinare" il tipo di file (es. eseguire script
  # caricati da utenti mascherati da immagini).
  Header always set X-Content-Type-Options "nosniff"

  # Anti-Clickjacking: Vieta l'inserimento del sito all'interno di <iframe> su siti terzi.
  Header always set X-Frame-Options "SAMEORIGIN"

  # CSP (Content Security Policy): Istruisce il browser a caricare risorse (script, immagini, CSS)
  # esclusivamente dallo stesso dominio ('self'), bloccando attacchi XSS.
  # 'unsafe-inline' è inserito per consentire i CSS interni nella demo
  Header always set Content-Security-Policy "default-src 'self';"

  # PERMESSI DIRECTORY GENERALE
  <Directory /var/www/mini-site/public>
    AllowOverride None
    Require all granted
  </Directory>

  # PROTEZIONE CARTELLE E FILE SENSIBILI
  <Directory /var/www/mini-site/public/config>
    Require all denied
  </Directory>

  <FilesMatch "^\.env">
    Require all denied
  </FilesMatch>

  # RESTRIZIONI IP DEDICATE
  <Directory /var/www/mini-site/public/admin>
    Require ip 127.0.0.1
  </Directory>

  ErrorLog ${APACHE_LOG_DIR}/mini-site_error.log
  CustomLog ${APACHE_LOG_DIR}/mini-site_access.log combined
</VirtualHost>
```

```bash
# 1. Disabilita moduli obsoleti/incompatibili
sudo a2dismod mpm_prefork 2>/dev/null
sudo a2dismod php* 2>/dev/null

# 2. Abilita MPM Event e i moduli proxy/headers necessari
sudo a2enmod mpm_event proxy proxy_fcgi headers

# 3. Attiva la configurazione del mini-sito e disabilita quella di default
sudo a2dissite 000-default.conf
sudo a2ensite mini-site.conf

# 4. Verifica sintassi
sudo apache2ctl configtest

# 5. Riavvia i servizi
sudo systemctl restart php8.1-fpm
sudo systemctl restart apache2
```

```bash
# Verifica il funzionamento da terminale
curl -i http://localhost/index.php # test della pagina
curl -i http://localhost/config/ # test di un path vietato
curl -I http://localhost/index.php # test degli header
```

## Configurazione completa (HTTPS, HTTP/2, Autenticazione)

### 1. Generazione Certificato SSL e Utente .htpasswd

```bash
# Generazione certificato Self-Signed per i test
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/mini-site-selfsigned.key \
  -out /etc/ssl/certs/mini-site-selfsigned.crt \
  -subj "/CN=localhost"

# Creazione del file .htpasswd per l'area protetta
sudo htpasswd -c /etc/apache2/.htpasswd admin-corso
```

### 2. Moduli Requisiti per HTTP/2 e SSL

```bash
sudo a2enmod ssl http2
```

### 3. VirtualHost Avanzato

```bash
sudo vim /etc/apache2/sites-available/mini-site.conf
```

```apache
<VirtualHost *:80>
  ServerName localhost
  # Reindirizzamento permanente verso HTTPS
  Redirect permanent / https://localhost/
</VirtualHost>

<VirtualHost *:443>
  ServerName localhost
  DocumentRoot /var/www/mini-site/public

  # ABILITA HTTP/2: Multiplexing su singola connessione TCP (richiede a2enmod http2 e MPM Event/Worker)
  Protocols h2 http/1.1

  # CONFIGURAZIONE TLS/SSL
  SSLEngine on
  SSLCertificateFile /etc/ssl/certs/mini-site-selfsigned.crt
  SSLCertificateKeyFile /etc/ssl/private/mini-site-selfsigned.key

  # -all disabilita tutti i vecchi protocolli vulnerabili (SSLv2, SSLv3, TLS 1.0, TLS 1.1)
  # lasciando attivi solo i moderni e sicuri TLS 1.2 e TLS 1.3
  SSLProtocol -all +TLSv1.2 +TLSv1.3

  # INTEGRAZIONE PHP-FPM tramite Unix Socket
  <FilesMatch \.php$>
    SetHandler "proxy:unix:/run/php/php8.1-fpm.sock|fcgi://localhost"
  </FilesMatch>
  # fcgi://localhost indica ad Apache di usare il protocollo FastCGI.
  # proxy:unix:/run/php/php8.1-fpm.sock dice a proxy_fcgi di inviare i pacchetti tramite un file Unix Socket locale anziché aprire una connessione di rete TCP (127.0.0.1:9000).
  # Perché Unix Socket? Ha prestazioni superiori e minor overhead di CPU rispetto al socket TCP quando Apache e PHP-FPM girano sulla stessa macchina.

  # HEADER DI SICUREZZA
  # HSTS (1 anno): Forza l'uso esclusivo di HTTPS nel browser
  Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"

  # Anti-MIME-Sniffing: Impedisce al browser di "indovinare" il tipo di file (es. eseguire script
  # caricati da utenti mascherati da immagini).
  Header always set X-Content-Type-Options "nosniff"

  # Anti-Clickjacking: Vieta l'inserimento del sito all'interno di <iframe> su siti terzi.
  Header always set X-Frame-Options "SAMEORIGIN"

  # CSP (Content Security Policy): Istruisce il browser a caricare risorse (script, immagini, CSS)
  # esclusivamente dallo stesso dominio ('self'), bloccando attacchi XSS.
  # 'unsafe-inline' è inserito per consentire i CSS interni nella demo
  Header always set Content-Security-Policy "default-src 'self';"

  # PERMESSI DIRECTORY
  <Directory /var/www/mini-site/public>
    # Disabilita l'uso dei file .htaccess. Aumenta le prestazioni (Apache
    # non deve cercare .htaccess in ogni sottocartella) e impedisce modifiche di sicurezza non autorizzate.
    AllowOverride None

    # Concede l'accesso in lettura ai file contenuti in questa cartella.
    Require all granted
  </Directory>

  # Blocco di sicurezza sulla cartella dei file di configurazione
  <Directory /var/www/mini-site/public/config>
    Require all denied
  </Directory>

  # Blocco esplicito di file sensibili (es. file di ambiente)
  <FilesMatch "^\.env">
    Require all denied
  </FilesMatch>

  <# Restrizioni IP per l'area di amministrazione
  <Directory /var/www/mini-site/public/admin>
    Require ip 127.0.0.1
  </Directory>

  # Combinazione di più regole
  <Directory /var/www/mini-site/public/internal>
    <RequireAll>
        # Deve venire dalla rete aziendale...
        Require ip 192.168.1.0/24
        # ...MA NON da questo specifico IP compromesso/ospite
        Require not ip 192.168.1.100
    </RequireAll>
  </Directory>

  # Area ad accesso autenticato (.htpasswd)
  <Directory /var/www/mini-site/public/protected>
    AuthType Basic
    AuthName "Area Riservata Corso Security"
    AuthUserFile /etc/apache2/.htpasswd

    # Richiede che l'utente inserisca un login/password valido
    Require valid-user

    # Oppure limita l'accesso a uno specifico utente del file .htpasswd:
    # Require user admin
  </Directory>

  ErrorLog ${APACHE_LOG_DIR}/mini-site_error.log
  CustomLog ${APACHE_LOG_DIR}/mini-site_access.log combined
</VirtualHost>
```

```bash
# Verifica sintassi e riavvia Apache
sudo apache2ctl configtest
sudo systemctl restart apache2
```

### 4. Verifica dei Protocolli e degli Header

_(`-k` serve a ignorare il fatto che stiamo usando un certificato self-signed)_

```bash
# Test del redirect HTTP -> HTTPS
curl -I http://localhost/

# Test HTTP/2 (il flag --http2 verifica la negoziazione h2)
curl -I --http2 -k https://localhost/index.php

# Test accesso negato alla cartella riservata (risposta 403)
curl -I -k https://localhost/config/db.ini

# Test area protetta senza credenziali (risposta 401 Unauthorized)
curl -I -k https://localhost/protected/
```

## Cenni su Perl e CGI (Legacy)

- **CGI (Common Gateway Interface)**: Ad ogni richiesta HTTP per uno script Perl/Bash, il server web creava un nuovo processo di sistema (_fork_), eseguiva lo script e ne restituiva l'output.
- **Limiti**: Creare un processo di sistema per ciascuna chiamata generava un elevatissimo overhead di CPU/RAM, esponendo il sistema a semplici attacchi DoS.
- **Sostituto moderno**: Architetture basate su pool di processi pre-inizializzati come **FastCGI** (PHP-FPM) o **WSGI** (Python) mantengono i processi _worker_ attivi in memoria, ricevendo ed elaborando le richieste tramite socket senza ricreare il processo a ogni invocazione.

## Troubleshooting ed Errori Comuni

| Errore HTTP                                                                                               | Causa Principale                                                                               | Messaggio tipico in `error.log`                                                                                         | Soluzione                                                                                                                        |
| --------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| **503 Service Unavailable**                                                                               | PHP-FPM è spento, il file socket non esiste o ha permessi errati.                              | `(111)Connection refused: AH00957: FCGI: attempt to connect to 127.0.0.1...` oppure `(13)Permission denied: AH01079...` | Verificare che il servizio `php8.1-fpm` sia attivo (`systemctl status`) e che il socket in `/run/php/` esista.                   |
| **500 Internal Server Error**                                                                             | Sintassi errata nel file `.htaccess` o direttiva non autorizzata da `AllowOverride`.           | `AH00670: Options not allowed here` oppure `Invalid command 'Header'...`                                                | Verificare `AllowOverride` nel VirtualHost o abilitare il modulo mancante (`a2enmod headers`).                                   |
| **403 Forbidden**                                                                                         | Permessi del File System POSIX restrittivi o direttiva `Require` bloccante.                    | `(13)Permission denied: AH00035: client denied by server configuration`                                                 | Verificare la direttiva `Require all granted` e assicurarsi che l'utente `www-data` possa accedere/leggere le cartelle e i file. |
| **403 / 500 (Silenzioso)**                                                                                | AppArmor o SELinux bloccano l'accesso a directory non standard (es. `/srv/app` o `/opt/data`). | **AppArmor**: `apparmor="DENIED" operation="open" profile="/usr/sbin/apache2" in /var/log/syslog o dmesg.`              |
| **AppArmor**: modificare il profilo in `/etc/apparmor.d/` ricaricando le regole con `apparmor_parser -r`. |

## Esercitazioni di Troubleshooting

### 1. Simulazione Errore 503 (PHP-FPM Off)

```bash
# Provoca l'errore spegnendo PHP-FPM:
sudo systemctl stop php8.1-fpm

# Test:
curl -ik https://localhost/index.php # Risultato: HTTP 503 Service Unavailable

# Analisi log:
sudo tail -n 5 /var/log/apache2/mini-site_error.log

# Risoluzione:
sudo systemctl start php8.1-fpm
```

### Simulazione Errore 500 (.htaccess errato)

```bash
# Provoca l'errore abilitando AllowOverride e inserendo un comando errato:
sudo sed -i 's/AllowOverride None/AllowOverride All/' /etc/apache2/sites-available/mini-site.conf
sudo bash -c 'echo "DirettivaErrata Test" > /var/www/mini-site/public/.htaccess'
sudo systemctl reload apache2

# Test:
curl -ik https://localhost/index.php # Risultato: HTTP 500 Internal Server Error

# Analisi log:
sudo tail -n 5 /var/log/apache2/mini-site_error.log # Invalid command 'DirettivaErrata'

# Risoluzione:
sudo rm -f /var/www/mini-site/public/.htaccess
sudo sed -i 's/AllowOverride All/AllowOverride None/' /etc/apache2/sites-available/mini-site.conf
sudo systemctl reload apache2
```

### 3. Simulazione Errore 403 (Permessi POSIX)

```bash
# Provoca l'errore rimuovendo i permessi di lettura:
sudo chmod 000 /var/www/mini-site/public/index.php

# Test:
curl -ik https://localhost/index.php # Risultato: HTTP 403 Forbidden

# Analisi log:
sudo tail -n 5 /var/log/apache2/mini-site_error.log

# Risoluzione:
sudo chmod 644 /var/www/mini-site/public/index.php
```

### 4. Simulazione Blocco AppArmor su PHP-FPM

Un tentativo di generare un 403 vietando l'accesso ai file da Apache2 fallirà, perché i file sono gestiti da PHP-FPM, non direttamente da Apache.

```bash
# scassa:
sudo apt install libapache2-mod-apparmor
sudo rm /etc/apparmor.d/disable/usr.sbin.apache2 # il profilo è disabilitato di default
sudo aa-enforce /etc/apparmor.d/usr.sbin.apache2 # il profilo è complain di default
sudo vim /etc/apparmor.d/usr.sbin.apache2
```

```ini
  deny /var/www/mini-site/ r,
  deny /var/www/mini-site/** rwx,
```

```bash
sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.apache2
sudo systemctl restart apache2 # sempre 200!
```

Proviamo a bloccare PHP-FPM:

```bash
# scassa:
sudo apt install rsyslog # aa-genprof usa rsyslog invece di journald
sudo systemctl enable --now rsyslog # non necessario, ma per sicurezza
sudo aa-genprof /usr/sbin/php-fpm8.1
# in un altro terminale:
sudo systemctl restart php8.1-fpm
curl -ik https://localhost/index.php
# nel primo terminale, premi S, accetta tutto con A, infine scrivi con S e chiudi con F. Si è creato il file /etc/apparmor.d/usr.sbin.php-fpm8.1
sudo aa-enforce /etc/apparmor.d/usr.sbin.php-fpm8.1
sudo vim /etc/apparmor.d/usr.sbin.php-fpm8.1
```

```ini
# Last Modified: Wed Sep 16 10:26:46 2026
abi <abi/3.0>,

include <tunables/global>

/usr/sbin/php-fpm8.1 {
  include <abstractions/base>
  include <abstractions/dovecot-common>
  include <abstractions/openssl>
  include <abstractions/php>
  include <abstractions/postfix-common>
  include <abstractions/totem>

  capability chown,
  capability dac_override,
  capability net_admin,

  /usr/sbin/php-fpm8.1 mr,
  owner /etc/group r,
  owner /etc/nsswitch.conf r,
  owner /etc/passwd r,
  owner /etc/php/8.1/fpm/php-fpm.conf r,
  owner /etc/php/8.1/fpm/pool.d/www.conf r,
  owner /proc/sys/kernel/random/boot_id r,
  owner /run/php/php8.1-fpm.pid w,
  owner /run/php/php8.1-fpm.sock rw,
  owner /run/systemd/notify w,
  owner /run/systemd/userdb/io.systemd.DynamicUser rw,
  owner /var/log/php8.1-fpm.log w,
  # commenta via questa riga
  #owner /var/www/mini-site/public/index.php r,
  # aggiungi queste due righe
  deny /var/www/mini-site/ r,
  deny /var/www/mini-site/** rwx,
}
```

```bash
sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.php-fpm8.1
sudo systemctl restart php8.1-fpm
# in caso di errore (socket già in uso):
sudo rm /run/php/php8.1-fpm.sock
sudo systemctl restart php8.1-fpm

# testa:
curl -ik https://localhost/index.php # 403

# indaga:
sudo tail -n 5 /var/log/apache2/mini-site_error.log # Pemission denied

# correggi:
sudo vim /etc/apparmor.d/usr.sbin.php-fpm8.1
sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.php-fpm8.1
```
