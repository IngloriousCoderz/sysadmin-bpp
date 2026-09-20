# Tomcat

## Setup

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
```

```bash
sudo vim /etc/systemd/system/tomcat.service
```

```ini
[Unit]
# Descrizione sintetica del servizio visualizzata nei log e tramite 'systemctl status'
Description=Tomcat
# Garantisce che Tomcat venga avviato solo DOPO che lo stack di rete del sistema operativo è completamente attivo
After=network.target

[Service]
# Indica a systemd che il processo avviato farà il fork (spawn) di un processo figlio e poi il genitore terminerà (comportamento tipico degli script startup.sh)
Type=forking

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

# Comando o script da eseguire per avviare il servizio Tomcat
ExecStart=/opt/tomcat/bin/startup.sh
# Comando o script da eseguire per arrestare in modo pulito il servizio Tomcat
ExecStop=/opt/tomcat/bin/shutdown.sh

[Install]
# Definisce il "target" (livello di esecuzione) a cui agganciare il servizio quando viene abilitato con 'systemctl enable' (multi-user corrisponde alla normale modalità server senza GUI)
WantedBy=multi-user.target
```

## Architettura e Versioning

Tomcat non è un semplice web server, ma un Servlet Container. La sua architettura è gerarchica:

```bash
sudo vim /opt/tomcat/conf/server.xml
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!--
  =============================================================================
  APACHE TOMCAT - CONFIGURAZIONE ARCHITETTURALE MAIN (server.xml)
  =============================================================================
-->

<!--
  <Server>: Istanza radice dell'intera JVM Tomcat. Non è un contenitore di webapp.
  - port="8005": Porta TCP per i comandi amministrativi locali (loopback).
  - shutdown="SHUTDOWN": Stringa di controllo inviata sulla porta 8005 per lo spegnimento pulito.
  HARDENING: In produzione impostare port="-1" per disabilitare il socket di shutdown
  e gestire il ciclo di vita del processo esclusivamente via systemd.
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

      </Host>
    </Engine>
  </Service>
</Server>
```

## Differenze fra Tomcat 9 e Tomcat 10

La differenza fondamentale risiede nel passaggio di proprietà dei namespace dalle specifiche Java EE (Oracle) alle specifiche Jakarta EE (Eclipse Foundation).

| Caratteristica        | Tomcat 9.x           | Tomcat 10.0 / 10.1 / 11.x                      |
| --------------------- | -------------------- | ---------------------------------------------- |
| Specifica EE          | Java EE 8            | Jakarta EE 9 / EE 10                           |
| Namespace dei Package | javax.servlet.\*     | jakarta.servlet.\*                             |
| Compatibilità         | Applicazioni Legacy  | Applicazioni Moderne (Spring Boot 3+, Jakarta) |
| Stato                 | Mantenuto ma vecchio | Standard attuale                               |

### Il Problema Operativo

Un file .war sviluppato per Tomcat 9 (javax.servlet.\*) inviato su Tomcat 10 fallirà all'avvio sollevando un'eccezione java.lang.NoClassDefFoundError o ClassNotFoundException.

### Strategia di Migrazione

1. **Migrazione Codice Sorgente (soluzione ideale)**: Aggiornare il codice e le dipendenze in `pom.xml`/`build.gradle` sostituendo gli import `javax.` con `jakarta.`.
2. **Tomcat Migration Tool for Jakarta EE (soluzione sysadmin)**: Se hai solo il pacchetto .war compilato e non i sorgenti, Apache fornisce un tool da riga di comando che converte il bytecode in automatico convertendo le chiamate `javax` in `jakarta`:

```bash
java -jar jakartaee-migration-\*-shaded.jar /path/to/app-legacy.war /path/to/app-jakarta.war
```

3. **Deploy con Conversione Automatica (Tomcat 10.0 legacy)**: Nelle prime versioni di Tomcat 10 era possibile posizionare il file .war nella cartella webapps-javaee/ invece di webapps/: Tomcat eseguiva il tool di conversione al volo durante lo scompattamento del WAR.

## Hardening del Servizio (Systemd)

Applicazione dei vincoli di sicurezza a livello di Kernel Linux per l'unità del container Tomcat.

```bash
sudo vim /etc/systemd/system/tomcat.service
```

```ini
[Unit]
Description=Tomcat
After=network.target

[Service]
Type=forking

User=tomcat
Group=tomcat

Environment="JAVA_HOME=/usr/lib/jvm/default-java"
Environment="CATALINA_HOME=/opt/tomcat"
Environment="CATALINA_BASE=/opt/tomcat"
Environment="CATALINA_PID=/opt/tomcat/temp/tomcat.pid"

ExecStart=/opt/tomcat/bin/startup.sh
ExecStop=/opt/tomcat/bin/shutdown.sh

# RIAVVIO AUTOMATICO
Restart=on-failure
RestartSec=5s

# DIRECTORY CONCESSE IN SCRITTURA
# Rende scrivibili solo ed esclusivamente le cartelle specificate, necessarie al funzionamento di Tomcat
ReadWritePaths=/opt/tomcat/logs /opt/tomcat/temp /opt/tomcat/work

# HARDENING ISOLAMENTO
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

# HARDENING PRIVILEGI
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

# FILTRI SYSCALL
# Consente un set di chiamate di sistema (syscall) standard predefinite e sicure per i servizi di sistema
SystemCallFilter=@system-service
# Inverte il filtro (~) e blocca categoricamente i gruppi di syscall pericolose o non necessarie (gestione hardware, clock, reboot, swap, ecc.)
SystemCallFilter=~@resources @privileged @mount @debug @clock @module @reboot @swap
# Limita l'esecuzione delle chiamate di sistema esclusivamente all'architettura nativa della macchina (es. x86_64), bloccando quelle a 32-bit
SystemCallArchitectures=native

[Install]
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

$$\text{RAM Totale Processo Java} \approx \text{Heap (-Xmx)} + \text{MaxMetaspace} + (\text{Thread Max} \times \text{-Xss}) + \text{DirectMemory} + \text{CodeCache}$$

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

# 6. GC LOGGING (Tracciamento performance Garbage Collection)
CATALINA_OPTS="$CATALINA_OPTS -Xlog:gc*,gc+phases=debug:file=/opt/tomcat/logs/gc.log:time,uptime,pid:filecount=5,filesize=10m"

export CATALINA_OPTS
```

```bash
sudo chmod +x /opt/tomcat/bin/setenv.sh
sudo chown tomcat:tomcat /opt/tomcat/bin/setenv.sh
```

### Simulazione e Test di Memory Leak (OOM)

```bash
sudo mkdir -p /opt/tomcat/webapps/leak
sudo vim /opt/tomcat/webapps/leak/index.jsp
```

```jsp
<%@ page import="java.util.*" %>
<%!
    // Mantiene i riferimenti in memoria per impedire al GC di liberarli
    static List<byte[]> memoryBucket = new ArrayList<>();
%>
<%
    // Alloca 100MB di RAM ad ogni singola richiesta
    for (int i = 0; i < 20; i++) {
        memoryBucket.add(new byte[5 * 1024 * 1024]);
    }
    out.println("Allocati 100MB in memoria statica. Elementi totali nel bucket: " + memoryBucket.size());
%>
```

```bash
sudo chown -R tomcat:tomcat /opt/tomcat/webapps/leak
curl -ik http://localhost:8080/leak/index.jsp # errore 500 dopo 5 curl
sudo ls -lh /opt/tomcat/logs/ # dovrebbe esserci un file .hprof
# in teoria il servizio riparte da sé dopo 5 secondi, altrimenti:
sudo systemctl restart tomcat # svuota la RAM per nuovi test
```

```bash
sudo vim /opt/tomcat/bin/setenv.sh
```

```ini
# modifica questo:
# CATALINA_OPTS="$CATALINA_OPTS -Xms512m -Xmx512m"
CATALINA_OPTS="$CATALINA_OPTS -Xms64m -Xmx64m"
```

```bash
sudo systemctl restart tomcat
curl -ik http://localhost:8080/leak/index.jsp # errore 500 subito!
sudo ls -lh /opt/tomcat/logs/ # dovrebbe esserci un file .hprof (da 64MB o poco meno) e gc.log
# in teoria il servizio riparte da sé dopo 5 secondi, altrimenti:
sudo systemctl restart tomcat # svuota la RAM per nuovi test
```

## Alta Affidabilità: Load Balancing AJP con Apache Httpd

### Parte 1: Inizializzazione Multi-Istanze Tomcat

```bash
# Crea le due directory d'istanza
sudo mkdir -p /opt/tomcat-instance1 /opt/tomcat-instance2

# Copia la struttura base delle configurazioni
sudo cp -r /opt/tomcat/{conf,logs,temp,webapps,work} /opt/tomcat-instance1/
sudo cp -r /opt/tomcat/{conf,logs,temp,webapps,work} /opt/tomcat-instance2/

# Assegna i permessi all'utente tomcat
sudo chown -R tomcat:tomcat /opt/tomcat-instance1 /opt/tomcat-instance2

sudo vim /opt/tomcat-instance1/conf/server.xml # fare altrettanto con instance2
```

```ini
Shutdown port: 8005 # 8006
Connector HTTP: 8080 # 8081
Connector AJP:
  port 8009 # 8010
  address 127.0.0.1
  secret MiaPasswordAJP1 # 2
  secretRequired true
Engine jvmRoute: node1 # 2
```

```bash
sudo vim /etc/systemd/system/tomcat-node1.service # fare altrettanto con node2
```

```ini
[Unit]
Description=Tomcat Node 1 # 2
After=network.target

[Service]
Type=forking

User=tomcat
Group=tomcat

Environment="JAVA_HOME=/usr/lib/jvm/default-java"
Environment="CATALINA_HOME=/opt/tomcat"
Environment="CATALINA_BASE=/opt/tomcat-instance1" # 2
Environment="CATALINA_PID=/opt/tomcat-instance1/temp/tomcat.pid" # 2

ExecStart=/opt/tomcat/bin/startup.sh
ExecStop=/opt/tomcat/bin/shutdown.sh

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now tomcat-node1 tomcat-node2
```

### Parte 2: Reverse Proxy e Load Balancer su Apache Httpd

```bash
sudo a2enmod proxy proxy_ajp proxy_balancer lbmethod_byrequests status ssl rewrite
sudo systemctl restart apache2

# Genera un certificato SSL Self-Signed
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/tomcat-lb.key \
  -out /etc/ssl/certs/tomcat-lb.crt \
  -subj "/CN=localhost"

sudo vim /etc/apache2/sites-available/lb-tomcat.conf
```

```apache
# VirtualHost per il traffico in chiaro su porta 80 (HTTP)
<VirtualHost *:80>
    # Nome di dominio principale associato a questo VirtualHost
    ServerName localhost
    # Nome di dominio alternativo/secondario gestito dallo stesso VirtualHost
    ServerAlias lb.local

    # Attiva il motore di riscrittura delle URL di Apache
    RewriteEngine On
    # Verifica la condizione: la richiesta NON sta usando il protocollo HTTPS
    RewriteCond %{HTTPS} off
    # Reindirizza permanentemente (HTTP 301) qualsiasi URI verso la controparte cifrata HTTPS
    RewriteRule ^/(.*)$ https://%{HTTP_HOST}/$1 [R=301,L]
</VirtualHost>

# VirtualHost per il traffico cifrato su porta 443 (HTTPS)
<VirtualHost *:443>
    # Nome di dominio principale
    ServerName localhost
    # Nome di dominio alternativo/secondario
    ServerAlias lb.local

    # MOTORE SSL/TLS
    # Abilita la cifratura SSL/TLS su questo VirtualHost
    SSLEngine on
    # Percorso del file contenente il certificato X.509 pubblico
    SSLCertificateFile /etc/ssl/certs/tomcat-lb.crt
    # Percorso del file contenente la chiave privata associata al certificato
    SSLCertificateKeyFile /etc/ssl/private/tomcat-lb.key

    # CONFIGURAZIONE DEL BALANCER AJP
    # Definizione del gruppo logico di bilanciamento (cluster) con nome "tomcatcluster"
    <Proxy "balancer://tomcatcluster">
        # Primo nodo del cluster: connessione via AJP su porta 8009, identificato come node1, protetto da password AJP
        BalancerMember "ajp://127.0.0.1:8009" route=node1 secret=MiaPasswordAJP1
        # Secondo nodo del cluster: connessione via AJP su porta 8010, identificato come node2, protetto da password AJP
        BalancerMember "ajp://127.0.0.1:8010" route=node2 secret=MiaPasswordAJP2
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

    # INTERFACCIA DI GESTIONE DEL CLUSTER (OPZIONALE)
    # Crea un endpoint web all'URL /balancer-manager
    <Location "/balancer-manager">
        # Associa all'URL l'handler nativo di Apache per la GUI di monitoraggio del bilanciatore
        SetHandler balancer-manager
        # Restringe l'accesso alla dashboard di gestione unicamente alle richieste provenienti da localhost
        Require ip 127.0.0.1
    </Location>

    # FILE DI LOG
    # Percorso del log degli errori specifici per questo VirtualHost
    ErrorLog ${APACHE_LOG_DIR}/lb_ssl_error.log
    # Percorso del log degli accessi HTTP/HTTPS in formato 'combined'
    CustomLog ${APACHE_LOG_DIR}/lb_ssl_access.log combined
</VirtualHost>
```

```bash
sudo a2dissite mini-site.conf # se era ancora abilitato
sudo a2ensite lb-tomcat.conf
sudo systemctl reload apache2
sudo curl -ik http://localhost # test redirect HTTP -> HTTPS
sudo curl -ik https://localhost # test bilanciamento HTTPS
```
