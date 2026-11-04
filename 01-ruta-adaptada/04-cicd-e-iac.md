# 04 — CI/CD e IaC

Fuente: `02-containers-security/06-cicd-pipeline-security.md` (secciones 6.3, 12, 15 y 16, y el parche de la sección 18) y `02-containers-security/10-infrastructure-as-code-and-configuration.md` (secciones 3, 5 y 14).

## Paso 1 — Permisos del workflow

El valor por defecto que la sección 6.3 escribe es solo lectura, y el permiso de escritura se concede en el job que lo necesita:

```yaml
permissions:
  contents: read
  pull-requests: write
```

En el repositorio del producto, Settings → Actions → General → Workflow permissions queda en solo lectura. Un workflow que no declara `permissions` hereda ese default.

## Paso 2 — El dato del pull request no entra en el script

El parche de la sección 18 pasa el título por el entorno y lo imprime con `printf`. Bash ve una variable, no un fragmento de script. Este es el workflow parcheado del fuente, recortado al paso que importa:

```yaml
- name: Post greeting safely
  env:
    PR_TITLE: ${{ github.event.pull_request.title }}
    PR_NUMBER: ${{ github.event.pull_request.number }}
    REPO: ${{ github.repository }}
  run: |
    safe_title=$(printf '%s' "$PR_TITLE" | tr -cd '[:alnum:] .,:;!?_-' | head -c 200)
    gh api "/repos/$REPO/issues/$PR_NUMBER/comments" \
      -f body="Thanks for PR titled: $safe_title"
```

Los dos controles que la lección nombra: el dato del fork viaja en una variable de entorno, y el valor se recorta en longitud y en caracteres antes de salir a una API.

No construyas el título que rompe la comilla. El archivo original lo tiene. Aquí el entregable es el workflow parcheado y una corrida en la que el comentario muestra el título como texto.

## Paso 3 — Digest, no tag

La sección 12.2 fija la base por digest:

```dockerfile
FROM node:20-alpine@sha256:2d07db07a2df6830718ae2a47db6fedce6745f5bcd174c398f2acdda90a01c00
```

Sustituye el digest por el de la imagen que realmente construiste en `02-imagen-y-runtime.md` (`docker image inspect hardened-nginx:lab --format '{{index .RepoDigests 0}}'` si la publicaste, o el id local si todavía no hay registry). El tag `latest` no entra en el Dockerfile que el pipeline publica.

## Paso 4 — Firma y provenance

Los comandos son los de las secciones 12.3 y 15.4. La identidad y el digest son los de tu repositorio, no los de `example`:

```bash
cosign sign --yes ghcr.io/<org>/<app>@sha256:<digest>

cosign verify \
  --certificate-identity 'https://github.com/<org>/<app>/.github/workflows/release.yml@refs/tags/v1.2.3' \
  --certificate-oidc-issuer 'https://token.actions.githubusercontent.com' \
  ghcr.io/<org>/<app>@sha256:<digest>
```

Si todavía no publicas la imagen, deja el comando escrito en el README del lab y marca el paso como pendiente de registry. No firmes una imagen que no sea tuya.

La provenance de la sección 15.3 es `cosign attest --predicate provenance.json --type slsaprovenance`. La verificación es `cosign verify-attestation` con el issuer de GitHub Actions. SLSA, en la sección 12.4: L1 hay provenance, L2 está firmada por la plataforma, L3 el build está aislado.

## Paso 5 — Commits firmados

Sección 16.1:

```bash
git config --global user.signingkey <keyid>
git config --global commit.gpgsign true
git config --global tag.gpgsign true
git commit -S -m "feat: signed commit"
```

La lección lo dice: la firma no impide un commit malicioso. Prueba quién lo firmó.

## Paso 6 — IaC sin tocar producción

Sobre un directorio Terraform que sea tuyo, o sobre el ejemplo de la lección 10:

```bash
tfsec . --format json > tfsec-report.json
checkov -d . --framework terraform -o json > checkov.json
```

La sección 5.1 muestra el hallazgo `aws-s3-enable-versioning` como ejemplo de salida. No hace falta ese bucket. El informe es el JSON.

La sección 3 lista lo que el state guarda en claro: `password`, `master_password`, `secret_string`, `private_key_pem`, `access_key`. El `.gitignore` de esa sección incluye `terraform.tfstate`, `terraform.tfstate.backup`, `*.tfvars` y `.terraform/`. Comprueba que tu repo los ignora. No vuelques un state.

El backend remoto de la sección 3.3 lleva `encrypt = true` y un `kms_key_id`. Si tu backend es local, el entregable es el diff que lo pasa a remoto cifrado, no un `terraform apply` contra una cuenta que no sea de laboratorio.

## Paso 7 — El gate

Un job, después del build de la imagen y del `tfsec`:

1. `trivy image` con `exit-code: '1'` para HIGH y CRITICAL, como en el archivo 02.
2. El job falla si `tfsec-report.json` o `checkov.json` traen un resultado que la herramienta marca HIGH o CRITICAL.

No añadas un scanner que las lecciones 03, 06 y 10 no nombran. Los que sí nombran son hadolint, trivy, grype, checkov y tfsec.

## Entregable

- El workflow con `permissions` y el título por variable de entorno.
- El Dockerfile con digest.
- El JSON de tfsec o de checkov.
- La línea del `.gitignore` que cubre el state.
