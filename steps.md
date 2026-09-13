# Ubuntu

_su UTM_
crea nuovo indirizzo MAC

_sulla VM_
imposta keyboard in inglese

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

### Se per qualche ragione il comando dovesse fallire, possiamo farlo a mano

Ad esempio, se abbiamo fatto hardening prima ancora di copiare la chiave

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

### Sottosezione Didattica: Il motore apparmor_parser

AppArmor non legge i file di testo in `/etc/apparmor.d/` a ogni chiamata di sistema. L'utility `apparmor_parser` traduce la sintassi del profilo in una tabella binaria (automa a stati finiti) e la carica direttamente nello spazio di memoria del Kernel via securityfs.

- `apparmor_parser -a /etc/apparmor.d/profilo` (Add): Carica un nuovo profilo in memoria.
- `apparmor_parser -r /etc/apparmor.d/profilo` (Replace/Reload): Ricompila e sostituisce un profilo già attivo. È il comando fondamentale per applicare modifiche senza riavviare la macchina o il servizio.
- `apparmor_parser -R /etc/apparmor.d/profilo` (Remove): Scarica il profilo dalla memoria del kernel.
- `sudo apparmor_parser -S /etc/apparmor.d/profilo > /dev/null` (Stdout/Check): Compila il profilo in stdout (utilissimo per testare il profilo prima del deploy).

# Httpd

```bash
sudo apt update
sudo apt upgrade
sudo apt install apache2
sudo sysctl enable --now apache2
```

# Tomcat

```bash
sudo apt install default-jdk
java -version

sudo groupadd tomcat
sudo useradd -s /bin/false -g tomcat -d /opt/tomcat tomcat

cd /tmp
curl -O https://dlcdn.apache.org/tomcat/tomcat-10/v10.1.59/bin/apache-tomcat-10.1.59.tar.gz
sudo mkdir -p /opt/tomcat
sudo tar xzvf apache-tomcat-10.1.59.tar.gz -C /opt/tomcat --strip-components=1

cd /opt/tomcat
sudo chgrp -R tomcat /opt/tomcat
sudo chmod -R g+r conf
sudo chmod g+x conf
sudo chown -R tomcat webapps/ work/ temp/ logs/

sudo vim /etc/systemd/system/tomcat.service
```

```ini
[Unit]
Description=Apache Tomcat Web Application Container
After=network.target

[Service]
Type=forking

User=tomcat
Group=tomcat

Environment="JAVA_HOME=/usr/lib/jvm/default-java"
Environment="CATALINA_PID=/opt/tomcat/temp/tomcat.pid"
Environment="CATALINA_HOME=/opt/tomcat"
Environment="CATALINA_BASE=/opt/tomcat"

ExecStart=/opt/tomcat/bin/startup.sh
ExecStop=/opt/tomcat/bin/shutdown.sh

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now tomcat
```
