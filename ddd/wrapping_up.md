[back](observability_and_middleware.md) | [index](index.md)


## Wrapping up

Fantastic, we have a Go app ready to be built, and the resulting binary is compatible with whichever system it targets (most of them at least).

Taking it up a notch, it would be good to deploy it as a Docker container so that it's easy for any user to run an instance, with or without persistence, and as a manageable service.

I will create a `Dockerfile` at the root of the project to create the container:

```dockerfile
FROM golang:alpine AS builder

WORKDIR /go/src/github.com/zalgonoise/x/secr

COPY ./ ./

# this app's sqlite requires gcc 
RUN apk add build-base

RUN go mod download
RUN mkdir /build \
    && go build -o /build/secr . \
    && chmod +x /build/secr


FROM alpine:edge

RUN mkdir -p /secr/server

COPY --from=builder /build/secr /secr

CMD ["/secr"]
```

This Dockerfile is doing a multi-stage so that I can fetch all dependencies to build the application, and still avoid the overhead of having them installed in the container.

This SQLite implementation requires CGO, which requires `gcc` in the alpine container. To fetch `gcc` I am running `apk add build-base`.

Then it's a matter of downloading the (Go) dependencies with `go mod download` and to build the app.

The final stage will grab the binary and run it on container's startup.

As for a `docker-compose.yaml` file, I would only need to link the container's port with the host, and link a volume if I wanted to persist the database content (which is honestly advisable):


```yaml
version: '3.7'
services:

  secr:
    build:
      context: . 
      dockerfile: ./Dockerfile
    container_name: secr
    ports: 
      - 8080:8080
    restart: unless-stopped
    volumes:
      - /tmp/secr:/secr
```

From here, simply running `docker compose up --build` does the job, and after that I can start the app with `docker compose up`.


[next](final_thoughts.md)