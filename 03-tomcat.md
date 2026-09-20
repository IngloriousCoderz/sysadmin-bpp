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
Description=Tomcat
After=network.target

[Service]
Type=forking
User=tomcat
Group=tomcat
Environment="JAVA_HOME=/usr/lib/jvm/default-java"
Environment="CATALINA_HOME=/opt/tomcat"
Environment="CATALINA_BASE=/opt/tomcat"
ExecStart=/opt/tomcat/bin/startup.sh
ExecStop=/opt/tomcat/bin/shutdown.sh

[Install]
WantedBy=multi-user.target
```

## Architettura (/opt/tomcat/conf/server.xml)

Tomcat non è un semplice web server, ma un Servlet Container. La sua architettura è gerarchica:

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

## Differenze 9-10

La differenza fondamentale tra Tomcat 9 e Tomcat 10 risiede nel passaggio di proprietà dei namespace dalle specifiche Java EE (Oracle) alle specifiche Jakarta EE (Eclipse Foundation).

| Caratteristica             | Tomcat 9.x           | Tomcat 10.0 / 10.1 / 11.x                      |
| -------------------------- | -------------------- | ---------------------------------------------- |
| Specifica EE               | Java EE 8            | Jakarta EE 9 / EE 10                           |
| Namespace dei Package      | javax.servlet.\_     | jakarta.servlet.\_                             |
| Compatibilità Applicazioni | Applicazioni Legacy  | Applicazioni Moderne (Spring Boot 3+, Jakarta) |
| Supporto/EOL               | Mantenuto ma vecchio | Rilascio Principale (Standard attuale)         |

### Il Problema Operativo

Se prendi un file .war sviluppato per Tomcat 9 (che usa import javax.servlet.http.HttpServlet;) e lo deploy su Tomcat 10 senza modificarlo, l'applicazione fallirà all'avvio lanciando un'eccezione java.lang.NoClassDefFoundError o ClassNotFoundException.

## Strategia di Migrazione (Come gestire il passaggio a Tomcat 10)

Quando ci si trova a gestire la migrazione di server aziendali, esistono 3 approcci:

1. Migrazione del Codice Sorgente (Soluzione Ideale): Gli sviluppatori aggiornano le dipendenze nel pom.xml (Maven) o build.gradle e cambiano gli import javax. in import jakarta.
2. Uso dell'Apache Tomcat Migration Tool for Jakarta EE (Soluzione Sysadmin): Se hai solo il pacchetto .war compilato e non i sorgenti, Apache fornisce un tool da riga di comando che converte il bytecode in automatico convertendo le chiamate javax in jakarta: java -jar jakartaee-migration-\*-shaded.jar /path/to/app-legacy.war /path/to/app-jakarta.war
3. Deploy con Conversione Automatica (Tomcat 10.0 legacy): Nelle prime versioni di Tomcat 10 era possibile posizionare il file .war nella cartella webapps-javaee/ invece di webapps/: Tomcat eseguiva il tool di conversione al volo durante lo scompattamento del WAR.

## AJP: Httpd come bilanciatore di traffico

### Parte 1: Istanze Tomcat

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
Shutdown port: 8005/8006
Connector HTTP: 8080/8081
Connector AJP: 8009/8010, address 127.0.0.1, secret MiaPasswordAJP1/2
Engine jvmRoute: jvm1
```

```bash
sudo vim /etc/systemd/system/tomcat-node1.service # fare altrettanto con node2
```

```ini
[Unit]
Description=Tomcat Node 1
After=network.target

[Service]
Type=forking
User=tomcat
Group=tomcat
Environment="JAVA_HOME=/usr/lib/jvm/default-java"
Environment="CATALINA_HOME=/opt/tomcat"
Environment="CATALINA_BASE=/opt/tomcat-instance1"
ExecStart=/opt/tomcat/bin/startup.sh
ExecStop=/opt/tomcat/bin/shutdown.sh

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now tomcat-node1 tomcat-node2
```

### Parte 2: Load balancer Httpd

```bash
sudo a2enmod proxy proxy_ajp proxy_balancer lbmethod_byrequests status
sudo systemctl restart apache2

sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/tomcat-lb.key \
  -out /etc/ssl/certs/tomcat-lb.crt \
  -subj "/CN=localhost"

sudo a2enmod ssl rewrite

sudo vim /etc/apache2/sites-available/lb-tomcat.conf
```

```apache
<VirtualHost *:80>
    ServerName localhost
    ServerAlias lb.local

    RewriteEngine On
    RewriteCond %{HTTPS} off
    RewriteRule ^/(.*)$ https://%{HTTP_HOST}/$1 [R=301,L]
</VirtualHost>

<VirtualHost *:443>
    ServerName localhost
    ServerAlias lb.local

    # Motore SSL/TLS
    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/tomcat-lb.crt
    SSLCertificateKeyFile /etc/ssl/private/tomcat-lb.key

    # Configurazione del Balancer AJP
    <Proxy "balancer://tomcatcluster">
        BalancerMember "ajp://127.0.0.1:8009" route=node1 secret=MiaPasswordAJP1
        BalancerMember "ajp://127.0.0.1:8010" route=node2 secret=MiaPasswordAJP2
        ProxySet stickysession=JSESSIONID
    </Proxy>

    # Passaggio dei metadati HTTPS a Tomcat tramite AJP
    ProxyPreserveHost On
    ProxyPass / "balancer://tomcatcluster/"
    ProxyPassReverse / "balancer://tomcatcluster/"

    # Interfaccia di gestione del cluster (Opzionale)
    <Location "/balancer-manager">
        SetHandler balancer-manager
        Require ip 127.0.0.1
    </Location>

    ErrorLog ${APACHE_LOG_DIR}/lb_ssl_error.log
    CustomLog ${APACHE_LOG_DIR}/lb_ssl_access.log combined
</VirtualHost>
```

```bash
sudo a2dissite mini-site.conf # se era ancora abilitato
sudo a2ensite lb-tomcat.conf && sudo systemctl reload apache2
sudo curl -ik http://localhost
```

## Gestione della memoria

A differenza dei processi nativi C/C++, la JVM gestisce la memoria tramite aree logiche distinte con scopi ben precisi:

- Heap Memory, gestita dal Garbage Collector;
- Off-heap Memory, memoria nativa RAM (Metaspace e Thread Stack).

1. Heap Memory (-Xms, -Xmx):
   - Contiene tutti gli oggetti Java allocati a runtime (es. istanze di classi, collezioni).
   - È gestita direttamente dal Garbage Collector (GC).
   - Si divide in Young Generation (oggetti a vita breve) e Old Generation (oggetti persistenti, come le sessioni utente).

2. Metaspace (-XX:MetaspaceSize, -XX:MaxMetaspaceSize):
   - Sostituisce la vecchia PermGen da Java 8 in poi. Risiede nella RAM nativa del sistema operativo, non nell'Heap.
   - Contiene i metadati delle classi caricate (struttura delle classi, metodi, pool di costanti).
   - Rischio: Di default è illimitata (MaxMetaspaceSize non impostato). Se un'applicazione soffre di leaker di ClassLoader (frequente nei redeploy dinamici di file .war), la Metaspace saturerà la RAM fisica portando all'intervento del Linux OOM Killer.

3. Thread Stack (-Xss):
   - Ogni singolo thread creato da Tomcat (es. i worker HTTP/AJP) alloca una quantità fissa di memoria nativa per lo stack delle chiamate (frame di metodi, variabili locali).
   - Valore di default tipico: 1MB per thread. Con 200 thread attivi, l'impatto nativo è di circa 200MB fuori dall'Heap.

```bash
sudo vim /opt/tomcat/bin/setenv.sh
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

# 6. GC LOGGING (Tracciamento performance Garbage Collection)
CATALINA_OPTS="$CATALINA_OPTS -Xlog:gc*,gc+phases=debug:file=/opt/tomcat/logs/gc.log:time,uptime,pid:filecount=5,filesize=10m"

export CATALINA_OPTS
```

```bash
sudo chmod +x /opt/tomcat/bin/setenv.sh
sudo chown tomcat:tomcat /opt/tomcat/bin/setenv.sh
```

Quando calcoli la memoria totale richiesta dal server, non considerare solo -Xmx:
RAM Totale Processo Java ~= Heap (-Xmx) + MaxMetaspace + Thread Max \* (-Xss) + DirectMemory + CodeCache

Esempio di calcolo:

- **-Xmx**: 512MB
- **MaxMetaspace**: 256MB
- **Thread Pool (maxThreads="150") con -Xss512k**: 150 \* 0.5MB = 75MB
- **CodeCache/Nativa di servizio**: ~ 100MB

**Consumo di picco stimato per singola istanza**: ~ 943MB di RAM reale del sistema operativo. Se il server host ha 2GB di RAM e fai girare due istanze Tomcat senza limiti, il sistema operativo andrà in Swap Thrashing o attiverà l'OOM Killer del Kernel.

### Generare un Memory Leak

```bash
sudo mkdir -p /opt/tomcat/webapps/leak
sudo vim /opt/tomcat/webapps/leak/index.jsp
```

```jsp
<%@ page import="java.util.*" %>
<%!
    // Mantiene i riferimenti in memoria per impedire al Garbage Collector di liberarli
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
sudo systemctl restart tomcat # svuota la RAM per nuovi test
```

```bash
sudo vim /opt/tomcat/bin/setenv.sh
```

```ini
# modifica questo:
CATALINA_OPTS="$CATALINA_OPTS -Xms64m -Xmx64m"
```

```bash
sudo systemctl restart tomcat
curl -ik http://localhost:8080/leak/index.jsp # errore 500 subito!
sudo ls -lh /opt/tomcat/logs/ # dovrebbe esserci un file .hprof da 64MB o poco meno
sudo systemctl restart tomcat # svuota la RAM per nuovi test
```

## Hardening

```bash
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
Environment="CATALINA_HOME=/opt/tomcat"
Environment="CATALINA_BASE=/opt/tomcat"
Environment="CATALINA_PID=/opt/tomcat/temp/tomcat.pid"

ExecStart=/opt/tomcat/bin/startup.sh
ExecStop=/opt/tomcat/bin/shutdown.sh

# DIRECTORY CONCESSE IN SCRITTURA
ReadWritePaths=/opt/tomcat/logs /opt/tomcat/temp /opt/tomcat/work

# HARDENING ISOLAMENTO
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
PrivateDevices=true
ProtectKernelTunables=true
ProtectKernelModules=true
ProtectControlGroups=true

# HARDENING PRIVILEGI
NoNewPrivileges=true
CapabilityBoundingSet=
RestrictRealtime=true
RestrictSUIDSGID=true
LockPersonality=true
UMask=0027

# FILTRI SYSCALL
SystemCallFilter=@system-service
SystemCallFilter=~@resources @privileged @mount @debug @clock @module @reboot @swap
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

Il motivo per cui Tomcat si attesta a 2.9 (anziché lo 0.2 del servizio fittizio) risiede proprio nelle concessioni necessarie che abbiamo dovuto fare:

- L'accesso allo stack di rete
- Le directory montate in scrittura (ReadWritePaths)
- La rinuncia a DynamicUser in favore dell'utente di servizio statico tomcat
- La disattivazione di MemoryDenyWriteExecute per consentire il JIT della JVM

In ambito enterprise, scendere sotto il 3.0 per un application server Java senza romperne le funzionalità è esattamente il target a cui deve puntare un sysadmin o uno specialista di hardening.
