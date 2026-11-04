# 02 — Imagen y runtime

Fuente: `02-contenedores-pipelines-e-iac/01-docker-fundamentals-and-internals.md` sección 15, `02-contenedores-pipelines-e-iac/03-container-security-and-escape-techniques.md` secciones 13 y 14, y el setup de Trivy en la lección 12.

Este es el patrón de imagen que después el pipeline publica. No incluye el escape por el socket.

## Paso 1 — Construir la imagen de la lección 01

Crea `hardened-nginx.Dockerfile` con el Dockerfile de la sección 15.1 de esa lección: stage de build en `debian:12-slim`, Nginx 1.27.0 compilado hacia `/opt/nginx`, config que escucha en 8080 y responde `hardened`, stage final `gcr.io/distroless/base-debian12:nonroot`, `USER nonroot:nonroot`.

```bash
docker build -f hardened-nginx.Dockerfile -t hardened-nginx:lab .
docker images hardened-nginx
```

La lección mide del orden de 26.4 MB. Si tu tamaño difiere, anótalo. No cambies el stage final a una base con shell.

## Paso 2 — Arrancarla con el runtime de la lección

```bash
docker run -d --name hnginx \
  --read-only \
  --tmpfs /tmp:size=8m,mode=1777 \
  --cap-drop ALL \
  --security-opt no-new-privileges \
  -p 8080:8080 \
  hardened-nginx:lab

curl -s http://localhost:8080/
```

La respuesta esperada es `hardened`.

## Paso 3 — Comprobar el endurecimiento

Estos `docker inspect` son los de la sección 15.3:

```bash
docker inspect hnginx --format '{{.HostConfig.CapDrop}}'
docker inspect hnginx --format '{{.HostConfig.ReadonlyRootfs}}'
docker inspect hnginx --format '{{.HostConfig.SecurityOpt}}'
docker exec hnginx cat /proc/1/status | grep -E '^(Uid|CapEff)'
```

Tienes que ver `CapDrop` en `ALL`, raíz de solo lectura, `no-new-privileges`, uid 65532 y `CapEff` en cero. Si `docker exec` no entra porque la base es distroless y no tiene shell, la lección igual usa `cat` de `/proc/1/status`: si falla, quédate con los tres `inspect` y anota que el exec no tiene binario.

## Paso 4 — Scanner

La sección 14 de la lección 03 nombra estos comandos:

```bash
hadolint hardened-nginx.Dockerfile
trivy image hardened-nginx:lab
checkov -f hardened-nginx.Dockerfile
```

El ejercicio de esa lección pide `hadolint` y `trivy image` con cero hallazgos HIGH o CRITICAL. En CI, el mismo corte es el de la sección 14.5:

```yaml
- name: Trivy image scan
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: '${{ steps.build.outputs.image }}'
    severity: 'HIGH,CRITICAL'
    exit-code: '1'
```

Guarda la salida. Es el insumo del gate del archivo `04-cicd-e-iac.md`. `exit-code: '1'` es lo que hace fallar el build. Sin eso el scanner solo imprime.

## Paso 5 — Lo que el manifiesto no debe pedir

De la sección 13 de la lección 01 y de la sección 13 de la lección 03, el control es este: la imagen de aplicación no monta `/var/run/docker.sock`, no lleva `privileged: true` y no hace bind de `/` del host. Revisa el Compose del servicio y tacha esas tres líneas si aparecen. El health check de la lección 02 de Compose tiene que seguir en verde después del cambio.

## Entregable

- Dockerfile, salida de `docker inspect`, salida de `hadolint` y de `trivy image`.
- El fragmento de workflow con `exit-code: '1'`.
- Una línea que diga si el Compose del servicio monta el socket.
