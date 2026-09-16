# Httpd

## Setup

```bash
sudo apt update
sudo apt upgrade
sudo apt install php-fpm
sudo apt install apache2
sudo sysctl enable --now apache2
```

## Creazione di un mini-sito

```bash
# 1. Crea la cartella del progetto
sudo mkdir -p /var/www/mini-site/public

# 2. Crea un file index.php minimale ma informativo
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
# 1. Crea la cartella /config e un file sensibile dentro di essa
sudo mkdir -p /var/www/mini-site/public/config
sudo bash -c 'echo "DB_PASSWORD=secret" > /var/www/mini-site/public/config/db.ini'

# 2. Crea anche il file .env nella root per testare la seconda regola
sudo bash -c 'echo "SECRET_KEY=12345" > /var/www/mini-site/public/.env'

# 3. Assicura le proprietà POSIX
sudo chown -R www-data:www-data /var/www/mini-site
sudo chmod -R 755 /var/www/mini-site
```

## Multi-Processing Modules (MPM)

Apache gestisce le richieste dei client usando i MPM. La scelta del modulo determina le prestazioni e come l'applicazione (es. PHP) viene eseguita.

| MPM                         | Come gestisce le connessioni                                                                           | Integrazione con PHP                                         | Quando usarlo                                                                               |
| --------------------------- | ------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------------------------------------- |
| **Prefork**                 | Un processo per ogni richiesta (1 processo = 1 client). **Non usabile con thread**.                    | Usa **mod_php** (PHP è integrato dentro il processo Apache). | **Rarissimo/Obsoleto**. Utile solo con vecchi moduli C non thread-safe. Spreca molta RAM.   |
| **Worker**                  | Processi multipli, ciascuno con più thread. I/O sincrono / bloccante.                                  | Usa **PHP-FPM** tramite socket Unix/TCP (`proxy_fcgi`).      | Transizionale. Gestisce bene la concorrenza ma fatica con connessioni Keep-Alive lunghe.    |
| **Event** (Default moderno) | Simile a Worker, ma un thread dedicato gestisce l'I/O asincrono per Keep-Alive e connessioni inattive. | Usa **PHP-FPM** tramite socket Unix/TCP (`proxy_fcgi`).      | **Standard moderno**. Consuma pochissima RAM e gestisce migliaia di connessioni simultanee. |

_(FPM sta per FastCGI Process Manager)_

**Regola d'oro**: In qualsiasi installazione moderna con PHP, si usa **MPM Event + PHP-FPM**. Evitare `mod_php` (richiede Prefork, che satura la memoria con l'aumentare dei client).

## VirtualHost HTTP2

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

  # HEADER DI SICUREZZA (Disabilitiamo HSTS perché lavoriamo in HTTP locale)
  Header always set X-Content-Type-Options "nosniff"
  Header always set X-Frame-Options "SAMEORIGIN"
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
# 1. Disabilita MPM Prefork e mod_php (se attivi)
sudo a2dismod mpm_prefork 2>/dev/null
sudo a2dismod php* 2>/dev/null

# 2. Abilita MPM Event, Proxy FCGI e Headers
sudo a2enmod mpm_event proxy proxy_fcgi headers

# 3. Disabilita il sito di default di Apache e abilita il mini-sito
sudo a2dissite 000-default.conf
sudo a2ensite mini-site.conf

# 4. Verifica che la sintassi della configurazione sia corretta
sudo apache2ctl configtest

# 5. Riavvia Apache e PHP-FPM
sudo systemctl restart php*-fpm
sudo systemctl restart apache2
```

```bash
# Verifica il funzionamento da terminale
curl -i http://localhost/index.php # test della pagina
curl -i http://localhost/config/ # test di un path vietato
curl -I http://localhost/index.php # test degli header
```

## Configurazione completa

Una configurazione più completa richiede l'installazione di un certificato SSL:

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/mini-site-selfsigned.key \
  -out /etc/ssl/certs/mini-site-selfsigned.crt \
  -subj "/CN=localhost"
sudo a2enmod ssl
```

```bash
sudo vim /etc/apache2/sites-available/mini-site.conf
```

```apache
<VirtualHost *:80>
  ServerName mini-site.example.com
  Redirect permanent / https://mini-site.example.com/
</VirtualHost>

<VirtualHost *:443>
  ServerName mini-site.example.com
  DocumentRoot /var/www/mini-site/public

  # ABILITA HTTP/2: Riduce la latenza permettendo il multiplexing su un'unica connessione TCP.
  # Funziona solo con MPM Event o Worker (non con Prefork).
  Protocols h2 http/1.1

  # CONFIGURAZIONE TLS/SSL
  SSLEngine on
  SSLCertificateFile /etc/ssl/certs/mini-site-selfsigned.crt
  SSLCertificateKeyFile /etc/ssl/private/mini-site-selfsigned.key

  # -all disabilita tutti i vecchi protocolli vulnerabili (SSLv2, SSLv3, TLS 1.0, TLS 1.1)
  # lasciando attivi solo i moderni e sicuri TLS 1.2 e TLS 1.3
  SSLProtocol -all +TLSv1.2 +TLSv1.3

  # INTEGRAZIONE PHP-FPM: Unix Socket vs TCP
  <FilesMatch \.php$>
    SetHandler "proxy:unix:/run/php/php8.1-fpm.sock|fcgi://localhost"
  </FilesMatch>
  # fcgi://localhost indica ad Apache di usare il protocollo FastCGI.
  # proxy:unix:/run/php/php8.1-fpm.sock dice a proxy_fcgi di inviare i pacchetti tramite un file Unix Socket locale anziché aprire una connessione di rete TCP (127.0.0.1:9000).
  # Perché Unix Socket? Ha prestazioni superiori e minor overhead di CPU rispetto al socket TCP quando Apache e PHP-FPM girano sulla stessa macchina.

  # HEADER DI SICUREZZA
  # HSTS: Obbliga il browser a ricordare per 1 anno (31536000 sec) di connettersi SOLO in HTTPS,
  # ignorando qualsiasi link HTTP inserito dall'utente.
  Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"

  # Anti-MIME-Sniffing: Impedisce al browser di "indovinare" il tipo di file (es. eseguire script
  # caricati da utenti mascherati da immagini).
  Header always set X-Content-Type-Options "nosniff"

  # Anti-Clickjacking: Vieta l'inserimento del sito all'interno di <iframe> su siti terzi.
  Header always set X-Frame-Options "SAMEORIGIN"

  # CSP (Content Security Policy): Istruisce il browser a caricare risorse (script, immagini, CSS)
  # esclusivamente dallo stesso dominio ('self'), bloccando attacchi XSS.
  Header always set Content-Security-Policy "default-src 'self';"

  # PERMESSI DIRECTORY
  <Directory /var/www/mini-site/public>
    # AllowOverride None: Disabilita l'uso dei file .htaccess. Aumenta le prestazioni (Apache
    # non deve cercare .htaccess in ogni sottocartella) e impedisce modifiche di sicurezza non autorizzate.
    AllowOverride None

    # Require all granted: Concede l'accesso in lettura ai file contenuti in questa cartella.
    Require all granted
  </Directory>

  <Directory /var/www/mini-site/config>
    Require all denied
  </Directory>

  # Blocco di specifici file sensibili
  <FilesMatch "^\.env">
    Require all denied
  </FilesMatch>

  <Directory /var/www/mini-site/public/admin>
    # Consente l'accesso solo dal loopback locale e dalla VPN aziendale
    Require ip 127.0.0.1
    Require ip 192.168.1.0/24
    Require ip 10.8.0.50
  </Directory>

  <Directory /var/www/mini-site/public/internal>
    <RequireAll>
        # Deve venire dalla rete aziendale...
        Require ip 192.168.1.0/24
        # ...MA NON da questo specifico IP compromesso/ospite
        Require not ip 192.168.1.100
    </RequireAll>
  </Directory>

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
# Verifica il funzionamento da terminale
curl -i http://localhost/index.php # Moved Permanently
curl -ik https://localhost/index.php # la k consente connessioni non sicure (certificato self-signed)
```

## Cenni su Perl e CGI (Legacy)

- Che cos'era il CGI (Common Gateway Interface): Ad ogni richiesta HTTP per uno script Perl/Bash, il server web avviava un nuovo processo di sistema (fork), eseguiva lo script e ne restituiva l'output.
- Perché è stato abbandonato: Creare un nuovo processo di sistema per ogni singola chiamata genera un overhead di CPU/RAM enorme, esponendo il server a facili attacchi DoS.
- Sostituto moderno: Architetture come FastCGI (PHP-FPM, FastCGI Process Manager, per PHP) o WSGI (Web Server Gateway Interface, per Python) mantengono un pool di processi worker costantemente attivi in RAM, ricevendo le richieste tramite socket senza dover ricreare processi a ogni chiamata.

## Errori comuni

| Sintomo / Errore HTTP     | Causa Principale                                                                           | Cosa si legge nel file error.log                                                                                                                                                                                             | Come si risolve nell'esercizio                                                                                                                                                                     |
| ------------------------- | ------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 503 Service Unavailable   | PHP-FPM è spento, il socket Unix non esiste o i permessi sul socket sono errati.           | (111)Connection refused: AH00957: FCGI: attempt to connect to 127.0.0.1... oppure (13)Permission denied: AH01079: failed to make connection to backend                                                                       | Verificare che il servizio php8.1-fpm sia attivo (systemctl status) e che i permessi sul file .sock consentano la lettura/scrittura a www-data.                                                    |
| 500 Internal Server Error | Sintassi errata nel file .htaccess o direttiva non consentita da AllowOverride.            | AH00670: Options not allowed here oppure Invalid command 'Header', perhaps misspelled...                                                                                                                                     | Verificare la configurazione di AllowOverride nel VirtualHost o verificare se manca un modulo Apache (es. dimenticato a2enmod headers).                                                            |
| 403 Forbidden             | Permessi del File System POSIX insufficienti o direttiva Require restrittiva.              | (13)Permission denied: AH00035: client denied by server configuration oppure cannot open file for reading                                                                                                                    | Verificare che la direttiva nel VirtualHost sia Require all granted e che l'utente www-data abbia i permessi di lettura sui file e di esecuzione (+x) sulle directory genitrici.                   |
| 403 / 500 (Silenzioso)    | AppArmor o SELinux bloccano l'accesso a directory non standard (es. /srv/app o /opt/data). | **AppArmor**: audit: type=1400 ... apparmor="DENIED" operation="open" profile="/usr/sbin/apache2" (in /var/log/syslog o dmesg) § **SELinux**: type=AVC msg=audit... comm="httpd" name="public" ... scontext=... tcontext=... | **AppArmor**: aggiungere il percorso consentito in /etc/apparmor.d/local/usr.sbin.apache2. § **SELinux**: impostare il contesto corretto tramite chcon -t httpd_sys_content_t o semanage fcontext. |

### 503

```bash
# scassa:
sudo systemctl stop php8.1-fpm

# testa:
curl -ik https://localhost/index.php # 503

# indaga:
sudo tail -n 5 /var/log/apache2/mini-site_error.log # attempt to connect to Unix socket failed

# correggi:
sudo systemctl start php8.1-fpm
```

### 500

```bash
# scassa:
sudo vim /etc/apache2/sites-available/mini-site.conf # AllowOverride All
sudo bash -c 'echo "DirettivaInesistente Finta" > /var/www/mini-site/public/.htaccess'
sudo systemctl reload apache2

# testa:
curl -ik https://localhost/index.php # 500

# indaga:
sudo tail -n 5 /var/log/apache2/mini-site_error.log # Invalid command 'DirettivaInesistente'

# correggi:
sudo vim /etc/apache2/sites-available/mini-site.conf # AllowOverride None
sudo systemctl reload apache2

```

### 403

#### POSIX

```bash
# scassa:
sudo chmod 000 /var/www/mini-site/public

# testa:
curl -ik https://localhost/index.php # 403

# indaga:
sudo tail -n 5 /var/log/apache2/mini-site_error.log # Access to index.php denied

# correggi:
sudo chmod 755 /var/www/mini-site/public
```

#### AppArmor

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
