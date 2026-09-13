# Ubuntu

## SSH

_sul Mac_

```bash
ssh-keygen -t ed25519 -C "admin-corso"
ssh-copy-id -i ~/.ssh/admin-corso_ed25519.pub ubuntu@192.168.64.2
```

Se per qualche ragione il comando dovesse fallire, possiamo farlo a mano (ad esempio, se abbiamo fatto hardening prima ancora di copiare la chiave):

```bash
cat admin-corso_ed25519.pub # copia il testo della chiave pubblica
```

_sulla VM_

```bash
vim ~/.ssh/authorized_keys # incolla il testo della chiave pubblica
chmod 600 ~/.ssh/authorized_keys
```

### Hardening

_sulla VM_

```bash
vim /etc/ssh/sshd_config.d/hardening.conf
```

```ini
PermitRootLogin no # richiede un utente non-root, che può fare sudo
PasswordAuthentication no # previene attacchi brute-froce sulla password
PubkeyAuthentication yes # abilita l'autenticazione tramie chiave SSH
X11Forwarding no # riduce la superficie di attacco
MaxAuthTries 3 # limita il numero massimo di connessioni prima di interromperela comuncazione
ClientAliveInterval 300 # controlla ogni 5 minuti che la connessione sia attiva
ClientAliveCountMax 2 # dopo due tentativi, disconnette
```

```bash
sudo sshd -t # verifica che non ci siano errori sintattici`
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

UFW non è un firewall reale, ma un'interfaccia utente semplificata (frontend) scritta in Python per gestire la configurazione del filtro pacchetti del kernel Linux (iptables / nftables).

1. Default Deny (Principio del Minimo Privilegio): Bloccare tutto il traffico in ingresso tranne quello esplicitamente autorizzato.
2. Stateful Firewall: UFW sfrutta il tracciamento delle connessioni del kernel (conntrack). Quando consenti il traffico in uscita (outgoing), la risposta in ingresso viene automaticamente fatta passare senza dover aprire porte extra.
3. Integrazione con App Profiles: UFW legge i file da /etc/ufw/applications.d/ forniti dai software installati (come apache2 o openssh-server), permettendo di aprire i servizi tramite nome (es. Apache Full) anziché per numeri di porta singoli.

```bash
sudo apt install ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow 'Apache Full'
sudo nano /etc/ufw/applications.d/tomcat
```

```ini
[Tomcat]
title=Apache Tomcat Servlet Container
description=Apache Tomcat Application Server (Standalone)
ports=8080/tcp
```

```bash
sudo ufw app update Tomcat
sudo ufw allow Tomcat
sudo ufw enable
sudo ufw status verbose
```

Si può anche aggiungere una regola puntualmente sulla porta:

```bash
sudo ufw allow 8080/tcp comment 'Tomcat HTTP Connector'
```

Per eliminare una regola attiva:

```bash
sudo ufw status numbered
sudo ufw delete N
```

## AppArmor

AppArmor è un modulo di sicurezza del kernel Linux basato su Mandatory Access Control (MAC). A differenza dei classici permessi Unix (DAC), che dipendono dall'utente che esegue un file, AppArmor applica regole rigide legate direttamente al percorso del programma (path-based profiling).

Anche se un processo viene compromesso o gira come root, AppArmor gli impedirà di eseguire azioni non esplicitamente previste dal suo profilo (es. accedere a /etc/shadow o aprire una shell bash).

1. Enforce (Produzione): AppArmor blocca attivamente qualsiasi chiamata di sistema non autorizzata e la traccia nei log.
2. Complain (Testing/Sviluppo): AppArmor non blocca le violazioni, ma le permette registrandole nei log di sistema. È indispensabile per creare e collaudare nuovi profili senza interrompere il servizio.

```bash
sudo apt update
sudo apt install -y apparmor-utils
sudo aa-status
```

Per aggiungere cURL in complain:

```bash
sudo aa-autodep /usr/bin/curl # genera lo scheletro del profilo per curl
# oppure, per una generazione interattiva in base all'utilizzo:
sudo aa-genprof /usr/bin/curl
sudo aa-complain /usr/bin/curl # mette curl in modalità Complain
sudo aa-status # verifica che il profilo sia in Complain
```

Il file autogenerato da `aa-autodep` è il seguente:

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

Per impedire a cURL di accedere al file delle password:

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

```bash
sudo aa-enforce /usr/bin/curl
```

Per verificarne il funzionamento:

```bash
sudo journalctl -kf | grep -i apparmor
# oppure
sudo dmesg | grep -i apparmor
```

Su un altro terminale:

```bash
curl file:///etc/hosts # passato
curl file:///etc/passwd # bloccato
```

Per ripristinare:

```bash
sudo aa-disable /usr/bin/curl
sudo rm -f /etc/apparmor.d/usr.bin.curl
```

### Il motore apparmor_parser

Sotto la scocca, `aa-enforce` e `aa-complain` usano `apparmor_parser`.

AppArmor non legge i file di testo in `/etc/apparmor.d/` a ogni chiamata di sistema. L'utility `apparmor_parser` traduce la sintassi del profilo in una tabella binaria (automa a stati finiti) e la carica direttamente nello spazio di memoria del Kernel via securityfs.

- `apparmor_parser -a /etc/apparmor.d/profilo` (Add): Carica un nuovo profilo in memoria.
- `apparmor_parser -r /etc/apparmor.d/profilo` (Replace/Reload): Ricompila e sostituisce un profilo già attivo. È il comando fondamentale per applicare modifiche senza riavviare la macchina o il servizio.
- `apparmor_parser -R /etc/apparmor.d/profilo` (Remove): Scarica il profilo dalla memoria del kernel.
- `sudo apparmor_parser -S /etc/apparmor.d/profilo > /dev/null` (Stdout/Check): Compila il profilo in stdout (utilissimo per testare il profilo prima del deploy).

## POSIX

Notazione Ottale dei Permessi: Ciascuna delle tre cifre definisce i permessi per tre classi distinte: Utente/Proprietario (u), Gruppo (g), e Altri (o).

- 4 = Lettura (r)
- 2 = Scrittura (w)
- 1 = Esecuzione (x)
- Sommando i valori si compongono i permessi: ad esempio 7 (4 + 2 + 1) indica tutti i permessi, 6 (4 + 2) indica lettura e scrittura, 5 (4 + 1) indica lettura ed esecuzione.

Permessi Comuni da Memorizzare:

- 644 (rw-r--r--): Standard per i file di testo/configurazione.
- 755 (rwxr-xr-x): Standard per gli eseguibili e le directory (l'accesso ad una directory richiede il bit x).
- 700 (rwx------) / 600 (rw-------): Riservati a directory e file sensibili (es. la directory .ssh o le chiavi private).

```bash
sudo addgroup corso-group
sudo adduser --system --no-create-home --ingroup corso-group corso-user
```

L'opzione --system (o -system) indica ad adduser di creare un account di sistema anziché un utente umano standard.

Le differenze chiave tra Utente Umano e Utente di Sistema

- ID Utente (UID) riservato:
  Nei sistemi basati su Debian/Ubuntu, gli utenti umani ricevono un UID da 1000 in poi (il tuo primo utente ha UID 1000). Gli utenti di sistema ricevono un UID compreso nell'intervallo 100-999 riservato al sistema operativo.
- Nessun aggiornamento delle scadenze:
  Gli account di sistema non sono soggetti alle politiche di scadenza della password di /etc/login.defs.
- Creazione pulita per i servizi:
  Indica al sistema che l'account serve unicamente per isolare un processo o un demone (come nginx, postgres o il nostro tomcat), senza sovraccaricare la macchina con configurazioni da utente desktop.

```bash
touch ~/test_permessi.txt
# Imposta proprietario e gruppo in un solo comando
sudo chown corso-user:corso-group ~/test_permessi.txt
ls -l ~/test_permessi.txt
# In alternativa, per cambiare solo il gruppo:
sudo chgrp corso-group ~/test_permessi.txt
```

```bash
# 1. Permessi standard per file di testo (Proprietario: rw, Gruppo: r, Altri: r)
chmod 644 ~/test_permessi.txt
ls -l ~/test_permessi.txt
# 2. Rendi il file eseguibile solo per il proprietario (Proprietario: rwx, Gruppo: r, Altri: r)
chmod 744 ~/test_permessi.txt
# 3. Restringi l'accesso esclusivamente al proprietario
chmod 600 ~/test_permessi.txt
```

```bash
rm ~/test_permessi.txt
sudo deluser corso-user
sudo delgroup corso-group
```

### Bit speciali

| Bit                 | Valore Ottale | Notazione Simbolica | Effetto su File                                                                | Effetto su Directory                                                                               |
| ------------------- | ------------- | ------------------- | ------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------- |
| SUID (Set User ID)  | 4000          | u+s (rws------)     | Esegue il file con i permessi del proprietario del file, non di chi lo lancia. | Nessun effetto significativo su Linux.                                                             |
| SGID (Set Group ID) | 2000          | g+s (---rws---)     | Esegue il file con i permessi del gruppo del file.                             | I nuovi file/cartelle creati all'interno ereditano il gruppo della directory padre.                |
| Sticky Bit          | 1000          | o+t (------rwt)     | Nessun effetto sui file moderni.                                               | Solo il proprietario di un file (o root) può cancellarlo o rinominarlo all'interno della cartella. |

#### SUID

Il problema senza SUID: Un utente standard deve poter cambiare la propria password. La password cifrata risiede nel file `/etc/shadow`, che per motivi di sicurezza è leggibile e modificabile esclusivamente dall'utente `root`. Se un utente normale lanciasse il comando `/usr/bin/passwd`, il sistema operativo bloccherebbe l'operazione con un errore di "Permesso Negato", rendendo impossibile per chiunque modificare la propria password senza l'intervento di un amministratore.

La soluzione con SUID: Applicando il bit SUID al file eseguibile (`/usr/bin/passwd`), il sistema operativo esegue quel determinato programma con i privilegi del proprietario del file (`root`), anziché con i privilegi limitati dell'utente che lo ha digitato. Questo permette al comando di accedere temporaneamente a `/etc/shadow`, ma solo ed esclusivamente nei modi e nei limiti previsti dal codice di quel programma.

```bash
ls -l /etc/shadow # scrivibile solo da root
ls -l /usr/bin/passwd # SUID per modificare la propria password
```

Il Rischio di Sicurezza da Evitare: Se un amministratore imposta sbadatamente il bit SUID su un interprete di comandi o su un editor di testo (ad esempio vim, find o bash), qualsiasi utente non privilegiato potrà sfruttare quel programma per eseguire comandi arbitrari o leggere qualsiasi file di sistema come root, ottenendo la Privilege Escalation totale sulla macchina.

#### SGID

Il problema senza SGID: Se tre colleghi lavorano nella cartella `/progetti/` (di proprietà del gruppo `sviluppatori`), ogni volta che l'utente `mario` crea un file, quel file appartiene al suo gruppo primario (`mario`). Gli altri colleghi non possono modificarlo finché `mario` non cambia manualmente il gruppo del file.

La soluzione con SGID: Applicando il bit SGID alla cartella (`chmod g+s /progetti/`), il sistema operativo forza tutti i nuovi file creati all'interno ad ereditare automaticamente il gruppo della cartella padre (`sviluppatori`), indipendentemente da chi li crea.

```bash
mkdir ~/lab_special && cd ~/lab_special

# 2. Applica il bit SGID per forzare l'ereditarietà del gruppo
chmod 2775 .   # Corrisponde a chmod g+s .
ls -ld .       # Noterai drwxrwsr-x (la 's' nel gruppo)
```

#### Sticky Bit

Il problema senza Sticky Bit: Nella cartella temporanea `/tmp`, tutti gli utenti hanno i permessi di scrittura per poter creare i propri file di lavoro. Tuttavia, in POSIX standard, chiunque abbia i permessi di scrittura su una cartella può cancellare qualsiasi file al suo interno, anche se appartiene a un altro utente. Senza Sticky Bit, `mario` potrebbe cancellare i file temporanei di `luigi`.

La soluzione con Sticky Bit: Lo Sticky Bit impedisce agli utenti di cancellare o rinominare file che non gli appartengono. In una cartella con Sticky Bit attiva, solo il proprietario del singolo file (o `root`) può eliminarlo.

```bash
# 1. Applica lo Sticky Bit (es. come avviene in /tmp)
chmod 1777 .   # Corrisponde a chmod +t .
ls -ld .       # Noterai drwxrwxrwt (la 't' finale)
```
