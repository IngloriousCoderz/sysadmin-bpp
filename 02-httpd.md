# Httpd

## Setup

```bash
sudo apt update
sudo apt upgrade
sudo apt install apache2
sudo sysctl enable --now apache2
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

### Selezione MPM

```bash
# 0. Controlla quale modulo è attivo
sudo apache2ctl -M | grep mpm # mpm_event_module (shared)

# 1. Disabilita il modulo mod_php (se presente) e l'MPM corrente
sudo a2dismod php8.1
sudo a2dismod mpm_prefork

# 2. Abilita MPM Event
sudo a2enmod mpm_event

# 3. Abilita i moduli necessari per comunicare con PHP-FPM tramite FastCGI
sudo a2enmod proxy proxy_fcgi

# 4. Riavvia Apache per applicare la nuova architettura
sudo systemctl restart apache2
```

## VirtualHost HTTP2

```bash
vim /etc/apache2/sites-available/app.conf
```

```apache
<VirtualHost *:80>
  ServerName app.example.com
  Redirect permanent / https://app.example.com/
</VirtualHost>

<VirtualHost *:443>
  ServerName app.example.com
  DocumentRoot /var/www/app/public

  # ABILITA HTTP/2: Riduce la latenza permettendo il multiplexing su un'unica connessione TCP.
  # Funziona solo con MPM Event o Worker (non con Prefork).
  Protocols h2 http/1.1

  # CONFIGURAZIONE TLS/SSL
  SSLEngine on
  SSLCertificateFile /etc/letsencrypt/live/app.example.com/fullchain.pem
  SSLCertificateKeyFile /etc/letsencrypt/live/app.example.com/privkey.pem

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
  <Directory /var/www/app/public>
    # AllowOverride None: Disabilita l'uso dei file .htaccess. Aumenta le prestazioni (Apache
    # non deve cercare .htaccess in ogni sottocartella) e impedisce modifiche di sicurezza non autorizzate.
    AllowOverride None

    # Require all granted: Concede l'accesso in lettura ai file contenuti in questa cartella.
    Require all granted
  </Directory>

  <Directory /var/www/app/config>
    Require all denied
  </Directory>

  # Blocco di specifici file sensibili
  <FilesMatch "^\.env">
    Require all denied
  </FilesMatch>

  <Directory /var/www/app/public/admin>
    # Consente l'accesso solo dal loopback locale e dalla VPN aziendale
    Require ip 127.0.0.1
    Require ip 192.168.1.0/24
    Require ip 10.8.0.50
  </Directory>

  <Directory /var/www/app/public/internal>
    <RequireAll>
        # Deve venire dalla rete aziendale...
        Require ip 192.168.1.0/24
        # ...MA NON da questo specifico IP compromesso/ospite
        Require not ip 192.168.1.100
    </RequireAll>
  </Directory>

  <Directory /var/www/app/public/protected>
    AuthType Basic
    AuthName "Area Riservata Corso Security"
    AuthUserFile /etc/apache2/.htpasswd

    # Richiede che l'utente inserisca un login/password valido
    Require valid-user

    # Oppure limita l'accesso a uno specifico utente del file .htpasswd:
    # Require user admin
  </Directory>
</VirtualHost>
```

```bash
# Abilita la configurazione
sudo a2ensite app.conf
# Ricarica Apache
sudo systemctl reload apache2
```

## Cenni su Perl e CGI (Legacy)

- Che cos'era il CGI (Common Gateway Interface): Ad ogni richiesta HTTP per uno script Perl/Bash, il server web avviava un nuovo processo di sistema (fork), eseguiva lo script e ne restituiva l'output.
- Perché è stato abbandonato: Creare un nuovo processo di sistema per ogni singola chiamata genera un overhead di CPU/RAM enorme, esponendo il server a facili attacchi DoS.
- Sostituto moderno: Architetture come FastCGI (PHP-FPM, FastCGI Process Manager, per PHP) o WSGI (Web Server Gateway Interface, per Python) mantengono un pool di processi worker costantemente attivi in RAM, ricevendo le richieste tramite socket senza dover ricreare processi a ogni chiamata.
