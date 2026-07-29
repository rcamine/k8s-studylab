# ---- build stage: full Go toolchain ----
FROM golang:1.26-alpine AS build
WORKDIR /src
COPY go.mod ./
RUN go mod download
COPY . .
ARG APP_VERSION=dev
RUN CGO_ENABLED=0 go build -ldflags "-s -w" -o /web .

# ---- runtime stage: distroless, no shell, non-root ----
FROM gcr.io/distroless/static:nonroot
COPY --from=build /web /web
ARG APP_VERSION=dev
ENV APP_VERSION=${APP_VERSION}
EXPOSE 8080
USER nonroot:nonroot
ENTRYPOINT ["/web"]
