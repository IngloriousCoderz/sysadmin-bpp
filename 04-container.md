# Container

Passando a una containerizzazione moderna con Docker / Podman e immagini minimali (come Alpine Linux), l'approccio alla sicurezza cambia radicalmente. Non si butta via quanto visto finora: quasi ogni concetto delle lezioni precedenti (namespace, capabilities, AppArmor, cgroup, utenti di servizio) **è esattamente ciò su cui i container sono costruiti**. Cambia _chi_ lo configura e _dove_.

## Scaletta e Tempi

Il modulo è pensato a blocchi indipendenti: i blocchi **[base]** formano un percorso minimo di circa 2 ore, i blocchi **[esteso]** portano il totale a circa 4 ore e mezza.

| #   | Blocco                                     | Tipo   | Durata |
| --- | ------------------------------------------ | ------ | ------ |
| 1   | Cosa sparisce, cosa si sposta, cosa resta  | base   | 15'    |
| 2   | Setup Docker sulla VM                      | base   | 10'    |
| 3   | Un container è solo un processo            | base   | 20'    |
| 4   | Immagini minimali (Dockerfile multi-stage) | base   | 40'    |
| 5   | Hardening a runtime (da systemd a Docker)  | base   | 35'    |
| 6   | Docker scavalca UFW                        | base   | 20'    |
| 7   | La JVM nei container: memoria e OOM        | esteso | 40'    |
| 8   | AppArmor e seccomp nei container           | esteso | 25'    |
| 9   | Utenti, UID, volumi e rootless             | esteso | 30'    |
| 10  | Log e diagnostica                          | esteso | 15'    |
| 11  | Integrato: Httpd + 2 Tomcat con Compose    | esteso | 45'    |
| 12  | Esercitazioni di troubleshooting           | esteso | 20'    |
| 13  | Cenni su Kubernetes                        | esteso | 10'    |

> **Porte**: sulla VM sono già occupate 80/443 (Httpd), 8080-8082 e 8443 (Tomcat), 8009-8011 (AJP). In questo modulo i container pubblicano solo nel range **9080-9090**.

## 1. Cosa Sparisce, Cosa si Sposta, Cosa Resta

**Cosa SPARISCE con i container:**

- **Systemd e i file `.service`**: dentro un container non c'è systemd. Si esegue un singolo processo in foreground (PID 1), lanciando direttamente `java`. Non esistono più le direttive `ProtectSystem=`, `ProtectHome=`, `PrivateTmp=`: l'isolamento del filesystem, dei processi e dei namespace viene gestito dal container engine (Docker / Podman / Kubernetes).
- **Installazione manuale e layout custom**: non si scarica più Tomcat a mano in `/opt/`. Si parte da un'immagine ufficiale (es. `eclipse-temurin` o `tomcat`) oppure si crea un'immagine minima che contiene solo il JRE e l'applicazione.
- **Multi-istanza con `CATALINA_BASE`**: due istanze = due container dalla stessa immagine. La separazione binari/configurazione la fanno i layer dell'immagine e i volumi.

**Cosa SI SPOSTA:**

| Prima (lezioni 01-03)                       | Ora (container)                                             |
| ------------------------------------------- | ----------------------------------------------------------- |
| `User=tomcat` nella unit                    | `USER` nel Dockerfile / `--user`                            |
| `ProtectSystem=strict` + `ReadWritePaths=`  | `--read-only` + volumi / `--tmpfs` sui path scrivibili      |
| `PrivateTmp=true`                           | `--tmpfs /tmp` (ogni container ha già il suo filesystem)    |
| `ProtectHome=true`                          | Implicito: la home dell'host non esiste se non la monti     |
| `CapabilityBoundingSet=`                    | `--cap-drop=ALL` (+ `--cap-add` mirati)                     |
| `NoNewPrivileges=true`                      | `--security-opt no-new-privileges`                          |
| `SystemCallFilter=@system-service`          | Profilo seccomp di default di Docker (attivo di serie)      |
| `PrivateNetwork=true`                       | `--network none` / reti interne                             |
| `MemoryMax=1350M`                           | `--memory 1350m`                                            |
| `TasksMax=`                                 | `--pids-limit`                                              |
| `Restart=on-failure`                        | `--restart on-failure`                                      |
| `journalctl -u tomcat`                      | `docker logs` (oppure log driver `journald`)                |
| Profilo AppArmor per path dell'eseguibile   | Profilo AppArmor per container (`docker-default`)           |
| `systemd-analyze security`                  | Revisione dei flag di `docker run` / scanner di immagini    |

**Cosa RESTA (e diventa ancora più importante):**

- **Il kernel è condiviso**: un container non è una VM. Una vulnerabilità del kernel dell'host è una via di fuga da tutti i container. Patch dell'host (lezione 01) obbligatorie.
- **Hardening dell'host**: SSH, UFW, AppArmor sull'host restano tutti necessari.
- **I permessi POSIX**: gli UID dentro e fuori dal container sono **gli stessi numeri** (salvo user namespace). Root nel container è root sull'host, con meno capabilities.
- **Il sizing della memoria JVM**: la formula vista con Tomcat vale identica; cambia solo chi impone il tetto.

## 2. Setup Docker sulla VM

_sulla VM_

Usiamo i pacchetti di Ubuntu (nessun repository esterno da aggiungere):

```bash
sudo apt update
sudo apt install -y docker.io
# Plugin "docker compose" (v2). Se il pacchetto non fosse disponibile, si può usare il repository ufficiale Docker
sudo apt install -y docker-compose-v2

docker --version
docker compose version
uname -m # aarch64 su Mac Apple Silicon, x86_64 su PC Intel/AMD: servirà nel troubleshooting
```

Per usare `docker` senza `sudo` si aggiunge l'utente al gruppo `docker`:

```bash
sudo usermod -aG docker $USER
# Serve un nuovo login perché il gruppo venga applicato
exit
```

_sul Mac_

```bash
ssh ubuntu-lab
```

_sulla VM_

```bash
docker run --rm hello-world
```

> **ATTENZIONE**: far parte del gruppo `docker` **equivale ad avere root sull'host senza password**. Lo dimostreremo nel blocco 9. In produzione il gruppo `docker` va trattato come il gruppo `sudo`.

## 3. Un Container è Solo un Processo

Un container non è una macchina virtuale: è un normale processo Linux a cui il kernel applica namespace (cosa vede), cgroup (quanto consuma), capabilities, seccomp e AppArmor (cosa può fare). Sono gli stessi meccanismi che systemd usava per le direttive di hardening della lezione 01.

```bash
docker run -d --name sleeper alpine:3.20 sleep 1000

# Il processo è visibile dall'host come qualunque altro, e gira come root
ps -ef | grep 'sleep 1000'

# Dentro al container, invece, è il PID 1 e non vede nient'altro
docker exec sleeper ps

# Recupera il PID del processo sull'host
PID=$(docker inspect -f '{{.State.Pid}}' sleeper)

# Namespace dedicati: mnt, uts, ipc, pid, net (e cgroup)
sudo lsns -p $PID
sudo ls -l /proc/$PID/ns

# Il cgroup del container: qui vivono i limiti di memoria/CPU (come per le unit systemd)
cat /proc/$PID/cgroup

docker rm -f sleeper
```

Il filesystem del container è un'**immagine** composta da layer in sola lettura, più un sottile layer scrivibile che viene buttato via con il container:

```bash
docker image inspect alpine:3.20 --format '{{json .RootFS.Layers}}'
docker history alpine:3.20
```

> **Conseguenza operativa**: tutto ciò che il container scrive fuori da un volume **sparisce** con `docker rm`. Log, heap dump e dati vanno su volumi o su stdout.

## 4. Immagini Minimali

Meno roba c'è nell'immagine, meno CVE da patchare e meno strumenti a disposizione di un attaccante (niente shell, niente `curl`, niente compilatore).

### Applicazione di Test

Una mini-applicazione Java senza dipendenze esterne: un server HTTP che mostra quanta memoria vede la JVM, e un endpoint `/leak` che alloca memoria come la `leak.jsp` della lezione 03.

```bash
mkdir -p ~/container-lab && cd ~/container-lab
vim App.java
```

```java
import com.sun.net.httpserver.HttpExchange;
import com.sun.net.httpserver.HttpServer;
import java.io.IOException;
import java.io.OutputStream;
import java.net.InetAddress;
import java.net.InetSocketAddress;
import java.nio.charset.StandardCharsets;
import java.util.ArrayList;
import java.util.List;

public class App {
    // Collezione statica: impedisce al GC di liberare la memoria (come leak.jsp)
    static final List<byte[]> BUCKET = new ArrayList<>();

    public static void main(String[] args) throws Exception {
        HttpServer server = HttpServer.create(new InetSocketAddress(8080), 0);

        // Mostra cosa "vede" la JVM: hostname, utente, heap massimo, CPU
        server.createContext("/", ex -> {
            Runtime rt = Runtime.getRuntime();
            String body = String.format("host=%s user=%s maxHeap=%dMB cpus=%d%n",
                    InetAddress.getLocalHost().getHostName(),
                    System.getProperty("user.name"),
                    rt.maxMemory() / 1024 / 1024,
                    rt.availableProcessors());
            send(ex, body);
        });

        // Alloca 50MB a ogni invocazione
        server.createContext("/leak", ex -> {
            for (int i = 0; i < 10; i++) {
                BUCKET.add(new byte[5 * 1024 * 1024]);
            }
            send(ex, "Allocati 50MB. Totale: " + BUCKET.size() * 5 + "MB\n");
        });

        server.start();
        System.out.println("In ascolto su :8080");
    }

    static void send(HttpExchange ex, String body) throws IOException {
        byte[] bytes = body.getBytes(StandardCharsets.UTF_8);
        ex.sendResponseHeaders(200, bytes.length);
        try (OutputStream os = ex.getResponseBody()) {
            os.write(bytes);
        }
    }
}
```

### Versione 1: l'immagine "ingenua"

```bash
vim Dockerfile.v1
```

```dockerfile
# Immagine con JDK completo (compilatore incluso) su base Ubuntu
FROM eclipse-temurin:21-jdk

WORKDIR /app
COPY App.java .

EXPOSE 8080
# Single-file source launch: compila ed esegue al volo (serve il JDK a runtime)
CMD ["java", "App.java"]
```

```bash
docker build -f Dockerfile.v1 -t app:v1 .
docker run -d --name app-v1 -p 127.0.0.1:9080:8080 app:v1
curl http://127.0.0.1:9080/ # user=root!

# Cosa c'è a disposizione di un attaccante dentro questo container?
docker exec app-v1 sh -c 'id; which bash curl apt javac'
docker rm -f app-v1
```

Problemi: gira come **root**, contiene un compilatore, una shell, un package manager e un sistema operativo intero.

### Versione 2: Alpine multi-stage e utente non privilegiato

Il **multi-stage build** separa l'ambiente di compilazione da quello di esecuzione: il primo stage viene scartato e nell'immagine finale si copia solo l'artefatto.

```bash
vim Dockerfile.v2
```

```dockerfile
# ---- STAGE 1: build (scartato alla fine) ----
FROM alpine:3.20 AS build
RUN apk add --no-cache openjdk21-jdk
ENV PATH=/usr/lib/jvm/java-21-openjdk/bin:$PATH
WORKDIR /src
COPY App.java .
RUN javac -d out App.java \
 && jar --create --file app.jar --main-class App -C out .

# ---- STAGE 2: runtime ----
FROM alpine:3.20

# Installa solo il JRE headless (niente JDK, niente pacchetti superflui)
RUN apk add --no-cache openjdk21-jre-headless
ENV PATH=/usr/lib/jvm/java-21-openjdk/bin:$PATH

# Crea un utente di servizio non privilegiato con UID fisso (come useradd --system della lezione 01)
RUN addgroup -S -g 10001 app && adduser -S -D -H -u 10001 -G app app

WORKDIR /app
# Il jar resta di proprietà di root: l'utente applicativo può leggerlo ma non modificarlo
COPY --from=build /src/app.jar /app/app.jar

# Da qui in poi tutto gira come utente non-root (UID numerico: verificabile anche da Kubernetes)
USER 10001:10001

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

```bash
docker build -f Dockerfile.v2 -t app:v2 .
docker run -d --name app-v2 -p 127.0.0.1:9080:8080 app:v2
curl http://127.0.0.1:9080/ # user=app
docker rm -f app-v2
```

### Versione 3: JRE su misura con `jlink`

`jlink` costruisce un runtime Java contenente **solo i moduli usati** dall'applicazione. Alla nostra app servono `java.base` e `jdk.httpserver`.

```bash
vim Dockerfile.v3
```

```dockerfile
# ---- STAGE 1: build + runtime su misura ----
FROM alpine:3.20 AS build
# jmods: i moduli "sorgente" da cui jlink assembla il runtime
RUN apk add --no-cache openjdk21-jdk openjdk21-jmods
ENV PATH=/usr/lib/jvm/java-21-openjdk/bin:$PATH
WORKDIR /src
COPY App.java .
RUN javac -d out App.java \
 && jar --create --file app.jar --main-class App -C out .
# Per un'app reale, l'elenco dei moduli si ricava con: jdeps --print-module-deps app.jar
RUN jlink \
      --add-modules java.base,jdk.httpserver \
      --strip-debug --no-man-pages --no-header-files \
      --compress=zip-6 \
      --output /opt/jre

# ---- STAGE 2: runtime ----
FROM alpine:3.20
# Librerie C++ di runtime richieste dalla JVM compilata per Alpine (musl)
RUN apk add --no-cache libstdc++ \
 && addgroup -S -g 10001 app && adduser -S -D -H -u 10001 -G app app
COPY --from=build /opt/jre /opt/jre
COPY --from=build /src/app.jar /app/app.jar
USER 10001:10001
EXPOSE 8080
ENTRYPOINT ["/opt/jre/bin/java", "-jar", "/app/app.jar"]
```

```bash
docker build -f Dockerfile.v3 -t app:v3 .

# Confronto delle dimensioni
docker images app
```

_Esito atteso_: si passa da circa **450-500 MB** (v1) a circa **200 MB** (v2) a meno di **60 MB** (v3).

### Buone Pratiche per le Immagini

- **Tag fissati, mai `latest`**: `alpine:3.20`, non `alpine`. In produzione si fissa anche il digest, che è immutabile: `FROM alpine:3.20@sha256:...` (lo si legge con `docker images --digests`).
- **`.dockerignore`**: evita di copiare nell'immagine `.git`, chiavi, file `.env`.
- **Nessun segreto nell'immagine**: ogni `COPY` e ogni `ENV` resta nei layer ed è leggibile con `docker history` anche se cancellato in un layer successivo.
- **Rebuild periodico**: un'immagine non si "aggiorna" con `apt upgrade`. Si ricostruisce dalla base aggiornata e si sostituisce il container.
- **Distroless**: le immagini `gcr.io/distroless/java21-debian12` non hanno nemmeno shell e package manager. Sono il passo successivo ad Alpine.

_Opzionale_: scansione delle vulnerabilità con Trivy, eseguito anch'esso come container. Notare che gli si deve passare il socket di Docker, cioè **pieno controllo dell'host** (vedi blocco 9):

```bash
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image app:v1
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image app:v3
```

## 5. Hardening a Runtime

L'immagine definisce _cosa_ c'è nel container; i flag di `docker run` definiscono _cosa può fare_. Sono l'equivalente diretto della sezione `[Service]` della unit Tomcat.

### Prima: container senza restrizioni

```bash
docker run -d --name app-plain -p 127.0.0.1:9080:8080 app:v3

# Bounding set (CapBnd): le capabilities ottenibili da un processo del container. Docker ne concede 14 di default
# (CapEff dell'app è già 0 perché gira come utente 10001: le capabilities effettive spettano solo a root)
docker exec app-plain grep Cap /proc/1/status
capsh --decode=$(docker exec app-plain awk '/CapBnd/ {print $2}' /proc/1/status)

# Ma chi ottiene root nel container (qui simulato con docker exec --user 0) le ha tutte, e può scrivere ovunque
docker exec --user 0 app-plain grep CapEff /proc/self/status
docker exec --user 0 app-plain touch /app/infiltrato && echo "scrittura riuscita"

docker rm -f app-plain
```

### Dopo: container blindato

```bash
docker run -d \
  --name app-hardened \
  --user 10001:10001 \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=64m \
  --cap-drop=ALL \
  --security-opt no-new-privileges:true \
  --memory 256m --memory-swap 256m \
  --cpus 1 \
  --pids-limit 200 \
  --restart on-failure:5 \
  -p 127.0.0.1:9080:8080 \
  app:v3
```

Il significato di ogni flag:

- `--user 10001:10001`: forza l'utente non privilegiato anche se l'immagine non lo dichiarasse (equivale a `User=`).
- `--read-only`: filesystem del container in sola lettura (equivale a `ProtectSystem=strict`).
- `--tmpfs /tmp:rw,noexec,nosuid,size=64m`: unica area scrivibile, in RAM, senza esecuzione di binari né bit SUID (equivale a `PrivateTmp=` + `ReadWritePaths=`). La JVM scrive in `/tmp` i file `hsperfdata`.
- `--cap-drop=ALL`: rimuove tutte le Linux capabilities (equivale a `CapabilityBoundingSet=`).
- `--security-opt no-new-privileges:true`: un eventuale binario SUID nell'immagine non concede privilegi (equivale a `NoNewPrivileges=true`).
- `--memory 256m --memory-swap 256m`: tetto cgroup di memoria **senza swap** (equivale a `MemoryMax=`). Se si omette `--memory-swap`, il container può usare altrettanta swap.
- `--cpus 1`: massimo una CPU (equivale a `CPUQuota=100%`).
- `--pids-limit 200`: limita i processi/thread, protegge dalle fork bomb (equivale a `TasksMax=`).
- `--restart on-failure:5`: riavvio automatico in caso di uscita con errore, al massimo 5 volte (equivale a `Restart=on-failure`).
- `-p 127.0.0.1:9080:8080`: pubblica la porta **solo su loopback** (il perché lo vediamo nel blocco 6).

Verifiche:

```bash
curl http://127.0.0.1:9080/

# Bounding set vuoto: CapBnd = 0000000000000000
docker exec app-hardened grep CapBnd /proc/1/status

# Anche root nel container non ha più capabilities, e il filesystem è in sola lettura
docker exec --user 0 app-hardened grep CapEff /proc/self/status # 0000000000000000
docker exec --user 0 app-hardened touch /app/infiltrato # Read-only file system

# Il tmpfs nasce con permessi 1777: lo Sticky Bit della lezione 01
docker exec app-hardened ls -ld /tmp # drwxrwxrwt

# /tmp è scrivibile ma non eseguibile: un attaccante non può scaricare ed eseguire un binario
docker exec app-hardened sh -c 'cp /bin/busybox /tmp/bb && /tmp/bb echo ciao' # Permission denied

# Limiti effettivi applicati dal cgroup
docker stats --no-stream app-hardened
```

> **Nota su `noexec`**: alcune librerie Java (Netty, Snappy, JNA) estraggono librerie native in `/tmp` e le caricano. Con `noexec` falliscono: in quel caso si dedica loro un tmpfs separato tramite `-Djava.io.tmpdir`.

### Chi fa da init?

Senza systemd, il processo principale **è il PID 1** del container, con due responsabilità che normalmente spettano all'init:

1. **Gestire `SIGTERM`**: `docker stop` invia `SIGTERM` al PID 1 e, dopo 10 secondi, `SIGKILL`. La JVM gestisce correttamente `SIGTERM` (shutdown hook, chiusura dei connettori). Uno script shell **no**, a meno che non usi `exec java ...` come ultima istruzione.
2. **Raccogliere i processi zombie**: se l'applicazione lancia sottoprocessi, serve un init minimo. Docker lo fornisce con `--init` (tini).

```bash
# Forma "exec" (array JSON): java è il PID 1 e riceve SIGTERM
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
# Forma "shell" (stringa): il PID 1 è /bin/sh, che NON inoltra SIGTERM a java. Da evitare
ENTRYPOINT java -jar /app/app.jar
```

```bash
# Verifica: lo stop deve completarsi subito, non dopo 10 secondi
time docker stop app-hardened
docker rm app-hardened
```

## 6. Docker Scavalca UFW

Questa è la sorpresa più pericolosa per un sysadmin. Docker pubblica le porte scrivendo **direttamente** regole `iptables` (catene `DOCKER` e `DOCKER-FORWARD`) che il kernel valuta **prima** delle regole di UFW. Il traffico verso un container pubblicato non passa mai dalla catena `INPUT` gestita da UFW.

### Dimostrazione

_sulla VM_

```bash
# UFW è attivo con default deny, e la porta 9090 NON è autorizzata
sudo ufw status verbose

# Pubblichiamo un container su tutte le interfacce (il default di -p)
docker run -d --name bypass -p 9090:8080 app:v3
```

_sul Mac_

```bash
curl http://192.168.64.2:9090/ # risponde, nonostante UFW!
```

_sulla VM_

```bash
# Ecco perché: una regola DNAT verso il container, fuori dal controllo di UFW
sudo iptables -t nat -L DOCKER -n
sudo iptables -L FORWARD -n --line-numbers | head
```

### Rimedi

1. **Pubblicare solo su loopback** (la soluzione più semplice, quella usata in tutto il modulo). Il container è raggiungibile solo dall'host, ad esempio tramite l'Httpd della lezione 02 come reverse proxy:

   ```bash
   docker rm -f bypass
   docker run -d --name bypass -p 127.0.0.1:9090:8080 app:v3
   ```

2. **Rendere loopback il default per tutto il demone** (vale per ogni `-p` che non specifica l'indirizzo):

   ```bash
   sudo vim /etc/docker/daemon.json
   ```

   ```json
   {
     "ip": "127.0.0.1"
   }
   ```

   ```bash
   sudo systemctl restart docker
   ```

3. **Filtrare nella catena `DOCKER-USER`**, l'unica che Docker non sovrascrive e che viene valutata per prima. Attenzione: dopo il DNAT la porta di destinazione è quella **del container**, per questo si confronta la porta originale con `conntrack`:

   ```bash
   # Interfaccia di rete esterna della VM
   IFACE=$(ip route show default | awk '{print $5}')
   # Blocca dall'esterno le connessioni dirette alla porta pubblicata 9090
   sudo iptables -I DOCKER-USER -i $IFACE -p tcp -m conntrack --ctorigdstport 9090 -j DROP
   ```

   Le regole aggiunte così non sopravvivono al riavvio: in produzione vanno rese persistenti (es. in `/etc/ufw/after.rules`, o con strumenti come `ufw-docker`).

> **Regola pratica**: `docker run -p` va considerato come `ufw allow`. Ogni porta pubblicata senza indirizzo è aperta al mondo.

```bash
# Pulizia
docker rm -f bypass
```

## 7. La JVM nei Container: Memoria e OOM

### Container awareness

Dalla versione 10 (e dalla 8u191) la JVM legge i limiti del **cgroup** invece della RAM fisica dell'host (`-XX:+UseContainerSupport`, attivo di default). Ma il default è prudente: **heap massimo = 25% del limite del container**.

```bash
# Senza limiti: la JVM dimensiona l'heap sulla RAM della VM
docker run -d --name mem-test -p 127.0.0.1:9081:8080 app:v3
curl http://127.0.0.1:9081/
docker rm -f mem-test

# Con 512MB di limite: heap massimo ~128MB (25%)
docker run -d --name mem-test -m 512m --cpus 1 -p 127.0.0.1:9081:8080 app:v3
curl http://127.0.0.1:9081/ # maxHeap ~128MB, cpus=1
docker rm -f mem-test

# Con MaxRAMPercentage: heap proporzionale al limite, senza hardcodare -Xmx
docker run -d --name mem-test -m 512m \
  -e JAVA_TOOL_OPTIONS="-XX:MaxRAMPercentage=75" \
  -p 127.0.0.1:9081:8080 app:v3
curl http://127.0.0.1:9081/ # maxHeap ~384MB
docker logs mem-test # "Picked up JAVA_TOOL_OPTIONS: ..."
docker rm -f mem-test
```

`JAVA_TOOL_OPTIONS` è letta da qualunque JVM: è il modo standard per passare opzioni senza ricostruire l'immagine (l'equivalente di `setenv.sh`).

> **Quale percentuale?** La formula della lezione 03 vale ancora: il limite del container deve contenere Heap + Metaspace + Thread Stack + overhead nativo. Con limiti piccoli (< 1GB) un valore di 50-60% è prudente; con limiti grandi si può salire a 75%. Hardcodare `-Xmx` funziona, ma va aggiornato a mano ogni volta che cambia il limite.

### Esercitazione 1: OOM della JVM con Heap Dump su Volume

Ricreiamo l'Esercitazione 1 della lezione 03. Il dump va scritto su un **volume**, altrimenti sparisce con il container. La directory deve essere scrivibile dall'UID del container.

```bash
mkdir -p ~/dumps
sudo chown 10001:10001 ~/dumps

docker run -d --name app-oom \
  -m 512m --memory-swap 512m \
  -e JAVA_TOOL_OPTIONS="-XX:MaxRAMPercentage=50 -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps -XX:+ExitOnOutOfMemoryError" \
  -v ~/dumps:/dumps \
  -p 127.0.0.1:9082:8080 \
  app:v3

# Heap di ~256MB: bastano 5-6 chiamate
for i in $(seq 1 8); do curl -s http://127.0.0.1:9082/leak; done

docker ps -a --filter name=app-oom # Exited (3)
docker inspect -f 'ExitCode={{.State.ExitCode}} OOMKilled={{.State.OOMKilled}}' app-oom # OOMKilled=false
docker logs app-oom | tail # "Terminating due to java.lang.OutOfMemoryError: Java heap space"
ls -lh ~/dumps # java_pid1.hprof: il PID è 1, perché la JVM è il PID 1 del container
```

### Esercitazione 2: OOM Killer del Kernel (cgroup)

Ricreiamo l'Esercitazione 2: un `-Xmx` più grande del limite del container. La JVM crede di avere spazio, il kernel no.

```bash
# In un secondo terminale, osserviamo gli eventi del demone
docker events --filter event=oom --filter event=die
```

```bash
docker run -d --name app-killed \
  -m 256m --memory-swap 256m \
  -e JAVA_TOOL_OPTIONS="-Xmx512m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps" \
  -v ~/dumps:/dumps \
  -p 127.0.0.1:9083:8080 \
  app:v3

for i in $(seq 1 8); do curl -s http://127.0.0.1:9083/leak; done

docker ps -a --filter name=app-killed # Exited (137)
docker inspect -f 'ExitCode={{.State.ExitCode}} OOMKilled={{.State.OOMKilled}}' app-killed # 137 true
sudo dmesg -T | grep -i 'out of memory' # "Memory cgroup out of memory: Killed process ... (java)"
ls -lh ~/dumps # nessun nuovo dump: SIGKILL non lascia alla JVM il tempo di scriverlo
```

| Scenario                  | Chi interviene | Exit code | `OOMKilled` | Heap dump |
| ------------------------- | -------------- | --------- | ----------- | --------- |
| Heap pieno (`-Xmx` < limite) | JVM         | 3 (con `ExitOnOutOfMemoryError`) | false | Sì |
| Limite cgroup superato    | Kernel         | 137 (128 + 9, `SIGKILL`) | true | No |

```bash
# Pulizia
docker rm -f app-oom app-killed
```

## 8. AppArmor e seccomp nei Container

### AppArmor: il profilo `docker-default`

Docker applica automaticamente a ogni container il profilo AppArmor `docker-default`, che tra l'altro impedisce di montare filesystem e di scrivere in `/proc/sys` e `/sys`.

```bash
docker info --format '{{.SecurityOptions}}' # name=apparmor name=seccomp,profile=builtin ...

docker run -d --name aa-test app:v3
docker inspect -f '{{.AppArmorProfile}}' aa-test # docker-default
sudo aa-status | grep -A2 docker
docker exec aa-test cat /proc/1/attr/current # docker-default (enforce)
docker rm -f aa-test
```

### Profilo personalizzato: niente shell nel container

Come per `curl` nella lezione 01, scriviamo un profilo restrittivo. Obiettivo: anche se l'applicazione venisse compromessa, l'attaccante non deve poter avviare una shell. In Alpine tutti i comandi (`sh`, `cat`, `ls`, `wget`...) sono link a `/bin/busybox`.

```bash
sudo vim /etc/apparmor.d/docker-app
```

```ini
# Stessa struttura del profilo docker-default generato da Docker
#include <tunables/global>

profile docker-app flags=(attach_disconnected,mediate_deleted) {
  #include <abstractions/base>

  # Permessi di base, analoghi a docker-default
  network,
  capability,
  file,
  umount,

  # Segnali: docker stop (unconfined) deve poter inviare SIGTERM
  signal (receive) peer=unconfined,
  signal (send,receive) peer=docker-app,

  # Protezioni ereditate da docker-default
  deny mount,
  deny @{PROC}/sys/** w,
  deny /sys/** w,

  # REGOLA DELLA DEMO: nessun binario busybox (quindi nessuna shell, cat, wget...)
  deny /bin/busybox x,
}
```

```bash
# Carica il profilo nel kernel (vedi "Il motore apparmor_parser" nella lezione 01)
sudo apparmor_parser -r /etc/apparmor.d/docker-app

docker run -d --name app-aa --security-opt apparmor=docker-app -p 127.0.0.1:9084:8080 app:v3
curl http://127.0.0.1:9084/ # l'applicazione funziona

# docker exec eredita il profilo del container
docker exec app-aa sh # Permission denied
docker exec app-aa ls / # Permission denied

# Il blocco è tracciato nei log del kernel
sudo journalctl -k | grep -i 'apparmor="DENIED"' | tail -3
```

```bash
# Pulizia
docker rm -f app-aa
sudo apparmor_parser -R /etc/apparmor.d/docker-app
sudo rm /etc/apparmor.d/docker-app
```

### seccomp

Docker applica di default anche un filtro **seccomp** che blocca circa 50 syscall pericolose (es. `mount`, `reboot`, `kexec_load`, `init_module`). È l'equivalente di `SystemCallFilter=` e, a differenza di systemd, è attivo senza scrivere nulla.

```bash
# Seccomp: 2 = filtro attivo
docker run --rm alpine:3.20 grep Seccomp /proc/1/status
# Seccomp: 0 = nessun filtro. Da non fare mai in produzione
docker run --rm --security-opt seccomp=unconfined alpine:3.20 grep Seccomp /proc/1/status
```

### Il flag da non usare mai: `--privileged`

`--privileged` disattiva **tutto** in un colpo: restituisce tutte le capabilities, disattiva AppArmor e seccomp, espone tutti i dispositivi dell'host. Un container privilegiato è root sull'host a tutti gli effetti.

```bash
docker run --rm alpine:3.20 sh -c 'ls /dev | wc -l' # pochi dispositivi
docker run --rm --privileged alpine:3.20 sh -c 'ls /dev | wc -l' # tutti i dispositivi dell'host
```

## 9. Utenti, UID, Volumi e Rootless

### Gli UID sono condivisi con l'host

Senza user namespace, l'UID dentro il container **è lo stesso numero** sull'host:

```bash
docker run -d --name as-root alpine:3.20 sleep 1000
docker run -d --name as-user --user 10001 alpine:3.20 sleep 1000

# Il primo processo appartiene a root, il secondo all'UID 10001 (senza nome, perché non esiste in /etc/passwd)
ps -o user,pid,cmd -C sleep

docker rm -f as-root as-user
```

### Esercitazione: Permessi sui Bind Mount

```bash
mkdir -p ~/data
echo "segreto" > ~/data/file.txt
chmod 600 ~/data/file.txt
ls -ln ~/data # proprietario UID 1000 (l'utente ubuntu)

# Il container gira come 10001: i permessi POSIX dell'host valgono anche dentro
docker run --rm --user 10001 -v ~/data:/data:ro alpine:3.20 cat /data/file.txt # Permission denied

# Soluzione 1: far girare il container con l'UID proprietario dei file
docker run --rm --user $(id -u):$(id -g) -v ~/data:/data:ro alpine:3.20 cat /data/file.txt

# Soluzione 2: assegnare i file all'UID del container (come fatto per ~/dumps)
sudo chown 10001 ~/data/file.txt
docker run --rm --user 10001 -v ~/data:/data:ro alpine:3.20 cat /data/file.txt
```

Il suffisso `:ro` monta il volume in sola lettura: va usato ogni volta che il container non deve scrivere (configurazioni, certificati, webapp).

### Il gruppo `docker` equivale a root

Chi può parlare con il demone Docker può chiedergli di montare la root dell'host in un container che gira come root:

```bash
# Lo esegue l'utente ubuntu, senza sudo
docker run --rm -v /:/host alpine:3.20 cat /host/etc/shadow
```

Per lo stesso motivo, montare `/var/run/docker.sock` in un container (come abbiamo fatto con Trivy) gli consegna il controllo completo dell'host.

### Container Rootless con Podman

**Podman** non ha un demone che gira come root: i container sono processi dell'utente che li avvia. Grazie agli **user namespace**, root dentro il container corrisponde all'utente non privilegiato sull'host.

```bash
sudo apt install -y podman

# Gli intervalli di UID "subordinati" assegnati all'utente
grep $USER /etc/subuid

# Con Podman conviene usare nomi di immagine completi (registro incluso)
podman run --rm docker.io/library/alpine:3.20 id # uid=0(root) dentro il container...

podman run -d --name rootless docker.io/library/alpine:3.20 sleep 1000
ps -o user,pid,cmd -C sleep # ...ma sull'host è l'utente ubuntu

# Mappatura degli UID: lo 0 del container corrisponde al 1000 dell'host
podman unshare cat /proc/self/uid_map

# Lo stesso attacco di prima ora fallisce
podman run --rm -v /:/host docker.io/library/alpine:3.20 cat /host/etc/shadow # Permission denied

podman rm -f rootless
```

La CLI di Podman è compatibile con quella di Docker (`alias docker=podman` funziona per la maggior parte dei comandi). Docker offre un risultato simile con la modalità _rootless_ o con l'opzione `userns-remap` del demone.

## 10. Log e Diagnostica

### Log

Un container deve scrivere i log su **stdout/stderr**, non su file. Docker li raccoglie, di default in file JSON **senza rotazione**: una causa classica di disco pieno.

```bash
docker run -d --name app-log -p 127.0.0.1:9085:8080 app:v3
docker logs app-log
docker logs -f --since 5m app-log

# Dove finiscono fisicamente
sudo ls -lh $(docker inspect -f '{{.LogPath}}' app-log)
```

Configurare la rotazione per tutto il demone (vale solo per i container creati dopo):

```bash
sudo vim /etc/docker/daemon.json
```

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

```bash
sudo systemctl restart docker
```

In alternativa, il driver `journald` riporta i log dei container nel flusso di `journalctl` visto con systemd:

```bash
docker run -d --name app-journal --log-driver journald app:v3
sudo journalctl CONTAINER_NAME=app-journal
```

### Diagnostica senza shell nel container

In un'immagine minimale (o con il profilo AppArmor del blocco 8) non c'è una shell in cui entrare. Si lavora dall'esterno:

```bash
# Stato, exit code, limiti, mount, rete: tutto in un JSON
docker inspect app-log

# Consumo di risorse in tempo reale
docker stats

# Entrare solo nel namespace di rete del container con gli strumenti dell'host (come nsenter nella lezione 03)
PID=$(docker inspect -f '{{.State.Pid}}' app-log)
sudo nsenter -t $PID -n ss -ltnp

# Un container "di debug" che condivide il namespace PID del container target
docker run --rm --pid container:app-log alpine:3.20 ps
```

Per usare `jcmd` (NMT, thread dump) con un runtime creato da `jlink` bisogna includere il modulo `jdk.jcmd`, oppure usare un'immagine JDK come container di debug.

```bash
# Pulizia
docker rm -f app-log app-journal
```

## 11. Integrato: Httpd + 2 Tomcat con Docker Compose

Ricostruiamo l'architettura finale della lezione 03 (Httpd bilanciatore con sticky session davanti a due Tomcat), ma con container, in un unico file dichiarativo.

Differenze rispetto alla versione su VM:

- **Rete**: i Tomcat stanno su una rete **interna** senza accesso a Internet e senza porte pubblicate. Solo Httpd è raggiungibile, e solo da loopback.
- **Sticky session**: invece di AJP e `jvmRoute` (che richiederebbero di modificare il `server.xml` dell'immagine), usiamo HTTP e un cookie di routing aggiunto da Apache.
- **Hardening**: tutti i container senza capabilities, read-only e non-root.

```bash
mkdir -p ~/compose-lb/httpd ~/compose-lb/webapps/ROOT
cd ~/compose-lb
```

### 1. Applicazione di Test

```bash
vim webapps/ROOT/session.jsp
```

```jsp
<%@ page contentType="text/plain; charset=UTF-8" %>
Nodo:       <%= java.net.InetAddress.getLocalHost().getHostName() %>
Sessione:   <%= session.getId() %>
Utente JVM: <%= System.getProperty("user.name") %>
Max heap:   <%= Runtime.getRuntime().maxMemory() / 1024 / 1024 %> MB
```

### 2. Configurazione Httpd Minimale

L'immagine ufficiale `httpd` legge `/usr/local/apache2/conf/httpd.conf`. Lo sostituiamo con una configurazione minima che carica **solo** i moduli necessari:

```bash
vim httpd/httpd.conf
```

```apache
ServerRoot "/usr/local/apache2"
# Porta non privilegiata: il processo gira come utente non-root
Listen 8080
ServerName localhost

# Solo i moduli indispensabili
LoadModule mpm_event_module modules/mod_mpm_event.so
LoadModule unixd_module modules/mod_unixd.so
LoadModule authz_core_module modules/mod_authz_core.so
LoadModule log_config_module modules/mod_log_config.so
LoadModule headers_module modules/mod_headers.so
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so
LoadModule proxy_balancer_module modules/mod_proxy_balancer.so
LoadModule slotmem_shm_module modules/mod_slotmem_shm.so
LoadModule lbmethod_byrequests_module modules/mod_lbmethod_byrequests.so

# PID e memoria condivisa del balancer su /tmp (tmpfs): il resto del filesystem è read-only
PidFile /tmp/httpd.pid
DefaultRuntimeDir /tmp

# Log su stdout/stderr, raccolti da "docker compose logs"
ErrorLog /proc/self/fd/2
# Formato con il nodo che ha servito la richiesta
LogFormat "%h %t \"%r\" %>s %b route=%{BALANCER_WORKER_ROUTE}e" balanced
CustomLog /proc/self/fd/1 balanced

# Non rivelare versione e sistema operativo (come nella lezione 02)
ServerTokens Prod
ServerSignature Off

<Directory />
    Require all denied
</Directory>

# Aggiunge un cookie con il nome del nodo quando il balancer sceglie (o cambia) nodo
Header add Set-Cookie "ROUTEID=.%{BALANCER_WORKER_ROUTE}e; path=/" env=BALANCER_ROUTE_CHANGED

<Proxy "balancer://tomcatcluster">
    # I nomi dei servizi Compose sono risolti dal DNS interno di Docker
    BalancerMember "http://node1:8080" route=node1
    BalancerMember "http://node2:8080" route=node2
    # Legge il suffisso dopo il punto nel cookie ROUTEID (".node1" -> node1)
    ProxySet stickysession=ROUTEID
</Proxy>

ProxyPreserveHost On
ProxyPass / "balancer://tomcatcluster/"
ProxyPassReverse / "balancer://tomcatcluster/"
```

### 3. File Compose

```bash
vim compose.yaml
```

```yaml
# Blocco riutilizzato da entrambi i nodi Tomcat (YAML anchor)
x-tomcat: &tomcat
  image: tomcat:10.1-jre21-temurin
  # Utente non privilegiato
  user: "10001:10001"
  # Filesystem read-only: scrivibili solo le directory di lavoro di Tomcat, in RAM
  read_only: true
  tmpfs:
    - /usr/local/tomcat/logs
    - /usr/local/tomcat/work
    - /usr/local/tomcat/temp
    - /tmp
  volumes:
    # La webapp è montata in sola lettura
    - ./webapps:/usr/local/tomcat/webapps:ro
  environment:
    # Heap proporzionale al limite del container (vedi blocco 7)
    CATALINA_OPTS: "-XX:MaxRAMPercentage=60 -XX:+ExitOnOutOfMemoryError"
  cap_drop: [ALL]
  security_opt:
    - no-new-privileges:true
  mem_limit: 512m
  memswap_limit: 512m
  cpus: 1.0
  pids_limit: 300
  restart: unless-stopped
  # Il container è "healthy" quando la porta 8080 accetta connessioni (bash è presente nell'immagine)
  healthcheck:
    test: ["CMD-SHELL", "bash -c '</dev/tcp/127.0.0.1/8080' || exit 1"]
    interval: 10s
    timeout: 3s
    retries: 5
    start_period: 30s
  # Solo rete interna: nessuna porta pubblicata, nessun accesso a Internet
  networks: [backend]

services:
  node1:
    <<: *tomcat
    hostname: node1

  node2:
    <<: *tomcat
    hostname: node2

  httpd:
    image: httpd:2.4
    # "daemon" (UID 1) è l'utente previsto dall'immagine
    user: "1:1"
    read_only: true
    tmpfs:
      - /tmp
    volumes:
      - ./httpd/httpd.conf:/usr/local/apache2/conf/httpd.conf:ro
    cap_drop: [ALL]
    security_opt:
      - no-new-privileges:true
    mem_limit: 128m
    restart: unless-stopped
    # Pubblicato solo su loopback (blocco 6)
    ports:
      - "127.0.0.1:9088:8080"
    # Avvia Httpd solo quando i Tomcat rispondono
    depends_on:
      node1:
        condition: service_healthy
      node2:
        condition: service_healthy
    # Httpd sta su entrambe le reti: fa da ponte tra l'esterno e i Tomcat
    networks: [frontend, backend]

networks:
  frontend: {}
  backend:
    # Rete senza gateway verso l'esterno
    internal: true
```

### 4. Avvio e Test

```bash
docker compose up -d
docker compose ps # node1 e node2 "healthy", httpd "running"

# Prima chiamata: il balancer sceglie un nodo e imposta il cookie ROUTEID
curl -s -i -c cookie.txt http://127.0.0.1:9088/session.jsp

# Chiamate successive con il cookie: sempre lo stesso nodo (sticky session)
curl -s -b cookie.txt http://127.0.0.1:9088/session.jsp
curl -s -b cookie.txt http://127.0.0.1:9088/session.jsp

# Senza cookie: i nodi si alternano (round robin)
for i in 1 2 3 4; do curl -s http://127.0.0.1:9088/session.jsp | head -1; done

# Log di accesso con il nodo che ha risposto
docker compose logs httpd
```

Verifiche di sicurezza:

```bash
# I Tomcat non sono raggiungibili dall'host (nessuna porta pubblicata)...
curl -m 3 http://127.0.0.1:8080/session.jsp # se risponde, è il Tomcat della lezione 03, non il container

# ...e non possono uscire su Internet (rete interna)
docker compose exec node1 bash -c '</dev/tcp/1.1.1.1/443' # Network is unreachable

# Gira come utente non-root
curl -s http://127.0.0.1:9088/session.jsp | grep Utente
```

### 5. Failover

```bash
# Spegniamo il nodo su cui siamo "incollati" (leggere il nome nel cookie.txt)
grep ROUTEID cookie.txt
docker compose stop node1

# La richiesta viene servita da node2, con una nuova sessione (le sessioni non sono replicate)
curl -s -b cookie.txt -c cookie.txt http://127.0.0.1:9088/session.jsp

docker compose start node1
```

_Integrazione opzionale_: l'Httpd della VM (lezione 02) può fare da terminatore TLS e inoltrare a `http://127.0.0.1:9088/` con un semplice `ProxyPass`, mantenendo i certificati sull'host.

```bash
# Pulizia
docker compose down
```

## 12. Esercitazioni di Troubleshooting

Stesso schema della lezione 02: provoca, testa, analizza, risolvi.

### 1. Tomcat in Read-only senza tmpfs

```bash
# Provoca l'errore: Tomcat in read-only senza aree scrivibili
docker run -d --name ts-ro --read-only tomcat:10.1-jre21-temurin

# Test: il container è ancora in esecuzione o è uscito?
docker ps -a --filter name=ts-ro

# Analisi log: errori di scrittura su logs/, work/, temp/
docker logs ts-ro 2>&1 | grep -iE 'read-only|denied|unable|failed' | head

# Risoluzione: tmpfs sulle directory di lavoro (come nel compose.yaml)
docker rm -f ts-ro
docker run -d --name ts-ro --read-only \
  --tmpfs /usr/local/tomcat/logs --tmpfs /usr/local/tomcat/work \
  --tmpfs /usr/local/tomcat/temp --tmpfs /tmp \
  tomcat:10.1-jre21-temurin
docker logs ts-ro 2>&1 | tail -3 # "Server startup in [...] milliseconds"
docker rm -f ts-ro
```

### 2. Porta già occupata

```bash
# Provoca l'errore: la porta 8081 è già usata da tomcat@node1 della lezione 03
docker run -d --name ts-port -p 127.0.0.1:8081:8080 app:v3
# Errore: "address already in use"

# Analisi: chi occupa la porta?
sudo ss -ltnp | grep 8081

# Risoluzione: un'altra porta sull'host (la porta del container non cambia)
docker rm -f ts-port
docker run -d --name ts-port -p 127.0.0.1:9086:8080 app:v3
docker rm -f ts-port
```

### 3. Exit Code 137

```bash
# Provoca l'errore: limite di memoria troppo basso per l'avvio di Tomcat
docker run -d --name ts-oom -m 64m --memory-swap 64m tomcat:10.1-jre21-temurin

# Test (attendere qualche secondo):
docker ps -a --filter name=ts-oom # Exited (137)?

# Analisi:
docker inspect -f 'ExitCode={{.State.ExitCode}} OOMKilled={{.State.OOMKilled}}' ts-oom
sudo dmesg -T | grep -i 'out of memory' | tail -2

# Risoluzione: dimensionare il limite (blocco 7)
docker rm -f ts-oom
```

> Se il container con 64MB riesce comunque ad avviarsi, provocare l'OOM con `/leak` come nell'Esercitazione 2 del blocco 7.

### 4. `exec format error` (Architettura Sbagliata)

Problema frequentissimo per chi sviluppa su Mac Apple Silicon (ARM) e distribuisce su server x86, o viceversa.

```bash
uname -m # aarch64 oppure x86_64

# Provoca l'errore: forza un'immagine dell'altra architettura
# (su aarch64 usare linux/amd64, su x86_64 usare linux/arm64)
docker run --rm --platform linux/amd64 alpine:3.20 uname -m
# "exec format error"

# Analisi: architetture disponibili per un'immagine
docker manifest inspect alpine:3.20 | grep architecture
```

Risoluzione: costruire l'immagine per l'architettura di destinazione (`docker buildx build --platform linux/amd64,linux/arm64 ...`) o eseguire la build direttamente su un host con l'architettura giusta (tipicamente in CI).

### 5. Bind mount: Permission Denied

Vedi l'esercitazione del blocco 9: verificare con `ls -ln` l'UID proprietario dei file sull'host e con `docker inspect -f '{{.Config.User}}'` l'UID del container.

## 13. Cenni su Kubernetes

In Kubernetes gli stessi controlli si dichiarano nel `securityContext` del Pod. Chi ha capito i flag di `docker run` sa già leggere questo file:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-hardened
spec:
  securityContext:
    # USER 10001 / --user
    runAsNonRoot: true
    runAsUser: 10001
    runAsGroup: 10001
    # Profilo seccomp di default del runtime
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: registry.example.com/app:v3
      securityContext:
        # --read-only
        readOnlyRootFilesystem: true
        # --security-opt no-new-privileges
        allowPrivilegeEscalation: false
        # --cap-drop=ALL
        capabilities:
          drop: ["ALL"]
        # --security-opt apparmor=docker-app (profilo caricato sul nodo)
        appArmorProfile:
          type: Localhost
          localhostProfile: docker-app
      env:
        - name: JAVA_TOOL_OPTIONS
          value: "-XX:MaxRAMPercentage=60"
      resources:
        # --memory / --cpus
        limits:
          memory: 512Mi
          cpu: "1"
      volumeMounts:
        - name: tmp
          mountPath: /tmp
  volumes:
    # --tmpfs /tmp
    - name: tmp
      emptyDir:
        medium: Memory
        sizeLimit: 64Mi
```

Le **Pod Security Standards** (livello `restricted`) rendono obbligatori la maggior parte di questi campi a livello di namespace, rifiutando i Pod che non li rispettano.

## Pulizia Finale

```bash
docker ps -a
docker rm -f $(docker ps -aq)
docker system prune -a # rimuove immagini, reti e cache di build non usate
sudo rm -rf ~/container-lab ~/compose-lb ~/dumps ~/data
```
