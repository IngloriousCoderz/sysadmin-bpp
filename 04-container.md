# Container

Passando a una containerizzazione moderna con Docker / Podman e immagini minimali (come Alpine Linux), l'approccio alla sicurezza cambia radicalmente.

Cosa SPARISCE completamente con i Container:

- Systemd Hardening (addio systemd e file .service):
- Alpine Linux dentro un container non usa systemd (nei container si esegue un singolo processo in foreground, spesso gestito tramite OpenRC se si usa un init, o lanciando direttamente java).
- Non esistono più le direttive ProtectSystem=, ProtectHome=, PrivateTmp=. Tutta l'isolamento del filesystem, dei processi e dei namespace viene gestito dal Container Engine (Docker/Podman/Kubernetes).
- Installazione manuale di dependencies e layout custom:
- Non scarichi più Tomcat a mano in /opt/. Si parte da un'immagine ufficiale (es. eclipse-temurin o tomcat:alpine) oppure si crea un pacchetto minimo contenente solo il JRE e il file .war dell'applicazione.

```dockerfile
FROM alpine:3.19

# Installa solo OpenJDK (niente pacchetti superflui)
RUN apk add --no-cache openjdk11-jre-headless

# Crea un utente di servizio non privilegiato
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app
COPY my-app.jar /app/app.jar

# Imposta l'utente non-root
USER appuser

EXPOSE 8080
CMD ["java", "-jar", "/app/app.jar"]
```

```bash
docker run -d \
  --name my-hardened-app \
  --user 10001:10001 \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid \
  --cap-drop=ALL \
  --security-opt no-new-privileges:true \
  --security-opt apparmor=docker-default \
  -p 8080:8080 my-alpine-app
```
