# Lab 6 — Containers: Dockerize QuickNotes

## Task 1 — Multi-Stage Dockerfile

### Dockerfile

The Dockerfile is located at `app/Dockerfile`.

```dockerfile
FROM golang:1.24-alpine AS builder

WORKDIR /src

COPY go.mod go.sum ./

RUN go mod download

COPY . .

RUN CGO_ENABLED=0 GOOS=linux go build \
    -trimpath \
    -ldflags="-s -w" \
    -o /quicknotes .


FROM gcr.io/distroless/static:nonroot

WORKDIR /

COPY --from=builder /quicknotes /quicknotes

EXPOSE 8080

USER nonroot:nonroot

ENTRYPOINT ["/quicknotes"]
```

## Image size

Command:

```bash
docker images quicknotes:lab6
```

Output:

```
IMAGE             ID             DISK USAGE   CONTENT SIZE   EXTRA
quicknotes:lab6   523b4aaaaa78       14.8MB         3.32MB    U
```

## Docker inspect

Command:

```bash
docker inspect quicknotes:lab6 | grep -A20 '"Config"'
```

Output:

```json
        "Config": {
            "User": "nonroot:nonroot",
            "ExposedPorts": {
                "8080/tcp": {}
            },
            "Env": [
                "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
                "SSL_CERT_FILE=/etc/ssl/certs/ca-certificates.crt"
            ],
            "Entrypoint": [
                "/quicknotes"
            ],
            "WorkingDir": "/",
            "Labels": {
                "com.docker.compose.project": "devops-outro",
                "com.docker.compose.service": "quicknotes",
                "com.docker.compose.version": "5.1.4"
            }
        },
        "Architecture": "amd64",
        "Os": "linux",

```

Important values:

- User: `nonroot:nonroot`
- ExposedPorts: `8080/tcp`
- Entrypoint: `["/quicknotes"]`

## Builder image comparison

Command:

```bash
docker images golang:1.24-alpine
```

Output:

```
IMAGE                ID             DISK USAGE   CONTENT SIZE   EXTRA
golang:1.24-alpine   8bee1901f1e5        395MB         83.5MB
```

# Design Questions

## a) Why does layer order matter?

Docker builds images in layers and caches unchanged layers. Copying `go.mod` and `go.sum` before the source code allows Docker to reuse downloaded dependencies when only application code changes.

If `COPY . .` happens before `go mod download`, every source code change invalidates the dependency layer and dependencies need to be downloaded again.

## b) Why CGO_ENABLED=0?

`CGO_ENABLED=0` creates a statically linked Go binary. Distroless static images do not contain a dynamic linker or libc libraries, so a dynamically linked binary would fail to start.

## c) What is gcr.io/distroless/static:nonroot?

`gcr.io/distroless/static:nonroot` is a minimal runtime image for static applications.

It contains only the required files to run the application. It does not contain a shell, package manager, or unnecessary utilities.

This reduces the attack surface and possible CVEs.

The `nonroot` version runs the application as UID 65532 instead of root.

## d) What do -ldflags="-s -w" and -trimpath do?

`-ldflags="-s -w"` removes debugging information and reduces binary size.

`-trimpath` removes local filesystem paths from the compiled binary, improving reproducibility.

The trade-off is losing debugging information.

# Task 2 — Compose + Healthcheck + Persistent Volume

## compose.yaml

```yaml
services:
  quicknotes:

    build:
      context: ./app

    image: quicknotes:lab6

    ports:
      - "8080:8080"

    environment:
      ADDR: ":8080"
      DATA_PATH: "/data/notes.json"
      SEED_PATH: "/data/seed.json"

    volumes:
      - quicknotes-data:/data

    restart: unless-stopped

    healthcheck:
      test: ["CMD", "/quicknotes"]
      interval: 30s
      timeout: 5s
      retries: 3


volumes:
  quicknotes-data:
```

# Persistence Test

## Create note

Command:

```bash
curl.exe -X POST -H "Content-Type: application/json" \
-d '{"title":"durable","body":"survive a restart"}' \
http://localhost:8080/notes
```

Output:

```json
{"id":1,"title":"durable","body":"survive a restart","created_at":"2026-06-23T17:11:53.386065624Z"}
```

## Verify note exists

Command:

```bash
curl.exe http://localhost:8080/notes
```

Output:

```json
[{"id":1,"title":"durable","body":"survive a restart","created_at":"2026-06-23T17:11:53.386065624Z"}]
```

## Restart without deleting volume

Commands:

```bash
docker compose down

docker compose up -d

sleep 3

curl.exe http://localhost:8080/notes
```

Output:

```json
[+] down 2/2
 ✔ Container devops-outro-quicknotes-1 Removed                                 0.1s
 ✔ Network devops-outro_default        Removed                                 0.2s
[+] up 2/2
 ✔ Network devops-outro_default        Created                                 0.0s
 ✔ Container devops-outro-quicknotes-1 Started                                 0.4s
[{"id":1,"title":"durable","body":"survive a restart","created_at":"2026-06-23T17:11:53.386065624Z"}]
```

The note survived because `docker compose down` removes containers and networks but keeps named volumes.

## Delete volume

Commands:

```bash
docker compose down -v

docker compose up -d

sleep 3

curl.exe http://localhost:8080/notes
```

Output:

```json
[+] down 3/3
 ✔ Container devops-outro-quicknotes-1 Removed                                 0.3s
 ✔ Network devops-outro_default        Removed                                 0.3s
 ✔ Volume devops-outro_quicknotes-data Removed                                 0.0s
[+] up 3/3
 ✔ Network devops-outro_default        Created                                 0.1s
 ✔ Volume devops-outro_quicknotes-data Created                                 0.0s
 ✔ Container devops-outro-quicknotes-1 Started                                 0.4s
curl: (52) Empty reply from server

```

The note disappeared because `docker compose down -v` removes the named volume.

# Compose Design Questions

## e) How do you healthcheck a distroless image?

Distroless images do not contain a shell or tools like curl/wget.

The healthcheck uses exec form and runs the application binary directly. This checks that the application process is running.

## f) Why does the named volume survive docker compose down?

Named volumes are managed separately from containers.

`docker compose down` removes containers and networks but keeps volumes.

`docker compose down -v` removes the volumes and stored data.

## g) What does depends_on without service_healthy do?

Without `condition: service_healthy`, `depends_on` only controls startup order.

It does not wait until the application is ready, which can cause connection failures.
