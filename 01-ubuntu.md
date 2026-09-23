# Ubuntu

## Setup

_sul Mac_

1. Installa UTM
2. Scarica una VM prebuilt dalla gallery
3. crea nuovo indirizzo MAC
4. Avvia la VM

_sulla VM_

imposta keyboard in italiano

```bash
sudo apt update
sudo apt upgrade
ip a # 192.168.64.2
```

## SSH

_sul Mac_

```bash
ssh-keygen -t ed25519 -C "admin-corso"
ssh-copy-id -i ~/.ssh/admin-corso_ed25519.pub ubuntu@192.168.64.2
```

Se per qualche ragione il comando dovesse fallire (ad esempio, se abbiamo fatto hardening prima ancora di copiare la chiave), possiamo farlo a mano:

```bash
cat ~/.ssh/admin-corso_ed25519.pub # copia il testo della chiave pubblica
```

_sulla VM_

```bash
vim ~/.ssh/authorized_keys # incolla il testo della chiave pubblica
chmod 600 ~/.ssh/authorized_keys
```

### Hardening SSH

_sulla VM_

I file dentro /etc/ssh/sshd_config.d/ vengono letti in **ordine alfabetico** e il parser di OpenSSH **applica il primo valore trovato** per ogni direttiva. Su molte immagini cloud (es. Ubuntu Cloud / AWS / UTM), esiste già un file tipo 50-cloud-init.conf che imposta `PasswordAuthentication yes`. Per assicurarci che la nostra configurazione prevalga, usiamo il prefisso `00-`.

> Nota di sicurezza: Prima di riavviare o ricaricare il servizio SSH, **mantieni sempre aperta una seconda sessione terminale** per evitare di rimanere chiuso fuori dal server in caso di errore.

```bash
vim /etc/ssh/sshd_config.d/00-hardening.conf
```

```ini
# Richiede un utente non-root che possa usare sudo
PermitRootLogin no
# Previene attacchi brute-force sulle password
PasswordAuthentication no
# Abilita l'autenticazione tramite chiave SSH
PubkeyAuthentication yes
# Riduce la superficie di attacco disabilitando interfacce grafiche
X11Forwarding no
# Limita il numero massimo di tentativi di autenticazione per connessione
MaxAuthTries 3
# Controlla ogni 5 minuti che la connessione sia attiva
ClientAliveInterval 300
# Disconnette il client se non risponde a 2 controlli consecutivi
ClientAliveCountMax 2
```

```bash
# Test sintattico del file di configurazione
sudo sshd -t
# Verifica quali impostazioni effettive (valutate) sta usando SSH
sudo sshd -T | grep -iE 'passwordauthentication|permitrootlogin|pubkeyauthentication'
# Riavvia il servizio solo dopo aver verificato che tutto sia corretto
sudo systemctl restart ssh
```

### Connessione

_sul Mac_

```bash
ssh -i ~/.ssh/admin-corso_ed25519 ubuntu@192.168.64.2
```

_per semplificare il collegamento_

```bash
vim ~/.ssh/config
```

```ini
Host ubuntu-lab
    HostName 192.168.64.2
    User ubuntu
    IdentityFile ~/.ssh/admin-corso_ed25519
```

```bash
ssh ubuntu-lab
```

## UFW

UFW (_Uncomplicated Firewall_) non è un firewall autonomo, ma un'interfaccia utente semplificata (frontend) scritta in Python per gestire il filtro pacchetti del kernel Linux (`iptables`/`nftables`).

1. **Default Deny (Principio del Minimo Privilegio)**: Blocca tutto il traffico in ingresso tranne quello esplicitamente autorizzato.
2. **Stateful Firewall**: UFW sfrutta il tracciamento delle connessioni del kernel (`conntrack`). Quando si consente il traffico in uscita (_outgoing_), la risposta in ingresso viene fatta passare automaticamente.
3. **Integrazione con App Profiles**: UFW legge i file da `/etc/ufw/applications.d/` forniti dai pacchetti software (come `apache2` o `openssh-server`), permettendo di aprire le porte tramite il nome del servizio.

```bash
sudo apt install ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
 # Abilitazione di Httpd, una volta installato apache2 (02-httpd.md)
sudo ufw allow 'Apache Full'

# Creazione di un profilo UFW personalizzato per Tomcat, utile per quando faremo l'esercitazione singola istanza (03-tomcat.md)
sudo vim /etc/ufw/applications.d/tomcat
```

```ini
[Tomcat]
title=Tomcat
description=Apache Tomcat Application Server (Standalone)
ports=8080/tcp
```

```bash
sudo ufw app update Tomcat
sudo ufw allow Tomcat
sudo ufw enable
sudo ufw status verbose
```

Si può anche aggiungere una regola puntuale direttamente sulla porta con un commento esplicativo:

```bash
sudo ufw allow 8080/tcp comment 'Tomcat HTTP Connector'
```

Per eliminare una regola attiva:

```bash
sudo ufw status numbered
sudo ufw delete N
```

## AppArmor

AppArmor è un modulo di sicurezza del kernel Linux basato su _Mandatory Access Control_ (MAC). A differenza dei classici permessi Unix (_Discretionary Access Control_ - DAC), che dipendono dall'utente che esegue un file, AppArmor applica regole rigide legate direttamente al percorso dell'eseguibile (_path-based profiling_).

Anche se un processo viene compromesso o gira come `root`, AppArmor gli impedirà di eseguire azioni non esplicitamente previste dal suo profilo (es. accedere a `/etc/shadow` o aprire una shell `/bin/sh`).

1. **Enforce (Produzione)**: AppArmor blocca attivamente qualsiasi chiamata di sistema non autorizzata e la traccia nei log.
2. **Complain (Testing/Sviluppo)**: AppArmor non blocca le violazioni, ma le esegue comunque registrandole nei log di sistema. È indispensabile per collaudare nuovi profili senza interrompere i servizi.

```bash
sudo apt update
sudo apt install -y apparmor-utils
sudo aa-status
```

Per aggiungere `curl` in modalità _complain_:

```bash
sudo aa-autodep /usr/bin/curl # genera lo scheletro del profilo per curl
# oppure, per una generazione interattiva:
sudo aa-genprof /usr/bin/curl

sudo aa-complain /usr/bin/curl # imposta la modalità Complain
sudo aa-status # verifica che il profilo sia in Complain
```

Visualizziamo il file autogenerato:

```bash
sudo cat /etc/apparmor.d/usr.bin.curl
```

```ini
abi <abi/3.0>,

include <tunables/global>

# L'opzione flags=(complain) indica che il profilo nasce in modalità log/complain
/usr/bin/curl flags=(complain) {
  include <abstractions/base>

  # Permette a curl di mappare in memoria (m) e leggere (r) il suo stesso eseguibile
  /usr/bin/curl mr,
}
```

Per impedire a `curl` di accedere al file `/etc/passwd`:

```bash
sudo vim /etc/apparmor.d/usr.bin.curl
```

```ini
abi <abi/3.0>,

include <tunables/global>

/usr/bin/curl {
  include <abstractions/base>
  include <abstractions/nameservice>

  # Regola di blocco esplicita per la demo
  deny /etc/passwd r,
}
```

Applichiamo le regole in modalità Enforce:

```bash
sudo aa-enforce /usr/bin/curl
```

Per verificarne il funzionamento, apri un terminale per il monitoraggio dei log:

```bash
sudo journalctl -kf | grep -i apparmor
# Oppure:
sudo dmesg | grep -i apparmor
```

Da un altro terminale esegui i test:

```bash
curl file:///etc/hosts # consentito
curl file:///etc/passwd # bloccato da AppArmor!
```

Per ripristinare ed eliminare il profilo di test:

```bash
sudo aa-disable /usr/bin/curl
sudo rm -f /etc/apparmor.d/usr.bin.curl
```

### Il motore `apparmor_parser`

Sotto il cofano, `aa-enforce` e `aa-complain` utilizzano l'utility `apparmor_parser`.

AppArmor non rilegge i file di testo in `/etc/apparmor.d/` a ogni chiamata di sistema. `apparmor_parser` traduce la sintassi del profilo in una tabella binaria (automa a stati finiti) e la carica direttamente nello spazio di memoria del Kernel tramite `securityfs`.

- `sudo apparmor_parser -a /etc/apparmor.d/profilo` (_Add_): Carica un nuovo profilo in memoria.
- `sudo apparmor_parser -r /etc/apparmor.d/profilo` (_Replace/Reload_): Ricompila e sostituisce un profilo già attivo. È il comando fondamentale per applicare modifiche senza riavviare il servizio.
- `sudo apparmor_parser -R /etc/apparmor.d/profilo` (_Remove_): Scarica il profilo dalla memoria del kernel.
- `sudo apparmor_parser -S /etc/apparmor.d/profilo > /dev/null` (_Stdout/Check_): Verifica la sintassi del profilo senza caricarlo nel kernel.

## Permessi POSIX e Utenti di Servizio

### Notazione Ottale dei Permessi

Ciascuna delle tre cifre definisce i permessi per tre classi: **Proprietario (u)**, **Gruppo (g)**, e **Altri (o)**.

- `4` = Lettura (`r`)
- `2` = Scrittura (`w`)
- `1` = Esecuzione (`x`)

Sommando i valori si compongono i permessi: ad esempio `7` (`4 + 2 + 1`) indica tutti i permessi, `6` (`4 + 2`) indica lettura e scrittura, `5` (`4 + 1`) indica lettura ed esecuzione.

#### Valori comuni

- `644` (`rw-r--r--`): File di configurazione/testo standard.
- `755` (`rwxr-xr-x`): Eseguibili e directory (l'accesso/attraversamento di una directory richiede il bit `x`).
- `700` (`rwx------`) / 600 (`rw-------`): Directory e file sensibili (es. chiavi private SSH).

```bash
sudo addgroup corso-group
sudo adduser --system --no-create-home --ingroup corso-group corso-user
```

L'opzione `--system` crea un account dedicato esclusivamente a un servizio o demone.

### Differenze chiave tra Utente Umano e Utente di Sistema

- **UID riservato**: Su Debian/Ubuntu gli utenti umani partono da UID `1000`. Gli utenti di sistema usano l'intervallo `100`-`999`.
- **Nessuna scadenza password**: Non sono soggetti alle politiche di cambio/scadenza password di `/etc/login.defs`.
- **Isolamento dei processi**: Riduce i rischi se il processo viene compromesso.

```bash
touch ~/test_permessi.txt
# Imposta proprietario e gruppo
sudo chown corso-user:corso-group ~/test_permessi.txt
ls -l ~/test_permessi.txt
# In alternativa, per cambiare solo il gruppo:
sudo chgrp corso-group ~/test_permessi.txt
```

```bash
# Modifica dei permessi
chmod 644 ~/test_permessi.txt
ls -l ~/test_permessi.txt
chmod 744 ~/test_permessi.txt
ls -l ~/test_permessi.txt
chmod 600 ~/test_permessi.txt
ls -l ~/test_permessi.txt
```

```bash
# Pulizia
rm ~/test_permessi.txt
sudo deluser corso-user
sudo delgroup corso-group
```

### Bit speciali

| Bit                       | Valore Ottale | Notazione Simbolica | Effetto su File                                                 | Effetto su Directory                                                                    |
| ------------------------- | ------------- | ------------------- | --------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| **SUID** (_Set User ID_)  | 4000          | `u+s` (`rws------`) | Esegue il file con i privilegi del proprietario del file.       | Nessun effetto significativo su Linux.                                                  |
| **SGID** (_Set Group ID_) | 2000          | `g+s` (`---rws---`) | Esegue il file con i permessi del gruppo proprietario del file. | I nuovi file/cartelle creati all'interno ereditano il gruppo della cartella padre.      |
| **Sticky Bit**            | 1000          | `o+t` (`------rwt`) | Nessun effetto sui file moderni.                                | Solo il proprietario di un file (o `root`) può eliminarlo o rinominarlo nella cartella. |

#### SUID in azione

Un utente deve poter cambiare la propria password, ma le cifrature risiedono in `/etc/shadow` (accessibile solo a `root`). Il binario `/usr/bin/passwd` ha il bit SUID attivo, quindi viene eseguito con i privilegi di `root`, permettendo l'aggiornamento controllato del file.

> **Rischio di sicurezza**: Se si imposta il bit SUID su interpreti di comandi o editor (`bash`, `vim`, `find`), qualsiasi utente potrà effettuare una _Privilege Escalation_ diretta a `root`.

```bash
ls -l /etc/shadow # Accessibile solo a root
ls -l /usr/bin/passwd # Noterai la 's' nei permessi del proprietario ('rwsr-xr-x')
```

#### SGID per cartelle condivise

Se tre colleghi lavorano nella cartella `/lab_special/` (di proprietà del gruppo `sviluppatori`), ogni volta che l'utente `mario` crea un file, quel file appartiene al suo gruppo primario (`mario`). Gli altri colleghi non possono modificarlo finché `mario` non cambia manualmente il gruppo del file.

Applicando il bit SGID alla cartella (`chmod g+s /lab_special/`), il sistema operativo forza tutti i nuovi file creati all'interno ad ereditare automaticamente il gruppo della cartella padre (`sviluppatori`), indipendentemente da chi li crea.

```bash
mkdir ~/lab_special && cd ~/lab_special
# Applica il bit SGID per forzare l'ereditarietà del gruppo
chmod 2775 .   # Corrisponde a chmod g+s .
ls -ld .       # Noterai 'drwxrwsr-x' (la 's' nel gruppo)
```

#### Sticky Bit per directory temporanee

Nella cartella temporanea `/tmp`, tutti gli utenti hanno i permessi di scrittura per poter creare i propri file di lavoro. Tuttavia, in POSIX standard, chiunque abbia i permessi di scrittura su una cartella può cancellare qualsiasi file al suo interno, anche se appartiene a un altro utente. Senza Sticky Bit, `mario` potrebbe cancellare i file temporanei di `luigi`.

Lo Sticky Bit impedisce agli utenti di cancellare o rinominare file che non gli appartengono. In una cartella con Sticky Bit attiva, solo il proprietario del singolo file (o `root`) può eliminarlo.

```bash
# Applica lo Sticky Bit (es. come avviene in /tmp)
chmod 1777 .   # Corrisponde a chmod +t .
ls -ld .       # Noterai 'drwxrwxrwt' (la 't' finale)
```

## Systemd e Hardening dei Servizi

I servizi Linux moderni non dovrebbero girare con permessi illimitati. Systemd consente di applicare direttive di isolamento (_sandboxing_) direttamente nell'unit file del servizio.

> **IMPORTANTE SULLA SINTASSI SYSTEMD:**
> I file di unità systemd **NON supportano i commenti a fine riga** (inline) contrassegnati dal carattere `#`.
> Qualsiasi carattere `#` inserito a fine riga viene considerato parte del valore della direttiva, compromettendo la configurazione o causando il mancato funzionamento della direttiva stessa. Tutti i commenti devono risiedere su **righe dedicate**.

### Le principali direttive di isolamento

- `ProtectSystem=strict`: Monta l'intero file system in sola lettura (`/usr`, `/boot`, `/etc`). Per consentire la scrittura in percorsi specifici (es. log o dati), si usano le direttive `ReadWritePaths=` o `StateDirectory=`.
- `ProtectHome=true`: Rende del tutto invisibili e inaccessibili le directory `/home`, `/root` e `/run/user`.
- `PrivateTmp=true`: Isola la cartella `/tmp` del servizio tramite un mount namespace dedicato, impedendo a processi malevoli di leggere o manipolare i file temporanei dell'applicazione.
- `NoNewPrivileges=true`: Impedisce al processo (e ai suoi figli) di acquisire nuovi privilegi (ignora SUID/GUID).
- `CapabilityBoundingSet=`: Definisce quali Linux Capabilities riservare al processo (es. impedisce di associare porte sotto la 1024 o di modificare l'orologio di sistema).

### Demo Pratica Hardening

Creiamo una prima versione del servizio senza protezioni:

```bash
sudo vim /etc/systemd/system/test-hardening.service
```

```ini
[Unit]
Description=Test Hardening Systemd

[Service]
# Service tipo oneshot per eseguire una singola azione
Type=oneshot
ExecStart=/bin/bash -c "echo 'Infiltrato' > /root/test.txt"
```

```bash
sudo systemctl daemon-reload
sudo systemctl start test-hardening.service
ls -l /root/test.txt # Il file è stato creato!
sudo rm -f /root/test.txt
```

Ora applichiamo le direttive di hardening al servizio (notare che i commenti sono su righe separate):

```bash
sudo vim /etc/systemd/system/test-hardening.service
```

```ini
[Unit]
Description=Test Hardening Systemd

[Service]
Type=oneshot
ExecStart=/bin/bash -c "echo 'Infiltrato' > /root/test.txt"

# DIRETTIVE DI HARDENING

# Monta l'intero sistema operativo in modalità Read-Only per questo processo
ProtectSystem=strict
# rende le cartelle /root e /home del tutto invisibili e inaccessibili al servizio
ProtectHome=true
```

Ricarichiamo e testiamo l'errore:

```bash
sudo systemctl daemon-reload
sudo systemctl start test-hardening.service # Il comando fallirà con errore di Permesso Negato!

# Analisi della sicurezza dell'unità
systemd-analyze security test-hardening.service # 9.0 UNSAFE!
```

### Profilo di Hardening Completo (Sandbox Avanzata)

Ecco l'esempio di un file di unità systemd con hardening massimo configurato correttamente (commenti su righe proprie):

```bash
sudo vim /etc/systemd/system/test-hardening.service
```

```ini
[Unit]
Description=Test Hardening Systemd Avanzato

[Service]
Type=oneshot
ExecStart=/bin/bash -c "echo 'Infiltrato' > /root/test.txt"

# ISOLAMENTO UTENTE E RETE
# Genera un utente e gruppo effimeri e dedicati per la durata del servizio
DynamicUser=yes
# Isola completamente lo stack di rete (crea un loopback vuoto dedicato)
PrivateNetwork=true
# Crea un namespace utente separato dal resto del sistema
PrivateUsers=true
# Blocca qualsiasi traffico IP in ingresso e in uscita
IPAddressDeny=any
# Permessi file visibili solo dall'utente del servizio
UMask=0077

# PROTEZIONI FILE SYSTEM
# Rende l'intero file system in sola lettura (/usr, /boot, /etc, ecc.)
ProtectSystem=strict
# Nasconde completamente le directory /home, /root e /run/user
ProtectHome=true
# Assegna una cartella /tmp e /var/tmp privata e isolata dagli altri processi
PrivateTmp=true
# Blocca l'accesso ai dispositivi fisici in /dev
PrivateDevices=true
# Applica la policy restrittiva per l'accesso ai dispositivi
DevicePolicy=closed
# Rende in sola lettura le variabili del kernel in /proc/sys e /sys
ProtectKernelTunables=true
# Impedisce al servizio di caricare o scaricare moduli del kernel
ProtectKernelModules=true
# Rende in sola lettura la gerarchia dei cgroups in /sys/fs/cgroup
ProtectControlGroups=true

# ISOLAMENTO KERNEL E PROC
# Impedisce al servizio di modificare l'ora di sistema
ProtectClock=true
# Blocca l'accesso al buffer dei log del kernel (dmesg)
ProtectKernelLogs=true
# Nasconde i processi degli altri utenti in /proc
ProtectProc=invisible
# Restringe /proc mostrando solo i dati dei PID
ProcSubset=pid
# Impedisce al servizio di cambiare l'hostname della macchina
ProtectHostname=true

# PROTEZIONI PRIVILEGI ED ESECUZIONE
# Impedisce al processo di acquisire nuovi privilegi (ignora SUID/SGID)
NoNewPrivileges=true
# Rimuove tutte le Linux Capabilities (nessun potere da root)
CapabilityBoundingSet=
# Impedisce l'uso dello scheduling in tempo reale
RestrictRealtime=true
# Vieta la creazione di file con bit SUID o SGID attivi
RestrictSUIDSGID=true
# Blocca il cambio dell'ABI del kernel
LockPersonality=true
# Blocca pagine di memoria contemporaneamente scrivibili ed eseguibili
MemoryDenyWriteExecute=true

# FILTRO SYSCALL E NAMESPACES
# Permette solo le system call minime indispensabili per i servizi standard
SystemCallFilter=@system-service
# Blocca esplicitamente gruppi di chiamate di sistema pericolose
SystemCallFilter=~@resources @privileged @mount @debug @clock @module @reboot @swap
# Disabilita la creazione di nuovi namespace Linux
RestrictNamespaces=true
# Impedisce l'apertura di qualsiasi socket di rete
RestrictAddressFamilies=none
# Disabilita le system call per architetture diverse da quella nativa
SystemCallArchitectures=native
```

Verifica il punteggio di sicurezza:

```bash
sudo systemctl daemon-reload
systemd-analyze security test-hardening.service # 0.2 SAFE!
```

Per ripristinare il sistema ed eliminare la demo:

```bash
sudo rm -f /etc/systemd/system/test-hardening.service
sudo systemctl daemon-reload
```
