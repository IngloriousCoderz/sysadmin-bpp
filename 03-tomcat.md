# Tomcat

## Setup e Installazione

Per garantire che gli script e gli URL rimangano validi nel tempo durante i laboratori, utilizziamo il repository d'archivio ufficiale di Apache.

```bash
sudo apt update
sudo apt install -y default-jdk
java -version

# Creazione del gruppo e utente di sistema per Tomcat
sudo groupadd --system tomcat
sudo useradd --system -s /usr/sbin/nologin -g tomcat -d /opt/tomcat tomcat

# Download della release specifica da archive.apache.org
cd /tmp
wget https://archive.apache.org/dist/tomcat/tomcat-10/v10.1.60/bin/apache-tomcat-10.1.60.tar.gz

sudo mkdir -p /opt/tomcat
sudo tar xzvf apache-tomcat-10.1.60.tar.gz -C /opt/tomcat --strip-components=1

# Configurazione dei permessi di base
cd /opt/tomcat
sudo chgrp -R tomcat /opt/tomcat
sudo chmod -R g+r conf
sudo chmod g+x conf
sudo chown -R tomcat webapps/ work/ temp/ logs/
```

### Clean-Up di Sicurezza Iniziale (Hardening Applicativo)

In un'installazione di produzione, i pacchetti di default inclusi nella cartella webapps rappresentano un vettore di attacco (vulnerabilità nei sample, esposizione di credenziali su manager/host-manager):

```bash
# RIMOZIONE APPLICAZIONI DEFAULT E FILE INUTILI
# Rimuove Manager, Host-Manager, Documentazione ed Esempi
sudo rm -rf /opt/tomcat/webapps/*
```

## Architettura e Hardening di `server.xml`

Tomcat è un Servlet Container basato su un'architettura gerarchica definita in `conf/server.xml`.

```bash
sudo vim /opt/tomcat/conf/server.xml
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!--
  =============================================================================
  APACHE TOMCAT - ARCHITETTURA CORE (server.xml)
  =============================================================================
-->

<!--
  <Server>: Istanza radice dell'intera JVM Tomcat. Non è un contenitore di webapp.
  - port="8005": Porta TCP per i comandi amministrativi locali (loopback).
  - shutdown="SHUTDOWN": Stringa di controllo inviata sulla porta 8005 per lo spegnimento pulito.
  HARDENING:
  - port="-1" in produzione per disabilitare il socket di shutdown e gestire il ciclo di vita del processo esclusivamente via systemd.
-->
<Server port="8005" shutdown="SHUTDOWN">

  <!--
    ===========================================================================
    1. LIFECYCLE LISTENERS (Hook del Ciclo di Vita)
    ===========================================================================
    Intercettano gli eventi di avvio/arresto dell'istanza per eseguire operazioni
    di inizializzazione, prevenzione memory leak o caricamento di librerie native.
  -->

  <!-- Stampa nei log di avvio le versioni esatte di sistema operativo, JVM e Tomcat -->
  <Listener className="org.apache.catalina.startup.VersionLoggerListener" />

  <!-- Inizializza le librerie native APR/OpenSSL per velocizzare la cifratura TLS rispetto a JSSE -->
  <Listener className="org.apache.catalina.core.AprLifecycleListener" />

  <!-- Previene i Memory Leak nella Metaspace causati da certe API Java/javax durante il redeploy -->
  <Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener" />

  <!-- Inizializza l'albero JNDI globale descritto in <GlobalNamingResources> -->
  <Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener" />

  <!-- Previene i Memory Leak legati ai ThreadLocal rimasti appesi dopo l'arresto di una webapp -->
  <Listener className="org.apache.catalina.core.ThreadLocalLeakPreventionListener" />

  <!--
    ===========================================================================
    2. GLOBAL JNDI RESOURCES (Risorse Condivise)
    ===========================================================================
    Definisce oggetti Java (DataSource DB, credenziali, code JMS) accessibili
    a tutte le applicazioni o ai moduli di autenticazione dell'Engine.
  -->
  <GlobalNamingResources>
    <!-- Database utenti in memoria basato su conf/tomcat-users.xml (usato dal Realm) -->
    <Resource name="UserDatabase" auth="Container"
              type="org.apache.catalina.UserDatabase"
              description="User database che legge/scrive su conf/tomcat-users.xml"
              factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
              pathname="conf/tomcat-users.xml" />
  </GlobalNamingResources>

  <!--
    ===========================================================================
    3. SERVICE (Catalina)
    ===========================================================================
    Raggruppa uno o più Connettori di rete (HTTP, HTTPS, AJP) attorno a un singolo
    motore di elaborazione delle richieste (Engine).
  -->
  <Service name="Catalina">

    <!--
      POOL DI THREAD CONDIVISO (Opzionale, decommentare per la produzione)
      Consente a più connettori di attingere allo stesso pool di thread per risparmiare RAM.
      - maxThreads="150": Limite massimo di thread worker concorrenti.
      - minSpareThreads="4": Thread minimi sempre attivi e pronti in idle.
    -->
    <!--
    <Executor name="tomcatThreadPool" namePrefix="catalina-exec-"
              maxThreads="150" minSpareThreads="4"/>
    -->

    <!--
      =========================================================================
      CONNECTOR 1: HTTP/1.1 (Porta 8080)
      =========================================================================
      Interfaccia di rete primaria per il traffico web in chiaro.
      - port="8080": Porta di ascolto TCP.
      - protocol="HTTP/1.1": Usa il motore NIO (Non-blocking I/O) Coyote.
      - connectionTimeout="20000": Drop della connessione inattiva dopo 20 secondi.
      - redirectPort="8443": Porta verso cui reindirizzare se l'app richiede HTTPS (CONFIDENTIAL).
      - maxParameterCount="1000": Protezione contro attacchi DoS basati su form con troppi parametri.
      HARDENING:
      - server="Apache": Sovrascrive l'header Server nascondendo la versione di Tomcat
    -->
    <Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443"
               maxParameterCount="1000"
               />

    <!--
      CONNECTOR HTTP/1.1 Con Pool di Thread Condiviso (Alternativa all'uso del connettore standard)
    -->
    <!--
    <Connector executor="tomcatThreadPool"
               port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443"
               maxParameterCount="1000"
               />
    -->

    <!--
      =========================================================================
      CONNECTOR 2: HTTPS / TLS (Porta 8443 con HTTP/2)
      =========================================================================
      Interfaccia di rete cifrata.
      - SSLEnabled="true": Attiva il motore di cifratura (JSSE o OpenSSL).
      - UpgradeProtocol: Abilita la negoziazione nativa del protocollo HTTP/2.
      - SSLHostConfig: Definisce Keystore, password e certificati SSL/TLS.
    -->
    <!--
    <Connector port="8443" protocol="org.apache.coyote.http11.Http11NioProtocol"
               maxThreads="150" SSLEnabled="true"
               maxParameterCount="1000"
               >
        <UpgradeProtocol className="org.apache.coyote.http2.Http2Protocol" />
        <SSLHostConfig>
            <Certificate certificateKeystoreFile="conf/localhost-rsa.jks"
                         certificateKeystorePassword="changeit" type="RSA" />
        </SSLHostConfig>
    </Connector>
    -->

    <!--
      =========================================================================
      CONNECTOR 3: AJP/1.3 (Porta 8009 - Apache JServ Protocol)
      =========================================================================
      Protocollo binario ad alte prestazioni per Reverse Proxy (es. Apache httpd via mod_proxy_ajp).
      HARDENING:
      - address="::1" o "127.0.0.1": Deve ascoltare SOLO in loopback per prevenire attacchi (Ghostcat).
      - In produzione aggiungere obbligatoriamente l'attributo secret="VostraPasswordSicura".
    -->
    <!--
    <Connector protocol="AJP/1.3"
               address="::1"
               port="8009"
               redirectPort="8443"
               maxParameterCount="1000"
               />
    -->

    <!--
      =========================================================================
      4. ENGINE (Catalina)
      =========================================================================
      Il motore core che analizza gli header HTTP giunti dai Connettori e li smista
      all'Host virtuale (VirtualHost) di competenza.
      - defaultHost="localhost": Host di fallback se l'header Host: non combacia con nessun <Host>.
      - jvmRoute="jvm1": Impostazione fondamentale nei cluster per la Session Stickiness.
    -->
    <Engine name="Catalina" defaultHost="localhost">

      <!-- CONFIGURAZIONE CLUSTER (Per la replica delle sessioni in RAM tra più nodi Tomcat) -->
      <!--
      <Cluster className="org.apache.catalina.ha.tcp.SimpleTcpCluster"/>
      -->

      <!--
        =======================================================================
        REALM (Modulo Autenticazione e Sicurezza)
        =======================================================================
        Fornisce l'infrastruttura per la gestione di Utenti, Password e Ruoli.
        - LockOutRealm: Wrapper di sicurezza che blocca temporaneamente l'utente dopo
          ripetuti tentativi di login falliti (protezione anti-Brute Force).
        - UserDatabaseRealm: Sotto-realm che consulta la risorsa JNDI UserDatabase.
      -->
      <Realm className="org.apache.catalina.realm.LockOutRealm">
        <Realm className="org.apache.catalina.realm.UserDatabaseRealm"
               resourceName="UserDatabase"/>
      </Realm>

      <!--
        =======================================================================
        5. HOST (VirtualHost)
        =======================================================================
        Equivale a un <VirtualHost> di Apache. Associa un nome di dominio a una cartella di webapp.
        - name="localhost": Nome del dominio.
        - appBase="webapps": Cartella relativa/assoluta da cui caricare le applicazioni (.war o cartelle).
        - unpackWARs="true": Scompatta automaticamente i file .war all'avvio per velocizzare l'esecuzione.
        - autoDeploy="true": Monitora appBase ed esegue il deploy a caldo dei nuovi .war senza riavviare Tomcat.
        HARDENING:
        - autoDeploy="false" e unpackWARs="false" (se possibile) in produzione impediscono il caricamento incontrollato o la modifica a caldo di pacchetti WAR.
      -->
      <Host name="localhost" appBase="webapps"
            unpackWARs="true" autoDeploy="true">

        <!-- VALVE: Middleware per la condivisione della sessione Single Sign-On tra più webapp dell'Host -->
        <!--
        <Valve className="org.apache.catalina.authenticator.SingleSignOn" />
        -->

        <!--
          VALVE: Access Log Middleware
          Intercetta ogni request/response erogata dall'Host ed elabora il log degli accessi.
          - pattern="%h %l %u %t &quot;%r&quot; %s %b": Formato identico allo standard Common Log Format di Apache.
        -->
        <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix="localhost_access_log" suffix=".txt"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />

        <!-- HARDENING:

        VALVE 1: Offuscamento degli Errori (Previene Stack Trace ed esposizione versioni)
        <Valve className="org.apache.catalina.valves.ErrorReportValve"
               showServerInfo="false"
               showReport="false" />

        VALVE 2: Gestione Corretta degli IP Reali dietro Reverse Proxy / Load Balancer
        <Valve className="org.apache.catalina.valves.RemoteIpValve"
               internalProxies="127\.0\.0\.1|10\.\d+\.\d+\.\d+"
               remoteIpHeader="x-forwarded-for"
               protocolHeader="x-forwarded-proto" />

        VALVE 3: Restrizione IP per eventuali app/manager (Esempio RemoteCIDRValve)
        <Valve className="org.apache.catalina.valves.RemoteCIDRValve"
               allow="127.0.0.1, 10.0.0.0/8"
               denyStatus="403" />
        -->
      </Host>
    </Engine>
  </Service>
</Server>
```

## Differenze fra Tomcat 9, 10 e 11

Il punto di svolta nelle versioni recenti è il cambio di namespace imposto dalla transizione da Java EE (Oracle) a Jakarta EE (Eclipse Foundation).

| Caratteristica         | Tomcat 9.x                    | Tomcat 10.x                  | Tomcat 11.x            |
| ---------------------- | ----------------------------- | ---------------------------- | ---------------------- |
| **Specifica EE**       | Java EE 8                     | Jakarta EE 10                | Jakarta EE 11          |
| **Namespace Package**  | `javax.servlet.*`             | `jakarta.servlet.*`          | `jakarta.servlet.*`    |
| **Target Applicativo** | Applicazioni Legacy / Java 8+ | App Moderne / Spring Boot 3+ | App Moderne / Java 17+ |

### Strategia di Migrazione (.WAR Legacy)

Un file `.war` sviluppato per Tomcat 9 con pacchetti `javax.servlet.*` solleverà eccezioni `ClassNotFoundException`su Tomcat 10 o 11.

1. **Migrazione Codice Sorgente (soluzione ideale)**: Aggiornare il codice e le dipendenze in `pom.xml`/`build.gradle` sostituendo gli import `javax.` con `jakarta.`.
   - **CAMBIANO**: `javax.servlet.*`, `javax.persistence.*`, `javax.websocket.*`, `javax.faces.*`, `javax.ejb.*`, `javax.mail.*`, `javax.jms.*`, `javax.annotation.*`, `javax.inject.*`, `javax.validation.*`, `javax.ws.rs.*`, `javax.transaction.*`, `javax.xml.bind.*`, `javax.xml.ws.*`, `javax.xml.soap.*`.
   - **NON CAMBIANO**: I pacchetti facenti parte di SE/JDK core restano `javax.sql.*`, `javax.crypto.*`, `javax.naming.*`, `javax.net.*`, il resto dei `javax.xml.*`.
   - Spring Framework 5 $\rightarrow$ **Spring Framework 6** / **Spring Boot 3**.
   - Hibernate 5 $\rightarrow$ **Hibernate 6**.
2. **Deploy con Conversione Automatica (Tomcat 10.0 legacy)**: Nelle prime versioni di Tomcat 10 era possibile posizionare il file `.war` nella cartella `webapps-javaee/` invece di `webapps/`: Tomcat eseguiva il tool di conversione al volo durante lo scompattamento del WAR.
3. **Tomcat Migration Tool for Jakarta EE (soluzione sysadmin)**: Se hai solo il pacchetto `.war` compilato e non i sorgenti, Apache fornisce un tool da riga di comando che converte il bytecode in automatico convertendo le chiamate `javax` in `jakarta`:

```bash
java -jar jakartaee-migration-*-shaded.jar /path/to/app-legacy.war /path/to/app-jakarta.war
```

> **ATTENZIONE**: Il tool a riga di comando (`jakartaee-migration-*.jar`) converte il bytecode nei file `.class` ed esegue la sostituzione dei nomi di pacchetto, ma **NON intercetta e NON converte**:
>
> - Stringhe nei file di configurazione caricati tramite Reflection (`Class.forName("javax.servlet...")`).
> - Proprietà in file di configurazione custom (`.properties`, `.yaml`).

## Systemd Hardening (Single Instance)

Sfruttiamo **Type=simple** eseguendo `catalina.sh run` in foreground. Questo consente a systemd di monitorare direttamente il processo della JVM e gestire il restart automatico in caso di `OutOfMemoryError`.

```bash
sudo vim /etc/systemd/system/tomcat.service
```

```ini
[Unit]
# Descrizione sintetica del servizio visualizzata nei log e tramite 'systemctl status'
Description=Apache Tomcat Web Application Container
# Garantisce che Tomcat venga avviato solo DOPO che lo stack di rete del sistema operativo è completamente attivo
After=network.target

[Service]
Type=simple

# Utente di sistema non privilegiato con cui verrà eseguito il processo Java
User=tomcat
# Gruppo di sistema associato all'utente per la gestione dei permessi su file e directory
Group=tomcat

# Variabile d'ambiente che indica la root della Java Development Kit (JDK) utilizzata per eseguire Tomcat
Environment="JAVA_HOME=/usr/lib/jvm/default-java"
# Variabile d'ambiente che definisce la directory principale in cui è installato il binario di Tomcat
Environment="CATALINA_HOME=/opt/tomcat"
# Variabile d'ambiente che definisce la directory di lavoro dell'istanza specifica (conf, logs, webapps, temp, work)
Environment="CATALINA_BASE=/opt/tomcat"
# Definisce la posizione del file PID (Process ID) per consentire agli script di arrestare o killare con certezza il processo Java
Environment="CATALINA_PID=/opt/tomcat/temp/tomcat.pid"

# Esecuzione in foreground diretta gestita da systemd
ExecStart=/opt/tomcat/bin/catalina.sh run
# Niente ExecStop
# 128 = processo fermato da un signal + 15 = signal SIGTERM (graceful shutdown)
SuccessExitStatus=143

# Gestione ciclo di vita tramite segnali
Restart=on-failure
RestartSec=10s

# LIMITI CGROUP DEDICATI PER ISTANZA
# Togliamo MemoryHigh, che non fa fallire il processo ma lo rallenta soltanto (il kernel comincia a reclamare aggressivamente la page cache facendo throttling)
# MemoryMax con margine sopra il fabbisogno reale di questa istanza (Xmx512m + Metaspace256m + Thread Stack + overhead nativo ≈ 1170M teorici).
# Useremo lo stesso valore, ricavato con lo stesso criterio, nel sizing dettagliato della sezione Multi-Istanza.
MemoryMax=1350M

# DIRECTORY SCRIVIBILI (Principio del Minimo Privilegio)
# Rende scrivibili solo ed esclusivamente le cartelle specificate, necessarie al funzionamento di Tomcat
ReadWritePaths=/opt/tomcat/logs /opt/tomcat/temp /opt/tomcat/work /opt/tomcat/webapps

# ISOLAMENTO FILE SYSTEM E KERNEL
# Rende l'intero file system del sistema operativo in sola lettura per il servizio
ProtectSystem=strict
# Impedisce del tutto l'accesso alle directory personali degli utenti (/home, /root, /run/user)
ProtectHome=true
# Monta un file system /tmp privato e isolato, inaccessibile agli altri processi di sistema
PrivateTmp=true
# Nasconde i dispositivi fisici (/dev) al servizio, lasciando visibili solo i pseudo-dispositivi essenziali (es. /dev/null, /dev/urandom)
PrivateDevices=true
# Rende in sola lettura le variabili del kernel in /proc/sys e /sys, impedendo modifiche a runtime
ProtectKernelTunables=true
# Blocca il caricamento e la rimozione esplicita di moduli del kernel Linux da parte del servizio
ProtectKernelModules=true
# Rende in sola lettura le gerarchie di cgroups (/sys/fs/cgroup) per evitare modifiche ai limiti di risorsa
ProtectControlGroups=true

# ISOLAMENTO PRIVILEGI
# Impedisce al servizio (e a eventuali processi figli) di acquisire nuovi privilegi tramite execution di binary SetUID/SetGID
NoNewPrivileges=true
# Rimuove completamente tutte le capabilities di Linux dal processo (impedisce azioni da superuser anche a livello root)
CapabilityBoundingSet=
# Disabilita la possibilità di richiedere lo scheduling real-time del kernel Linux
RestrictRealtime=true
# Ignora i bit SetUID e SetGID sui file eseguibili lanciati dall'applicazione
RestrictSUIDSGID=true
# Blocca la modifica dell'architettura di esecuzione o della personalità del kernel via syscall (es. emulazione 32-bit)
LockPersonality=true
# Imposta la maschera dei permessi predefinita per i nuovi file creati (rwxr-x---: lettura/scrittura per tomcat, lettura per il gruppo)
UMask=0027

# SYSCALL FILTERING
# Consente un set di chiamate di sistema (syscall) standard predefinite e sicure per i servizi di sistema
SystemCallFilter=@system-service
# Inverte il filtro (~) e blocca categoricamente i gruppi di syscall pericolose o non necessarie (gestione hardware, clock, reboot, swap, ecc.)
SystemCallFilter=~@resources @privileged @mount @debug @clock @module @reboot @swap
# Limita l'esecuzione delle chiamate di sistema esclusivamente all'architettura nativa della macchina (es. x86_64), bloccando quelle a 32-bit
SystemCallArchitectures=native

[Install]
# Definisce il "target" (livello di esecuzione) a cui agganciare il servizio quando viene abilitato con 'systemctl enable' (multi-user corrisponde alla normale modalità server senza GUI)
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now tomcat
systemd-analyze security tomcat.service # 2.9 OK!
```

Un risultato di 2.9 (OK) su un servizio complesso come Tomcat è eccellente. Dimostra come si possa stringere la morsa della sicurezza tramite il kernel mantenendo al contempo un'applicazione reale del tutto operativa.

Le concessioni necessarie che impediscono il punteggio 0.2 sono la rete, le `ReadWritePaths`, l'uso dell'utente statico `tomcat` e la disattivazione di `MemoryDenyWriteExecute` (indispensabile per il JIT della JVM).

## Gestione della Memoria e Profilazione JVM

### Architettura Memoria JVM

- **Heap Memory (`-Xms`, `-Xmx`)**: Gestita dal Garbage Collector (Young Gen / Old Gen).
- **Off-Heap Memory (Memoria Nativa RAM)**:
  - **Metaspace (`-XX:MetaspaceSize`, `-XX:MaxMetaspaceSize`)**: Metadati delle classi. Se illimitata, i leaker di ClassLoader portano all'OOM Killer del Kernel.
  - **Thread Stack (`-Xss`)**: Allocazione nativa per thread (default 1MB/thread).

### Formula della Memoria Occupata dal Processo Java

$$\text{RAM Totale Processo} \approx \text{Heap (-Xmx)} + \text{MaxMetaspace} + (\text{MaxThreads} \times \text{-Xss}) + \text{DirectMemory} + \text{CodeCache}$$

### Impatti Operativi su Swap e OOM Killer

1. **RSS (Resident Set Size) vs `-Xmx`**: La RAM allocata nel sistema operativo dal processo Java (evidenziata con `top` o `ps aux`) sarà sempre maggiore di `-Xmx`, perché include le aree native (Thread Stack, Metaspace, GC Overhead, JIT Code Cache).
2. **Swap e Risposte del Server**: Quando la RAM fisica della macchina finisce e il sistema va in Swap, la JVM subisce rallentamenti drammatici durante le fasi di Stop-The-World del Garbage Collector, trasformando pause di millisecondi in blocchi di diversi secondi (con conseguenti timeout HTTP 504 dei proxy).
3. **Out-of-Memory: Java vs Kernel Linux**:
   - **`java.lang.OutOfMemoryError` (In-JVM)**: La JVM è viva ma ha esaurito lo spazio nell'Heap (o nel Metaspace). Genera il dump `.hprof` e si arresta se configurata con `-XX:+ExitOnOutOfMemoryError`.
   - **Kernel OOM Killer (OS)**: Il Kernel Linux esaurisce la RAM totale e lo Swap della macchina e uccide brutalmente il processo java inviando un segnale `SIGKILL` (Exit Code 137). In questo scenario non viene generato alcun file `.hprof`.
   - Per diagnosticare un attacco dell'OOM Killer del kernel:

```bash
# Per verificare se il processo è stato ucciso dal MemCG OOM Killer (limite MemoryMax di Systemd)
# oppure dall'OOM Killer di sistema (RAM fisica esaurita):

# 1. Ricerca case-insensitive nei log di sistema tramite journalctl
sudo journalctl -k -g 'out of memory'

# 2. In alternativa, ricerca case-insensitive tramite dmesg
sudo dmesg -T | grep -i oom
```

### NMT (Native Memory Tracking)

Per diagnosticare dove la JVM stia allocando la RAM nativa al di fuori dell'Heap:

1. Aggiungi a `setenv.sh`: `-XX:NativeMemoryTracking=summary` (vedi sotto)
2. Esegui la diagnosi a runtime via CLI:

```bash
# Per eseguire jcmd con PrivateTmp=true e permessi utente corretti:
# 1. Recupera il PID dell'istanza
PID=$(pgrep -f "tomcat.*node1")

# 2. Esegui jcmd all'interno del namespace mount della unit Systemd usando l'utente tomcat
sudo nsenter -t $PID -m sudo -u tomcat jcmd $PID VM.native_memory summary
```

### Tuning tramite setenv.sh

```bash
sudo vim /opt/tomcat/bin/setenv.sh # file nuovo, automaticamente richiamato dagli script di avvio
```

```bash
#!/bin/sh
# ==============================================================================
# TOMCAT JVM TUNING CONFIGURATION
# ==============================================================================

# 1. HEAP MEMORY SETTINGS
# -Xms: Dimensione iniziale dell'Heap
# -Xmx: Dimensione massima dell'Heap (Regola d'oro: Xms = Xmx per evitare reshrink/resize overhead)
CATALINA_OPTS="$CATALINA_OPTS -Xms512m -Xmx512m"

# 2. METASPACE SETTINGS
# Evita la saturazione incontrollata della RAM nativa del sistema operativo
CATALINA_OPTS="$CATALINA_OPTS -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=256m"

# 3. THREAD STACK SIZE
# Riduce la dimensione dello stack per thread se l'app non ha ricorsioni profonde (risparmia RAM nativa)
CATALINA_OPTS="$CATALINA_OPTS -Xss512k"

# 4. GARBAGE COLLECTOR (G1GC - Standard consigliato da Java 9+)
CATALINA_OPTS="$CATALINA_OPTS -XX:+UseG1GC -XX:MaxGCPauseMillis=200"

# 5. DIAGNOSTICA AUTOMATICA OOM (Out Of Memory)
# Forza la JVM a generare un Heap Dump al momento esatto del crash per analisi post-mortem
CATALINA_OPTS="$CATALINA_OPTS -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/opt/tomcat/logs/heap_dump.hprof"
# Termina automaticamente al primo OOM
CATALINA_OPTS="$CATALINA_OPTS -XX:+ExitOnOutOfMemoryError"

# 6. NATIVE MEMORY TRACKING
CATALINA_OPTS="$CATALINA_OPTS -XX:NativeMemoryTracking=summary"

# 7. GC LOGGING (Tracciamento performance Garbage Collection)
CATALINA_OPTS="$CATALINA_OPTS -Xlog:gc*,gc+phases=debug:file=/opt/tomcat/logs/gc.log:time,uptime,pid:filecount=5,filesize=10m"

export CATALINA_OPTS
```

```bash
sudo chmod +x /opt/tomcat/bin/setenv.sh
sudo chown tomcat:tomcat /opt/tomcat/bin/setenv.sh
```

### 1. Interpretazione dei Log del Garbage Collector (`gc.log`)

Analizzando il file `/opt/tomcat/logs/gc.log`, i punti cardine da osservare sono:

- **`Allocation Failure` come causa di una `Pause Young (Normal)`**: La Young Generation è piena, la JVM avvia una pulizia Minor GC. Con G1 (il collector che usiamo) è un evento fisiologico e molto frequente.
- **`Pause Young (Normal)`**: Pausa breve in cui vengono ripuliti gli oggetti a vita breve.
- **`Allocation Failure` come causa di una `Pause Full` (o `Full GC`)**: Qui la stessa etichetta è tutt'altro che innocua. Con G1 una Full GC scatta solo come ultima risorsa, quando il collector non riesce a liberare spazio abbastanza in fretta con le normali pause incrementali: il Garbage Collector deve fermarsi e scansionare l'intero Heap (Young + Old Gen) in modalità Stop-The-World, con pause molto più lunghe. Segnale d'allarme: se la frequenza delle Full GC aumenta e lo spazio libero nell'Old Generation dopo ogni ciclo non cresce, l'applicazione soffre di un Memory Leak.
- **Pause Time Execution**: Verificare che le pause non superino mai il target (`MaxGCPauseMillis=200`). Pause superiori a 1-2 secondi indicano cattivo sizing o swapping della memoria da parte del sistema operativo.

### 2. Analisi dell'Heap Dump (.hprof) con Eclipse MAT (Memory Analyzer Tool)

Quando l'applicazione va in Crash per OOM e genera il file `/opt/tomcat/logs/heap_dump.hprof`, scaricalo ed esaminalo via **Eclipse MAT**:

1. **Leak Suspects Report**: È la prima dashboard generata da MAT. Identifica automaticamente le istanze o i thread che occupano una percentuale anomala dell'Heap (es. _"One instance of `java.util.ArrayList` loaded by `org.apache.catalina.loader.ParallelWebappClassLoader` occupies 85% of the memory"_).
2. **Dominator Tree**: Mostra l'elenco degli oggetti ordinati per **Retained Heap** (la quantità di memoria che verrebbe liberata se l'oggetto venisse rimosso ed escluso dal Garbage Collection).
3. **Path to GC Roots**: Cliccando col tasto destro su un oggetto sospetto nel _Dominator Tree $\rightarrow$ Path to GC Roots $\rightarrow$ exclude weak/soft references_. Rivela la catena esatta di riferimenti (es. una variabile `static` o un thread rimasto appeso) che impedisce al Garbage Collector di distruggere l'oggetto.

### Esercitazione 1: OOM della JVM e Riavvio Automatico

```bash
sudo mkdir -p /opt/tomcat/webapps/leak
sudo vim /opt/tomcat/webapps/leak/index.jsp
```

```jsp
<%@ page import="java.util.*" %>
<%!
    // Collezione statica: impedisce al GC di liberare la memoria
    static List<byte[]> memoryBucket = new ArrayList<>();
%>
<%
    // Alloca 100MB di RAM ad ogni invocazione
    for (int i = 0; i < 20; i++) {
        memoryBucket.add(new byte[5 * 1024 * 1024]);
    }
    out.println("Allocati 100MB in RAM. Elementi totali: " + memoryBucket.size());
%>
```

```bash
sudo chown -R tomcat:tomcat /opt/tomcat/webapps/leak

# Pulisci eventuali precedenti dump prima del test
sudo rm -f /opt/tomcat/logs/*.hprof

sudo systemctl restart tomcat
# Esegui chiamate ripetute per saturare l'heap
curl -ik http://localhost:8080/leak/index.jsp # errore 500 dopo 5 curl
```

Al verificarsi dell'OOM: 1. La JVM scrive il file di dump `/opt/tomcat/logs/heap_dump.hprof`. 2. La flag `-XX:+ExitOnOutOfMemoryError` termina immediatamente il processo Java. 3. Systemd rileva la chiusura con errore e riavvia automaticamente l'istanza dopo 10 secondi.

```bash
# Verifica della presenza del dump e dei log di riavvio
ls -lh /opt/tomcat/logs/heap_dump.hprof
sudo journalctl -u tomcat.service -n 20 --no-pager
```

Riduciamo ulteriormente la dimensione dell'heap per generare subito OOM:

```bash
sudo vim /opt/tomcat/bin/setenv.sh
```

```ini
# modifica questo:
# CATALINA_OPTS="$CATALINA_OPTS -Xms512m -Xmx512m"
CATALINA_OPTS="$CATALINA_OPTS -Xms64m -Xmx64m"
```

```bash
# Pulisci eventuali precedenti dump prima del test
sudo rm -f /opt/tomcat/logs/*.hprof

sudo systemctl restart tomcat
curl -ik http://localhost:8080/leak/index.jsp # errore 500 subito!
sudo ls -lh /opt/tomcat/logs/ # dovrebbe esserci un file .hprof (da 64MB o poco meno) e gc.log
```

### Esercitazione 2: OOM Killer del Kernel (Systemd MemCG vs Heap Dump)

Per mostrare la differenza fondamentale tra un crash gestito dalla JVM e una terminazione forzata dal sistema operativo, configuriamo uno scenario in cui il limite impostato dal Kernel/Systemd è inferiore alla memoria richiesta o allocabile dall'applicazione.

#### 1. Configurazione del Test

Imposta temporaneamente un parametro `-Xmx` superiore al limite `MemoryMax` della unit Systemd dell'istanza di test (oppure riduci `MemoryMax` su tomcat@node1.service):

```bash
# Impostiamo il vincolo cgroup di Systemd a 1200MB per node1
sudo systemctl edit tomcat@node1 --drop-in=override.conf
```

```ini
[Service]
MemoryMax=1200M
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart tomcat@node1
```

Configura `setenv.sh` di `node1` per consentire un Heap fino a 1400MB (superiore ai 1200MB del cgroup):

```bash
sudo vim /var/lib/tomcat/instances/node1/bin/setenv.sh
```

```ini
CATALINA_OPTS="$CATALINA_OPTS -Xms512m -Xmx1400m"
```

#### 2. Esecuzione del Test e Generazione dello Stress

Invocando la pagina di leak (`/leak/index.jsp`) o allocando memoria nativa, la JVM tenterà di espandere l'Heap verso i 1400MB.

```bash
curl -ik http://localhost:8081/leak/index.jsp # una decina di volte, probabilmente
```

#### 3. Diagnostica e Verifica delle Differenze

Al superamento dei 1200MB, si noteranno i seguenti comportamenti:

1. Assenza di Heap Dump:

```bash
ls -la /var/lib/tomcat/instances/node1/logs/*.hprof # non c'è
```

2. Verifica Exit Code di Systemd:

```bash
sudo systemctl status tomcat@node1 # servizio riavviato dopo essere uscito con status=137/KILL
```

3. Ispezione dei Log del Kernel:

```bash
sudo journalctl -k -g 'out of memory' # qualcosa come "Memory cgroup out of memory: Kill process (java) score 950 or sacrifice child Killed process (java) total-vm:... anon-rss:1200100kB"
```

## Architettura Multi-Istanza Scalabile (tomcat@.service)

Invece di duplicare manualmente gli script e perdere le impostazioni di hardening, si utilizza la configurazione **CATALINA_HOME** / **CATALINA_BASE** separando i binari dalle singole istanze operative.

- **CATALINA_HOME** (`/opt/tomcat`): Contiene solo i binari e le librerie condivise (`bin/`, `lib/`).
- **CATALINA_BASE** (`/var/lib/tomcat/instances/`): Contiene la configurazione specifica dell'istanza (`conf/`, `logs/`, `temp/`, `webapps/`, `work/`).

### Sizing della Memoria (Esempio Numerico Reale)

Ipotizziamo di disporre di una Virtual Machine con **4 GB** di RAM fisicamente installata:

- Riserva OS + Apache httpd + SSH/monitoring: **1200 MB**
- Margine di sicurezza non allocato a livello di host (page cache, picchi non pianificati, stabilità del kernel): **196 MB**
- Budget totale per le 2 Istanze Tomcat: **2700 MB** → **1350 MB** di `MemoryMax` (cgroup) per istanza.

> **Nota**: il margine va sottratto dal budget totale _prima_ di dividerlo tra le istanze, non aggiunto dopo a ciascun `MemoryMax`. Se lo aggiungi dopo (es. "+15% a testa"), la somma dei tetti cgroup può superare la RAM fisica della VM: i singoli servizi restano entro il proprio limite, ma è il kernel a rischiare l'OOM sull'intera macchina se entrambe le istanze lo raggiungono insieme.

Sizing per Singola Istanza Tomcat (il tetto cgroup lascia ~50 MB di margine sopra il fabbisogno teorico, per la Page Cache e i picchi del GC):

$$\begin{aligned} \text{RAM Teorica Istanza} &= \text{Heap (-Xmx)} + \text{MaxMetaspace} + (\text{MaxThreads} \times \text{-Xss}) + \text{Overhead Nativo} \\ 1300\text{ MB} &= 700\text{ MB (-Xmx)} + 256\text{ MB (Metaspace)} + (100 \times 1\text{ MB (-Xss)}) + 244\text{ MB (Native/JIT/NIO)} \end{aligned}$$

Configureremo le istanze con: `-Xms700m -Xmx700m -XX:MaxMetaspaceSize=256m -Xss1024k`, e `MemoryMax=1350M` nella unit systemd. Verifica del budget: $1200 + 196 + (2 \times 1350) = 4096$ MB, cioè l'intera RAM fisica della VM.

### 1. Preparazione dell'Albero delle Istanze

```bash
# Disabilita il servizio singolo
sudo systemctl stop tomcat
sudo systemctl disable tomcat

# Crea le strutture per le istanze node1 e node2
sudo mkdir -p /var/lib/tomcat/instances/node1
sudo mkdir -p /var/lib/tomcat/instances/node2

# Copia la struttura di base per ogni istanza
sudo cp -r /opt/tomcat/{conf,logs,temp,webapps,work} /var/lib/tomcat/instances/node1/
sudo cp -r /opt/tomcat/{conf,logs,temp,webapps,work} /var/lib/tomcat/instances/node2/

# Crea cartelle bin dedicate alle istanze per i loro setenv.sh specifici
sudo mkdir -p /var/lib/tomcat/instances/node1/bin
sudo mkdir -p /var/lib/tomcat/instances/node2/bin

# Assegna la proprietà all'utente tomcat
sudo chown -R tomcat:tomcat /var/lib/tomcat/instances/
```

### 2. Differenziazione delle Porte nei file `server.xml``

```bash
sudo vim /var/lib/tomcat/instances/node1/conf/server.xml # fare altrettanto con node2
```

```ini
Shutdown port: -1
Connector HTTP:
  address: 127.0.0.1
  port: 8081 # 8082
  maxThreads: 100
Connector AJP:
  address: 127.0.0.1
  port: 8009 # 8010
  maxThreads: 100
  secret: SecretNodo1 # 2
  secretRequired: true
Engine jvmRoute: node1
```

### 3. Tuning della memoria

```bash
sudo vim /var/lib/tomcat/instances/node1/bin/setenv.sh # fare altrettanto con node2
```

```ini
#!/bin/sh
CATALINA_OPTS="$CATALINA_OPTS -Xms700m -Xmx700m -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=256m -Xss1024k"
CATALINA_OPTS="$CATALINA_OPTS -XX:+UseG1GC -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/var/lib/tomcat/instances/node1/logs/heap_dump.hprof -XX:+ExitOnOutOfMemoryError"
export CATALINA_OPTS
```

```bash
sudo chmod +x /var/lib/tomcat/instances/node1/bin/setenv.sh # fare altrettanto con node2
sudo chown tomcat:tomcat /var/lib/tomcat/instances/node1/bin/setenv.sh # fare altrettanto con node2
```

### 4. Creazione del Template Systemd (`tomcat@.service`)

Il carattere `%i` nel template viene sostituito dinamicamente da systemd con il nome dell'istanza passata dopo la `@` (es. `node1`, `node2`).

```bash
sudo vim /etc/systemd/system/tomcat@.service
```

```ini
[Unit]
Description=Apache Tomcat Instance %i
After=network.target

[Service]
Type=simple

User=tomcat
Group=tomcat

Environment="JAVA_HOME=/usr/lib/jvm/default-java"
Environment="CATALINA_HOME=/opt/tomcat"
Environment="CATALINA_BASE=/var/lib/tomcat/instances/%i"
Environment="CATALINA_PID=/var/lib/tomcat/instances/%i/temp/tomcat.pid"

ExecStart=/opt/tomcat/bin/catalina.sh run
# 128 = processo fermato da un signal + 15 = signal SIGTERM (graceful shutdown)
SuccessExitStatus=143

Restart=on-failure
RestartSec=10s

MemoryMax=1350M

# ISOLAMENTO DINAMICO: Rende scrivibile solo la cartella dell'istanza specifica %i
ReadWritePaths=/var/lib/tomcat/instances/%i/logs /var/lib/tomcat/instances/%i/temp /var/lib/tomcat/instances/%i/work /var/lib/tomcat/instances/%i/webapps

ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
PrivateDevices=true
ProtectKernelTunables=true
ProtectKernelModules=true
ProtectControlGroups=true

NoNewPrivileges=true
CapabilityBoundingSet=
RestrictRealtime=true
RestrictSUIDSGID=true
LockPersonality=true
UMask=0027

SystemCallFilter=@system-service
SystemCallFilter=~@resources @privileged @mount @debug @clock @module @reboot @swap
SystemCallArchitectures=native

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload

# Avvio e abilitazione delle due istanze
sudo systemctl enable --now tomcat@node1
sudo systemctl enable --now tomcat@node2

# Verifica dello stato
sudo systemctl status tomcat@node1 tomcat@node2
```

## Integrato: Load Balancing AJP con Apache Httpd

Ora colleghiamo le due istanze Tomcat a un bilanciatore Apache `httpd` con protocollo binario AJP e _Sticky Sessions_.

### 1. Applicazione per Test Sticky Sessions

Crea su entrambe le istanze un file JSP per visualizzare la rotta del bilanciamento e la sessione attiva:

```bash
sudo mkdir -p /var/lib/tomcat/instances/node1/webapps/ROOT
sudo vim /var/lib/tomcat/instances/node1/webapps/ROOT/session.jsp # fare altrettanto con node2
```

```jsp
<%@ page language="java" contentType="text/html; charset=UTF-8" pageEncoding="UTF-8"%>
<%
    // Recupera la rotta dal cookie JSESSIONID (es. xxxxx.node1 -> node1)
    String activeNode = "N/D (Primo Accesso)";
    String sessionId = request.getSession().getId();

    if (sessionId != null && sessionId.contains(".")) {
        activeNode = sessionId.substring(sessionId.indexOf(".") + 1);
    } else {
        // Fallback: estrae il nome della cartella dell'istanza da catalina.base
        String catalinaBase = System.getProperty("catalina.base");
        if (catalinaBase != null) {
            activeNode = catalinaBase.substring(catalinaBase.lastIndexOf("/") + 1) + " (da catalina.base)";
        }
    }
%>
<!DOCTYPE html>
<html>
<head><title>Test Sticky Session</title></head>
<body>
    <h2>Info Nodo Tomcat</h2>
    <p><strong>JSESSIONID Completo:</strong> <%= sessionId %></p>
    <p><strong>Nodo Rispondente:</strong> <%= activeNode %></p>
    <p><strong>Porta Locale Server:</strong> <%= request.getLocalPort() %></p>
    <p><strong>Data/Ora Server:</strong> <%= new java.util.Date() %></p>
</body>
</html>
```

### 2. VirtualHost Apache Httpd

```bash
# Abilitazione dei moduli di bilanciamento su Apache
sudo a2enmod proxy proxy_ajp proxy_balancer lbmethod_byrequests status ssl rewrite

sudo vim /etc/apache2/sites-available/lb-tomcat.conf
```

```apache
<VirtualHost *:80>
    ServerName localhost

    # Attiva il motore di riscrittura delle URL di Apache
    RewriteEngine On
    # Verifica la condizione: la richiesta NON sta usando il protocollo HTTPS
    RewriteCond %{HTTPS} off
    # Reindirizza permanentemente (HTTP 301) qualsiasi URI verso la controparte cifrata HTTPS
    RewriteRule ^/(.*)$ https://%{HTTP_HOST}/$1 [R=301,L]
</VirtualHost>

<VirtualHost *:443>
    ServerName localhost

    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/mini-site-selfsigned.crt
    SSLCertificateKeyFile /etc/ssl/private/mini-site-selfsigned.key

    # CONFIGURAZIONE DEL BALANCER AJP
    # Definizione del gruppo logico di bilanciamento (cluster) con nome "tomcatcluster"
    <Proxy "balancer://tomcatcluster">
        # Primo nodo del cluster: connessione via AJP su porta 8009, identificato come node1, protetto da password AJP
        BalancerMember "ajp://127.0.0.1:8009" route=node1 secret=SecretNodo1

        # Secondo nodo del cluster: connessione via AJP su porta 8010, identificato come node2, protetto da password AJP
        BalancerMember "ajp://127.0.0.1:8010" route=node2 secret=SecretNodo2

        # Mantiene l'utente legato allo stesso nodo Tomcat leggendo il suffisso del cookie JSESSIONID (Sticky Sessions)
        ProxySet stickysession=JSESSIONID
    </Proxy>

    # PASSAGGIO TRAFFICO E METADATI
    # Mantiene l'header Host originale inviato dal client inoltrandolo intatto a Tomcat
    ProxyPreserveHost On
    # ESCLUSIONE PROXY: Dice ad Apache di gestire internamente /balancer-manager e NON inviarlo a Tomcat
    ProxyPass /balancer-manager !
    # Mappa la radice del sito web (/) verso il cluster di bilanciamento AJP appena definito
    ProxyPass / "balancer://tomcatcluster/"
    # Riscrive le intestazioni degli URL di risposta inviati da Tomcat per nascondere la struttura interna
    ProxyPassReverse / "balancer://tomcatcluster/"

    # DASHBOARD DI MONITORAGGIO (Accessibile solo in locale)
    # Crea un endpoint web all'URL /balancer-manager
    <Location "/balancer-manager">
        # Associa all'URL l'handler nativo di Apache per la GUI di monitoraggio del bilanciatore
        SetHandler balancer-manager
        # Restringe l'accesso alla dashboard di gestione unicamente alle richieste provenienti da localhost
        Require ip 127.0.0.1
    </Location>

    ErrorLog ${APACHE_LOG_DIR}/lb_ssl_error.log
    CustomLog ${APACHE_LOG_DIR}/lb_ssl_access.log combined
</VirtualHost>
```

```bash
sudo a2dissite mini-site.conf # se era ancora abilitato
sudo a2ensite lb-tomcat.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

### 3. Test Iniziale Sticky Session (via cURL)

```bash
# Esegui la prima chiamata salvando i cookie nel file cookie.txt
curl -i -k -c cookie.txt https://localhost/session.jsp

# L'output mostrerà il cookie JSESSIONID con il suffisso del nodo (es. xxxxxxxx.node1)
# Esegui le chiamate successive riutilizzando lo stesso cookie:
curl -i -k -b cookie.txt https://localhost/session.jsp
curl -i -k -b cookie.txt https://localhost/session.jsp
```

_Esito atteso_: Il nodo che risponde rimarrà fisso su node1 (o node2), dimostrando l'efficacia dello _sticky routing_.
