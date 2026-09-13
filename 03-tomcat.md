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

Environment="JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64"
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
systemd-analyze security test-hardening.service # 2.9 OK!
```

Un risultato di 2.9 (OK) su un servizio complesso come Tomcat è eccellente. Dimostra come si possa stringere la morsa della sicurezza tramite il kernel mantenendo al contempo un'applicazione reale del tutto operativa.

Il motivo per cui Tomcat si attesta a 2.9 (anziché lo 0.2 del servizio fittizio) risiede proprio nelle concessioni necessarie che abbiamo dovuto fare:

- L'accesso allo stack di rete
- Le directory montate in scrittura (ReadWritePaths)
- La rinuncia a DynamicUser in favore dell'utente di servizio statico tomcat
- La disattivazione di MemoryDenyWriteExecute per consentire il JIT della JVM

In ambito enterprise, scendere sotto il 3.0 per un application server Java senza romperne le funzionalità è esattamente il target a cui deve puntare un sysadmin o uno specialista di hardening.
